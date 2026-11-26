# AI inference systems briefing — November 26, 2026

## Executive view

The strongest new work since the November 25 briefing is **Shifter Quantization**, which attacks a recurring systems failure mode in low-bit inference: a compact checkpoint can still execute inefficiently when group metadata, unpacking, corrections, or datatype conversions prevent the hardware from using its native integer matrix path across a full dot product.

Two other developments matter, but at different levels:

- **Draft calibration** is a low-cost serving optimization: tuning only the draft distribution’s temperature improved speculative acceptance length by up to about 10% across nine target/draft pairs.
- **Top-K attention has a structural limitation**: even an exact per-token importance oracle can choose a poor fixed-size support because attention positions interact after renormalization. This is more important as a design warning than as an immediately deployable speedup.
- **llama.cpp 1.1.0** adds useful production machinery—budget-targeted per-tensor quantization and a server slot-scheduling refactor—but publishes no end-to-end CPU performance evidence for those changes. Treat it as an engineering release, not a demonstrated inference advance.

| Development | Assessment | CPU relevance | Deployment readiness |
|---|---|---:|---:|
| Shifter Quantization | Substantive representation/execution co-design | High conceptually; CPU results absent | Prototype |
| Draft calibration | Useful, low-risk incremental optimization | Medium-high for speculative CPU serving | Easy to test |
| Joint-support limits of Top-K attention | Substantive negative result | Medium for long-context CPU inference | Research-stage |
| llama.cpp 1.1.0 | Important implementation update, not a proven speedup | High ecosystem relevance | Available now |

## 1. Shifter Quantization: optimize the representation for the integer datapath

**Importance: High**

