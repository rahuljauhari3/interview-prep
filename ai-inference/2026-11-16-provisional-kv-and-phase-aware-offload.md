# AI inference systems briefing — November 16, 2026

## Executive view

The strongest new result since Friday’s briefing is **Callo**, which changes the role of invalidated KV state after an agent edits its context. Instead of discarding the stale suffix or waiting for exact re-prefill, Callo uses that state to draft tokens on otherwise idle decode capacity, then verifies those tokens against the exact KV before committing anything. The mechanism is lossless and hides roughly half of the cache-miss penalty in its single-request study, although its end-to-end gain is a more modest 1.05–1.19× at moderate load.

Two other papers sharpen the same systems lesson at lower layers. Zero-SM NCCL moves collectives through copy engines and RDMA rather than consuming GPU compute units, dramatically reducing interference with overlapping GEMMs—but it loses on small transfers and host-bound workloads. TailGuard-GDS finds that direct NVMe-to-GPU I/O is frequently worse at tail latency than an optimized host-staged path. Together, these results argue for **phase-aware resource routing**, not blanket use of a theoretically “more direct” path.

## 1. Callo turns invalidated KV into provisional compute state

**Assessment: Genuinely important for agentic inference**

[Callo: Serving Cache-Miss Requests with Speculative Decoding on Invalidated KV under Prefill-Decode Disaggregation](https://arxiv.org/abs/2611.18459), submitted November 13, targets a common agent workload that ordinary prefix caching handles poorly: a long session edits, masks, or summarizes earlier context. The edit invalidates KV for the retained suffix, forcing exact re-prefill before generation can begin.

Callo observes that the decode worker often still holds the previous request’s invalidated KV while the prefill worker recomputes the exact state. That stale KV cannot safely be committed, but it is close enough to the new state to act as a speculative draft source.

The system:

1. aligns unchanged token runs between the old and edited prompts;
2. copies their KV into provisional blocks and re-rotates keys for new positions;
3. computes KV for newly inserted tokens in small chunks;
4. starts target-model drafting on this provisional state;
5. streams exact KV from the prefill worker layer by layer, overwriting provisional blocks in place;
6. verifies all draft tokens using the completed exact KV before committing the accepted prefix.

The verification step makes the output lossless with respect to the configured greedy or sampling rule. Across 1,928 verification events and 44,792 committed tokens, the authors report no fingerprint, acceptance, output, or rollback violations.

### Why stale KV works as a drafter

On the paper’s Llama-3.3-70B SWE-agent replay:

- invalidated-KV predictions matched exact-KV next tokens at 98.3% of positions;
- 80% of next-token distributions were within 0.01 total variation;
- a complete 32-token draft window survived fixed-tie verification on 65% of cache-miss requests;
- low-confidence, near-tied logits accounted for most first mismatches.

That final property enables early termination: the scheduler tracks cumulative acceptance likelihood and preempts a speculation when further drafts are unlikely to survive verification.

The provisional state also “heals” while exact KV arrives. Layer-major streaming performed better than position-major refresh, so Callo pushes each completed attention layer rather than waiting to transfer the entire suffix at the end.

### Measured impact

For the main single-request configuration, Callo:

- hid 49% of the estimated miss penalty;
- produced 22.6 accepted drafts per miss request on average;
- accelerated miss requests by 1.15×;
- retained 75% of all drafted tokens through verification.

Across the live two-GPU tests, miss-request speedup was 1.06–1.24× and all-request speedup was 1.05–1.19× at moderate load. The smaller end-to-end number is the credible one for capacity planning.

Callo deliberately yields to exact decoding as load rises. Without its staging admission budget, speculation removed the benefit for non-speculating requests at 90% simulated load. This is not extra throughput from nowhere; it converts **decode slack during prefill waits** into useful provisional work.

### Practical implications for CPU and hybrid serving

The idea transfers beyond GPUs. In a CPU or CPU–accelerator deployment, invalidated KV could be used by a lower-cost draft lane while an exact prefill runs elsewhere:

| Component | Callo implementation | CPU/hybrid analogue |
|---|---|---|
| Exact re-prefill | Prefill GPU | Compute-optimized CPU socket or accelerator |
| Provisional drafting | Decode GPU using stale KV | Bandwidth-local CPU instance or spare cores |
| KV repair | Layer-wise GPU transfer | Layer-/block-wise NUMA-local overwrite |
| Admission control | Decode batch and staging budget | Core, bandwidth, and LLC-pressure budget |
| Verification | Target model on exact KV | Exact model lane after repaired state arrives |

For EPYC, admission should be driven by more than idle cores. Drafting can consume memory bandwidth and LLC capacity needed by exact decode. A useful policy would admit speculation only when:

[
E[	ext{accepted tokens}] cdot T_{	ext{decode-step}}
>
T_{	ext{staging}} + T_{	ext{verification}} + T_{	ext{interference}}
]

Measure (T_{	ext{interference}}) per NUMA node using bandwidth counters and p99 inter-token latency. A stale-KV drafter on remote memory may look computationally cheap while degrading exact requests through fabric traffic.

### Caveats

The real testbed has only one prefill and one decode GPU. Cluster-scale conclusions come from a calibrated simulator. The workload is SWE-agent with three model families, so acceptance may differ for conversational agents, RAG, aggressive summarization, or edits that rewrite semantically important early context. The system also depends on retaining invalidated blocks long enough to stage them, creating a capacity trade-off against ordinary prefix caching.

**Reading priority:** High. Read Sections 3–5 and the decode-side policy appendix.

## 2. Zero-SM communication reduces compute interference—but only in the right regime

**Assessment: Strong communication-systems work; indirectly relevant to inference**

[Zero-SM NCCL: A Systematic Guide to Designing and Deploying SM-Free GPU Communication](https://arxiv.org/abs/2611.18075), submitted November 13, removes GPU streaming multiprocessors from host-initiated communication.

Conventional NCCL kernels use SMs to copy data, reduce values, poll flags, and enforce stream order. That competes with GEMMs even when communication is nominally asynchronous. Zero-SM NCCL instead uses:

- CUDA stream memory operations for ordering;
- copy engines for movement inside a scale-up domain;
- GPUDirect RDMA driven by a stream-ordered CPU proxy across domains;
- symmetric memory registration for direct peer addressing.

On GB200 NVL72 and B200 NVL8, it reports up to 1.36× higher AllGather bandwidth and reduces median slowdown of an overlapping GEMM from 33% to 4%. End-to-end gains are smaller: up to 3.5% for FSDP and 15% for deep pipeline-parallel workloads.

The important negative result is explicit: zero-SM paths have higher per-operation latency and lose on small messages or host-bound workloads.

### Why it matters for inference

Inference systems increasingly overlap collectives with attention, MoE dispatch, KV movement, or speculative verification. The proper selection variable is not just tensor size:

[
	ext{path} =
f(	ext{message size}, 	ext{compute occupancy}, 	ext{host load}, 	ext{pipeline slack})
]

For CPU-centric systems, the analogue is avoiding a “helper” path that steals cores, cache, or memory bandwidth from inference. An offloaded DMA or DSA path is beneficial only if setup and synchronization cost are below the interference it removes. Benchmark the overlapped application, not isolated copy bandwidth.

**Reading priority:** Medium-high. The decision boundaries and negative regimes are more valuable than the peak bandwidth claim.

## 3. Direct storage paths are not automatically tail-optimal

**Assessment: Rigorous systems evidence; inference application remains to be demonstrated**

[TailGuard-GDS: Tail-Safe Control of GPU Storage Data Movement](https://arxiv.org/abs/2611.17824), submitted November 13, compares GPUDirect Storage with optimized host staging and finds that the “direct” path is often not the p999 winner.

Application-visible latency includes request submission, buffer registration, filesystem and NVMe service, PCIe movement, and completion synchronization. Avoiding one host copy does not eliminate the other costs. The preferred path changes with operation type, request size, concurrency, PCIe placement, and contention; bandwidth and tail-latency winners can disagree.

TailGuard profiles semantically equivalent delivery plans and selects per application phase:

- backend: GDS or host staging;
- request size and concurrency;
- GPU/storage placement;
- tail-latency target and throughput/deadline constraints.

In FSDP and DeepSpeed ZeRO-3 experiments, it reduces p95 step time by 7.9–42.6% relative to fixed paths, depending on baseline and workload. Those are training results, not proof of inference gains, but the mechanism maps directly to weight paging and KV spill/restore.

### Practical implication

For a tiered CPU/GPU inference stack, storage routing should be attached to transfer intent:

- synchronous KV restore: minimize foreground stall;
- speculative prefetch: maximize useful bandwidth without hurting decode;
- model-segment load: optimize deadline probability;
- cache eviction/writeback: throttle against foreground memory pressure.

On EPYC, include host-staged paths through socket-local pinned memory rather than treating “host staging” as one uniform option. PCIe root-complex locality and NUMA placement can reverse the result.

**Reading priority:** Medium. Read for the methodology and controller boundary, not the training-specific headline gains.

## What changed from the previous briefing

Friday’s briefing focused on **topology-aware placement and sharding**: preserving GEMM shape, KV capacity, and hierarchical collectives across chiplets. The new work adds a complementary dimension: **temporal and phase-aware scheduling**.

- Callo asks whether invalid state can perform provisional work while exact state is rebuilt.
- Zero-SM NCCL asks whether communication should consume compute resources while kernels run.
- TailGuard asks which physically equivalent data path minimizes application-visible tail latency in the current phase.

The common principle is that a resource path cannot be judged in isolation. Its value depends on which critical-path work it overlaps, delays, or displaces.

## Bottom line

Callo is the standout paper. It introduces a useful new state category—**invalid for commitment but valid for speculation**—and integrates it into a lossless serving path with explicit admission and preemption.

For an experienced CPU inference engineer, the highest-value prototype is a stale-KV speculative lane with three hard guards:

1. NUMA-local provisional state;
2. an admission model that prices memory-bandwidth interference, not just core occupancy;
3. exact verification that preserves output semantics.

More broadly, the three papers reinforce the same deployment rule: choose communication and storage mechanisms using end-to-end tail behavior under overlap. “Direct,” “asynchronous,” and “offloaded” are implementation properties, not performance guarantees.
