# AI inference systems briefing — November 25, 2026

## Executive assessment

The most important new result is [LiveLoRA](https://arxiv.org/abs/2611.31105), which addresses a lifecycle problem that serving systems usually avoid: how to update an adapter while thousands of long-lived agent sessions retain KV state computed under the old version.

Reusing the old cache silently mixes model versions. Recomputing every active history preserves correctness but creates a publication-time compute and latency spike that grows with session count and context length. LiveLoRA constrains where and how LoRA updates are applied so the runtime can reconstruct affected KV state and switch the adapter, cache, and next-token logits atomically. Under its assumptions, continuation is equivalent to full-history re-prefill in exact arithmetic.

The second useful result is [DisSparse](https://arxiv.org/abs/2611.28522). Yesterday’s LongCat briefing focused on making sparse retrieval page-regular and sublinear. DisSparse addresses a different deployment obstacle: adaptive top-p attention produces a variable number of KV blocks, conflicting with static scheduler allocation, while selection itself can cost more than attention. It pipelines selection off the scheduler’s critical path and uses a single-pass histogram kernel.

The broader theme is that inference optimization now extends into state evolution. A production runtime must manage not only where KV lives, but also which model version produced it, how it can be transformed, and whether adaptive sparsity can be admitted without destabilizing scheduling.

## 1. LiveLoRA: exact KV migration across adapter publication

**Importance: High for persistent agents, multi-LoRA serving, and continuous model updates**

[LiveLoRA: Exact KV Cache Migration Across LoRA Updates for LLM Agents](https://arxiv.org/abs/2611.31105), published November 23, targets persistent sessions that accumulate long histories while their LoRA adapter continues to evolve.

The conventional choices are both poor:

| Update policy | Correctness | Cost |
|---|---|---|
| Keep old KV and switch adapter | Incorrect version mixing | Low immediate cost |
| Re-prefill full history under new adapter | Equivalent to fresh execution | Grows with every active session and token |
| Drain sessions before update | Correct | Delays rollout indefinitely for persistent agents |
| Pin each session to old adapter | Correct per session | Retains many versions and fragments capacity |

LiveLoRA introduces a fifth option: restrict the adapter structure so the cache can be migrated exactly.

### Model–runtime contract

The design freezes lower decoder blocks and constrains the final-block K/V LoRA updates. Fixed down-projections preserve low-rank coordinates that remain valid across adapter versions.

At publication time, the runtime:

1. reconstructs candidate K and V state for the new adapter;
2. reevaluates the most recent token through the final block;
3. computes the corresponding next-token logits;
4. atomically commits the new adapter version, migrated cache, and logits.

The atomic commit matters. Updating weights without cache and logits creates an intermediate state in which different parts of the continuation belong to different model versions.

The authors prove continuation equivalence to full-history re-prefill in exact arithmetic under the stated architectural assumptions.

### Reported evidence

On Qwen2.5-3B-Instruct with 32 active sessions of 8K tokens each, LiveLoRA reports a median paired adapter-publication speedup averaging **68.98×** over full-history re-prefill across five runs.

The continuing cost is modest but not free:

- approximately **4.8% decoding overhead** relative to native cached decoding;
- **3.42× less additional session storage** at 8K than retained-boundary replay using the compact-retention design;
- similar measured publication-plus-decoding cost to the retained-boundary alternative.

The large publication speedup should be interpreted in context: the baseline recomputes 256K session tokens. It does not imply a 69× improvement to normal token generation.

### Why it matters for large-scale serving

This changes adapter rollout from a control-plane event into a state-migration transaction.

A production system needs versioned identities for:

- base model;
- adapter parameters;
- cache transformation contract;
- tokenizer and prompt template;
- quantization and KV layout;
- migration algorithm version;
- session commit epoch.

Requests arriving during publication need defined semantics. They should either complete under the old epoch or begin under the new one; no batch should combine partially migrated session state.

Failure recovery also matters. If migration fails after candidate cache construction but before commit, the old adapter and cache must remain valid. Candidate storage should be isolated until the transaction succeeds.

### CPU and EPYC implications

The exact LiveLoRA transformation is model-specific, but the systems concept transfers directly to CPU inference.

CPU fleets often host many adapters over a shared base model. Persistent session caches may reside in socket-local DRAM, shared memory, or an external tier. Re-prefilling all histories after an adapter update consumes GEMM capacity, pollutes LLC, and competes with live decode traffic.

A CPU-oriented implementation should:

- retain invariant low-rank coordinates in a compact, NUMA-local format;
- reconstruct migrated K/V with batched GEMM or GEMV grouped by adapter and sequence length;
- pin migration work to the socket owning each session’s cache;
- rate-limit background migration to preserve p99 TPOT;
- use copy-on-write candidate pages and an epoch-based atomic swap;
- expose adapter publication as a measurable workload class rather than hidden maintenance.

The critical benchmark is not just migration throughput. Sweep active-session count and context length while measuring:

- publication wall time;
- extra bytes stored per token;
- live-request p99 TTFT and TPOT during migration;
- DRAM bandwidth and remote-NUMA traffic;
- old/new adapter memory overlap;
- rollback time after injected failure;
- numerical equivalence to full re-prefill.

### Assessment and limitations

LiveLoRA is a substantive model–systems contribution because it provides an exactness argument, runtime transaction, storage tradeoff, and serving measurements for a real operational problem.

Its constraints are significant:

- only the final block is updated under the demonstrated scheme;
- lower blocks remain frozen;
- K/V adapter down-projections must remain fixed;
- exact-arithmetic equivalence does not automatically guarantee bitwise equality under every precision and kernel ordering;
- evaluation centers on one 3B model and moderate session scale;
- the restricted update space may reduce adaptation quality for some tasks.

The paper demonstrates that exact cache migration is possible under a designed update contract. It does not establish that arbitrary LoRA updates can be migrated.

**Read:** [LiveLoRA: Exact KV Cache Migration Across LoRA Updates for LLM Agents](https://arxiv.org/abs/2611.31105)

## 2. DisSparse: adaptive top-p attention without scheduler destabilization

**Importance: Medium-high for long-context serving; incremental algorithmically, substantive in integration**

[DisSparse: Pipelined Top-p Sparse Attention for Long-Context LLM Serving](https://arxiv.org/abs/2611.28522), submitted November 21 and accepted at NeurIPS 2026, starts from a tension between attention quality and serving predictability.

Fixed top-k sparse attention selects the same block count even when attention distributions differ across heads, layers, and steps. It can waste bandwidth on concentrated distributions or discard meaningful mass on diffuse ones.

Top-p selection adapts the retained block count to the distribution, but creates two systems problems:

1. variable block counts conflict with deterministic memory allocation;
2. selection kernels can cost more than the sparse attention they enable.

DisSparse integrates top-p selection into vLLM with three mechanisms.

### Selection leaves the scheduler’s critical path

Block selection runs asynchronously in a two-stream pipeline and overlaps with layer computation. The scheduler no longer waits for every variable-size selection result before making forward progress.

This is the most transferable insight. An adaptive policy can improve average work yet reduce end-to-end performance if its decision latency serializes the serving loop.

### Single-pass histogram selection

Instead of a more expensive general selection procedure, DisSparse uses a histogram-based top-p kernel that determines the threshold in one pass. It reports **1.17–1.80× faster sparsity analysis** than compared top-p approaches.

These are selection-stage gains, not end-to-end model speedups.

### Token relocation reduces block inflation

Paged KV systems fetch blocks, not individual tokens. If high-importance tokens are scattered across blocks with low-importance tokens, token-level sparsity does not translate into block-level bandwidth savings.

DisSparse accumulates attention information during prefill and relocates tokens so selected tokens are packed into fewer blocks. This makes logical sparsity match physical transfer granularity.

Under the same memory budget, the system reports up to **8× larger maximum batch sizes** than dense vLLM, with negligible reported accuracy loss across its evaluated models and benchmarks.

### CPU relevance

The token-relocation idea is especially relevant to CPU memory systems. Sparse indices are ineffective when every selected token pulls in a different cache line, page, or NUMA allocation.

A CPU design should optimize the physical unit that the hardware transfers:

| Logical decision | Physical consequence to optimize |
|---|---|
| Select token | Avoid a separate cache-line or gather miss |
| Select block | Keep block contiguous and socket-local |
| Change top-p budget | Preserve bounded staging and metadata allocation |
| Relocate tokens | Maintain position mapping without pointer chasing |
| Pipeline selection | Avoid competing with attention for memory bandwidth |

The selection thread or task should be placed carefully. Running it concurrently may hide latency but increase bandwidth contention with decode. On a bandwidth-saturated EPYC socket, time overlap is not resource overlap.

Measure:

- exposed versus overlapped selection time;
- memory-controller utilization during overlap;
- selected tokens per fetched block;
- gather count and LLC miss rate;
- variable-budget distribution;
- metadata and relocation overhead;
- p99 TPOT rather than average kernel latency.

### Relationship to yesterday’s LRSA paper

LRSA and DisSparse address different bottlenecks:

- **LRSA:** avoid scanning the full history with an incremental representative index.
- **DisSparse:** adapt the retained attention mass while making variable selection schedulable and physically compact.

They could be complementary in principle: an incremental index nominates candidates, top-p determines the adaptive budget, and relocation packs chosen members. Combining them would require careful quality validation because each approximation changes the support set.

### Assessment and limitations

DisSparse is more than a new selection rule because it handles scheduler integration, kernel cost, and block-level physical layout.

The top-p policy remains approximate and model-dependent. Token relocation complicates position mapping, prefix reuse, and cache-tier migration. The headline 8× batch-capacity gain is against dense vLLM, not necessarily the strongest fixed-budget sparse baseline. End-to-end improvements should be checked across realistic context distributions rather than maximum-context cases alone.

**Read:** [DisSparse: Pipelined Top-p Sparse Attention for Long-Context LLM Serving](https://arxiv.org/abs/2611.28522)

## 3. Secondary signal: thermal-aware precision control

[GreenDynaMix](https://arxiv.org/abs/2611.31737) jointly applies DVFS thermal control and runtime mixed precision for MoE inference. Router-derived expert-importance signals retain higher precision for critical experts while other experts move to lower precision to recover throughput lost to thermal throttling.

Across server, desktop, and embedded GPU platforms, the authors report a **25–33% reduction in temperature-induced aging rate**. Throughput recovery costs 0.9–1.6% perplexity with 8/4-bit mixtures and up to 9.4% where memory constraints force 4/2-bit precision.

This is an interesting reliability-aware control loop, but not yet a high-priority CPU result. CPU servers already expose mature thermal, power, and frequency telemetry, yet dynamically changing expert precision introduces quality and kernel-layout complexity. The most transferable idea is to include sustained thermal state in inference benchmarking: a configuration that wins a short cold run can lose after frequency settles.

For EPYC testing, record effective frequency, socket power, memory temperature, throttling counters, and throughput over long steady-state runs. Do not infer production performance from a short warmup followed by a brief measurement window.

**Read:** [GreenDynaMix: Thermal-Aware Dynamic Mixed-Precision MoE LLM Inference Systems](https://arxiv.org/abs/2611.31737)

## Recommended engineering actions

1. **Version KV state explicitly.** Cache identity should include base model, adapter, transformation contract, layout, and commit epoch.
2. **Make adapter publication transactional.** Construct candidate cache state separately and atomically commit weights, KV, and next-token state.
3. **Benchmark model updates as live workloads.** Measure migration interference and rollback, not only steady-state inference.
4. **Keep adaptive selection off the critical path.** Pipeline it only after checking whether it contends for the same memory bandwidth as attention.
5. **Align semantic sparsity with physical transfer units.** Pack selected tokens into fewer cache lines, pages, or KV blocks.
6. **Separate selection-stage and end-to-end claims.** A faster top-p kernel matters only if scheduler, relocation, and attention costs improve overall TPOT.
7. **Run thermal steady-state tests.** Include frequency and power telemetry in long-duration performance qualification.

## Bottom line

Yesterday’s briefing focused on deciding which context to retain and how to retrieve it. Today’s strongest result adds model-version evolution: retained state must remain consistent when the model changes.

LiveLoRA shows that cache migration can be exact when adapter design and runtime commit semantics are co-designed. DisSparse shows that adaptive sparsity becomes deployable only when its variable work and selection latency are reconciled with scheduler and memory-layout constraints.

For a CPU inference stack, the immediate opportunity is a versioned state-migration layer: NUMA-local candidate reconstruction, copy-on-write cache pages, epoch-based atomic publication, and explicit measurement of live traffic during updates. The same runtime should treat sparse-selection output as a physical layout problem—optimizing bytes and cache lines moved, not merely tokens discarded.