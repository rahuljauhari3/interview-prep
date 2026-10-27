# AI inference systems briefing — October 27, 2026

## BellowsKV compresses across layers without reducing latent rank

**Importance: High for KV-capacity planning; promising but not yet a production-ready serving primitive**

A new October 26 paper, [BellowsKV: One Latent for Many Layers, Expanded on Demand](https://arxiv.org/abs/2610.35023), reports a training-free method that stores one shared latent representation for a block of adjacent attention layers instead of retaining a separate key and value tensor for every layer.

Its main result is unusually strong: approximately 99% of full-cache task accuracy at roughly 3.6× compression, and 97–98% at about 13–14× across Llama-3.1-8B, Mistral-7B, and Qwen2.5-7B. The strongest compared cross-layer method retains at most 84% at a similar high-compression point. The authors also provide [an implementation](https://github.com/MorrisZJ/BellowsKV). genui{"citation":{"ref":"turn77view0"}}

The contribution is not merely another KV quantizer. It changes what is cached.

## What changed

Most KV-compression approaches act on each layer’s cache independently:

- quantization reduces bits per key/value element;
- eviction stores fewer tokens;
- low-rank projection stores fewer dimensions;
- cross-layer methods factorize or reuse representations derived from existing caches.

BellowsKV starts from a different empirical finding:

- adjacent layers’ keys and values have near-zero cosine similarity;
- adjacent layers’ input hidden states are highly similar—reported around 0.87 for Llama and 0.82 for Qwen;
- neighboring layers rank token importance similarly, with Spearman correlations around 0.76–0.80;
- reducing the shared representation’s dimension damages accuracy much more than reducing its bit width. genui{"citation":{"ref":"turn78view0"}}

The resulting rule is:

> Share the representation before the per-layer K/V projections, retain its full rank, and obtain compression through layer sharing and lower precision before considering eviction.

For a block of \(n\) layers, BellowsKV replaces the block’s per-layer keys and values with shared key and value latents:

\[
\mathbf{c}_t =
\mathbf{W}_{\downarrow}^{\top}\mathbf{h}_{\mathcal{B},t},
\qquad
\hat{\mathbf{k}}_{\ell_i,t} =
\mathbf{W}_{\uparrow}^{(i)\top}\mathbf{c}_t
\]

Each layer receives its own up-projection, so layers do not incorrectly share reconstructed keys. The down- and up-projections are fitted once from activation statistics using a closed-form factorization and then folded into the model weights. A small per-sequence low-rank “whisper” corrects key-reconstruction residuals. genui{"citation":{"ref":"turn78view0"}}

## Where the compression comes from

| Configuration | Mechanism | Measured compression | Reported behavior |
|---|---|---:|---|
| \(n=2\), INT8 | Share across two layers and quantize latent | ~3.6× | About 99% accuracy retention |
| \(n=4\), INT4 | Share across four layers and quantize latent | ~13–14× | About 97–98% retention |
| \(n=4\), INT3 | More aggressive latent quantization | ~16.5× | About 97% mean retention in the cited comparison |
| Quantization plus block eviction | Fewer bits and fewer shared tokens | ~26–30× | Strong on two models; one compared Qwen result exposes instability in a baseline |

All metadata is included in the reported compression accounting. The paper evaluates five models overall, with LongBench-style tasks and a BrowseComp-Plus retrieve-then-read workload. Its high-compression comparisons are much broader than perplexity-only evaluation. genui{"citation":{"ref":"turn79view3"}}

A particularly useful negative result is that lowering latent rank is the wrong lever at tight memory budgets. At approximately 16× compression, a rank-reduced xKV configuration retains 83.5%, 79.5%, and 54.4% on the three reported models, while BellowsKV’s full-rank INT4 latent retains 98.0%, 98.1%, and 97.3%. Quantization preserves the representational directions; rank reduction deletes them. genui{"citation":{"ref":"turn79view3"}}

## The most important systems caveat: compression buys capacity, not speed

BellowsKV’s current decode path runs at 0.995× the dense-cache speed on an H100 with Hugging Face at a 32K context. The authors’ cost model predicts parity.

That is not a failed result. It confirms that reconstructing per-layer K/V does not impose measurable slowdown in the tested configuration. But it also means the headline 13.8× byte reduction does **not** become a 13.8× attention speedup.

The paper makes the boundary explicit:

\[
\text{decode speedup}
=
\frac{d_{\mathrm{KV}}}{R}
\cdot
\frac{2}
{\text{bytes per latent value at read}}
\]

BellowsKV fixes \(R=d_{\mathrm{KV}}\) to preserve accuracy and consumes the latent in FP16 after dequantization. Cross-layer sharing removes persistent copies but does not reduce the number of reconstructed values each layer uses. The immediate benefits are therefore:

- more concurrent sequences;
- longer resident contexts;
- less cache eviction or offload;
- fewer capacity-triggered preemptions;
- reduced persistent memory per NUMA node or accelerator.

In the reported implementations, the freed memory supports a 2× batch in Hugging Face and a 14.8× larger KV pool in vLLM, although the authors have not isolated why the latter slightly exceeds the measured 13.8× byte ratio. genui{"citation":{"ref":"turn79view0"}}

## Why it matters for CPU and EPYC inference

On CPU servers, KV capacity and KV bandwidth are related but distinct bottlenecks. BellowsKV clearly attacks capacity. Whether it improves tokens/s depends on the reconstruction path.

A CPU implementation would introduce this tradeoff:

\[
T_{\mathrm{Bellows}}
=
T_{\mathrm{latent\ read}}
+
T_{\mathrm{dequant}}
+
T_{\mathrm{up\ projection}}
+
T_{\mathrm{attention}}
\]

versus:

\[
T_{\mathrm{dense}}
=
T_{\mathrm{KV\ read}}
+
T_{\mathrm{attention}}
\]

If the latent is expanded into a temporary FP16 K/V buffer, much of the bandwidth benefit disappears and the temporary allocation can become a new NUMA problem. The useful CPU design is a fused path that consumes the compressed latent directly or reconstructs tiles inside the attention kernel without materializing the full cache.

This suggests three implementation tiers:

1. **Capacity-only integration:** store the compressed latent, reconstruct per-layer K/V into workspace, and validate correctness and concurrency gains.
2. **Tiled reconstruction:** dequantize and up-project only the K/V tile currently consumed by attention.
3. **Fused latent attention:** fold value up-projections into the attention output path and fuse key reconstruction, positional transformation, scoring, and reduction.

The paper already folds value up-projections into the output projection because attention is linear in values. Keys still require reconstruction because positional rotation lies between the latent and attention score. That key path is the main CPU-kernel opportunity. genui{"citation":{"ref":"turn78view0"}}

## NUMA-aware deployment implications

BellowsKV can change the preferred placement strategy on a multisocket EPYC server.

Without compression, a large cache may be partitioned across sockets, forcing attention to read remote pages or perform partial reductions. With 13× lower persistent state, more requests may fit entirely inside one socket’s DRAM. The benefit can therefore be discontinuous:

\[
\Delta T
=
\Delta T_{\mathrm{reconstruction}}
-
\Delta T_{\mathrm{remote\ reads}}
-
\Delta T_{\mathrm{cross\ socket\ reduction}}
-
\Delta T_{\mathrm{preemption}}
\]

A reconstruction kernel that is slightly slower locally may still win end to end if compression moves a request from cross-socket execution to socket-local execution.

For a ZenDNN-style prototype, test four variants:

| Variant | Persistent cache | Reconstruction | Placement question |
|---|---|---|---|
| Dense baseline | Per-layer BF16/FP16 | None | Does the request spill across sockets? |
| Bellows materialized | Shared low-bit latent | Full temporary K/V | Is capacity gain erased by workspace traffic? |
| Bellows tiled | Shared low-bit latent | Per-attention tile | Can reconstruction stay in LLC? |
| Bellows socket-local | Per-socket compressed latent | Fused/tiled | Does replication beat remote access? |

Measure:

- persistent and peak transient bytes per request;
- local and remote DRAM reads;
- latent dequantization and up-projection cycles;
- LLC residency and TLB misses;
- attention throughput by context length;
- batch capacity before preemption;
- p50/p99 TPOT and worst streaming gap;
- TTFT impact from the per-sequence whisper fit;
- accuracy under multi-turn and retrieval-heavy traces.

The right headline metric is not compression ratio. It is SLO-compliant concurrency per socket:

\[
C_{\mathrm{useful}}
=
\max C
\quad\text{such that}\quad
\mathrm{TPOT}_{p99}(C)\leq S_{\mathrm{TPOT}}
\land
\mathrm{TTFT}_{p99}(C)\leq S_{\mathrm{TTFT}}
\]

## Operational costs and integration risks

The method is described as training-free, but that should not be confused with deployment-free.

Model preparation requires one calibration forward pass over 32 windows of 2,048 tokens and one SVD per layer block—reported at about 20 minutes on one GPU per model. Each sequence also requires the whisper sketch during prefill, costing milliseconds per block in the authors’ implementation. Decode adds dequantization and reconstruction operations. genui{"citation":{"ref":"turn79view4"}}

Production integration must also answer:

- How are latent pages represented in a paged KV allocator?
- Can prefix-cache blocks be shared across requests without duplicating whisper metadata?
- Does block-level eviction remain compatible with cache hashing and partial-prefix reuse?
- How are mixed exact and compressed windows transferred between prefill and decode workers?
- Can the latent remain compressed over RDMA, CXL, or PCIe?
- What happens when a model uses sliding-window, recurrent, or hybrid attention layers?
- How are projection factors versioned with quantized model artifacts?

These questions determine whether the memory saving survives contact with the serving stack.

## What is genuinely important versus preliminary

**Genuinely important:**

- The paper identifies that cross-layer redundancy lies in hidden-state inputs, not in the K/V tensors themselves.
- It separates rank preservation from bit-width reduction with controlled equal-budget comparisons.
- It reports strong accuracy retention at compression ratios where prior cross-layer methods degrade sharply.
- It includes metadata in capacity accounting, multiple model families, agent-style retrieval evaluation, negative results, and public code.
- It states honestly that layer sharing buys capacity rather than decode acceleration in the current path.

**Still preliminary:**

- Decode performance is measured on one H100, one 32K context, Hugging Face, and no custom kernel.
- The paper does not provide an end-to-end online-serving evaluation with realistic continuous batching, prefix reuse, preemption, or latency SLOs.
- CPU, NUMA, CXL, and disaggregated-transfer behavior are not evaluated.
- Accuracy retention is an aggregate benchmark measure; rare long-horizon failures may matter more than the mean for agents.
- The vLLM KV-pool increase is reported without a fully explained accounting discrepancy.
- The operational complexity of latent pages plus per-sequence correction metadata remains unproven.

## Reading priority

1. [BellowsKV paper](https://arxiv.org/abs/2610.35023) — read Sections 2, 3.3, 4.2, and 4.4 first.
2. [BellowsKV implementation](https://github.com/MorrisZJ/BellowsKV) — inspect the cache representation and decode reconstruction path.
3. [Local Inference of LLMs on Long Contexts: A Data Systems Perspective](https://arxiv.org/abs/2610.28223) — useful complementary characterization showing that long-context attention can underutilize both compute and bandwidth, especially on newer hardware. genui{"citation":{"ref":"turn77view1"}}

## Bottom line

BellowsKV is a genuinely interesting capacity result because it reaches high compression without throwing away latent dimensions. Its most transferable lesson is “hold rank, reduce bits, then evict”—but the systems payoff depends on avoiding full K/V materialization at read time.

For CPU inference, the next step is not merely to port the representation. It is to design a socket-local, tiled or fused reconstruction-attention path and measure whether the compressed state converts into higher SLO-compliant concurrency, lower remote-memory traffic, or reduced preemption. Until that exists, BellowsKV should be viewed as a strong memory-capacity technique with near-zero measured decode penalty—not as a demonstrated decode-throughput accelerator.
