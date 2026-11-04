# AI inference systems briefing — November 4, 2026

## Executive take

The most CPU-relevant development since yesterday’s cluster-efficiency briefing is a Euro-Par paper on **layer-wise CPU–GPU scheduling for models that exceed GPU memory**.

[Layer-wise CPU-GPU Scheduling for LLM Inference on Memory-Limited Consumer GPUs](https://arxiv.org/abs/2611.01393), posted November 1, makes a useful conceptual shift: CPU execution is not merely a fallback for layers that do not fit in VRAM. A deliberately chosen CPU layer can provide a compute window during which the next layer’s weights move asynchronously over PCIe.

On an RTX 5090 desktop with an eight-thread Intel Core Ultra 7 host, the authors report that this schedule hides 99.6–99.8% of weight-transfer time for OPT-30B and OPT-66B. For OPT-30B, it reduces end-to-end latency by 3.5× versus Hugging Face Accelerate and energy by 3.1×.

The negative result is equally important: on Llama-2-70B, host-memory pressure destabilizes the profiled timings, almost all transfers become exposed, and the proposed plan is 22% slower than a static CPU/GPU split for the short-prompt case. That failure makes the paper useful for systems engineers: an offline overlap schedule is only as good as the stability of memory allocation, transfer, and CPU execution time.

**Assessment:** a substantive heterogeneous-execution result with a clear and portable scheduling abstraction. The reported speedups are narrow—batch-one FP16 decoding on one desktop—and should not be projected to high-throughput serving or modern quantized models without new measurements.

## 1. What changed: CPU computation becomes a transfer-hiding resource

Conventional weight offload keeps most weights in host DRAM, transfers one layer to the GPU, executes it, and reuses the device buffer for the next layer. When every layer is streamed, PCIe dominates decode latency because batch-one GPU computation is too short to cover the weight copy.

The paper classifies each Transformer layer into three modes:

| Mode | Weight location and execution | Role |
|---|---|---|
| GPU-resident | Weights remain in VRAM; compute runs on GPU | Fast path for the layers that fit |
| CPU | Weights remain in host memory; compute runs on CPU | Creates a useful interval for overlapping another transfer |
| Stream | Weights move from host to a reusable GPU buffer; compute runs on GPU | Uses GPU speed without permanent VRAM residency |

The important difference from a static split is temporal. A CPU layer is valuable not only when its own CPU latency is acceptable, but when that latency covers a stream layer’s transfer:

[
T_{mathrm{pair}}
approx
maxleft(
T_{mathrm{CPU layer}},
T_{mathrm{H2D transfer}}
ight)
+
T_{mathrm{GPU layer}}
]

rather than:

[
T_{mathrm{serial}}
=
T_{mathrm{CPU layer}}
+
T_{mathrm{H2D transfer}}
+
T_{mathrm{GPU layer}}
]

The runtime therefore chooses a sequence of CPU, resident, and stream blocks from offline measurements of per-layer CPU compute, GPU compute, and transfer time. It relaxes detailed temporal constraints into an approximate optimization problem that determines the required ratio of CPU work to streamed work, then synthesizes a reusable execution plan.

This is a useful extension of the principle from yesterday’s briefing. Cluster efficiency asks whether communication cost destroys the benefit of pooling economical devices. This paper asks the same question within one host: can CPU compute and PCIe movement be scheduled so that the heterogeneous resources behave like one pipeline rather than two serial devices?

## 2. Why the result matters in practice

The authors measure weight transfer at roughly 98% of decode time for conventional OPT-30B offload. In that regime, accelerating the GPU kernel has almost no end-to-end value:

[
	ext{speedup}_{mathrm{system}}
le
rac{1}
{f_{mathrm{transfer}} + rac{1-f_{mathrm{transfer}}}{S_{mathrm{kernel}}}}
]

With (f_{mathrm{transfer}}=0.98), even an infinitely fast GPU kernel improves total latency by only about 2%.

The useful optimization target is therefore **exposed transfer time**:

[
T_{mathrm{decode}}
=
T_{mathrm{CPU}}
+
T_{mathrm{GPU}}
+
T_{mathrm{transfer,exposed}}
+
T_{mathrm{runtime}}
]

Total bytes moved may remain unchanged. The system wins when CPU computation covers the transfer on the critical path.

This distinction matters for EPYC systems. A CPU backend often appears slower than a GPU in isolation, yet can improve heterogeneous end-to-end latency if it executes work that would otherwise leave the GPU waiting on PCIe. The correct comparison is not CPU-layer time versus GPU-layer time; it is:

[
	ext{CPU layer cost}
quad	ext{versus}quad
	ext{transfer stall eliminated}
]

A CPU layer is beneficial when the stall it hides exceeds the extra latency caused by moving that layer off the GPU.

## 3. Reported evaluation

The evaluation platform contains:

- NVIDIA RTX 5090 with 32 GiB VRAM;
- Intel Core Ultra 7 265F at 2.40 GHz;
- 128 GiB host DRAM;
- PCIe Gen5 ×16;
- eight CPU computation threads;
- CUDA 12.4;
- FP16 weights and computation.

Models are OPT-30B, OPT-66B, and Llama-2-70B. Prompts contain 256 or 1,024 tokens, generation is fixed at 32 tokens, and effective batch size is one.

For OPT-30B with a 256-token prompt, the paper reports:

- 3.5× lower end-to-end latency than Hugging Face Accelerate;
- 6.3× lower latency than DeepSpeed ZeRO-Inference;
- 2.8× lower latency than CPU-only execution;
- 1.5× lower latency than FlexInfer’s static SplitGen policy.

For OPT-66B, it reports 3.5× lower latency than Accelerate, 6.1× lower latency than CPU-only, and 1.4× lower latency than SplitGen. DeepSpeed is omitted for OPT-66B and Llama-2-70B because it runs out of memory.

The plan hides 99.6% of transfers for OPT-30B and 99.8% for OPT-66B. Plan construction takes about 3.2 seconds and is reusable.

For OPT-30B with a 1,024-token prompt and 32 output tokens, the method consumes 3.1× less total energy than Accelerate, despite drawing more average power. The improvement comes from finishing sooner, a reminder that energy per request is power multiplied by time—not minimum instantaneous wattage.

## 4. The Llama-2-70B failure is the most valuable caveat

For Llama-2-70B with a 256-token prompt, the proposed method is 22% slower than SplitGen. Almost all planned transfers remain exposed.

The authors attribute this to host-memory pressure from the larger model and pinned buffers. CPU computation and transfer times become unstable, so a plan based on average offline measurements no longer matches runtime behavior.

This exposes the core limitation of static overlap scheduling:

[
T_i^{mathrm{actual}}
=
T_i^{mathrm{profiled}}
+
epsilon_i(
	ext{NUMA},
	ext{paging},
	ext{contention},
	ext{frequency},
	ext{allocator}
)
]

When the accumulated error exceeds overlap slack, the pipeline collapses into serialized execution.

On a multisocket EPYC host, the error sources would be larger:

- a streamed layer’s weights may be allocated on the remote socket;
- pinned buffers may consume local memory and change allocator behavior;
- CPU frequency changes with AVX-512 or sustained memory traffic;
- DMA and CPU kernels contend for the same memory channels;
- GPU and NIC traffic may share PCIe roots;
- background requests may perturb the calibrated CPU-layer time;
- simultaneous KV-cache traffic may consume the bandwidth reserved for weights.

A production scheduler therefore needs runtime feedback, not only an offline plan.

## 5. How to adapt the idea to EPYC and ZenDNN

The paper uses only eight host threads, so it does not explore the topology and bandwidth controls available on a large EPYC server. A CPU-oriented implementation should extend the plan in five ways.

### 5.1 Make placement NUMA-aware

For each layer (l), profile:

[
P_l =
(
T_l^{mathrm{CPU,socket0}},
T_l^{mathrm{CPU,socket1}},
T_l^{mathrm{GPU}},
T_l^{mathrm{H2D,local}},
T_l^{mathrm{H2D,remote}},
M_l
)
]

The scheduler should bind CPU workers, final packed weights, and the GPU’s source buffer to the NUMA node nearest the PCIe root. A remote first-touch can erase the overlap benefit before the GPU transfer begins.

### 5.2 Stream final packed or quantized representations

The paper evaluates FP16, making transfers unusually expensive. Modern deployments will use INT8, INT4, FP8, or mixed precision. The CPU loader should quantize and pack weights into the exact GPU-consumable format before streaming, avoiding:

1. host read of a canonical tensor;
2. temporary quantized tensor;
3. second packing pass;
4. PCIe transfer.

For CPU-executed layers, keep a separate ZenDNN/oneDNN packed representation only when its reuse justifies the extra DRAM.

### 5.3 Re-plan from observed slack

Track:

[
S_l =
T_{mathrm{cover},l}
-
T_{mathrm{transfer},l}
]

A positive (S_l) means the transfer is hidden; a negative value is exposed stall. Use exponentially weighted runtime measurements to replace CPU layers, change stream grouping, or reduce pinned-buffer pressure when slack deteriorates.

### 5.4 Separate prefill and decode plans

The paper already uses on-demand GPU loading for prefill, but a production system should explicitly maintain different plans:

- prefill favors GPU-resident compute and larger transfers amortized over many tokens;
- batch-one decode may benefit from CPU transfer-hiding windows;
- batched decode changes CPU GEMM efficiency and the amount of GPU work available to cover transfers.

The plan should be keyed by prompt bucket, decode batch, model format, and current memory pressure.

### 5.5 Integrate KV traffic into the bandwidth budget

Weight streaming and long-context attention both consume host memory and PCIe bandwidth. The scheduler should enforce:

[
B_{mathrm{weights}}
+
B_{mathrm{KV}}
+
B_{mathrm{CPU kernels}}
le
B_{mathrm{local sustainable}}
]

not assume each subsystem independently receives peak bandwidth.

## 6. A practical validation experiment

Use a model that exceeds GPU memory and compare:

1. sequential weight streaming;
2. maximum GPU residency plus sequential streaming;
3. static CPU/GPU layer split;
4. offline CPU-window overlap scheduling;
5. feedback-controlled, NUMA-aware overlap scheduling.

Sweep:

- batch sizes 1, 2, 4, 8, and 16;
- prompt lengths from 256 to at least 32K;
- FP16, INT8, and 4-bit weight formats;
- one and two CPU sockets;
- local versus remote pinned-memory placement;
- KV cache resident on GPU versus host offload.

Record:

- TPOT p50/p95/p99;
- total and exposed H2D time;
- transfer/CPU overlap percentage;
- local and remote DRAM bytes;
- PCIe payload bandwidth and copy-engine utilization;
- CPU memory-bandwidth saturation;
- pinned-memory footprint and allocation failures;
- CPU-layer cycles, packed-weight reuse, and thread scaling;
- plan prediction error per layer;
- energy per generated token.

The success metric should be the largest region of workload space where overlap remains stable—not the best single speedup.

## 7. What is genuinely important versus narrow

### Genuinely important

- The work treats CPU execution as an instrument for hiding communication, rather than a residual placement for layers that do not fit.
- It optimizes whole-layer residency, CPU execution, and streaming together.
- It evaluates latency and wall-meter energy, not only kernel throughput.
- It includes a clear failure case where memory pressure invalidates the schedule.
- The technique is exact: it changes execution placement but not model arithmetic or output quality.
- The paper is associated with Euro-Par 2026 rather than being only an unreviewed performance claim.

### Narrow or not yet established

- The evaluation is batch one with only 32 generated tokens; it does not establish throughput or tail latency under concurrent serving.
- Models are older dense FP16 checkpoints. Quantization changes both transfer time and CPU/GPU compute balance, potentially shrinking the available overlap window.
- The host is an eight-thread consumer CPU configuration, not a many-core or multisocket server.
- Only one NVIDIA GPU and one PCIe topology are tested.
- The plan uses average offline timing and does not adapt when runtime conditions change.
- Prompt lengths stop at 1,024 tokens, so weight streaming is not evaluated alongside truly long-context KV traffic.
- No public implementation is linked from the arXiv page.
- The large headline gains are against general offload frameworks; the improvement over the stronger static SplitGen baseline is 1.4–1.5× for the successful cases.

## Secondary signal: Jet-QSparse combines 4-bit storage with sparse high-precision aggregation

[Jet-QSparse](https://arxiv.org/abs/2611.01022), posted October 31 and updated November 3, retains the full KV history in NVFP4, performs a dense FP4 query–key scan, selects 128-token blocks, and computes the output-forming value aggregation in FP16 with FP32 accumulation.

This split is technically sensible: use low precision for storage and selection, but preserve higher precision where attention probabilities combine values into the final output. Contiguous 128-token blocks align sparsity with efficient tile loads rather than arbitrary gathers.

The authors report 1.94× end-to-end decode speedup for Qwen3.8-27B at 128K context and batch 16 on one GB300 with a 64-block budget, plus 2.58–2.76× the BF16 KV-token capacity for the evaluated Qwen models. A 128-block budget shows no detected loss across the tested tasks and four hybrid models.

For CPU inference, the representation idea is interesting, but the demonstrated path depends on native Blackwell NVFP4 tensor-core scoring. A CPU port would require an efficient packed 4-bit dot-product or dequantized scoring path, fast block-mass reduction, and sparse value aggregation that preserves contiguous access. Without that kernel co-design, selection overhead can exceed the bytes saved.

Treat Jet-QSparse as a substantive GPU result and a useful algorithmic pattern—not evidence of an immediate CPU speedup.

## Reading priority

1. **[Layer-wise CPU-GPU Scheduling for LLM Inference on Memory-Limited Consumer GPUs](https://arxiv.org/abs/2611.01393)** — Read the execution-mode formulation and approximation model, then the Llama-2-70B failure analysis in §5.2.
2. **[Jet-QSparse](https://arxiv.org/abs/2611.01022)** — Prioritize the cache representation, block-selection pipeline, accuracy-budget sweeps, and kernel evaluation.
3. **[FlexInfer](https://proceedings.mlsys.org/paper_files/paper/2025/hash/cc546335f00e1b15aa0569868010d53a-Abstract-Conference.html)** — Useful baseline for choosing among CPU-only, streamed GPU, and static split execution.
4. **[FlexGen](https://proceedings.mlr.press/v202/sheng23a.html)** — Background on throughput-oriented offload scheduling and weight reuse across batches.

## Bottom line

Yesterday’s briefing established that fabric latency determines whether pooled accelerators are economical. Today’s result applies the same principle within one machine: PCIe movement should not be treated as an unavoidable serial phase when CPU computation can cover it.

For CPU inference engineering, the highest-value idea is **latency-shaped heterogeneous scheduling**. Place work according to what it hides on the critical path, not merely which processor executes it fastest in isolation.

The production version should be NUMA-aware, quantization-aware, and feedback-controlled. If profiling remains static while host memory pressure, KV traffic, and request concurrency change, the carefully planned overlap can disappear—as the paper’s own Llama-2-70B result demonstrates.
