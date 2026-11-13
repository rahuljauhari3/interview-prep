# AI inference systems briefing — November 13, 2026

## Executive view

The most important change since the previous briefing is a shift from **choosing a parallelism degree that fits memory** to **making parallelism match the physical memory hierarchy inside an accelerator**. AMD’s new Starscream paper shows that exposing MI300X/MI355X chiplets as more logical GPUs is actively harmful if the runtime simply extends one-dimensional tensor parallelism. The useful result is not the reported kernel speedup by itself; it is a concrete decomposition of sharding into inter-GPU and intra-GPU dimensions that preserves KV capacity, GEMM shape, and collective locality.

Two other papers strengthen a related operational point: capacity outside the accelerator is valuable only when its movement and setup costs are modeled as scheduled work. Host-tier KV can recover overloaded agent workflows, but synchronous stores can worsen p99. Likewise, serverless startup is dominated by compilation, profiling, imports, and runtime initialization—not checkpoint I/O—so optimizing weight loading alone misses most of the cold-start budget.

## 1. Starscream makes chiplet topology part of the sharding contract

**Assessment: Genuinely important for AMD inference systems**

[Starscream: Reimagining LLM Sharding on AMD Instinct GPUs with GPU Partitioning](https://arxiv.org/abs/2611.15951), submitted November 11, studies MI300X in CPX/NPS4 mode and MI355X in DPX/NPS2 mode. Its central observation is that GPU partitioning exposes a hierarchy that current frameworks flatten: intra-package links can provide roughly 10× the bandwidth of inter-GPU XGMI, while memory placement is NUMA-local.

A naïve runtime sees eight MI300X XCD partitions as eight ordinary devices and increases tensor parallelism accordingly. That produces three pathologies:

- **KV replication:** GQA models can have fewer KV heads than logical devices. Extending head-parallel sharding forces replication, reaching 8× in the paper’s MI300X example. This reduces batch/context capacity and wastes bandwidth during decode.
- **Arithmetic-intensity collapse:** continuing to shard a row-parallel GEMM along (K) turns (K/P 	imes N) tiles into increasingly skinny matrices. A compute-bound GEMM can become bandwidth-bound even though the physical compute resource has not changed.
- **Flat collective inflation:** logical-device count rises without equivalent interconnect capacity. Flat all-reduces increase peer count and run at the slowest link’s rate, ignoring the much faster intra-package fabric.

Starscream instead applies an **orthogonal sharding dimension inside each physical GPU**:

1. Attention shards sequence length across XCDs rather than replicating KV heads.
2. GEMMs use different dimensions across physical GPUs and XCDs, preserving balanced tiles.
3. All-reduce becomes a hierarchy: intra-GPU reduce-scatter, inter-GPU all-reduce among corresponding XCDs, then intra-GPU all-gather.
4. Compute kernels write directly into the target NUMA region using symmetric memory, fusing communication with attention, GEMM, or normalization work.

The reported gains are up to 4× for attention and about 1.43× for fused GEMM-plus-communication kernels against chiplet-oblivious baselines. More important, naïve CPX/NPS4 is sometimes worse than SPX/NPS1 and can fail from KV-cache exhaustion at batch sizes the topology-aware implementation still supports.

### Why this matters in practice

For an EPYC/ZenDNN engineer, the transferable rule is precise:

> A partition is not another homogeneous worker. It is a locality domain whose useful parallel dimension depends on tensor shape and communication semantics.

That suggests a runtime interface richer than “TP size”:

| Decision | Physical-GPU analogue | CPU/EPYC analogue |
|---|---|---|
| First sharding dimension | Across GPUs | Across sockets |
| Orthogonal local dimension | Across XCDs | Across CCDs/NUMA nodes |
| State placement | KV pages in local HBM | KV/weights in local DRAM |
| Hierarchical reduction | USR before XGMI | CCD/socket-local before inter-socket |
| Shape constraint | Avoid skinny GEMMs | Preserve microkernel-friendly (M,N,K) |

The next useful CPU experiment is not simply socket-aware tensor parallelism. It is **two-dimensional tensor placement with a shape constraint**: choose the inter-socket split first, then choose a different intra-socket split that keeps ZenDNN/BLAS tiles above the arithmetic-intensity knee. Compare this with flat TP at equal core count while recording local/remote DRAM bytes, reduction traffic, achieved bandwidth, GEMM efficiency, and p99 inter-token latency.

### Caveats

The paper evaluates optimized kernels rather than a broadly integrated production server, and the implementation is promised only “upon acceptance.” Several results are described as conservative because CPX-specific tuning was not completed, but that also means independent reproduction is not yet possible. Treat the architectural diagnosis as strong; treat end-to-end deployment claims as pending.

## 2. Host-tier KV is a scheduling trade, not a free capacity tier

**Assessment: Substantive and methodologically strong, but hardware-specific**

[Recompute or Restore? Host-Tier KV for Multi-Model Agent Workflows on a Static-Graph NPU Serving Stack](https://arxiv.org/abs/2611.15126), submitted November 11, evaluates a Qwen3-8B planner and Qwen3-0.6B executor on Rebellions ATOM+ NPUs with a synchronous host-DRAM KV tier.

The paper’s useful contribution is its refusal to equate “cache hit” with “latency win.” On this stack, KV transfer occupies the same engine thread as prefill and decode. Restoring an evicted prefix therefore avoids recomputation but can delay unrelated work.

Under overload, 89.5% of executed planner prompt tokens had already been materialized once. Adding the host tier raised SLO attainment from 22.8% to 100% in one condition and from 9.4% to 99.8% in another. Confirmed sustainable-load gains were 11.8% for chain workflows and 18.2% for fork workflows. Yet fork-workflow p99 rose 16–18%, largely because synchronous store work blocked the serving thread.

The authors also built a frozen, scheduler-aware event model. It stayed within a 15% error tolerance for median and p99 across 24 common-load conditions, but misordered three policy comparisons. A store-path optimization that cut isolated store time substantially improved serving p99 by only 3.1%, versus roughly 11% predicted, because other requests’ prefills dominated the tail.

### Practical implication

A CPU or CPU–accelerator KV tier needs a cost model over **work occupancy**, not just bytes:

[
	ext{restore if } T_{	ext{recompute}} >
T_{	ext{copy}} + T_{	ext{queue interference}} + T_{	ext{placement penalty}}
]

For EPYC hosts, split the last term into local-socket, remote-socket, and CXL/PCIe costs. A host-cache policy should be allowed to choose among restore, recompute, defer, or migrate—and should charge synchronous store work to the request that causes it. Instrument engine-thread occupancy and p99 interference alongside hit rate.

## 3. Serverless cold starts are mostly engine initialization

**Assessment: Important deployment engineering; CXL result is preliminary**

[Cairn: Fast Serverless LLM Cold Starts by Sharing and Prewarming Startup Work](https://arxiv.org/abs/2611.16180), submitted November 12, measures roughly minute-long serverless LLM cold starts and finds that weight loading accounts for only 5–11%. Compilation, memory profiling, process creation, imports, CUDA initialization, and runtime setup dominate.

Cairn separates startup work into:

- **shareable:** weights, metadata, compiled code, model inspection, compiler keys, and graph-memory estimates;
- **prewarmable:** processes, imports, and accelerator/runtime initialization;
- **process-bound:** work that still must occur in the final worker.

The first server publishes shareable artifacts into a cluster memory pool; later workers read them in place. Model-independent setup runs in warm containers replenished from a zygote. On vLLM v0.20 with six 1B–14B models, cold start fell from 57–85 seconds to 11–21 seconds (4.1–5.2×), and median TTFT on an Azure-functions-derived trace fell from 32.5 to 11.8 seconds.

### Why it matters

For scalable CPU inference, this reframes autoscaling work. Weight mmap and page-cache reuse are necessary but insufficient. Persist or precompute:

- oneDNN/ZenDNN primitive caches and packed weights;
- graph specialization and shape-selection results;
- allocator and KV-capacity profiles;
- tokenizer/model metadata;
- process imports and runtime initialization;
- NUMA placement decisions that are invariant for a host class.

The cold-start metric should be decomposed by stage and measured separately for “first model anywhere,” “first model on this host,” and “new worker for an already active model.” Otherwise checkpoint improvements can look impressive while barely changing TTFT.

### Caveats

The shared CXL pool was emulated with remote NUMA memory between two VMs on one host; the 16-server result is simulated. The paper also says code will be released only upon publication. The startup-work taxonomy is immediately useful, but the cross-host CXL performance claim needs real-switch validation.

## What to read first

1. **[Starscream](https://arxiv.org/abs/2611.15951)** — read Sections III–V. The analysis of KV replication, skinny GEMMs, collective interchange, and hierarchical communication is the highest-value material this week.
2. **[Recompute or Restore?](https://arxiv.org/abs/2611.15126)** — read the event model and Sections 4.3–4.5. The negative result around store-path optimization is especially useful.
3. **[Cairn](https://arxiv.org/abs/2611.16180)** — read the startup-stage taxonomy and implementation details before the headline speedup.

## Watchlist, not recommendation

[Broadcast, Don’t Replicate: Eliminating Weight HBM in Datacenter LLM Serving](https://arxiv.org/abs/2611.17097) proposes optical broadcast memory in which a centralized server streams weights to 80,000 accelerators. The idea attacks genuine HBM replication cost, but the evaluation is architectural modeling at an extreme scale, not a deployable inference stack. It is worth tracking for memory-system direction, not using for near-term capacity planning.

## Bottom line

This week’s strongest lesson is that **topology must be reflected in the algebra of sharding and in the scheduler’s accounting model**. Starscream shows the performance cost of pretending chiplet partitions are homogeneous devices. Host-tier KV shows the latency cost of pretending data movement is free. Cairn shows the deployment cost of pretending model startup is mostly I/O.

For a CPU inference stack, the highest-value prototype is a topology-aware planner that jointly selects tensor partition, tile shape, state placement, and reduction hierarchy, then rejects configurations that cross either a local-memory-capacity threshold or a kernel-efficiency threshold. That is a more promising optimization frontier than another isolated microkernel gain measured outside the serving runtime.
