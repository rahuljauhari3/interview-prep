# AI inference systems update — October 21, 2026

## Humming separates quantization semantics from kernel execution

**Importance: High for quantized-inference architecture; directly applicable to GPU serving and strategically relevant to CPU backends**

Ant Group and the vLLM community released Humming on October 21, an Apache-2.0 quantized GEMM library already integrated into vLLM. Its most important contribution is not another fast INT4 kernel. It introduces a common representation for quantization semantics and then specializes execution for the model shape, batch regime and target architecture.

That separation addresses a growing systems problem: every combination of weight format, activation type, scaling granularity, zero-point policy, dense/MoE layout and hardware generation otherwise tends to acquire its own conversion path, dispatch rules and hand-maintained kernel family. genui{"citation":{"refs":["turn64view0","turn65view0"]}}

**Read:** [Humming engineering writeup](https://vllm.ai/blog/2026-10-21-humming) and [open-source repository](https://github.com/vllm-project/humming)

### What changed

Humming divides the execution path into three layers:

1. **Quantization description.** Export formats are converted into a standard representation of weights, scales, zero points, secondary scales and activation quantization.
2. **Workload description.** Layer shape, GEMM type, batch-invariance assumptions, scale layout and target architecture describe what must execute.
3. **Execution specialization.** Tile sizes, pipeline stages, warp specialization, memory-access strategy, MMA instructions, scheduling and concurrency are selected, then compiled with NVRTC or NVCC and cached.

The same component model covers dense, indexed-MoE and grouped-MoE GEMMs. It supports weights from 1–8 bits with multiple integer and floating-point encodings, and activation paths including FP16, BF16, FP8, FP4, INT8 and INT4 where supported by the hardware. Current device coverage starts at NVIDIA Turing, with newer low-precision paths enabled according to each architecture’s capabilities. genui{"citation":{"refs":["turn64view0","turn65view0"]}}

The runtime can also select different kernels for different input-size ranges. The authors report that small batches often favor a multistage cp.async plus MMA pipeline, while larger batches can favor TMA/UMMA. That is an important design choice: decode and prefill do not merely use different shapes; they may need different data-movement mechanisms.

### Reported performance

The microbenchmarks cover powers-of-two \(M\) values from 1 to 8,192, use warmed kernels and cold L2 measurements, and compare against vLLM Marlin, FlashInfer and DeepGEMM paths.

Selected reported results include:

- Dense W4A16 reaches roughly 3–4× Marlin performance at several small-batch points and about 2× on large-batch H200 cases.
- Grouped MoE W4A16 reaches approximately 5× Marlin in some large-batch B300 cases.
- W8A8 reaches about 2.5–2.6× DeepGEMM Contiguous at selected small and medium H20/H200 shapes.
- Humming’s Hopper W4A8 path dequantizes MXFP4 weights while loading them and computes through FP8 Tensor Cores; it reaches about 4.2×–4.5× the cited W8A8 baseline at selected small/medium H20 and H200 shapes.
- On B300, the reported W8A8, W4A8 and W4A4 gains over the corresponding DeepGEMM baselines are approximately 1.25×, 1.6× and 1.4× at selected small/medium sizes. genui{"citation":{"ref":"turn64view0"}}

More importantly, the authors provide an end-to-end vLLM comparison on DeepSeek-V4.1-Flash using eight H20 GPUs with TP8. Replacing Marlin W4A16 MoE execution with Humming W4A8 raised input and output throughput by 14.1% at concurrency 1 and 43.3% at concurrency 64. At concurrency 64, output throughput increased from 442.37 to 633.94 tokens/s, mean TTFT fell from 5,467 to 3,883 ms and mean TPOT fell from 123.25 to 85.70 ms. genui{"citation":{"ref":"turn64view0"}}

| Concurrency | Output throughput gain | TTFT reduction | TPOT reduction |
|---:|---:|---:|---:|
| 1 | 14.1% | 29.3% | 8.5% |
| 8 | 33.4% | 32.3% | 22.2% |
| 32 | 40.5% | 30.5% | 28.5% |
| 64 | 43.3% | 29.0% | 30.5% |

The scaling trend matters: reduced weight traffic and activation precision become more valuable as concurrency raises pressure on memory movement and expert execution.

### Why this matters for CPU inference

Humming is currently a CUDA library, so its kernels cannot be dropped into ZenDNN. The transferable architecture is nevertheless strong.

A CPU quantization stack usually entangles at least four concerns:

- the model’s exported quantization format;
- the packed layout expected by a microkernel;
- the execution precision and ISA;
- the shape regime used by prefill, decode or MoE experts.

Separating them would let a CPU backend normalize many external formats once, then choose an internal packing and execution plan for AVX2, AVX-512, VNNI, BF16 or future narrow-precision instructions. The model format should describe numerical meaning; it should not permanently dictate the kernel’s physical layout.

For EPYC, an analogous dispatch key could be:

\[
K =
(
M,N,K,
Q_w,Q_a,
G,
\text{ISA},
\text{CCD topology},
B,
\text{phase}
)
\]

where \(Q_w\) and \(Q_a\) describe weight and activation quantization, \(G\) is grouping/scaling geometry, \(B\) is effective batch, and phase distinguishes prefill, decode and expert execution.

The key addition beyond Humming is topology. A configuration that wins on one CCD may lose across a socket if it increases packed-weight replication, LLC pressure or cross-NUMA reductions.

### A practical ZenDNN prototype

A useful first step would be a thin quantization intermediate representation independent of any one microkernel:

\[
Q =
(
\text{value format},
\text{scale format},
\text{scale granularity},
\text{zero-point policy},
\text{group size},
\text{logical layout}
)
\]

Then:

1. Convert model-export formats into \(Q\).
2. Benchmark candidate packing and kernel configurations by shape bucket.
3. Cache the selected plan per ISA, NUMA placement and workload phase.
4. Keep packed weights socket-local when capacity permits.
5. Fall back to a conservative precompiled path while tuning or JIT compilation runs.
6. Validate every generated plan against a reference dequantize-plus-GEMM implementation.

The benchmark objective should include preparation and residency costs:

\[
T_{\mathrm{effective}} =
T_{\mathrm{pack}}/R +
T_{\mathrm{dispatch}} +
T_{\mathrm{kernel}} +
T_{\mathrm{remote\ memory}} +
T_{\mathrm{reduction}}
\]

where \(R\) is the expected number of reuses of the packed weights. A highly specialized kernel is a poor choice if packing dominates a short-lived model replica or if its layout multiplies per-socket memory consumption enough to evict KV state.

### JIT economics and deployment implications

Humming reports compiling one W4A16 kernel on H200/CUDA 13.4 in approximately 1.82 seconds with NVCC or 1.88 seconds with NVRTC, versus 23.35 seconds for the compared CUTLASS NVCC build, with caches disabled. Its Python wheel is roughly 500 KB because kernels are generated on demand rather than shipped as a large precompiled collection. genui{"citation":{"ref":"turn64view0"}}

That model improves flexibility but introduces operational requirements:

- compiled artifacts must be keyed by library, compiler, driver, device and quantization configuration;
- cold-start SLOs need a safe fallback;
- production images should prewarm common shapes;
- cache invalidation must be deterministic;
- generated kernels need numerical, bounds and concurrency testing;
- multi-tenant systems should not permit unbounded compilation requests.

For CPU deployments, ahead-of-time generation during model staging may be preferable to runtime JIT. The same specialization system can support both: tune during qualification, persist the winning binaries and load them without placing a compiler on production hosts.

### Caveats

The release is substantive, but the performance claims need careful interpretation:

- Humming is currently CUDA-focused; there is no x86 or ROCm backend.
- Most published results are microbenchmarks selected across many format and architecture combinations.
- Baselines vary by chart, so headline maxima should not be combined into one universal speedup.
- The end-to-end test changes both weight/activation execution—from W4A16 to W4A8—and the kernel library. It does not isolate kernel architecture from the precision change.
- The serving evaluation uses one model, eight H20 GPUs and TP8; quality validation for tokenwise FP8 activation quantization is not detailed in the post.
- Heuristic configuration selection is shipped today, while broader autotuning remains future work.
- JIT compilation expands the qualification surface even when compilation itself is fast.

### Assessment

Humming is a genuinely important kernel-infrastructure development, not simply a promotional model benchmark. Its strongest idea is the abstraction boundary: represent quantization semantics once, then independently specialize data layout, movement, scheduling and compute for the actual workload.

For CPU inference, the immediate opportunity is to adopt the same separation while making ISA and NUMA topology first-class tuning dimensions. That would reduce format-specific integration work and allow decode, prefill and MoE shapes to select different optimized paths without fragmenting the entire backend into unrelated kernel stacks.
