# AI inference systems briefing — November 17, 2026

## Executive view

The most important change since yesterday is a deeper treatment of **KV-cache preemption under long reasoning outputs**. Stint shows that optimizing one offload or recomputation event is insufficient: a request can repeatedly recover its state, emit only a few tokens, and be evicted again. The resulting KV thrashing leaves more than a fifth of requests stalled for over a minute in the paper’s main replay. Stint connects victim selection, CPU-memory layout, transfer kernels, and protected readmission so that recovery guarantees fresh output rather than merely restoring old state.

A second paper provides a strong kernel-engineering lesson that transfers directly to CPU inference. In high-throughput speech serving, neither persistent megakernels nor chains of fused kernels win across all batch regimes. Selecting between them at measured crossovers beats always using either strategy. This is the serving-level version of choosing between microkernels, fusion plans, and execution graphs based on dynamic shape rather than static model structure.

## 1. Stint makes preemption guarantee progress

**Assessment: Genuinely important for long-output serving and CPU-host KV design**

[Making Preemption Productive in Long-Output LLM Serving](https://arxiv.org/abs/2611.20106), submitted November 15, identifies **KV thrashing**: a long-running request is preempted, waits, restores or recomputes its KV, produces little new output, and is preempted again as other requests continue growing.

In the paper’s 399-request olympiad-math replay, vLLM recomputation, LMCache, and SGLang with HiCache each left 21–23% of requests stalled for more than one minute and roughly one in six stalled for more than five minutes. This exposes a limitation in treating preemption as a single cost:

[
T_{	ext{preemption}}

eq
T_{	ext{save}} + T_{	ext{restore}}
]

The application-visible cost also includes time waiting for readmission and the probability of another eviction before meaningful progress:

[
T_{	ext{effective}}
=
T_{	ext{wait}}
+
T_{	ext{recovery}}
+
T_{	ext{fresh decode}}
+
P(	ext{re-preempt}) cdot T_{	ext{future recovery}}
]

Stint attacks all four terms.

### System design

Stint combines four mechanisms:

1. **Prediction-guided victim selection.** A lightweight model uses decoding hidden states to estimate remaining output length. The scheduler balances immediate memory relief against the opportunity to let a nearly completed request finish and release its entire cache.

2. **Contiguity-first CPU KV pool.** GPU KV grows during decoding, but an offloaded snapshot has a fixed size until the request resumes. Stint allocates exactly that extent, preferring one contiguous region without reserving space for future growth.

3. **Bulk KV movement.** Fused K/V gather and scatter kernels pack paged GPU blocks into contiguous staging buffers. Double buffering overlaps GPU packing with host-device copies.

4. **Age-based protected recovery.** The oldest stalled request eventually receives priority. Once restored, it is protected through a quantum of fresh decoding. Rebuilding or reloading existing KV does not count as progress; only newly emitted tokens do.

The last mechanism is the most important conceptual contribution. Fast restoration alone can make thrashing cheaper without preventing it. Stint turns recovery into a scheduling contract: a restored request must receive useful forward progress before it can become a victim again.

### Results

On Qwen3-32B with H200 GPUs and olympiad-math traces, Stint reports:

- no generation pause above 50 seconds, versus worst pauses of 11–14 minutes across the evaluated baselines;
- a 50.7% reduction in p99 time per output token relative to recomputation in the same engine;
- 10.7% shorter replay completion time;
- 12.0% higher throughput;
- 14.2% higher throughput than LMCache;
- 48.5% higher throughput than SGLang with HiCache, although this comparison also crosses serving engines.

Across AIME and LiveCodeBench settings, p99 TPOT fell 35–75%, while throughput ranged from 0.5% lower to 3.6% higher. That range is more representative of the general trade-off than the main 12% result.

### Why this matters for EPYC and CPU inference

This is unusually relevant to CPU-based KV tiers because host memory is not just a larger cache. Its allocation and scheduling policy determine whether accelerator or CPU execution makes progress.

A multisocket EPYC implementation should extend Stint’s pool from “contiguous CPU memory” to **contiguous, socket-local extents**:

| Stint concern | EPYC-specific extension |
|---|---|
| Exact-size CPU extent | Exact-size extent within one NUMA node |
| Contiguous transfer buffer | Pinned, hugepage-backed local buffer |
| Victim selection | Include socket-local capacity released |
| Readmission age | Include projected local-bandwidth availability |
| Protected decoding | Reserve cores and DRAM bandwidth, not only KV space |
| Transfer pipeline | Overlap packing, DMA, and remote-to-local migration |

The allocator should avoid silently satisfying a large snapshot from remote-socket memory. A nominally successful restore can then place every decode step behind Infinity Fabric traffic. Track both logical state and physical placement through the request lifecycle.

A practical victim score could combine:

[
S_i =
alpha cdot 	ext{reclaimable local bytes}_i
-
eta cdot widehat{	ext{remaining tokens}}_i
-
gamma cdot 	ext{restore cost}_i
+
delta cdot 	ext{stall age}_i
]

The coefficients should be calibrated against p99 streaming pauses and completion time, not only aggregate tokens/s.

### Costs and caveats

Stint does not dominate everywhere:

- it uses more pinned host memory;
- it replaces rare, unbounded stalls with more frequent but bounded pauses;
- pooled inter-token and first-token latency tails can increase;
- prediction-guided victim selection improves output continuity but can reduce throughput;
- short-output summarization and BurstGPT traces showed no benefit.

The main evaluation centers on Qwen3-32B and H200. The paper discusses other attention designs and provides a recompute-versus-offload model, but deployment results on MLA, recurrent attention, CPU-only decode, CXL memory, or multisocket hosts remain open.

**Reading priority:** Highest. Read Sections 3–6, especially the protected-recovery policy and CPU-pool design.

## 2. Fusion granularity should switch at runtime

**Assessment: Strong engineering contribution; model- and NVIDIA-specific implementation**

[High-Throughput Serving of Nemotron 3.5 Transcribe with Regime-Aware Megakernels and Kernel Fusion](https://arxiv.org/abs/2611.20095), submitted November 15, builds a serving engine that selects between persistent megakernels and specialized fused-kernel chains for every pass.

Megakernels reduce launch overhead and intermediate-memory traffic, but they constrain registers, occupancy, tiling, and scheduling. Independent or partially fused kernels have higher launch and synchronization overhead but can use better GEMM shapes and specialized launch configurations as work grows.

The key result is not the speech model’s absolute throughput. It is the measured crossover behavior:

- MoE uses a persistent megakernel below 448 routed rows.
- Above 448 rows, grouped FP8 GEMMs win.
- Above 1,200 rows, larger tiles and TMA multicast become profitable.
- At 128 concurrent requests, regime-aware selection delivered 23% more throughput than always using megakernels.
- At eight concurrent requests, it delivered 9% more than always using fused kernels.

On the full workload, the engine reached 1.39× vLLM’s peak throughput on H100 SXM and 1.64× at the highest measured B200 concurrency. Within a 400 ms p95 completion budget, it served eight times as many concurrent requests.

### Negative results worth reading

The paper is valuable because it reports optimizations that failed:

- an expert-major persistent FP8 GEMM was 1–3% slower because the power cap reduced SM clocks;
- INT4 experts moved fewer bytes but were slower than FP8 grouped GEMMs;
- statically partitioning SMs between draft and encoder never beat resource sharing;
- decode-only megakernels improved decode steps by 7.6–16.4% but changed end-to-end throughput by only 0.35%, because those steps carried just 0.3% of tokens;
- speeding the shorter of two parallel encoder branches did not shorten the critical path.

These are strong warnings against extrapolating isolated kernel wins.

### Practical implications for CPU inference optimization

This maps directly to ZenDNN and oneDNN dispatch. The runtime should treat fusion plans as candidates with measured operating regions, not permanent graph rewrites.

For each pass or continuous-batching iteration, select based on:

- active token rows;
- prompt/decode composition;
- routed tokens per expert;
- matrix shapes and packing reuse;
- LLC and memory-bandwidth pressure;
- thread-team size and NUMA placement;
- launch or task-scheduling overhead;
- power/frequency behavior.

A useful CPU dispatch table could look like:

| Regime | Likely strategy |
|---|---|
| Very small (M) | Persistent worker/task graph; aggressive row-op fusion |
| Medium (M) | Fused projections with shape-specialized GEMM |
| Large (M) | Library GEMMs plus selective epilogue fusion |
| Sparse MoE expert population | Persistent expert kernel or packed batched GEMM |
| Dense expert population | Grouped GEMM with shared packing and larger tiles |

The crossovers must be benchmarked under the serving thread topology. A kernel that wins alone may lose after it lowers all-core frequency, increases remote memory traffic, or blocks concurrent request work.

### Accuracy boundary

The accuracy-preserving path is the relevant comparison. Optional expert masking, reduced top-k routing, and layer skipping add 13–16% throughput but increase WER from about 1.92% to 3.44–3.48%. Those are explicit quality trades, not free systems improvements.

**Reading priority:** High for kernel and runtime engineers. Focus on Sections 5, 7.2, and 8.

## 3. Multimodal disaggregation adds an energy-control surface

**Assessment: Useful operational study; secondary to the two systems contributions above**

[A Matter of Stages: Energy-Latency Trade-offs in Multimodal LLM Serving](https://arxiv.org/abs/2611.21165), submitted November 16, compares three fused replicas with a one-encoder, one-prefill, one-decode pipeline under the same three-GPU budget.

Across four multimodal models, E/P/D disaggregation reduced GPU energy per successful request by 13.3–37.0% at nearly identical output-token throughput, but increased mean TTFT. Moderate uniform downclocking to 80% reduced energy by 11.7–18.6% with small mean-latency changes.

The important result is non-monotonicity. Lower frequency can increase total energy when queueing extends execution. In one Qwen3-VL workload, reducing prefill frequency to 40% raised TTFT by 4,756% near saturation. Uniform 40% consumed less average power than 50% but more total energy because the benchmark ran longer.

For CPU deployments, the analogue is per-stage core and frequency control. Prefill can cross a queue-stability boundary abruptly, while moderate decode downclocking may save energy because decode often has bandwidth slack. Optimize joules per SLO-compliant request, not socket power or joules per isolated token.

**Reading priority:** Medium. Useful for capacity and energy policy, but the exact frequency settings are not portable.

## What changed from the previous briefing

Yesterday’s briefing showed how to use idle decode capacity and invalidated KV for provisional work. Today’s papers address what happens when the system has **too much admitted long-running work**:

- Stint prevents repeated preemption from turning into starvation.
- Regime-aware kernels adapt execution strategy as dynamic batch composition changes.
- Stage-wise energy control avoids configurations that save instantaneous power but destabilize queues.

The shared theme is progress-aware control. Memory, kernel, and frequency policies should be evaluated by whether they advance end-to-end requests under contention.

## Bottom line

Stint is today’s standout. Its most transferable insight is that state recovery is incomplete unless the scheduler guarantees new output afterward. For a CPU or hybrid inference stack, the immediate prototype should combine:

1. NUMA-local, exact-extent KV snapshots;
2. bulk gather/scatter with pipelined copies;
3. age-bounded readmission;
4. a protected fresh-token quantum;
5. victim selection that considers both reclaimable bytes and predicted remaining work.

The Nemotron serving paper supplies the complementary kernel lesson: keep multiple fusion strategies alive and select them at empirically measured crossovers. Static fusion and isolated microbenchmarks are insufficient when continuous batching changes shapes, power behavior, and critical paths every iteration.
