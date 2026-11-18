# AI inference systems briefing — November 18, 2026

## Executive assessment

The important change since the previous briefing is a move from treating inference memory as a single capacity pool to treating it as several state classes with different semantics and access patterns.

Two new papers make that shift concrete:

1. **XM-Sparse** separates unpredictable sparse-KV fetches from predictable K-index scans and assigns each a different transfer path.
2. **Gemma 4 K=V compaction** exploits an exact algebraic relationship in the model to eliminate redundant KV storage before applying lossy compression or eviction.

A third paper, on porting the Mirage graph superoptimizer to Ascend, supplies a useful warning: a portable graph representation does not make optimization decisions portable. Partitioning, fusion, launch geometry, and search policy still need a target-specific machine model.

For CPU inference, the practical theme is straightforward: classify state by reuse, predictability, and recomputation cost; preserve model-level aliasing in the runtime; and make placement and fusion decisions with a NUMA- and shape-aware cost model.

## 1. XM-Sparse: external memory changes the bottleneck, but does not remove it

**Importance: High for long-context sparse attention and heterogeneous memory systems**

[XM-Sparse](https://arxiv.org/abs/2611.22280), submitted November 17, studies sparse-attention serving with a scale-up-attached memory pool. Sparse attention reduces arithmetic, but it does not eliminate the full KV history. Moving that history out of HBM can relieve capacity and PCIe bandwidth pressure, yet the authors find that local HBM remains the concurrency limiter because weights, hot KV buffers, and the resident key index still compete for space.

The central contribution is not simply “use faster external memory.” It is the decision to split memory traffic by access behavior:

| State or traffic class | Access pattern | Mechanism |
|---|---|---|
| Selected KV misses | Fine-grained, irregular, latency-sensitive | TMA-based retrieval fused into the attention tile pipeline |
| Key index | Full-history, predictable, bandwidth-heavy | GPU-initiated DMA prefetch |
| MoE communication | Collective, latency-sensitive | Priority over bulk DMA traffic |
| Hot KV working set | Reused locally | Capacity-tuned HBM buffer |

That decomposition matters because a single copy mechanism performs poorly across all four cases. The paper reports that TMA can sustain 394 GB/s with 16 copy blocks at a 6.3% decode-throughput cost. Raising the allocation to 32 blocks reaches 612 GB/s but costs 29.4% of decode throughput. Bulk DMA has different problems: small-page CPU submission becomes a bottleneck, and unconstrained transfers interfere with NCCL and MoE collectives. XM-Sparse therefore generates descriptors on the GPU, batches CPU doorbells, and pauses lower-priority copy traffic at semaphore checkpoints.

On SGLang with DeepSeek-V3.2 and GLM-5.3 workloads spanning 16K to 1M tokens, the paper reports up to 4.98× the throughput of a PCIe-based ECHO configuration, 4.52× HiSparse, and 1.34× a naive integration on the same scale-up platform. At 128K context, it reaches 94.2% of compute-only throughput. Yet only about 3 TB of a nominal 24 TB external pool is used because HBM fills first; in representative configurations the K-index alone consumes 33.7% of HBM for DeepSeek and 24.9% for GLM.

### Why it matters for CPU and EPYC systems

The transferable result is the traffic classification, not the GPU-specific TMA implementation.

On a multisocket CPU server, the analogous design would be:

- keep latency-sensitive, irregular KV pages in local NUMA memory or fetch them into per-socket staging buffers;
- stream predictable index scans with asynchronous prefetch or DMA;
- schedule bulk CXL or remote-socket transfers below synchronization and collective traffic;
- size the local hot set by measured miss cost rather than by a fixed fraction of DRAM;
- account separately for metadata/index footprint, which can become the new local-capacity bottleneck after values move outward.

This also changes benchmark design. “External-memory bandwidth” is insufficient as a headline metric. Measure request concurrency, local-buffer occupancy, index footprint, remote-read amplification, copy-engine or memory-controller contention, and p99 inter-token latency while sweeping hot-set size.

### Assessment and caveat

This is a substantive architecture paper because it identifies distinct data paths and quantifies interference between them. The largest caveat is that the scale-up memory pool is emulated using additional B300 GPUs as memory nodes. The results establish a design target, not yet the economics or behavior of a production CXL or vendor scale-up memory system.

**Read:** [XM-Sparse: Efficient Sparse Attention Serving with Scale-Up Memory](https://arxiv.org/abs/2611.22280)

## 2. Gemma 4 exposes exact KV redundancy that runtimes currently miss

**Importance: High conceptually; model-specific in immediate applicability**

[Exact KV Cache Compaction for Gemma 4](https://arxiv.org/abs/2611.23035), also submitted November 17, observes that Gemma 4’s global-attention layers tie keys and values algebraically. The value vector is the normalized key projection; the key is that same vector with scaling and RoPE applied to a subset of dimensions.

A conventional runtime materializes and stores both tensors. The prototype instead stores a single 640-value row per head and exposes overlapping strided K and V views to an otherwise unchanged paged-attention kernel. The usual representation stores 1,024 values per head and token. The transformation is exact in real arithmetic and avoids an accuracy-versus-capacity tradeoff.

In a vLLM prototype for the 31B model in 16-bit precision:

- steady-state decode KV memory per request improves by 1.34–1.40× at 32K–64K contexts;
- a two-A100 prefill/decode deployment using NIXL and CPU offload admits 1.38× more decode tokens per step for 16K prompts and 1.24× more for 8K prompts;
- eager-mode throughput improves by 1.30× and 1.11% in the reported cases;
- at equal batch size, an individual step can be 4–9% slower, and one decode-node case is 11% slower, because the kernel consumes a less conventional layout.

The last point is the most useful systems lesson. The representation wins by increasing feasible concurrency, not by making every kernel invocation faster. A microbenchmark that fixes batch size would understate or even reverse the deployment-level result.

### Why it matters for CPU inference

Before quantizing, paging, or evicting state, inspect whether the model graph contains exact redundancy that export and lowering have erased. A CPU runtime should be able to represent:

- aliased or tied state tensors;
- transformed views rather than duplicated materializations;
- query-side folding of deterministic key transformations;
- capacity-aware dispatch that accepts a slightly slower kernel when the compact representation raises useful batch size.

For ZenDNN or another CPU graph backend, this argues for carrying semantic relationships through the IR instead of lowering K and V immediately into independent opaque buffers. On a bandwidth-bound decode path, eliminating redundant DRAM traffic can be more valuable than another small improvement in the dot-product microkernel.

A useful validation sequence would compare:

1. conventional K+V storage;
2. exact shared storage with reconstructed views;
3. the same compact representation plus lower-precision KV;
4. each configuration at equal batch and at maximum SLA-compliant concurrency.

Record memory-controller bandwidth, LLC misses, local versus remote NUMA traffic, tokens/s, and p99 time per output token. The equality test should cover long generations and all execution modes, not only a short greedy decode.

### Assessment and caveat

The idea is genuinely important because it attacks state volume without approximation. Its immediate scope is narrow: the exact relationship is specific to Gemma 4 variants, the evaluation uses one vLLM prototype and A100 systems, and the compact layout does not yet compose cleanly with every 8-bit KV-cache path. Treat this as a prompt to search model algebra systematically, not as a generally available 1.4× KV reduction.

**Read:** [Exact KV Cache Compaction for Gemma 4 via Key–Value Equality](https://arxiv.org/abs/2611.23035)

## 3. Mirage on Ascend: portable graph search still needs a hardware-specific optimizer

**Importance: Medium-high for compiler and kernel-stack design**

[Porting a Graph Superoptimizer Across Accelerator Architectures](https://arxiv.org/abs/2611.22844) adapts the CUDA-oriented Mirage optimizer to Huawei Ascend 910B3 through Triton. The paper reports gains as high as 87% on some fused workloads, but also substantial regressions on others. For example, one feed-forward configuration improves by 16.1%, while larger configurations are 51.5% and 151.8% slower. Two attention cases improve by 34.6% and 52.7%; another regresses by 8.2%.

The failures are more informative than the best speedups:

- tensor partitioning does not model the target’s unified-buffer capacity;
- grid selection does not properly account for the 40 Ascend AI Cores;
- the backend lacks the CUDA path’s threadblock-fusion lowering;
- attention lacks an equivalent online-softmax lowering;
- verification and search costs change with the target.

The reference kernels for some fused workloads are generated rather than expert-tuned, so the headline gains do not prove superiority over a mature native library. What the work does show is that semantic search rules can transfer while profitability rules do not.

### Practical implication for CPU optimization

Do not transplant a GPU fusion or superoptimization policy directly into a CPU backend. Preserve the algebraic search space, but rebuild the cost model around:

- cache hierarchy and NUMA placement;
- vector width, register pressure, and AMX or AVX-512 tile constraints;
- thread-team size and barrier cost;
- packing amortization across prompt and decode shapes;
- memory-bandwidth saturation;
- production shape distributions rather than isolated square matrices.

Every generated rewrite should pass both differential correctness testing and a shape/ISA-specific performance gate. The right artifact is not one “best” fused graph; it is a dispatch table whose regions are learned and then constrained by interpretable hardware limits.

### Assessment and caveat

This is a useful negative-results paper and a good engineering read. It demonstrates optimizer portability limits more convincingly than it demonstrates state-of-the-art Ascend performance.

**Read:** [Cross-Architecture Graph Superoptimization: Porting Mirage to Ascend](https://arxiv.org/abs/2611.22844)

## Secondary signal: hardware-aware speculative-decoding search

[SpecSearch](https://arxiv.org/abs/2611.22296) jointly selects speculative-decoding strategy, draft length, and precision for a target workload and device. It reports that a configuration delivering 3.5× autoregressive throughput on an RTX 5090 at low concurrency falls to 0.76× under high concurrency with short outputs. Across RTX 5090, H100, L40, and MI300X tests, its predictor reaches 97.36% of fully calibrated search performance.

This is credible supporting evidence for regime-aware dispatch, but it is less novel than the three items above because yesterday’s briefing already covered the same broader lesson: optimization rankings change with concurrency, sequence mix, and hardware. The useful action is to include speculative parameters in the serving autotuner rather than standardizing one draft policy cluster-wide.

**Read:** [SpecSearch: Hardware-Aware Search for Speculative Decoding](https://arxiv.org/abs/2611.22296)

## Recommended engineering actions

1. **Add a state-class inventory to the runtime.** For every cache, index, activation, and staging buffer, record size, reuse distance, access predictability, recomputation cost, and locality requirement.
2. **Make local capacity a first-class constraint.** Model HBM or socket-local DRAM separately from aggregate external capacity; metadata and indexes can become the limiting resident set.
3. **Search for exact semantic compaction before lossy methods.** Preserve aliasing and deterministic tensor relationships through export, graph lowering, and cache allocation.
4. **Benchmark capacity and speed together.** Report equal-batch kernel time and maximum SLA-compliant concurrency; either metric alone can select the wrong representation.
5. **Use target-specific profitability models.** Share rewrite correctness across architectures, but specialize partitioning, fusion, thread geometry, and dispatch thresholds for each CPU and memory topology.

## Bottom line

The strongest new work is not another isolated kernel speedup. It changes how inference state should be represented and moved.

XM-Sparse shows that irregular KV misses, predictable index scans, and collectives require different transfer and priority mechanisms. The Gemma 4 result shows that some cache pressure should be removed at the semantic level before the runtime reaches for quantization or eviction. The Mirage port shows why these decisions cannot be delegated to a supposedly architecture-neutral optimizer without rebuilding its machine model.

For an experienced CPU inference team, the highest-value near-term prototype is a semantic, NUMA-aware state planner: preserve exact tensor aliasing, classify traffic by predictability and urgency, place each class in the appropriate memory tier, and select kernels using maximum SLA-compliant concurrency rather than equal-batch latency alone.