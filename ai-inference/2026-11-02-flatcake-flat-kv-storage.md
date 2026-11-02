# AI inference systems briefing — November 2, 2026

## Executive take

The most important change since the October 30 briefing is **FlatCake**, a KV-cache storage design that challenges the default HBM → host DRAM → NVMe hierarchy for long-lived, agentic workloads. Its core claim is architectural: once session working sets exceed DRAM and SSD reads are routinely on the critical path, treating DRAM as a mandatory cache and staging tier adds migration, fragmentation, and queue contention without providing useful temporal filtering.

This is a substantive systems result, not a new kernel trick. For CPU-oriented inference stacks, the implication is uncomfortable but useful: host DRAM capacity and bandwidth should not automatically be consumed by KV state. In some regimes, reserving DRAM for weights, activations, CPU decode, allocator headroom, and metadata—while placing selected KV scopes directly on SSD—can produce a better end-to-end system.

## 1. FlatCake makes DRAM and NVMe peer KV media

**Importance: High for long-context and agentic serving; high architectural relevance to CPU/GPU systems**

FlatCake profiles eight production traces and finds two reuse regimes that both undermine conventional tiering:

| Workload regime | Observed behavior | Why DRAM tiering fails |
|---|---|---|
| Conversational/API | 53–94% of blocks are one-shot; reusable blocks can have reuse distances of hundreds to thousands of requests | Admitting every new block pollutes DRAM and evicts prefixes before reuse |
| Agentic coding | Reuse is dense and immediate, but a session carries multi-gigabyte state | Concurrent active sessions exceed DRAM, so SSD remains on the read-critical path |

The Claude Code trace is the sharpest example: requests average about 219K input tokens. For the evaluated DeepSeek-V4-Flash configuration, history state averages 0.80 GiB per GPU rank, or 6.4 GiB across eight ranks, with 16 GiB at P90. Even a large host-memory pool is quickly oversubscribed once many sessions are active.

FlatCake uses three coordinated mechanisms:

1. **Write-once placement.** Each immutable KV block is assigned directly to DRAM or SSD at creation and never migrates between them. A 64-bit global virtual address records its medium, node, and offset.
2. **Session-contiguous, layer-major SSD layout.** Blocks that will be restored together are written together. Reads collapse from roughly (O(	ext{layers} 	imes 	ext{blocks})) fragmented operations to (O(	ext{layers})) sequential transfers.
3. **Media-aware scheduling and bounded direct reads.** DRAM-ready requests and SSD-prefetching requests use separate queues, preventing SSD latency from blocking fast-path work. SSD data streams through a bounded GPU window via GPUDirect Storage rather than occupying context-sized host staging buffers.

This is a stronger idea than merely adding GPUDirect Storage. The paper's Mooncake-GDS baseline bypasses host staging but retains eviction-ordered SSD placement. At 64K tokens, it reports 6.60 s mean TTFT versus 3.74 s for FlatCake, showing that transport acceleration does not repair a fragmented physical layout.

### Reported results

The primary system uses eight B200 GPUs, 3 TiB host DRAM, and eight Samsung PM9A3 NVMe drives in RAID 0, with equal 1 TiB DRAM and 4 TiB SSD KV budgets across systems. On a production Claude Code trace with DeepSeek-V4-Flash:

- FlatCake sustains a 94.16–94.57% delivered cache-hit rate across load, close to the trace's 94.72% reuse ceiling.
- Mooncake falls from 87.9% at low load to 64.26% at higher load because staging contention prevents cached blocks from reaching HBM before prefill.
- At 1 QPS, FlatCake reports 7.57 s mean TTFT versus 12.24 s for Mooncake, alongside 1.6× higher total-token throughput.
- It eliminates Mooncake's reported 6.5–6.7× SSD read amplification.

A second experiment uses Llama-3.1-70B with tensor parallelism across four A100 GPUs and 8K–128K inputs. FlatCake reports 1.5–2.4× lower mean TTFT than Mooncake and up to 1.8× lower TTFT than Mooncake-GDS.

