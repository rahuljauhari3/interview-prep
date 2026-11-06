# AI inference systems briefing — November 6, 2026

## Executive take

The most important new result since yesterday’s InferaSim briefing is **StrataFlux**, a system that treats numerical precision as a memory-hierarchy resource rather than a fixed property of a tensor.

[StrataFlux: Memory-Hierarchy-Native Low-Bit LLM Inference](https://arxiv.org/abs/2611.08363), submitted November 5, keeps aggressively quantized weights and the complete KV cache in the fastest tier, while retaining compact refinement bits in host RAM or SSD. At each decoding step it fetches refinements only for activation-important weight channels and attention-important KV tokens, overlaps those transfers with other Transformer work, and reuses fetched refinements across later steps.

The result is not ordinary offload. The fast tier always contains a valid low-bit model and cache, so a late refinement can be skipped rather than stalling generation. The slower tier improves fidelity opportunistically.

On an RTX 5090, the paper reports that refining 2-bit Llama-3.1-8B-Instruct weights lowers WikiText-2 perplexity from 9.46 to 8.40 while retaining 95.4% of the low-bit base’s throughput. With 2-bit weights and 2-bit KV, Qwen3-8B AIME 2024/2025 accuracy rises from 11.0% to 47.1% without increasing GPU-memory use. At 128K context, a 3-bit-weight/2-bit-KV configuration is reported to match the perplexity and speed of vLLM’s 4-bit AWQ weights plus 8-bit KV while using roughly half the GPU memory.

For CPU inference engineering, the significant idea is that **DRAM, CXL memory, or SSD need not hold a second complete tensor representation**. They can hold only the residual information most likely to change the answer, with the base computation remaining locally executable. This opens a new design point between static low-bit inference and synchronous high-precision offload.

**Assessment:** a substantive representation-and-runtime co-design with an end-to-end llama.cpp implementation and unusually broad model/platform evaluation. It is still an arXiv v1 without a linked public implementation, and its strongest end-to-end results are on consumer NVIDIA GPUs. The architecture is highly relevant to CPU systems; the reported speedups are not directly portable to them.

## 1. What changed: precision becomes a cacheable overlay

Conventional quantization commits each tensor to one resident precision. Conventional offload moves a higher-precision tensor between tiers. StrataFlux instead decomposes a value into:

[
x approx Q_b(x) + R_{Delta b}(x)
]

where (Q_b(x)) is the always-resident low-bit base and (R_{Delta b}(x)) is a compact refinement stored in a slower tier.

This creates three useful properties:

| Property | Conventional offload | Selective precision streaming |
|---|---|---|
| Fast-tier state | Incomplete until offloaded data arrives | Complete low-bit representation is always usable |
| Slow-tier payload | Full tensor or full-precision replacement | Additional bits for selected channels or tokens |
| Miss behavior | Wait, recompute, or fail the request | Continue at base precision; cache refinement for later |
| Reuse unit | Tensor, layer, or KV block | Precision overlay for recurring selections |

That last distinction matters. In agentic decoding, adjacent steps often select similar weight channels and attend to similar tokens. The paper shows that caching a fraction of possible refinements can yield high hit rates because importance is temporally correlated. A refinement that arrives too late for step (t) can still help step (t+1).

The system therefore optimizes delivered precision under a latency budget, rather than assuming that every requested byte must arrive before computation proceeds.

## 2. Weight refinement: fetch early, apply late

For a quantized weight matrix (W_c), with residual (R = W-W_c), the projection error for activation (x) is:

[
(W-W_c)x = Rx = sum_i x_i R_{:,i}
]

Large (|x_i|) identifies channels whose residuals are most likely to matter. StrataFlux selects those channels dynamically and fetches only their refinement bits.

The scheduling contribution is more important than the selection rule. Fetching after the exact projection input is known leaves only that projection’s execution time to hide the transfer. StrataFlux expands the interval in both directions:

- **Fetch lead:** an earlier residual-stream activation predicts important channels for a later Q/K/V or gate/up projection.
- **Fetch lag:** selected refinements can be applied at a later point where the resulting correction remains useful.
- **Cross-block overlap:** transfers and refinement kernels overlap intervening Transformer operations rather than only the target GEMM.

The paper’s tuner searches channel budgets and lead/lag distances under a specified end-to-end slowdown budget. The main weight experiments use a 5% budget. On the RTX 5090 example, 2-bit Llama-3.1-8B-Instruct runs at 234.4 tokens/s after refinement versus 245.8 tokens/s for the unrefined base; partial BF16 llama.cpp offload at the same 3.9 GB GPU-memory budget reaches only 5.1 tokens/s.

This is the strongest practical contrast in the paper: transferring a small, computation-relevant residual can be two orders of magnitude cheaper than making full-precision placement the unit of movement.

## 3. KV refinement: rescore a small selected set

The full KV history remains resident at low precision. Attention first scores all tokens from the low-bit keys, then selects a small number of contiguous token groups. The runtime fetches nested refinement bits for those groups, reconstructs finer keys and values, and corrects their contribution to the softmax output.

The main configuration selects 1/128 of token groups and adds four refinement bits per selected key and value element. Tokens are grouped contiguously—two per group on dedicated GPUs and four on unified-memory systems—to make reads and kernels less gather-heavy.

The correction preserves contributions from unselected tokens. This differs from sparse attention, which omits them, and from fetching complete FP16 replacements, which moves more data than necessary.

For Llama-3.1-8B-Instruct at 128K on RTX 5090, 2-bit keys and values improve from 68.6% to 78.1% on RULER after refinement; the BF16 reference is 77.9%. Across the batch sweep, however, refinement overhead increases: the paper reports throughput reductions of 11.2% at batch 1 and 18.8% at batch 16 on RTX PRO 6000 Max-Q, and 7.0% and 25.4% respectively on DGX Spark.

That is an important boundary. The technique is not free, and selection, rescoring, metadata, and cache access become harder to hide as concurrency changes.

## 4. The miss policy is the architectural breakthrough

On dedicated-GPU systems, refinements are fetched directly from pinned host RAM. On unified-memory systems, the compact base is in shared RAM, refinement bits reside on SSD, and CPU loader threads fill separate weight and KV refinement caches.

On a cache miss, the runtime can either:

1. wait and apply the refinement now; or
2. skip it for the current step, execute with the resident base, and retain the fetched bits for later steps.

StrataFlux normally chooses the second policy on SSD-backed configurations. This turns storage latency from a hard dependency into an accuracy opportunity.

A useful way to model the decision is:

[
	ext{wait if }
Delta Q_i cdot V_Q
>
T_{mathrm{stall},i}cdot V_L
]

where (Delta Q_i) is the expected quality improvement from refinement (i), (T_{mathrm{stall},i}) is the exposed wait, and (V_Q,V_L) translate quality and latency into the deployment objective.

The current system uses configured speed budgets and cache policies rather than a per-request value model, but the architecture makes such a policy possible. Different tenants or request classes could choose different points:

- latency-critical interactive requests skip late refinements;
- accuracy-critical reasoning waits for high-value refinements;
- bulk jobs enlarge the overlay cache or prefetch more aggressively;
- speculative paths use base precision while the committed path receives refinements.

## 5. Implications for EPYC, ZenDNN, and CPU-only inference

The paper’s implementation targets CUDA and Metal, but the abstraction maps well to CPU memory hierarchies.

### 5.1 Treat local DRAM as the base tier

On a multisocket EPYC server, keep the low-bit base and hot refinement cache socket-local. Place colder overlays in remote DRAM, CXL memory, or NVMe. The hierarchy should be explicit:

[
	ext{LLC/local DRAM}
ightarrow
	ext{remote DRAM}
ightarrow
	ext{CXL memory}
ightarrow
	ext{NVMe}
]

A refinement request should carry its expected quality gain, byte count, reuse probability, and deadline. The scheduler can then decide whether it is worth crossing a NUMA boundary.

### 5.2 Fuse base and refinement arithmetic

A naïve CPU port would dequantize the base, materialize corrections, and then add a second GEMV. That can erase the bandwidth savings. The desirable kernel is:

[
y =
operatorname{GEMV}_{b}(W_b,x)
+
operatorname{SparseRefine}(R_S,x_S)
]

with the selected residual columns packed for contiguous vector loads and accumulated directly into the output buffer.

For ZenDNN or oneDNN, the engineering question is whether the refinement term should be fused into the epilogue, run as a separate sparse microkernel, or assigned to a sibling CCD while the main GEMV consumes bandwidth on another CCD.

### 5.3 Make refinement caching topology-aware

A global hit rate is insufficient. Track:

| Outcome | Cost model |
|---|---|
| Local overlay hit | LLC/DRAM latency and local bandwidth |
| Remote-socket hit | Fabric latency plus remote bandwidth |
| CXL hit | Device latency, link bandwidth, and contention |
| NVMe hit before deadline | I/O plus decompression/packing |
| Late arrival | Future reuse value only |
| Miss and skip | Base-precision quality cost |

The overlay cache should optimize **quality recovered per local byte-second**, not simple LRU hit rate. A rarely reused refinement with a large expected accuracy benefit may be worth retaining over a frequently reused but low-impact one.

### 5.4 Use spare CPU capacity deliberately in heterogeneous serving

In CPU–GPU deployments, CPU threads do more than issue I/O. They can rank candidates, assemble packed refinement slabs, decompress overlay data, and run sparse corrections. This should be scheduled against the same DRAM channels used for GPU DMA. Otherwise, CPU refinement preparation can slow the base weight transfer it is meant to complement.

The bandwidth constraint becomes:

[
B_{mathrm{base}}
+
B_{mathrm{overlay}}
+
B_{mathrm{KV}}
+
B_{mathrm{DMA}}
le
B_{mathrm{socket,sustainable}}
]

and should be evaluated per socket, not for the host in aggregate.

## 6. Engineering experiment worth running

A useful CPU-oriented validation would compare four representations on Llama-3.1-8B or Qwen3-8B:

1. static 4-bit weights with 8-bit KV;
2. static 2-bit weights with 2-bit KV;
3. 2-bit base plus synchronous high-precision fetching;
4. 2-bit base plus asynchronous, skippable precision overlays.

Run on one and two EPYC sockets, with overlays in local DRAM, remote DRAM, CXL if available, and NVMe. Sweep batch 1–32, contexts 8K–128K, and cache budgets from 64 MiB to several GiB.

Record:

- base and refined quality on perplexity, long-context retrieval, and reasoning tasks;
- tokens/s and TTFT/TPOT p50/p95/p99;
- overlay requests, hits, late arrivals, skips, and reuse distance;
- bytes fetched per recovered quality point;
- local and remote DRAM traffic;
- memory-controller saturation and Infinity Fabric traffic;
- NVMe request size, queue depth, and read amplification;
- cycles spent selecting, packing, decoding, and applying overlays;
- quality variance across requests caused by timing-dependent misses.

The last metric is essential. If a refinement is opportunistic, identical prompts can experience different numerical paths under load. Production use needs determinism controls, observability, and a policy for whether cached overlays may affect output reproducibility.

## 7. Evidence quality and caveats

### Genuinely important

- The work co-designs representation, selection, delivery, and caching; it is more than a new quantizer.
- The low-bit base is always executable, removing slow-tier arrival from the correctness critical path.
- Weight and KV refinements share one system abstraction but use different selection and scheduling mechanisms.
- The evaluation covers seven model families or sizes, dedicated GPUs, NVIDIA unified memory, and Apple unified memory.
- The implementation is end-to-end in llama.cpp rather than an isolated kernel simulation.
- The paper reports accuracy, memory, and throughput together and includes batch-dependent overheads and cache-policy ablations.

### Treat cautiously

- This is an arXiv v1, and no public code repository is linked from the paper page.
- Most headline throughput results are on RTX 5090; AMD GPU and server CPU results are absent.
- Dynamic weight selection adds control and sparse-access overhead that may behave differently on CPUs without GPU-style latency hiding.
- The main quality claims mix perplexity, RULER, MMLU-500, and AIME. Larger deployment-specific evaluations are needed before selecting a 2-bit base.
- Skipping late refinements makes delivered precision dependent on cache warmth and system load. The paper does not fully characterize output reproducibility or p99 quality under contention.
- Backing-tier capacity includes refinement data, metadata, and some duplicated values. Fast-tier memory falls substantially, but total-system storage is not reduced by the same factor.
- SSD endurance, multi-tenant interference, crash recovery, and overlay-cache persistence are not production-evaluated.
- The technique is tuned separately for each model, quantizer, and platform. It does not eliminate calibration or hardware-specific search.

## Secondary signal: ServerBench evaluates agents that optimize complete serving systems

[ServerBench](https://arxiv.org/abs/2611.06841), submitted November 4, gives coding agents a stateful serving workload, a latency-qualified value-per-dollar objective, and independent verification of the resulting system. Across 11 agent models and three deployments, 22 of 33 runs produced verified systems and 13 beat the strongest baseline; the best reported Gemma deployment reached 4.87× the vLLM reference on the benchmark’s objective.

The useful contribution is the evaluation harness, not the claim that agents can replace inference engineers. It forces optimizations to survive session dependencies, workload replay, correctness checks, and cost constraints. This complements yesterday’s InferaSim result: simulation can search declared configuration spaces, while an experimental agent can change implementation and policy—but only a strong verifier can distinguish a real optimization from a fast broken path.

Treat the reported agent ranking and speedup as benchmark-specific. The broader lesson is credible: future inference autotuning should verify outputs and exercised code paths, not merely accept better throughput.

## Reading priority

1. **[StrataFlux paper](https://arxiv.org/abs/2611.08363)** — Read §3.1–3.4 for representation, cross-block delivery, KV correction, and miss handling; §4 for end-to-end results; Appendix C for cache, batch, and refinement-budget sweeps.
2. **[ServerBench paper](https://arxiv.org/abs/2611.06841)** — Read for its stateful replay objective and independent verification methodology, not as evidence that one agent model will generalize to production systems work.
3. **[ROCm InferaSim](https://arxiv.org/abs/2611.04032)** — Revisit alongside ServerBench: it searches a known deployment space from measured anchors, whereas ServerBench evaluates systems that can change the implementation itself.
4. **[Any-Precision LLM](https://proceedings.mlr.press/v235/park24s.html)** — Background for nested refinement representations.
5. **[KTransformers](https://doi.org/10.1145/3731569.3764843)** — Related CPU–GPU precedent for deferring work across Transformer blocks to create overlap.

## Bottom line

Yesterday’s briefing focused on predicting the best serving configuration from a small number of measured anchors. Today’s result changes the object being configured: precision itself can be scheduled across the memory hierarchy.

The key insight is not “store extra bits on SSD.” It is to keep a complete, fast, low-bit path resident and make higher precision **selective, asynchronous, reusable, and skippable**. That converts slow memory from a blocking tensor store into a quality cache.

For CPU inference systems, the most promising next step is a NUMA-aware precision-overlay prototype: local low-bit weights and KV, sparse refinements in remote or tiered memory, a fused correction kernel, and a cache policy that maximizes recovered quality under explicit p99 latency and bandwidth constraints. If that policy works, the usual choice between a fast inaccurate representation and a slow accurate one becomes a workload-dependent continuum rather than a build-time decision.
