# AI inference systems briefing — November 11, 2026

## Executive assessment

Two new papers change the practical memory-management picture established by the November 10 [Evolving Infra briefing](https://github.com/rahuljauhari3/interview-prep/blob/main/ai-inference/2026-11-10-agent-adaptive-serving.md):

1. **SlackTier** shows that enormous, sparsely accessed model tables should be placed according to each layer’s prefetch deadline—not kept uniformly in host DRAM and not cached according to popularity alone.
2. **Bursts That Outlast Themselves** shows that enlarging the KV tier does not by itself make agent serving stable. Prefix caching creates a feedback loop in which a short burst can push the server into a persistent low-hit-rate, low-throughput regime after the burst has ended.

Together, the papers sharpen a deployment principle:

> Capacity must be allocated by deadline and protected by working-set admission.

SlackTier moves 203 GB of DeepSeek-V4.1 Engram tables from host DRAM to NVMe or remote memory while keeping mean decode overhead at no more than 1.4% in the reported replays. Host memory falls from 212 to 34 GiB. Reusing the freed memory as a host KV tier raises median throughput by 36% on a KV-constrained agent workload and reduces median p90 TTFT from 38 to 24 seconds.

The companion metastability paper explains the failure mode that remains. With prefix-cached agent sessions, a brief overload can evict active prefixes; misses then expand prefill work by roughly 10–40×, lowering throughput, increasing the number of concurrent unfinished sessions, and further expanding the live working set. Simple admission throttling after collapse is often too late. The effective control is to park already-admitted sessions at turn boundaries until the active sessions’ estimated KV demand fits the cache.

**Assessment:** both are substantive. SlackTier is a rigorous end-to-end placement result, not merely an NVMe-offload claim. The metastability paper’s feedback loop is not wholly new, which its authors acknowledge, but its controlled paired experiments, persistence result, and recovery analysis are operationally important.

**Primary sources:**

- [Placing LLM Lookup Tables by Per-Layer Prefetch Deadlines](https://arxiv.org/abs/2611.13413), submitted November 10, 2026 ([PDF](https://arxiv.org/pdf/2611.13413)).
- [Bursts That Outlast Themselves: Persistent Degradation in Prefix-Cached Agent Serving](https://arxiv.org/abs/2611.13412), submitted November 10, 2026 ([PDF](https://arxiv.org/pdf/2611.13412)).

---

## 1. What changed: model state now has per-layer delivery deadlines

DeepSeek-V4.1-Flash contains two Engram n-gram lookup tables totaling 203 GB. Each token hashes its recent n-grams into 24 rows per Engram layer, and each row contains 256 FP8 values plus scales. Only about 0.8 MB is touched per layer in a 128-session decode step, but the existing serving path keeps the complete tables in pinned host DRAM.

On the authors’ four-H200 node, the serving process uses 223 GiB of host memory, of which 190 GiB belongs to the tables. The table access itself is inexpensive from DRAM—20–160 microseconds per step—so this is a capacity problem rather than a baseline latency problem.

The key observation is that the row indices depend only on input tokens. All required reads are therefore known at the beginning of the decode step, before the layers that consume them execute. But the two Engram layers have very different slack:

| Engram layer | Approximate prefetch window | Uncached NVMe read | Consequence |
|---|---:|---:|---|
| Layer 1 | 0.3 ms | 2.3 ms | NVMe misses are exposed; needs cache or faster tier |
| Layer 14 | 9.3 ms | 3.6 ms | Entire uncached read fits behind earlier layers |
| Either layer over RDMA | Deadline-dependent | about 0.23 ms | Both fit without a row cache in the tested setup |

A table’s useful placement is therefore not determined by size or average bandwidth alone. It is determined by whether its tail read time fits the layer-specific window:

[
T_{	ext{lookup},ell}^{p99}
le
W_ell
]

where (W_ell) is the time from issuing the lookup to the layer consuming its rows.

This is a stronger rule than “keep hot rows in HBM.” A layer with a long window may need no cache even when its accesses are cold. A layer with a short window may need nearly all available cache capacity even if another layer has a higher aggregate hit rate.

### Why this matters for CPU inference

CPU inference stacks often group state by representation:

- all embeddings in DRAM;
- all expert weights in CXL;
- all lookup tables on NVMe;
- all KV in one host tier.

SlackTier argues for grouping by **consumption deadline** instead.

On an EPYC server, the same object may have different valid homes depending on where it is consumed in the layer graph:

[
	ext{valid tiers}(ell)
=
left{
j mid
T_{	ext{fetch}}^{p99}(j,	ext{load})
le
W_ell
ight}
]

A late-layer table can live in remote-socket DRAM, CXL memory, or NVMe because many preceding operators hide its retrieval. An early-layer table may need socket-local DRAM or LLC residency. This applies to:

- n-gram and product-key lookup tables;
- MoE expert weights;
- adapter weights;
- sparse attention metadata;
- quantization scales and refinement overlays;
- model-specific recurrent state.

The planner should measure deadlines under the actual batch, thread placement, and contention—not infer them only from graph position.

---

## 2. SlackTier: issue every lookup early, prioritize the tight deadline

SlackTier modifies SGLang so both Engram lookups are submitted at the beginning of the decode step. CPU I/O threads perform O_DIRECT asynchronous reads from raw NVMe devices or RDMA reads from another node’s memory into pinned pages. A side stream inside the CUDA graph manages descriptors, copies, and events.

Several implementation details are important:

- Layer 1’s I/O is submitted before layer 14’s.
- Each layer waits only on its own completion event.
- A GPU row cache serves hits and is filled by misses.
- The CPU directory maps each table row to its cache slot.
- CLOCK replacement pins rows used by the current step.
- One CPU pass performs hit lookup, page deduplication, and I/O-list construction.
- Directory processing takes 80–160 microseconds at 128 sessions and overlaps with layer 0.
- The resulting gather is bit-exact with the original host-table path; the checker found no mismatch over 790,000 lookups.

The result is not simply “NVMe is fast enough.” Naively placing both tables on one disk slows decode by roughly 6–7% at 128 sessions. Caching only layer 14—the table with the comfortable deadline—reduces the slowdown only modestly. Splitting the same one-million-row cache between the layers reduces overhead to roughly 1.2–1.3%, because the scarce cache protects the short-window layer.

Device contention matters as well. When both layers share a disk, layer 14’s reads can delay layer 1. SlackTier’s model places both on a common device timeline and predicts later-run decode slowdown across NVMe and RDMA, from 8 to 256 sessions, with 0.22 ms RMSE after fitting one constant.

### CPU-specific translation

A CPU implementation should separate three quantities:

1. **issue time** — when an address becomes known;
2. **need time** — when the consuming kernel reaches it;
3. **service distribution** — not only median tier latency, but p95/p99 under concurrent reads.

The usable slack is:

[
S_{ell,j}
=
W_ell
-
T_{	ext{fetch},j}^{p99}
-
T_{	ext{software},ell}
]

A positive (S_{ell,j}) makes tier (j) feasible. Negative slack estimates exposed stall.

For EPYC, include:

- remote-NUMA read latency and sustained bandwidth;
- Infinity Fabric contention;
- CXL switch and device contention;
- NVMe queue depth and page amplification;
- directory lookup, deduplication, and descriptor construction;
- cache-line and page locality of sparse rows;
- thread scheduling variance for I/O completion.

A tier that looks valid at median latency can still destroy p99 TPOT if several sockets or inference instances synchronize on the same device.

---

## 3. Session-local reuse beats static hot sets

SlackTier’s access characterization contains a useful cache-policy result. For agent traces, a per-layer LRU holding one million rows—about 264 MB, only 0.26% of a table—hits 74% of decode lookups. At five million rows, it reaches roughly 81–85%.

Static hot sets built offline achieve only 29–51%. Warming with unrelated repositories does not materially help. Chat traffic has lower reuse because many n-grams are first-seen.

The engineering implication is that the useful reuse domain is the active session, not a global corpus-level popularity distribution.

For CPU deployments:

- partition or tag cache state by tenant/session when interference is high;
- include prefill accesses when warming a session’s row cache;
- avoid spending local DRAM on a global static hot set unless measured traces show transferability;
- make cache admission aware of the consuming layer’s slack;
- charge duplicate rows only once when several requests in the same step need them.

A reasonable cache priority is:

[
	ext{priority}(r,ell)
=
rac{
Pr(	ext{reuse}_{r,ell}mid	ext{session})
cdot
max(0,T_{	ext{fetch},ell}-W_ell)
}{
	ext{bytes}(r)
}
]

Rows for long-window layers receive little value from caching even if frequently reused, because their misses are already hidden.

---

## 4. Freed host memory is most valuable as KV capacity

With both tables placed on four NVMe devices, SlackTier reduces the serving process’s host memory from 212 to 34 GiB while increasing mean TPOT by at most 1.4% over unmodified SGLang in the reported agent and chat replays.

With RDMA, one table copy in another node’s DRAM serves two inference instances without a GPU row cache. This is a meaningful deployment option for memory-heavy models: disaggregate immutable sparse state and share it, rather than replicating hundreds of gigabytes in every inference host.

The most valuable use of the released DRAM is not necessarily running another model replica. On a KV-constrained agent workload, reallocating it to host KV improves median throughput by 36% and reduces median p90 TTFT from 38 to 24 seconds at the same memory budget.

For a CPU inference server, this suggests a memory-budget exchange:

[
M_{	ext{host}}
=
M_{	ext{weights}}
+
M_{	ext{tables}}
+
M_{	ext{KV}}
+
M_{	ext{runtime}}
]

Moving a deadline-tolerant table to CXL or NVMe can raise (M_{	ext{KV}}), preserving prefixes and avoiding expensive recomputation. But the next paper shows why simply maximizing (M_{	ext{KV}}) is incomplete.

---

## 5. Prefix caching creates a metastable failure mode

Agent sessions are closed loops. Each turn appends model output and tool observations; the next request reuses the prior turn as a prefix. A cache hit requires prefill only for the newly appended tokens. A miss recomputes the entire accumulated context.

In the metastability experiments, a cached mid-session turn with a 21.6K-token context costs about 0.15 seconds of prefill, while a miss costs roughly 2 seconds. As sessions grow, that ratio worsens.

The feedback loop is:

1. A short arrival burst expands the active-session working set.
2. Prefixes are evicted because the KV pool no longer holds all active sessions.
3. Later turns miss and perform full-context prefill.
4. Service rate falls.
5. Sessions remain unfinished longer.
6. More sessions coexist, increasing live KV demand.
7. The cache keeps thrashing after arrivals return to normal.

In paired vLLM experiments with identical base arrivals, nine of twelve burst runs entered the degraded state while none of their no-burst twins did. The collapsed server remained near a 10% hit ratio, delivered about half the throughput, and accumulated backlog for the remaining 70 minutes.

This is a history-dependent operating point. A load can be sustainable when the cache is warm and unsustainable after a burst has destroyed locality.

### Why CPU offload is not a complete fix

The paper reports that adding a CPU offload tier delays collapse but does not prevent it. That is an important correction to capacity-first thinking.

A slower tier adds bytes, but it also changes hit cost. If retrieval or recomputation keeps response time high enough, the number of unfinished sessions can still exceed the combined tier’s effective working set. Capacity and service rate are coupled:

[
N
approx
lambda T(Z+R)
]

where (N) is the concurrent session population, (lambda) the session arrival rate, (T) turns per session, (Z) tool/think time, and (R) response time. Increasing (R) increases (N), which increases KV demand and can increase (R) again.

For EPYC/CXL serving, admission must therefore use **effective resident capacity under the latency objective**, not raw bytes across all tiers.

---

## 6. Recovery requires parking admitted sessions, not only blocking new ones

Once collapse occurs, stopping new admissions is insufficient because already-admitted sessions continue issuing turns and cycling through a cache too small for them. Each session may need to complete many expensive missed turns before the population naturally falls.

The paper’s strongest controller parks sessions at turn boundaries. Parked sessions issue no new inference requests, and their prefix may be evicted. The remaining active set fits the KV pool, begins hitting again, restores throughput, and drains. Parked sessions are later readmitted FIFO, ahead of new sessions.

The proposed budgeter tracks the conservative resident demand of admitted sessions:

[
D
=
sum_{sin A}
(L_s + o_s)
]

where (L_s) is the current context length and (o_s) the output-token reservation. It admits while demand fits below 75% of KV capacity and parks sessions when demand exceeds 95%. The hysteresis absorbs growth between turn boundaries.

The thresholds were fixed rather than tuned per workload. In the reported collapsed runs, capacity-based parking restored a 0.93–0.95 prefix hit ratio and throughput within six minutes. Admission-only controls recovered rarely and much more slowly.

### Practical control loop

For a production CPU or heterogeneous server, track demand in bytes rather than tokens when models, KV formats, or layer structures differ:

[
D_s
=
L_s
cdot
B_{	ext{KV/token},s}
+
R_s
+
H_s
]

where (R_s) is reserved growth for the next turn and (H_s) captures model-specific recurrent state.

Use tier-weighted capacity:

[
C_{	ext{effective}}
=
C_{	ext{local}}
+
alpha C_{	ext{remote}}
+
eta C_{	ext{CXL}}
+
gamma C_{	ext{NVMe}}
]

with (0 le gamma le eta le alpha le 1), calibrated from the service rate each tier supports under the TPOT/TTFT SLO. A byte in remote or storage-backed memory is not equivalent to a byte of local DRAM.

The controller should expose two modes:

- **prevention:** deny or queue new sessions before the active working set crosses the stable boundary;
- **recovery:** park admitted sessions at safe turn boundaries until the remaining working set fits comfortably.

These require different actions. Autoscaling or admission-only logic may prevent entry but cannot guarantee fast recovery.

---

## 7. Combined design for an EPYC inference stack

The two papers imply a joint memory and admission planner.

### Step 1: classify immutable model state by deadline

For each sparse table, expert shard, adapter, or overlay:

- record when its address becomes known;
- measure its p99 consumption deadline;
- measure tier service time under contention;
- place it in the slowest tier whose tail latency fits;
- allocate scarce local cache only to negative-slack layers.

### Step 2: allocate released DRAM to state with multiplicative reuse value

KV deserves priority when a miss causes full-context recomputation. Recurrent state or shared prefixes may have similar value. Allocate by avoided service time per byte, not solely by hit frequency.

### Step 3: admit sessions against effective capacity

Estimate current and near-term KV bytes for each active session. Include output reservations, branch fan-out, and compaction behavior. Keep a hysteresis margin for growth and allocator fragmentation.

### Step 4: prefetch during graph slack and tool time

Issue sparse immutable-state reads as soon as indices are known. For agent KV, use the queue-slack technique from Evolving Infra and tool-wait predictions where reliable. Treat remote-tier bandwidth as a scheduled resource.

### Step 5: recover by reducing the active working set

When prefix hit rate and throughput collapse together, do not merely add queueing at the front door. Park selected sessions at turn boundaries, preserve fairness, and restore the remaining set’s cache locality.

### Measurements worth adding

| Layer | Metrics |
|---|---|
| Immutable-state placement | per-layer deadline, p50/p95/p99 fetch time, exposed stall, cache value per byte |
| Host hierarchy | local/remote/CXL/NVMe bytes, queue depth, bandwidth contention, NUMA ownership |
| KV stability | active-session demand, effective capacity, hit ratio, recomputed tokens, evictions per turn |
| Control | admissions, parked sessions, recovery time, hysteresis crossings, fairness delay |
| User impact | TTFT/TPOT p50/p95/p99, task completion time, session abandonment |
| Failure detection | hit-rate slope, service-rate drop, unfinished-session growth, cache churn |

A useful early-warning invariant is:

[
rac{dN}{dt} > 0
quadlandquad
rac{dh}{dt} < 0
quadlandquad
D > eta C_{	ext{effective}}
]

where (N) is unfinished sessions, (h) prefix hit ratio, and (eta) the prevention threshold. This distinguishes ordinary queue growth from a cache-driven collapse.

---

## 8. Evidence quality and caveats

### Genuinely important

- SlackTier derives placement from measured per-layer deadlines and shared-device contention rather than generic “hot/cold” classification.
- The implementation stays inside SGLang’s decode CUDA graph and verifies bit-exact output against the original gather.
- The evaluation measures host-memory reduction, decode overhead, p99 step time, RDMA sharing, and downstream KV-cache value.
- The metastability paper uses paired experiments whose base arrivals are identical, isolating the effect of a short burst.
- The failure reproduces across vLLM and SGLang, Qwen3-8B and Qwen3-32B, synthetic closed-loop sessions and SWE-bench-shaped traces.
- The recovery study distinguishes preventing new work from reducing the already-active working set.
- The MI300X experiments make the cache-collapse result particularly relevant to AMD-oriented serving work.

### Treat cautiously

- Both papers are arXiv v1 preprints from the same two authors; no independent reproduction is yet available.
- SlackTier’s main deployment uses four H200 GPUs and high-end local NVMe. CPU-only inference will have different overlap windows and may expose more directory and dequantization cost.
- The SlackTier measurement runs are mostly short—30 seconds of warmup and 60 seconds of measurement—although the workload replays and configuration sweep are extensive.
- The 0.23 ms RDMA result depends on the tested fabric, load, and row volume. Multi-tenant tail latency could change the placement decision.
- Raw NVMe devices, O_DIRECT, pinned pages, and multiple I/O threads are more controlled than many production Kubernetes storage paths.
- The metastability paper’s main workload fixes turn shapes, output lengths, and think-time distributions. Real agents introduce branch fan-out, cancellation, compaction, and model-dependent output variance.
- The budgeter double-counts shared prefixes and uses conservative output reservations. This protects stability but may sacrifice utilization.
- Parking can increase per-session completion latency and raises fairness and timeout questions that the throughput recovery metric does not fully capture.
- Neither paper establishes a universally correct threshold. Effective capacity must be recalibrated when quantization, offload tier, scheduler, or SLO changes.

---

## Reading priority

1. **[Placing LLM Lookup Tables by Per-Layer Prefetch Deadlines](https://arxiv.org/abs/2611.13413)** — Read Sections 3–4 for access reuse and the deadline model, Section 5 for the CUDA-graph/I/O path, and Section 6 for cache allocation, NVMe, RDMA, and KV-capacity results.
2. **[Bursts That Outlast Themselves](https://arxiv.org/abs/2611.13412)** — Read Sections 3–4 for the controlled collapse and workload-balance condition, then Sections 5–6 for admission-only versus session-parking recovery.
3. **[Evolving Infra](https://arxiv.org/abs/2611.12304)** — Revisit its queue-overlapped KV fetching and branch-aware cache management. Today’s papers add tier deadlines and a stability controller around that architecture.
4. **[FlashOffloading](https://arxiv.org/abs/2611.07542)** — Compare its token-to-location directory and phase-specific KV paths with SlackTier’s row-to-slot directory and per-layer deadline placement.

## Bottom line

Yesterday’s briefing separated compute role from state ownership. Today’s results refine how that state should be placed and protected.

First, **not all large model state deserves DRAM**. If a lookup is known early and consumed late, NVMe or remote memory can be effectively free. Placement must be driven by the layer’s p99 prefetch deadline, and cache capacity should protect the layers with exposed misses rather than the globally hottest rows.

Second, **more KV capacity does not guarantee stability**. Prefix reuse changes service capacity so sharply that a brief burst can permanently move an agent server into a low-hit-rate operating regime. Recovery requires shrinking the active working set, not merely rejecting new sessions.

For CPU and EPYC inference, the immediate design target is a deadline-aware tier planner coupled to a byte-based session budgeter: move immutable sparse state to the slowest tier that meets its consumption deadline, spend local DRAM on high-value KV, and keep the admitted session working set below the capacity that can satisfy the latency SLO.