**Primary source:** [FlatCake: A Flat KVCache Storage Architecture for LLM Serving](https://arxiv.org/abs/2610.38355)

## Why this matters for an EPYC/ZenDNN-oriented stack

FlatCake's direct path is GPU-specific, but the placement argument is not. A CPU or heterogeneous server should classify KV storage by expected reuse and service cost rather than enforce a universal cache ladder.

A practical placement score should include:

[
	ext{benefit}(b,m) =
P_{	ext{reuse}}(b)cdot C_{	ext{recompute}}(b)
-
C_{	ext{read}}(b,m)
-
C_{	ext{capacity}}(b,m)
-
C_{	ext{NUMA}}(b,m)
]

where (b) is a KV scope and (m) is local DRAM, remote DRAM, CXL memory, or NVMe. This makes three costs explicit that a hit-rate-only policy hides:

- keeping a one-shot block in socket-local DRAM can evict weights or reusable state;
- a remote-NUMA “hit” can be worse than a local recomputation for a short prefix;
- an SSD hit can be valuable if its layout permits large sequential reads and overlaps with useful compute.

For a dual-socket EPYC system, I would prototype this as a scope-level allocator:

- reserve per-socket DRAM budgets for weights, activations, CPU decode state, and staging headroom;
- predict prefix reuse at session or tenant scope;
- bind the scope once to local DRAM, remote/CXL memory, or NVMe;
- keep SSD extents session-contiguous and layer-major;
- use independent admission queues for local-memory-ready and storage-bound requests;
- measure delivered reuse—the fraction restored before its compute deadline—not nominal cache hits.

The last metric is essential. FlatCake's strongest observation is that a block can exist on SSD yet behave like a miss if staging or scheduling cannot deliver it before prefill starts.

## What is genuinely new—and what is not

**Genuinely important**

- The paper derives its architecture from production reuse-distance data rather than a synthetic locality assumption.
- It co-designs placement, physical layout, scheduling, and transfer. The Mooncake-GDS comparison isolates why a faster I/O primitive alone is insufficient.
- It reframes host DRAM as a peer resource whose opportunity cost must be modeled, not as a mandatory waypoint.

**Not yet established**

- The main result depends on a high-end eight-drive RAID 0 array and GPUDirect Storage. Conventional PCIe servers, fewer drives, encryption, filesystem overhead, and shared-storage contention may shift the crossover point.
- Evaluation is NVIDIA-only. The write-once placement and queue isolation ideas should transfer, but the direct SSD-to-device path and overlap behavior require fresh measurement on AMD GPU or CPU-only deployments.
- Session-contiguous allocation trades read efficiency for space-management constraints. Long-lived sessions, partial-prefix sharing, branch-heavy agents, cancellation, and garbage collection could create fragmentation or stranded capacity not fully characterized here.
- The paper reports mean and median TTFT prominently; production adoption still needs p99 TTFT, inter-token latency, SSD endurance, failure recovery, and noisy-neighbor results.
- This is an arXiv paper with no linked public implementation on the abstract page, so the evaluation is not yet independently reproducible.

## Engineering experiment worth running

The highest-value validation is not “port FlatCake.” It is to determine whether your workload has crossed the point where DRAM tiering is counterproductive.

Replay a representative multi-turn trace under four policies:

1. DRAM-only KV caching;
2. conventional DRAM → NVMe eviction with host staging;
3. direct NVMe reads while retaining eviction-ordered layout;
4. write-once scope placement with session-contiguous layout.

For each socket and storage device, record:

- delivered cache-hit rate and recomputed tokens;
- local and remote DRAM bandwidth;
- staging bytes and aborted/late prefetches;
- SSD request size, queue depth, bandwidth, and read amplification;
- TTFT p50/p95/p99 and inter-token latency;
- CPU cycles spent in copying, allocation, completion handling, and storage queues;
- capacity lost to fragmentation and stale session scopes.

If policy 3 barely improves over policy 2 but policy 4 does, physical layout—not DMA overhead—is the bottleneck. If policy 1 remains best, your active session set still fits DRAM and a flat architecture is premature.

## Reading priority

1. **Read closely:** [FlatCake](https://arxiv.org/abs/2610.38355), especially the production-trace characterization (§2.2), address and placement design (§3.2), request scheduler (§3.3), coalesced I/O path (§3.4), and evaluation (§5).
2. **Read as related CPU context:** [DualPace: Decoupled Scheduling and Cooperative GPU–CPU Decoding for LLM Serving](https://arxiv.org/abs/2610.33501). It predates the previous briefing but is worth catching up on: it treats an AMX CPU as an independently scheduled decode engine rather than a GPU-controlled offload target. Its reported gains are promising, but hardware specificity and the absence of a linked implementation make it less actionable than FlatCake today.

## Bottom line

The week's meaningful change is a shift from **tier management** to **lifetime-aware placement**. Once agentic KV working sets exceed host memory, optimizing eviction policy or DMA transport may be solving the wrong problem. The more promising design is to decide a KV scope's home at creation, preserve the spatial order in which it will be restored, and schedule requests according to the medium that can actually meet their compute deadline.

For CPU inference engineering, that elevates storage layout, NUMA opportunity cost, and delivered-reuse latency to the same level as quantized GEMM throughput. The immediate next step is a trace-driven crossover study: identify when local DRAM stops being a useful KV cache and becomes the scarce resource your inference engine should protect.
