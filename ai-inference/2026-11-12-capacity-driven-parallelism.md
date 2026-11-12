# AI inference systems briefing — November 12, 2026

## Executive assessment

The most useful new development since yesterday’s [deadline-aware memory and KV-stability briefing](https://github.com/rahuljauhari3/interview-prep/blob/main/ai-inference/2026-11-11-deadline-aware-memory-and-kv-stability.md) is a reproducible AMD engineering study showing that **the optimal parallel layout changes when the workload crosses its KV-capacity ceiling**.

On one eight-GPU MI300X node serving GLM-5.3-Flash:

- TP8 is about 1.6× faster for a single request.
- TP8 replicates each request’s latent KV cache across all eight GPUs and can hold 76 concurrent 96K-input/32K-output sessions with BF16 KV.
- DP8+EP8 stores each request’s KV once, provides 5.75× the aggregate BF16 KV capacity, and can hold 441 such sessions.
- At concurrency 64, TP8 remains faster.
- At concurrency 128, DP8+EP8 becomes 1.54× faster.
- At concurrency 256, DP8+EP8 reaches 4,719 output tokens/s versus TP8’s 2,317, a 2.04× advantage.
- FP8 KV produces almost no throughput improvement while BF16 capacity is sufficient; its gains appear only after the BF16 pool fills.

This is not a new parallelism algorithm. The contribution is a clean capacity model, a full reproducible configuration, and measurements that expose a common deployment mistake: choosing topology from low-concurrency tokens/s rather than from the workload’s resident-state demand.

A second noteworthy development is vLLM’s expanded encoder/prefill/decode disaggregation for multimodal inference. The engineering contribution is the metadata contract and transport path, not the general idea of separating encoders. Downstream workers can now consume remote embeddings without re-fetching and reprocessing the original media, while encoder output caches can be shared across workers.

**Assessment:** the MI300X study is substantive engineering material and directly useful for capacity planning. Its results are vendor- and model-specific, and the benchmark uses random fixed-length traffic with prefix caching disabled; treat the crossover methodology as portable, not the numeric crossover point. Encoder disaggregation is strategically important for multimodal serving, but its current benchmark shows a load-dependent gain rather than a universal win.

**Primary technical references:**

- [Parallelism for Long-Context Serving: GLM-5.3-Flash on MI300X](https://rocm.blogs.amd.com/artificial-intelligence/glm53flash-mi300x-vllm/README.html), published November 11, 2026.
- [Encoder Disaggregation in vLLM: Scaling Multimodal Serving with EPD](https://vllm.ai/blog/2026-11-11-encoder-disaggregation-epd), published November 11, 2026.

---

## 1. What changed: parallelism is now a state-capacity decision

Tensor parallelism is usually chosen for latency: split each layer across several accelerators, reduce per-request compute time, and use fast collectives to assemble the result. That reasoning is incomplete for long-context serving because the parallel strategy also determines who owns the KV cache.

The AMD study compares two layouts on the same eight MI300X GPUs:

| Layout | Weight placement | Attention/KV ownership | Low-load behavior | Long-context capacity |
|---|---|---|---|---|
| TP8 | One model copy sharded across 8 GPUs | Each request’s latent KV is replicated on all 8 GPUs | Lowest single-request latency | Node capacity is effectively one GPU’s KV pool |
| DP8+EP8 | Eight DP ranks; routed experts sharded across ranks | Each request and its KV belong to one rank | Higher single-request latency | Aggregate capacity spans all 8 GPU-local pools |

GLM-5.3-Flash makes this comparison especially interesting. It is a 320B-parameter MoE with about 18B active parameters per token. Thirty-four of its 45 layers use Kimi Delta Attention with fixed recurrent state; only eleven sparse-MLA layers maintain a KV history that grows with context. The growing cache is therefore smaller than a conventional full-attention model’s, but it still becomes the dominant resource when hundreds of 128K-token sessions coexist.

The reported BF16 KV capacities are:

[
C_{	ext{TP8}} = 9{,}841{,}634 	ext{ tokens}
]

[
C_{	ext{DP8+EP8}} = 56{,}564{,}848 	ext{ tokens}
]

For a request reserving 96K input plus 32K output tokens:

[
N_{	ext{fit}}
=
leftlfloor
rac{C_{	ext{KV}}}{128{,}000}
ightfloor
]

which yields 76 TP8 sessions and 441 DP8+EP8 sessions.

Once the TP8 pool is full, additional requests wait even if aggregate compute throughput could have served them. The topology selected for low latency has converted seven-eighths of the node’s potential KV capacity into replication.

### Why this matters for CPU inference

The CPU analogue is weight replication versus state partitioning across sockets or processes.

A single multisocket process can obtain strong single-request performance by using cores across sockets for every layer, but that can also:

- replicate packed weights or metadata per socket;
- force every request’s KV to be visible across NUMA domains;
- increase remote reads during attention;
- collapse aggregate state capacity to the smallest locality domain;
- require cross-socket synchronization on every layer.

An alternative is to make each socket or NUMA group own complete requests and their KV while sharing or partitioning only expensive weight/expert state. Single-request latency can rise, but aggregate capacity and memory locality improve.

The topology decision should therefore include a replication factor:

[
C_{	ext{effective}}
=
rac{
sum_i M_{	ext{state},i}
}{
B_{	ext{request}}
cdot
r_{	ext{KV}}
}
]

where (r_{	ext{KV}}) is the number of physical copies created by the parallel layout. A topology with higher kernel throughput may lose at the system level when (r_{	ext{KV}}) reduces admitted concurrency.

---

## 2. The crossover is visible before and after the hard capacity wall

For the 96K/32K workload with BF16 KV:

| Concurrency | TP8 output tok/s | DP8+EP8 output tok/s | DP8+EP8 / TP8 |
|---:|---:|---:|---:|
| 1 | 78.74 | 49.89 | 0.63× |
| 32 | 1,392.09 | 1,118.29 | 0.80× |
| 64 | 2,104.91 | 1,890.31 | 0.90× |
| 128 | 2,084.59 | 3,218.28 | 1.54× |
| 256 | 2,317.08 | 4,719.20 | 2.04× |

At concurrency 64, both layouts fit and TP8 still wins. At 128, the workload exceeds TP8’s 76-session ceiling and the ordering reverses. DP8+EP8 continues scaling because its state capacity has not yet filled.

The medium 8K/1K workload reveals another effect. DP8+EP8 pulls ahead at concurrency 128 even though TP8 could theoretically hold about 1,067 such sessions. Here the gain comes from eight independent ranks decoding concurrently and avoiding tensor-parallel coordination, rather than from a hard KV-capacity limit.

That distinction is important. There are at least three crossover mechanisms:

1. **capacity crossover** — one layout stops admitting requests because replicated state fills memory;
2. **coordination crossover** — collective and synchronization costs dominate as batch/concurrency grows;
3. **locality crossover** — request-owned state improves cache/memory locality enough to offset slower per-request kernels.

A capacity calculator predicts the first but not the other two. Production topology selection still requires a concurrency sweep.

### Practical methodology

For each candidate layout:

1. Read the runtime’s actual allocatable KV-token count after weights, graphs, and workspaces.
2. Convert it to complete-request capacity using input plus reserved output length.
3. Include prefix sharing and state compression only if the workload actually provides them.
4. Benchmark below, near, and above the predicted ceiling.
5. Record throughput and TTFT/TPOT tail latency, not only tokens/s.
6. Repeat with the production arrival and length distributions.

A useful operating map is:

[
	ext{best layout}
=
f(
L_{	ext{input}},
L_{	ext{output}},
C,
B_{	ext{KV/token}},
r_{	ext{KV}},
T_{	ext{collective}},
B_{	ext{memory}}
)
]

There should not be one cluster-wide “best parallelism” flag for all tenants and request classes.

---

## 3. FP8 KV helps capacity, not necessarily bandwidth

The study’s cleanest result concerns KV precision.

Below the BF16 capacity ceiling, moving KV to FP8 changes throughput by less than 2%—within the reported run-to-run noise—for both TP8 and DP8+EP8. For example, TP8 at concurrency 64 moves from 2,104.91 to 2,083.36 output tokens/s, a slight measured decrease rather than a speedup.

Once BF16 capacity is exhausted, FP8 becomes valuable:

| Layout | Concurrency | BF16 output tok/s | FP8 output tok/s | Change |
|---|---:|---:|---:|---:|
| TP8 | 128 | 2,084.59 | 2,974.96 | +42.7% |
| TP8 | 256 | 2,317.08 | 2,824.43 | +21.9% |
| DP8+EP8 | 512 | 6,663.97 | 8,385.53 | +25.8% |
| DP8+EP8 | 768 | 6,795.66 | 8,245.74 | +21.3% |

The gain is not evidence that FP8 KV makes the attention kernel materially faster at these operating points. It increases the number of requests the scheduler can admit. Once the FP8 pool also fills, the benefit narrows.

This distinction matters for CPU optimization. KV quantization can have three independent effects:

[
Delta T
=
Delta T_{	ext{bandwidth}}
+
Delta T_{	ext{dequant}}
+
Delta T_{	ext{admission}}
]

Microbenchmarks usually measure the first two. End-to-end serving may be dominated by the third.

For EPYC inference:

- If the working set already fits in local DRAM, a narrower KV format may offer little benefit and can lose to dequantization overhead.
- If narrower KV keeps more sessions socket-local and avoids remote NUMA or CXL reads, the system-level benefit can be much larger than its kernel-level gain.
- If quantization merely lets the server admit work past a stable capacity boundary, it must be paired with the KV-demand admission control discussed yesterday; otherwise more admitted sessions can still trigger a cache-thrashing regime.
- Accuracy validation must include long-context retrieval and generation stability. The AMD study explicitly measures performance, not accuracy.

Quantization should therefore be evaluated as a **capacity and placement transformation**, not only as a vector-width optimization.

---

## 4. Hybrid attention changes the capacity model

GLM-5.3-Flash stores growing KV for only eleven of 45 layers. Thirty-four KDA layers keep fixed recurrent state. This lowers the slope of memory growth with sequence length, but the fixed state must still be included per request.

A general hybrid-model state equation is:

[
B_{	ext{request}}(L)
=
L
sum_{ellinmathcal{A}}
B_{	ext{KV/token},ell}
+
sum_{ellinmathcal{R}}
B_{	ext{recurrent},ell}
+
B_{	ext{metadata}}
]

where (mathcal{A}) is the set of attention layers with length-growing state and (mathcal{R}) the recurrent or linear-attention layers.

This matters because assuming “KV bytes per token × all layers” can overestimate capacity demand, while ignoring recurrent state can underestimate the fixed cost of high concurrency. The vLLM Qwen3.8 serving methodology discussed in an earlier briefing made the same point for Gated DeltaNet state.

For CPU systems, the capacity planner should consume model-declared state geometry:

- layers with full, sliding-window, sparse, or latent KV;
- cross-layer KV sharing;
- fixed recurrent state;
- indexer keys and sparse-selection metadata;
- alignment and block-allocation overhead;
- speculative-decoding draft and verification state.

A generic transformer formula is no longer sufficient for modern hybrid architectures.

---

## 5. What to reproduce on EPYC

A useful CPU experiment is not TP8 versus DP8+EP8 literally. It is **cross-socket request parallelism versus request ownership**.

Compare:

1. **Cross-socket tensor/operator parallelism**  
   Each request uses both sockets; weights are partitioned or jointly accessed; KV may be replicated or remotely visible.

2. **Socket-local request parallelism**  
   Each socket owns complete requests and their KV; model weights are replicated if they fit.

3. **Socket-local attention with shared/partitioned experts**  
   Attention and KV remain local; MoE experts are distributed across sockets, paying fabric traffic only for routed activations.

4. **Dynamic policy by request class**  
   Short, low-concurrency requests use cross-socket execution; long or high-concurrency requests use socket-owned state.

Sweep:

- context lengths from 1K to 128K;
- output reservations from 128 to 32K;
- concurrency from 1 through the measured capacity ceiling;
- BF16, FP8, INT8, and supported low-bit KV formats;
- one and two sockets;
- local DRAM, interleaved DRAM, and remote/CXL-backed KV;
- dense and MoE/hybrid models.

Measure:

| Dimension | Required metrics |
|---|---|
| Capacity | allocatable state bytes, complete sessions admitted, fragmentation, reserved output bytes |
| Compute | prefill tok/s, decode tok/s, GEMM/GEMV efficiency, expert utilization |
| Locality | local/remote DRAM bytes, Infinity Fabric traffic, LLC misses, NUMA page ownership |
| Coordination | barrier time, collectives, routed-activation traffic, completion skew |
| User impact | TTFT and TPOT p50/p95/p99, completed sessions/s, queue time |
| Stability | active KV demand, prefix hit ratio, evictions, parked sessions, recovery time |

The success metric should be SLO-qualified goodput:

[
G_{	ext{layout}}
=
max_C
left{
	ext{completed output tok/s}
mid
mathrm{TTFT}_{p99}le S_T, 
mathrm{TPOT}_{p99}le S_P
ight}
]

rather than single-request latency or peak unconstrained throughput.

---

## 6. Secondary development: encoder/prefill/decode disaggregation becomes operational

vLLM’s November 11 writeup documents the maturation of multimodal encoder disaggregation into a deployable path.

The architecture separates:

1. encoder execution for image, video, or audio;
2. language-model prefill;
3. language-model decode.

The technically important additions are:

- **metadata-only downstream requests:** prefill workers receive structural metadata and remote embeddings without refetching, decoding, resizing, normalizing, or patchifying the original media;
- **dedicated encoder runner:** encoder-only instances avoid language-model KV, sampling, and unnecessary checkpoint loading;
- **remote embedding connectors:** Mooncake and NIXL transfer encoder outputs;
- **cross-encoder reuse:** Mooncake Store can share cached embeddings across encoder workers;
- **independent orchestration:** vLLM Router, SGLang Model Gateway, and llm-d can scale encoder and language-model pools separately;
- **failure isolation:** a failed embedding transfer is associated with the affected request instead of silently poisoning another request’s state.

The metadata contract is more important than it appears. It must preserve the exact layout expected by the consumer—image grids, video timestamps and grids, audio token counts, and feature lengths—while the tensor payload travels on a separate data path.

On an audio evaluation using Qwen3-Omni-30B-A3B-Instruct at concurrency 64, metadata-only forwarding raises throughput from 62.21 to 73.13 requests/s and lowers mean latency from 980 to 853 ms. Accuracy shifts from 75.90% to 75.65%, which the authors say remains to be investigated.

For DeepSeek V4.1 Flash document QA, EPD adds four encoder GPUs to an existing four-prefill/four-decode deployment. At concurrency 64–128, it provides 6.6–9.5% higher throughput per GPU even after counting all twelve GPUs, and reduces P90 TTFT by 44–47%. At concurrency 16 it is less efficient per GPU; at 32 it is approximately break-even.

### CPU deployment implication

Encoder disaggregation creates a natural heterogeneous placement opportunity:

- run media decode, resize, tokenization, and lightweight encoders on CPU or integrated accelerators when their SLO permits;
- keep language-model prefill/decode on the scarce accelerator or optimized CPU pool;
- cache embeddings by content identity;
- transport compact embeddings and structural metadata, not raw media;
- scale encoder capacity from modality mix rather than text-token arrival rate.

But separation is not automatically beneficial. The decision must include:

[
T_{	ext{separate}}
=
T_{	ext{queue,E}}
+
T_{	ext{encode}}
+
T_{	ext{transfer}}
+
T_{	ext{queue,P}}
]

and compare it with colocated execution, including duplicate preprocessing and cache reuse.

The writeup also identifies unfinished work: observability, additional backends, clearer error propagation, and coordination between encoder caching and prefill-side KV reuse. An E-first pipeline may compute an embedding before prefill discovers it already has a reusable multimodal prefix. Avoiding that waste requires a cross-stage cache query without adding excessive cold-request latency.

---

## 7. Evidence quality and caveats

### Genuinely important

- The MI300X study reports the actual KV-token pools produced by each topology and connects them to complete-session capacity.
- It benchmarks below and above the predicted ceiling, making the throughput crossover explainable rather than anecdotal.
- The comparison holds model, hardware, context window, batch settings, and serving engine constant.
- The article provides container configuration, patches, benchmark commands, and every measured throughput point.
- It explicitly reports run-to-run and node-to-node noise and treats small differences as insignificant.
- The FP8 result separates capacity benefit from bandwidth benefit.
- vLLM’s encoder-disaggregation work specifies the metadata and transport contracts required for a real multimodal pipeline rather than stopping at an architecture diagram.

### Treat cautiously

- The AMD results are single-node MI300X measurements on one model and one vLLM/ROCm/AITER stack.
- Each reported throughput point is a single measured run; long-context high-concurrency points take an hour or more.
- Prompts are random fixed-length tokens, outputs are forced to the full requested length, and prefix caching is disabled. Production agent traffic will change batching, cache reuse, and completion distributions.
- DP8+EP8 relies on fast in-node XGMI for routed expert traffic. Cross-node expert parallelism would have a different crossover.
- The benchmark uses three patches beyond the official vLLM 0.31.0 image. Reproduction requires verifying the exact code path.
- FP8 KV accuracy is not evaluated.
- The vLLM EPD benchmark compares twelve-GPU EPD with eight-GPU PD and reports per-GPU throughput, but operational cost also depends on pool utilization and idle encoder capacity.
- The audio metadata-only evaluation shows a small accuracy difference that is unresolved.
- Neither item directly benchmarks CPU-only LLM generation; CPU implications are architectural inferences that require local validation.

---

## Reading priority

1. **[Parallelism for Long-Context Serving: GLM-5.3-Flash on MI300X](https://rocm.blogs.amd.com/artificial-intelligence/glm53flash-mi300x-vllm/README.html)** — Read the KV-capacity calculation, concurrency crossover, FP8 section, and reproducible setup. The appendix tables are more useful than the headline.
2. **[Decode Token Parallelism](https://arxiv.org/abs/2611.13293)** — Related research architecture that also separates weight placement from attention/KV ownership, evaluated on Llama 3.1 405B and Nemotron-4 340B.
3. **[Encoder Disaggregation in vLLM](https://vllm.ai/blog/2026-11-11-encoder-disaggregation-epd)** — Read the metadata-only contract, dedicated runner, connector semantics, and load-dependent DeepSeek evaluation.
4. **[Bursts That Outlast Themselves](https://arxiv.org/abs/2611.13412)** — Revisit when using FP8 or a new topology to admit more sessions: higher capacity still needs state-aware admission control.

## Bottom line

The strongest new evidence is that parallelism and precision should be selected from the **resident-state operating point**, not from isolated kernel speed or single-request latency.

TP8 is clearly better for one long request on the tested MI300X node. It is clearly worse once its replicated KV pool fills. FP8 KV is largely irrelevant before that boundary and highly valuable immediately after it. The same principle applies to multisocket CPU inference: a cross-socket layout that wins a microbenchmark can lose at production concurrency if it replicates state, increases remote traffic, or collapses effective capacity.

For an EPYC serving stack, the next practical step is a topology selector that predicts complete-session capacity per NUMA layout, benchmarks the coordination and locality crossover, and chooses between cross-socket execution and socket-owned requests by workload class. Pair it with the byte-based admission controller from yesterday’s briefing so that extra capacity becomes stable goodput rather than a larger cache-thrashing regime.