[Shifter Quantization: Compact Representations for Native Integer Computation](https://arxiv.org/abs/2611.33080), submitted November 24, introduces two related formats:

- **SQ** reconstructs native-width integer operands by appending and shifting bits shared within a group.
- **SHI** decomposes the representation into products that can execute through native INT4 matrix operations, with grouped correction terms.

The important change is not simply another accuracy-aware quantizer. The format is designed backward from the contraction primitive. Conventional groupwise quantization reduces storage and error, but group-local scales, offsets, and bit fields can force a kernel to perform multiple partial contractions or expensive dequantization. Shifter instead tries to recover operands that the hardware can consume in its regular integer MMA path.

### Reported results

On Qwen3.8-27B, using a single RTX 5090:

| Format and baseline | Workload | Reported result |
|---|---|---:|
| SQ2 vs llama.cpp Q2_K | 4,096-token prefill | 2.71× throughput |
| SHI2 vs llama.cpp Q2_K | 4,096-token prefill | 3.77× throughput |
| SQ4 vs NInfer groupwise INT, 27B | 7.7K / 260K prefill | 2.47× / 1.43× |
| SQ4 vs NInfer groupwise INT, 35B-A3B | 7.7K / 260K prefill | 1.81× / 1.67× |
| SQ8 vs vLLM online FP8 | Batch-one 4,096-token prefill | 27.4% higher throughput |

The two-bit released-checkpoint comparison also reports lower WikiText-2 KL divergence than Q2_K. At four bits, the advantage is strongly phase-dependent: SQ4 materially improves prefill, but the reported structured-decode results range from essentially tied to 1.13×. SHI4 is actually slower than the NVFP4 baseline in the serving comparison. That distinction matters: the paper demonstrates that the representation can remove groupwise-integer overhead, not that every Shifter variant dominates every low-bit format.

### Why it matters for EPYC and ZenDNN

The CPU lesson is architectural even though the evaluation is GPU-only. A useful low-bit CPU format must jointly minimize:

1. resident weight bytes;
2. bytes read per token;
3. unpack and correction instructions;
4. activation conversion and quantization cost;
5. the number of independent dot products required per logical GEMM;
6. layout transforms between operators.

Shifter’s core question transfers directly to AVX-512/VNNI and future low-bit CPU paths: **can the stored representation be expanded cheaply into operands that keep the native dot-product unit busy, or does compression merely move cost into the inner loop?**

For a ZenDNN experiment, compare SQ/SHI-like packing with Q2_K/Q4_K and a conventional groupwise INT4 path across both decode-like GEMV and prefill-like GEMM. Record more than tokens/s:

- retired integer vector instructions and IPC;
- L1/L2/LLC misses and DRAM bandwidth;
- bytes of weight, scale, correction, and prepared planes actually read;
- packing or activation-quantization time;
- socket-local versus remote memory traffic;
- energy per token;
- quality against the same BF16 logits.

The decisive CPU metric is likely **useful integer MACs per byte fetched**, including correction planes and metadata—not nominal bits per weight.

### Important caveats

- There is no CPU implementation or CPU benchmark.
- Most headline measurements use one RTX 5090 and custom kernels.
- Some comparisons differ in activation type and execution path; compact labels such as “2-bit” do not imply identical arithmetic.
- SHI can retain derived planes. The paper notes that a prepared three-plane G8 representation occupies 5.0 logical bits per weight even though the serialized base plus correction occupies 4.5.
- Several serving measurements use tuned workload-specific profiles, and the decode advantage is much smaller than the prefill advantage.
- The evaluation is extensive for a preprint, but independent reproduction and maintained code are still missing.

**Verdict:** genuinely important representation/execution co-design, especially as a template for CPU kernel work. Do not transfer the GPU speedups to EPYC without re-deriving the packing, correction, and cache costs for VNNI-class instructions.

## 2. Draft calibration: acceptance rate is partly an inference-time tuning problem

**Importance: Medium-high; immediately testable**

[Draft Calibration for Speculative Decoding](https://arxiv.org/abs/2611.33111), submitted November 24 and updated November 25, observes that released draft models are often systematically underconfident relative to their targets. Because lossless speculative decoding accepts tokens according to the overlap between the draft distribution and the target distribution, matching the top token is not enough; the full probability mismatch affects acceptance.

The paper applies post-hoc calibration while leaving model weights fixed. Standard draft-temperature scaling improves accepted length by up to roughly 10% with almost no added compute. The evaluation spans nine target/draft combinations using EAGLE-3, DFlash, and DSpark, with target models including Gemma-4-31B, Qwen3 variants, and GPT-OSS-120B.

### Practical implication

This is attractive for CPU inference because verification is expensive and draft sampling is relatively cheap. A few extra accepted tokens can eliminate full target-model passes, which may be worth more than a modest kernel improvement in a memory-bandwidth-bound decoder.

Add draft temperature to the serving control plane rather than freezing it as a model constant. Calibrate by workload class and generation policy, then measure:

- accepted tokens per target pass;
- target passes per generated token;
- draft cost and verification cost separately;
- p50 and p99 inter-token latency;
- throughput under continuous batching;
- stability as prompt domain, sampling temperature, and concurrency change.

A simple controller could choose among a small validated temperature grid using rolling acceptance statistics. Avoid unconstrained online adaptation until the rejection-sampling implementation is verified to use the calibrated distribution consistently in both proposal and acceptance calculations.

### Caveats

The best temperature in the reported online plots is selected retrospectively from a grid, so the result does not yet provide a production-ready adaptation policy. A 10% acceptance-length gain also does not imply 10% end-to-end throughput: the result depends on target/draft cost ratio, verification batch geometry, memory pressure, and scheduler behavior.

**Verdict:** substantive but incremental. It is one of the rare recent ideas that can be tested in an existing speculative server with little kernel work and low operational risk.

## 3. Exact marginal importance is insufficient for sparse attention

**Importance: High as a negative result; medium near-term deployment value**

[When Exact Token Importance Is Not Enough: Limits of Top-K Selection in LLM Attention](https://arxiv.org/abs/2611.32324), submitted November 24, proves that independently ranking tokens can produce an arbitrarily poor fixed-budget attention support—even when each token’s score is the exact downstream change caused by removing that token alone.

The cause is support interaction. Once a subset is chosen, softmax renormalization changes every retained token’s contribution. The best token considered alone may be a poor partner for the rest of the selected support.

The paper backs the construction with exhaustive audits over 24-position candidate sets. Exact-singleton Top-4 left at least 10% relative improvement in 55/60 SmolLM2 cases, 49/60 Gemma cases, and 60/60 Qwen cases. A one-swap audit, SwapCert, detected 157 of 164 supports with that amount of enumerated headroom.

### Why it matters in systems practice

This challenges a common sparse-attention engineering assumption: improving the marginal scorer will monotonically improve the chosen cache or attention support. It may not. For long-context CPU inference, where sparse attention is appealing because KV reads dominate, a more accurate independent score can still produce a worse joint set.

A practical design should separate:

- **candidate generation**, which can remain cheap and marginal;
- **joint-support refinement**, such as one or a few swaps;
- **cost-aware selection**, where remote-NUMA, CXL, or compressed-page retrieval cost is part of the objective;
- **runtime auditing**, which samples requests to estimate whether the current support has cheap improving exchanges.

For EPYC, the natural extension is not pure KL minimization. Optimize a combined objective over predictive loss and retrieval cost so that a slightly less faithful local-DRAM support can beat a higher-fidelity support dominated by remote-socket reads.

### Caveats

The paper deliberately isolates support choice over a resident full KV cache. It does not demonstrate physical KV eviction, memory savings, or end-to-end serving speedup. Exhaustive evaluation over small candidate universes is an audit, not a deployable selector. Swap-based refinement must earn back its extra scoring and data-movement cost.

**Verdict:** a rigorous and important boundary result. It should change how sparse-attention papers are evaluated: Top-K quality needs a joint-support baseline or at least a same-budget swap audit, not only a better marginal-importance metric.

## 4. llama.cpp 1.1.0: useful deployment plumbing, limited performance evidence

**Importance: Medium**

[llama.cpp 1.1.0](https://github.com/ggml-org/llama.cpp/releases/tag/v1.1.0), released November 26, adds two changes worth tracking:

- automatic per-tensor quantization targeting a requested bits-per-weight or total-size budget, with resumable state;
- a refactor of server slot scheduling using per-prompt running hashes, alongside fixes to cached-prompt restoration and KV-cell removal.

Budget-targeted quantization is operationally useful because heterogeneous per-tensor formats can now be selected against a deployment constraint rather than a single global type. The scheduler changes may improve maintainability and cache behavior under multi-request serving.

However, the release provides no controlled CPU throughput, latency, quality, or memory results for these features. The release also bundles many unrelated model, audio, UI, and API changes, including deprecation of the old batch API in favor of `llama_batch_ext` and `llama_process`.

**Verdict:** deployable engineering progress, but not evidence of a performance advance. Benchmark the quantization selector’s output and server behavior before attributing gains to the release.

## Recommended reading order

1. [Shifter Quantization](https://arxiv.org/abs/2611.33080) — read Sections 3, 5, and 6 plus the resident-memory accounting. This is the highest-value paper for kernel and format design.
2. [Draft Calibration for Speculative Decoding](https://arxiv.org/abs/2611.33111) — read the acceptance derivation, temperature sweep, and serving configuration. This is the easiest idea to operationalize.
3. [Limits of Top-K Selection in LLM Attention](https://arxiv.org/abs/2611.32324) — focus on the counterexample, support-conditioned exchange value, and SwapCert results.
4. [llama.cpp 1.1.0 release notes](https://github.com/ggml-org/llama.cpp/releases/tag/v1.1.0) — scan the quantization API and server changes; treat the rest as compatibility work.

## Bottom line

The most important new theme is that **format efficiency and selection quality must be judged after composition**. A low-bit tensor is not efficient if its representation fragments the native dot product; an individually accurate token score is not sufficient if the selected tokens interact badly; and a strong draft model still leaves speed on the table if its probability scale is mismatched to the target.

For CPU inference work, the highest-value next experiment is a native-integer, groupwise quantization prototype that accounts for every metadata and correction byte, benchmarked separately in prefill GEMM and decode GEMV. In parallel, draft-temperature calibration is a low-effort serving experiment likely to produce a quicker production win.