# AI inference systems briefing — October 23, 2026

## Apex makes CPU–GPU MoE placement a per-layer critical-path decision

**Importance: High for heterogeneous inference and CPU expert execution**

A new paper released today, [Efficient CPU-GPU Collaborative MoE Inference via Adaptive Expert Assignment](https://arxiv.org/abs/2610.23006), introduces Apex: a runtime that decides, for every layer and serving batch, whether each GPU-missed expert should execute from host-resident weights on the CPU or be transferred to the GPU.

The central observation is simple but consequential. All experts in a layer have approximately the same weight-transfer cost, while their token loads can differ sharply. Sending a one-token expert over PCIe is usually wasteful; transferring a heavily loaded expert may save enough CPU computation to justify the movement. Static policies that send every miss to one device ignore this asymmetry. genui{"citation":{"ref":"turn71view4"}}

Apex models the layer’s completion time as the slower of two concurrent paths:

[
T_i(p)=max!left(
T_{mathrm{CPU}}!left(mathcal{C}_i(p)ight),
T_{mathrm{transfer}}!left(mathcal{G}_i(p)ight)
+
T_{mathrm{GPU}}!left(mathcal{R}_icupmathcal{G}_i(p)ight)
ight)
]

Here, (mathcal{R}_i) is the resident GPU expert set, while the missed experts are divided between CPU execution (mathcal{C}_i) and transfer-plus-GPU execution (mathcal{G}_i). The runtime sorts missed experts by observed token load, estimates both paths from online rates, and chooses the transfer count that minimizes their maximum rather than optimizing CPU work or PCIe traffic independently. genui{"citation":{"refs":["turn71view3","turn71view4"]}}

### What changed relative to earlier expert offloading

| Policy | GPU-missed expert treatment | Main failure mode |
|---|---|---|
| Pure GPU offload | Transfer every missed expert | PCIe weight movement dominates |
| Static CPU collaboration | Execute every missed expert on CPU | CPU becomes the per-layer tail |
| Post-fetch caching | Execute the current miss on CPU, then transfer it for future reuse | Placement follows the previous iteration, not current demand |
| Apex | Transfer high-load misses; execute low-load misses on CPU | Requires accurate online cost estimates and coordinated execution |

Apex combines two mechanisms:

1. **Speculative assignment** predicts high-load experts one layer ahead and transfers them while the current layer is executing.
2. **Exact assignment** uses the actual router output, current GPU residency, and measured CPU/GPU/transfer rates to balance the remaining work.

Exact assignment alone reacts correctly but exposes transfer latency after routing. Speculation creates a longer overlap window but can be wrong. Their combination is the important contribution: prediction supplies overlap, while exact routing corrects the placement decision before the layer completes. genui{"citation":{"ref":"turn71view3"}}

## Results worth paying attention to

The authors implement Apex on KTransformers and evaluate BF16 versions of:

- Qwen3.5-35B-A3B: 256 total experts, eight active;
- Qwen3-Next-80B-A3B: 512 total experts, ten active;
- GLM-4.5-Air: 128 total experts, eight active.

The test server uses one 48 GB NVIDIA RTX A6000, one Intel Xeon Platinum 8358P, 512 GB of host memory, and PCIe 4.0 ×16. All complete expert weights remain in host memory; GPU memory holds non-expert parameters plus a fixed expert pool. Baselines were reimplemented in the same KTransformers stack and given the same GPU expert capacity, which makes the comparison more useful than cross-framework headline numbers. genui{"citation":{"ref":"turn72view0"}}

Across batches from 1 to 32, Apex reports:

- **31.2% average and 51.9% maximum prefill-speed improvement** over the strongest baseline;
- **31.1% average and 39.5% maximum decode-throughput improvement**;
- gains across every tested expert-pool capacity from (0.5G_0) to (1.5G_0);
- larger benefits for Qwen3-Next, whose many small experts provide finer placement choices, than for GLM-4.5-Air, whose fewer, larger experts make each transfer more expensive. genui{"citation":{"refs":["turn72view2","turn72view3"]}}

The ablation is especially informative. On Qwen3.5 at batch 16:

| Configuration | CPU path | GPU path | Critical path | Decode throughput |
|---|---:|---:|---:|---:|
| No assignment | 3.074 ms | 0.295 ms | 3.074 ms | 72.44 tok/s |
| Speculative only | 2.515 ms | 0.408 ms | 2.515 ms | 84.62 tok/s |
| Exact only | 2.220 ms | 2.320 ms | 2.320 ms | 90.31 tok/s |
| Full Apex | 2.130 ms | 1.930 ms | 2.130 ms | 103.35 tok/s |

The result shows why aggregate CPU utilization is the wrong target. Full Apex still leaves substantial expert work on the CPU; it wins by making the CPU and GPU paths finish at similar times. genui{"citation":{"ref":"turn71view3"}}

## Why this matters for EPYC and ZenDNN work

Apex treats the CPU as an execution peer rather than a passive weight store, but its CPU model is still too coarse for a multisocket EPYC system. “CPU time” should be decomposed by placement:

[
T_{mathrm{CPU}} =
T_{mathrm{local expert}}
+
T_{mathrm{remote weight}}
+
T_{mathrm{activation movement}}
+
T_{mathrm{thread scheduling}}
]

A high-load expert is not automatically a GPU-transfer candidate. If its weights are already local to an idle CCD and its activations are small, CPU execution may win. Conversely, even a moderate-load expert can be expensive on CPU if its weights sit in remote-socket DRAM or if dispatch wakes a contended thread pool.

The natural extension is topology-aware assignment with three decisions rather than two:

- execute on the GPU;
- execute on CPU socket 0;
- execute on CPU socket 1.

For each expert, the scheduler should estimate:

[
operatorname*{argmin}_{din{mathrm{GPU},S_0,S_1}}
left[
T_{mathrm{queue}}(d)
+
T_{mathrm{move}}(e,d)
+
T_{mathrm{compute}}(e,n_e,d)
+
T_{mathrm{merge}}(d)
ight]
]

subject to all concurrently selected experts sharing the layer barrier. The objective remains the maximum path completion time, not the sum of isolated expert times.

### Concrete implementation experiment

A useful ZenDNN/KTransformers prototype would:

1. Keep one canonical expert copy in host memory and optionally replicate the hottest experts per socket.
2. Measure BF16 expert throughput by token-load bucket on each socket and GPU.
3. Include PCIe-root and NUMA distance in the transfer model.
4. Pin CPU expert workers to CCD groups and allocate their packed weights locally.
5. Update execution rates with an exponentially weighted estimator rather than a fixed offline profile.
6. Compare load-aware placement against static hot-expert residency, CPU-only misses, and transfer-all policies.

Record:

- per-expert token load and chosen device;
- local and remote DRAM bytes;
- PCIe bytes and exposed transfer time;
- CPU/GPU path completion skew;
- expert packing and activation-copy cost;
- p50/p99 TTFT and TPOT;
- tokens/s per socket and watts per token;
- decision instability when routing demand changes.

The key diagnostic is path skew:

[
S_i =
rac{left|T_{mathrm{CPU},i}-T_{mathrm{GPU},i}ight|}
{max(T_{mathrm{CPU},i},T_{mathrm{GPU},i})}
]

A good policy should reduce both layer latency and (S_i). A policy that merely shifts the bottleneck between devices will show oscillating skew without stable end-to-end gains.

## Important limitations

Apex is a substantive systems contribution, but its portability is not yet established:

- The evaluation uses one Intel Xeon and one older PCIe-attached A6000; results may change materially with EPYC, AMX-class CPUs, HBM GPUs, CXL, NVLink-C2C, or faster PCIe generations.
- CPU-side kernel details and thread/NUMA placement are not characterized deeply enough to separate scheduling gains from the tested host implementation.
- Workloads come from MMLU with controlled batch sizes; production continuous batching produces more variable prefill/decode mixtures and routing distributions.
- All models use BF16. Weight-only quantization changes both expert-transfer size and CPU/GPU compute ratios, potentially moving the optimal assignment boundary.
- The online cost model assumes recent rates predict the next layer. Contention from other tenants or shared host services can invalidate that assumption.
- No implementation repository is linked in the paper at publication time, so independent reproduction is still pending.

**Assessment:** Apex is genuinely important because it formulates heterogeneous expert placement around the concurrent critical path, validates the design across three modern MoE architectures, and includes a useful mechanism ablation. The reported speedups should not be generalized beyond the tested PCIe system, but the scheduling abstraction is strong and directly applicable to EPYC-based heterogeneous inference.

---

## Reliability watch: vLLM identifies unsafe KV reclamation during RDMA failure

A same-day vLLM RFC documents a separate correctness problem in MoRIIO-based prefill/decode KV transfer. Several failure paths can release KV blocks while RDMA or xGMI operations may still access them. The consequences include silent wrong-output reads, reuse during live DMA, indefinite waits, and leaked peer resources.

The proposed contract separates request failure from memory reclamation:

- report a failed transfer immediately;
- recompute into fresh blocks if policy permits;
- keep the original blocks in an allocator hold;
- reclaim only after the transfer is sealed and every possible accessor is proven quiescent;
- use explicit claim/offer leases and bounded waits with positive, negative, and peer-down exits.

This is an RFC, not a merged fix, but it identifies a class of errors that performance testing rarely catches: a timeout proves that the control path stopped waiting; it does **not** prove that the data plane stopped touching memory. genui{"citation":{"ref":"turn70view1"}}

**Track:** [vLLM MoRIIO transfer-contract RFC #63336](https://github.com/vllm-project/vllm/issues/63336)

For any disaggregated serving implementation, block reclamation should require a data-plane completion proof rather than a scheduler timeout. Add fault-injection tests for delayed completions, partial batches, peer restart, notification loss, and request cancellation while DMA is active. Track held bytes and enforce backpressure so safety does not turn an unbounded failure into an out-of-memory event.

## Bottom line

Apex advances CPU–GPU collaboration from static offload policy to online critical-path control. The practical lesson for CPU inference is not simply to “use spare CPU cycles.” It is to price each expert using its current token load, device throughput, transfer cost, memory locality, and the completion time of every concurrent path.

For EPYC systems, the next step is to make socket and CCD locality part of that assignment. The MoRIIO RFC adds the corresponding reliability rule: when execution and state movement become dynamic, memory ownership must be governed by explicit completion evidence, not by elapsed time or scheduler state.
