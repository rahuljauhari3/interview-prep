# AI inference systems briefing — November 20, 2026

## Executive assessment

Two new systems results materially extend this week’s memory-hierarchy theme.

[TideCache](https://arxiv.org/abs/2611.26619) recognizes that hybrid language models do not have one uniform KV cache. Global-attention layers maintain token-growing state, while bounded-state layers carry recurrent or fixed-window state. TideCache dynamically redistributes GPU pages among these components and streams global-attention prefixes layer by layer from host memory, using bounded-state computation as the transfer-overlap window.

[KIWI](https://arxiv.org/abs/2611.23848), from AMD Research, challenges the assumption that GPU-initiated communication control should remain entirely on GPU threads. Its programmable reverse-offload layer sends fine-grained work to CPU proxy threads with sub-microsecond one-way latency. A workload-specific expert-parallel implementation reduces communication latency by up to 2.3× and vLLM time per output token by up to 29% on AMD GPUs.

Together, the papers make a useful point for CPU inference and heterogeneous serving: the CPU is not merely overflow memory or a slow fallback. It can be an active scheduling, transfer, and communication processor—but only when the interface is designed around fine-grained latency, locality, and overlap.

A third paper, [RoCoKV](https://arxiv.org/abs/2611.25433), is worth reading for its full-stack integration of reasoning-aware KV pruning. Its accuracy and performance claims are promising, but the method is lossy and model/workload dependent, so it ranks below the two systems contributions.

## 1. TideCache: hybrid models require cache-component scheduling

**Importance: High for CPU–GPU superchips, hybrid models, and long agent sessions**

[TideCache: Efficient Hybrid LLM Serving with Transparent Cache Management and Layer-Wise Streaming](https://arxiv.org/abs/2611.26619), submitted November 19, targets models that interleave global-attention layers with bounded-state layers such as state-space, recurrent, local-attention, or delta-rule components.

These layers create different memory-growth laws:

| Cache component | Growth pattern | Useful management policy |
|---|---|---|
| Global-attention KV | Grows with context and request count | Stream old prefixes; retain active generated KV |
| Local or sliding-window state | Bounded per request | Keep resident when its working set is repeatedly reused |
| Recurrent state | Fixed or architecture-dependent per request | Allocate according to concurrency rather than context length |
| Shared staging slots | Depends on streamed layer | Reuse across global-attention layers |

A static partition of HBM among these components becomes wrong as context length, concurrency, and request phase change. TideCache addresses this with three mechanisms.

### Transparent Cache Pool

The pool maps physical pages among cache components behind stable virtual addresses. That is more important than it sounds. Repartitioning memory normally requires changing cache layouts, pointer contracts, or attention backends. Stable virtual addresses let the runtime adjust physical ownership without forcing every kernel and connector to understand a new layout.

For CPU runtime design, the analogous mechanism is a stable logical tensor or cache address space whose backing pages can move among socket-local DRAM, remote NUMA memory, CXL memory, and pinned host buffers. The page manager—not the kernel API—should own those placement changes.

### Cache Streamer

Rather than keeping every global-attention layer’s historical KV resident, TideCache shares GPU slots across layers. While bounded-state layers execute, it prefetches the next global-attention layer’s prefix from host memory. Newly generated KV remains resident.

This exploits the model graph as an I/O schedule. The bounded-state layers are not just computation between attention layers; they provide deterministic transfer slack.

The same idea transfers to CPU inference even without a GPU:

- prefetch a later attention layer’s cold KV from CXL or remote NUMA memory while current bounded-state layers execute;
- reuse a socket-local staging arena across nonconcurrent layers;
- pin generated or high-reuse KV locally while streaming colder historical prefixes;
- choose the prefetch distance from measured layer time and tier latency rather than a fixed number of layers.

### Request Scheduler

TideCache selects between resident and streamed execution and adjusts cache capacity based on request size and queue state. Streaming is not always beneficial. For short contexts or low concurrency, transfer overhead can dominate; resident execution should win. At longer contexts or higher concurrency, releasing HBM admits more work and improves end-to-end completion.

This is consistent with the broader lesson from recent briefings: representation and placement policies need workload-regime dispatch, not one globally enabled optimization.

### Reported evidence

The system is implemented in SGLang and evaluated on GB300 and GH200 CPU–GPU superchips across seven model/platform configurations. Relative to SGLang with HiCache, TideCache reports:

- up to **2.04× as many completed agent turns**;
- **20–59% lower overall median per-token latency**;
- **1.40–3.05× better agent median TTFT**.

The use of agent turns is a better metric than isolated token throughput because cache capacity affects whether long-lived, stateful requests remain serviceable.

### Practical implications for EPYC and CPU inference

The most valuable near-term prototype is a layer-reused NUMA-local staging pool.

For each layer and state class, record:

[
(	ext{bytes}, 	ext{next-use time}, 	ext{reuse count}, 	ext{source tier}, 	ext{transfer latency})
]

Then choose resident versus streamed execution subject to:

[
T_{	ext{transfer}} leq T_{	ext{available overlap}}
]

and a local-capacity constraint. If the transfer cannot be hidden, the scheduler should compare its exposed cost with the concurrency gained by freeing local memory.

Measure:

- page migration and remapping cost;
- local, remote, and CXL bandwidth;
- staging-slot occupancy;
- prefetch accuracy and lateness;
- per-layer exposed transfer time;
- maximum SLA-compliant concurrency;
- p99 TTFT and inter-token latency.

For multisocket systems, do not build one global staging pool unless remote access is intentionally part of the design. A per-socket pool with explicit ownership is more likely to preserve predictable bandwidth.

### Assessment and caveats

TideCache is a substantive systems contribution because it jointly addresses allocation, transfer overlap, and request-level policy while preserving backend interfaces.

Its headline results are not portable to conventional PCIe servers. GB300 and GH200 provide tightly integrated CPU–GPU memory systems, and the ability to hide streaming depends on model layer timing and host-link bandwidth. The paper also targets hybrid architectures; a dense transformer with global attention in every layer has fewer natural overlap intervals.

**Read:** [TideCache: Efficient Hybrid LLM Serving with Transparent Cache Management and Layer-Wise Streaming](https://arxiv.org/abs/2611.26619)

## 2. KIWI: CPU reverse offload as a first-class communication path

**Importance: High for AMD GPU systems, MoE serving, and CPU participation in fine-grained communication**

[A Fast, Programmable GPU-to-CPU Invocation Layer for GPU-Initiated Communication](https://arxiv.org/abs/2611.23848), submitted November 18, revisits a basic communication-stack choice.

GPU-initiated libraries commonly put transport control on GPU threads, often described as GPUDirect Async. CPU proxy threads are treated as a compatibility fallback because crossing from device to host is assumed to be slow.

KIWI argues that reverse offload can be faster when designed as a low-latency programmable path. GPU work items are sent to CPU proxies that perform workload-defined communication operations. The framework addresses three classic proxy problems:

- contention on shared queues;
- CPU polling overhead;
- load imbalance across proxy threads.

The resulting interface reports sub-microsecond one-way GPU-to-CPU invocation latency and hundreds of millions of operations per second.

### Why the programmable interface matters

A generic CPU proxy can still lose to GPU-side control if each operation requires heavyweight dispatch, serialization, or a rigid transport verb. KIWI instead lets the workload define the host-side operation.

That enables KIWI-EP to redesign MoE dispatch and combine around the actual communication pattern rather than simply replacing one transport call with another. The paper reports:

- up to **2.3× lower expert-parallel communication latency** than the best compared backend on AMD GPUs;
- up to **29% lower vLLM time per output token**.

The [source code is available](https://github.com/AMDResearch/kiwi), which materially raises the value of the paper for practitioners.

### Why it matters for CPU specialists

The result is a reminder that CPU participation should be evaluated at the system boundary, not dismissed from peak arithmetic ratios. A CPU core can be the better control processor when it provides:

- cheaper complex branching and queue manipulation;
- direct access to host networking or memory-registration state;
- fewer GPU execution resources consumed by communication progress;
- more flexible workload-specific operations;
- independent progress while GPU kernels execute.

For an EPYC plus AMD GPU node, the topology details are decisive. Proxy threads should be pinned to cores near the GPU’s PCIe or xGMI attachment and near the relevant NIC. Queue memory should be allocated on that NUMA node. Polling cores must be budgeted explicitly; allowing the OS to migrate them can destroy tail latency.

A useful deployment experiment would sweep:

1. GPU-resident transport control;
2. one shared CPU proxy queue;
3. per-GPU or per-link proxy queues;
4. topology-aware workload-specific proxies.

Measure one-way invocation latency, message-rate saturation, CPU utilization, cache-line bouncing, remote-NUMA traffic, GPU occupancy recovered, collective tail latency, and end-to-end TPOT.

### Broader CPU-inference implication

KIWI’s principle also applies to CPU-only serving components. A dedicated control plane for page migration, cache lookup, network progress, or batch formation can outperform embedding those duties in compute worker threads. The key is not “use a CPU thread”; it is to create a low-overhead queueing contract and place the proxy where it can make progress without perturbing compute teams.

### Assessment and caveats

This is a strong and actionable systems result: it includes a programmable abstraction, microarchitectural scaling work, an end-to-end vLLM integration, and open code.

The largest caveat is scope. The reported end-to-end gain comes from expert-parallel communication on AMD GPU systems. It does not imply that CPU proxies beat device-side transport for large contiguous transfers, non-MoE models, or every interconnect topology. Polling also consumes host cores and power, which should be included in efficiency comparisons.

**Read:** [A Fast, Programmable GPU-to-CPU Invocation Layer for GPU-Initiated Communication](https://arxiv.org/abs/2611.23848)  
**Code:** [AMDResearch/kiwi](https://github.com/AMDResearch/kiwi)

## 3. RoCoKV: reasoning-aware pruning with serving-stack integration

**Importance: Medium-high for constrained reasoning workloads; lossy and workload-sensitive**

[RoCoKV](https://arxiv.org/abs/2611.25433), submitted November 19, argues that long reasoning traces contain both uncertainty-sensitive regions and semantic redundancy. Pure attention-score eviction can retain repetitive self-verification or loops while discarding less-attended state that still matters to the solution.

Its policy combines:

- **uncertainty-guided compression**, retaining more state when the model is less confident;
- **redundancy-aware selection**, penalizing tokens similar to recent reasoning content;
- **fused three-phase CUDA decode kernels** integrated with vLLM’s paged layout and CUDA graphs.

That last point distinguishes it from compression work that only reports reduced cache size. The authors attempt to convert the algorithm into end-to-end performance without breaking the serving engine’s memory and graph-capture machinery.

For DeepSeek-R1-Distill-Llama-8B, the paper reports:

- **1.27× speedup and 38% higher throughput** than Full-KV on AIME25;
- **1.43× speedup and 45% higher throughput** on a long synthetic benchmark;
- better accuracy under constrained KV budgets than compared compression methods across the evaluated reasoning tasks.

### CPU relevance

The selection algorithm is not inherently GPU-specific. A CPU implementation could use token uncertainty and approximate representation similarity to decide which KV blocks remain in local DRAM, move to a slower tier, or are discarded.

However, the accounting must include policy cost. Similarity scoring, compaction, metadata updates, and irregular attention layouts can erase the memory saving on CPU. Favor block-granular decisions that preserve vectorizable, contiguous attention access.

The strongest experiment would compare RoCoKV-style selection with:

- full KV;
- fixed sliding windows;
- attention-score eviction;
- exact low-bit KV;
- tiering without loss.

Evaluate pass rate, tokens generated before success, DRAM traffic, policy CPU time, and p99 TPOT. Reasoning accuracy should be measured at equal end-to-end resource budgets, not merely equal retained-token counts.

### Assessment and caveats

RoCoKV is promising because it co-designs the algorithm and runtime. It remains a lossy policy evaluated on two reasoning models and three benchmarks. Confidence estimates can be poorly calibrated, and semantic similarity during self-correction does not necessarily mean a token is dispensable. Treat it as a workload-specific optimization requiring online quality monitoring.

**Read:** [RoCoKV: Accelerating LLM Inference Serving through Uncertainty and Redundancy Aware KV Cache Compression](https://arxiv.org/abs/2611.25433)

## Recommended engineering actions

1. **Model caches by component, not by request alone.** Separate global-attention history, bounded state, recurrent state, generated KV, and streaming slots in capacity accounting.
2. **Exploit graph-defined transfer windows.** Use bounded-state or compute-heavy layers to prefetch the next memory-bound layer’s state.
3. **Build stable logical cache addresses over movable backing pages.** Keep kernels insulated from DRAM, CXL, and accelerator-memory placement changes.
4. **Prototype topology-aware CPU proxies.** Pin communication and transfer-control threads near the relevant accelerator and NIC; measure polling cost and NUMA traffic.
5. **Choose resident versus streamed execution per regime.** Short contexts and low concurrency may favor residency; long agent sessions may favor released capacity.
6. **Keep lossy pruning behind quality gates.** Validate reasoning accuracy, calibration drift, and end-to-end policy overhead per model and workload.

## Bottom line

The strongest new work gives the CPU a more active role in heterogeneous inference.

TideCache uses host memory and model-layer structure to make cache residency dynamic without rewriting backend interfaces. KIWI uses CPU proxies as a programmable, low-latency communication engine rather than a fallback. Both succeed because they co-design the control path with the workload instead of treating CPU participation as generic offload.

For a CPU inference engineer, the common design target is a topology-aware state-and-progress layer: stable logical cache objects, movable physical backing, per-layer prefetch schedules, and dedicated local proxy threads for communication and migration. That layer is likely to deliver more end-to-end value than another isolated microkernel improvement when long-context, hybrid, and MoE workloads dominate.