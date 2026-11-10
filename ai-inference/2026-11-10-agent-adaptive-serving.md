# AI inference systems briefing — November 10, 2026

## Executive assessment

The substantive new item since yesterday’s [FlashOffloading briefing](https://github.com/rahuljauhari3/interview-prep/blob/main/ai-inference/2026-11-09-phase-specialized-kv-offloading.md) is **Evolving Infra**, a serving design that makes the prefill/decode split an online control variable without tying KV-cache ownership to the current role of a node.

This is important because it attacks a systems-level coupling that fixed-role deployments usually accept: a node’s execution phase, scheduling loop, KV index, and cached state are treated as one unit. Changing the phase then requires either restarting the worker or abandoning useful KV. Evolving Infra separates those concerns. Every worker can execute either phase, while a distributed host-memory pool and directory keep KV reachable across role changes.

The paper is GPU-based, but its architecture is directly relevant to CPU inference. On a multisocket EPYC server, the analogous problem is not merely “how many cores run prefill versus decode?” It is whether thread teams, packed weights, allocator arenas, and KV pages can change execution roles without causing NUMA migration, cold starts, or loss of state locality.

**Assessment:** genuinely important systems work. The large headline speedups are partly a consequence of a weak dynamic baseline with expensive worker relaunches, so they should not be treated as portable. The enduring contribution is the decoupling of compute role from state ownership, combined with a planner that models actual uncached work and an eviction policy informed by the agent’s context lifecycle.

**Primary source:** [Evolving Infra: Agent-Adaptive LLM Serving at Scale](https://arxiv.org/abs/2611.12304), submitted November 9, 2026 ([PDF](https://arxiv.org/pdf/2611.12304)).

---

## 1. What changed: prefill/decode roles become elastic without invalidating KV

Conventional disaggregated serving assigns workers to prefill or decode at deployment time. Even systems with dynamic rebalancing commonly retain phase-specific engines and local KV metadata. That makes a role change expensive:

- Relaunching a worker reconstructs runtime state and removes capacity during the transition.
- Switching in place can strand or discard the KV indexed by the old phase.
- Routing for locality can overload the few prefill workers that own a popular prefix.
- Routing for load balance can force recomputation or a synchronous KV fetch.

Evolving Infra replaces phase-specific engines with a **symmetric engine** derived from SGLang. One scheduling loop supports both phases, and a Mooncake/RDMA-based transfer protocol can move KV between arbitrary workers. The router determines which requests a worker receives; changing its role does not require a process restart.

KV is stored separately:

- GPU memory is the L1 cache.
- Each node’s host DRAM is an L2 cache.
- The L2 caches form a cluster-wide pool addressable over RDMA.
- KV written by either phase is written through to host memory.
- A prefill worker can read a peer’s L2 directly, avoiding a separate staging store.
- The entire remote fetch is initiated while the request waits in the queue, rather than delaying prefill when execution begins.

The practical design rule is simple:

> Treat execution role as ephemeral and state ownership as durable.

Yesterday’s FlashOffloading result showed how to preserve exact long-context state in host memory while keeping a bounded GPU working set. Evolving Infra extends that separation across workers: state must remain discoverable and reusable even when the worker that produced it changes phase.

### Why it matters in practice

The authors report that a role change takes 6.2 seconds on average, including waiting for the worker’s running requests to finish while it continues serving them. Dynamo’s relaunch takes 754 seconds on average and removes the node from service. The dramatic gap is specific to the compared implementations, but it exposes a real operational issue: a planner is useless if applying its decision costs minutes or destroys the cache benefit it is trying to preserve.

For CPU inference, the transition cost has a different shape but the same structure. A nominally instant reassignment can still trigger:

- first-touch faults that relocate pages to the wrong NUMA node;
- eviction of packed-weight panels and JIT-generated kernels;
- thread-pool teardown and repinning;
- allocator-arena imbalance;
- loss of socket-local prefix-cache indexes;
- cache and TLB cold starts;
- remote DRAM traffic when KV ownership does not follow the new thread team.

A CPU implementation should therefore keep a symmetric runtime resident on every socket or worker group and switch only the admission policy. Packed weights, kernels, and thread teams should remain warm. KV ownership should be tracked independently of which group currently executes prefill or decode.

---

## 2. The planner models useful work, not prompt length or queue spikes

The second substantive contribution is the planner’s input signal.

For each request, Evolving Infra separates:

- uncached input tokens;
- cached prefix tokens;
- output tokens;
- accepted tokens per speculative-decoding iteration.

This distinction matters because full prompt length is a poor proxy for prefill work in agentic workloads. A 200,000-token request with a 198,000-token cache hit is not equivalent to a 200,000-token cold request. Likewise, a burst of newly spawned sub-agents increases queue depth but may not represent a persistent change in the cluster’s prefill/decode balance.

The paper fits per-node iteration-time models online. Its prefill model accounts for uncached tokens, causal attention work against the cached prefix, and host-memory restore cost. Its decode model captures the mostly bandwidth-bound cost of dense weights, routed MoE experts, and active requests. Coefficients are learned with nonnegative least squares during a collocated warmup; the controlled measurements report less than 7% mean error outside fitted settings.

Every ten-minute planning window, the planner predicts the throughput of candidate prefill/decode splits:

[
Theta(P,D)=minleft(sum_{nin P}mu_{P,n}, sum_{nin D}mu_{D,n}ight)
]

where each node’s service rate is evaluated on the average request work observed in the window. A candidate split is accepted only when its predicted gain exceeds twice the standard error across window slices. This hysteresis prevents a short queue burst from causing an expensive role oscillation.

### CPU translation

A ZenDNN-style planner should use measured service demand after cache reuse, not request counts or raw sequence lengths. At minimum, maintain separate online models for:

| Phase | Useful explanatory variables |
|---|---|
| Prefill | uncached tokens, cached-prefix length, attention pattern, batch/chunk size, quantization format, local/remote KV bytes |
| Decode | active sequences, accepted speculative tokens, MoE expert working set, weight bytes, KV bytes, memory-channel saturation |
| Transition | requests to drain, page migrations, packed-weight warmup, thread repinning, cache-directory changes |

On CPU, the model should be fitted per socket or NUMA domain, not per server. Two nominally identical sockets may expose different service rates because of memory placement, CXL attachment, NIC locality, or concurrent tenants.

A useful objective is not simply maximum predicted throughput. For production serving, optimize a constrained objective such as:

[
max  	ext{throughput}
quad 	ext{subject to} quad
p95(mathrm{TTFT}) leq S_P, 
p95(mathrm{TPOT}) leq S_D
]

The paper itself shows why this matters: at high concurrency on the decode-heavy Putnam workload, its throughput-oriented planner kept only one or two prefill nodes and increased P95 TTFT relative to Dynamo in some settings, even while mean end-to-end latency improved. An unconstrained throughput planner can make the wrong decision for an interactive SLO.

---

## 3. Agent context events are stronger cache signals than recency

The most transferable idea may be the **branch-reference directory**.

Agent KV reuse is determined by context lifecycle, not ordinary LRU recency. A branch can wait several minutes for a tool or sub-agent and then reuse almost its entire prefix. Conversely, compaction can make a recently accessed history permanently dead.

Evolving Infra tracks how many active branches reference each KV block copy. The directory changes as the harness reports:

- branch creation and forking;
- appended context;
- compaction and context-version replacement;
- sub-agent completion;
- branch termination;
- KV transfer between workers.

Compaction releases references to the replaced history. A completed branch releases all of its references. Transfers update the counts so duplicate copies do not appear equally valuable. Eviction first removes blocks with zero active references, then falls back to recency among still-referenced blocks.

This is more than an eviction tweak. It turns the agent harness into a source of storage-lifetime information, similar to compiler liveness. The serving layer no longer has to infer future reuse exclusively from block accesses.

### Practical implication for CPU and tiered memory

Expose explicit context-lifecycle events to the cache manager:

[
	exttt{fork(branch)},quad
	exttt{compact(branch, old_version, new_version)},quad
	exttt{finish(branch)},quad
	exttt{transfer(block, source, destination)}
]

Then maintain, for every block copy:

- active branch-reference count;
- context version;
- phase copies that still need it;
- NUMA location and retrieval cost;
- duplicate-copy count;
- last access time as a fallback signal.

For a multisocket or CXL hierarchy, “referenced” is necessary but insufficient. Eviction and placement should be cost-weighted:

[
	ext{value}(b)
approx
Pr(	ext{reuse}_b)
	imes
left(
T_{	ext{recompute},b}
-
T_{	ext{retrieve},b}
ight)
-
C_{	ext{capacity},b}
]

The branch count improves the reuse probability. NUMA locality and tier bandwidth determine retrieval cost. Prefix length and kernel throughput determine recomputation cost. This avoids preserving a remotely located short prefix whose retrieval is slower than recomputation while evicting an expensive long prefix that is idle but still live.

---

## 4. Queue time becomes useful transfer slack

Cache-aware routers usually face a bad choice: send work to the worker holding the prefix and accept a long queue, or choose an idle worker and pay recomputation or transfer latency.

Evolving Infra overlaps the remote host-memory fetch with time the request would have spent queued. Its router predicts TTFT for each prefill worker from:

1. the remaining time of the active batch;
2. the service time of queued requests;
3. the new request’s service time;
4. any fetch time not hidden by the queue;
5. the capacity cost of creating another KV copy.

It sends the request to the worker with the lowest predicted TTFT, then begins fetching immediately. The ablation reports that asynchronous fetching alone raises throughput by 12% and cuts prefill-worker idle time from 23.6% to 14.4%.

For CPU clusters, the same technique applies to:

- prefetching KV from a remote socket into local DRAM;
- pulling blocks from a CXL tier;
- warming packed weights or expert shards;
- migrating prefix metadata before a thread team becomes available.

The scheduler should treat expected queue delay as a prefetch budget. A remote fetch is not inherently harmful if its completion is hidden behind work already ahead of the request. What matters is the unhidden tail:

[
T_{	ext{visible fetch}}
=
max(0, T_{	ext{fetch}}-T_{	ext{queue slack}})
]

This also implies that routing, admission control, and data movement cannot be tuned independently. Queue estimates should launch placement actions, not merely choose a worker.

---

## 5. Evaluation: strong scale, but interpret the headline carefully

The primary evaluation uses 36 GB300 GPUs across nine four-GPU nodes, with GLM-5.2 or Kimi K2.6, NVFP4 weights, FP8 KV, speculative decoding, and a host-memory tier sized for four times as many tokens as GPU memory. Workloads cover decode-heavy math agents, prefill-heavy multimodal computer-use agents, and replayed Claude Code traces containing forks, compactions, sub-agents, and long pauses.

Reported results include:

- 1.10–2.73× Dynamo’s throughput across stationary workloads;
- 81–95% less lost KV reuse;
- 1.15–4.12× lower mean end-to-end latency;
- up to 13.92× lower P95 TTFT;
- 2.14× Dynamo’s throughput and 1.28× the best static split under a three-hour shift from math to computer-use agents;
- stable latency in tests up to 256 GPUs, with larger-scale results partly simulated.

The paper also evaluates heterogeneous parallelism: TP8 for prefill and attention data parallelism for decode. Keeping the union of required attention weights costs about 10 GB per B200 GPU, reported as 6% of memory, but allows switches without moving weights.

### What is genuinely convincing

- The paper identifies three concrete couplings—engine role, KV reachability, and agent context lifecycle—and addresses them together.
- The workloads exercise qualitatively different phase ratios rather than one synthetic request distribution.
- The evaluation includes a shifting composition, ablations, online model convergence, tail latency, and role-switch overhead.
- The reference-count semantics handle compaction and duplicate copies, cases that ordinary “keep waiting agents’ KV” policies miss.

### What should not be generalized

- The largest speedups are against Dynamo’s relaunch-based adaptation. A CPU runtime with persistent workers and cheap thread reassignment starts from a much stronger transition baseline.
- The main 36-GPU comparison gives host DRAM four times the GPU KV capacity and assumes fast RDMA. PCIe, NIC, NUMA, and CXL topology can change the break-even point materially.
- The online models are fitted after a 30-minute warmup and then largely frozen. Rapid hardware interference, DVFS, co-tenancy, or memory-bandwidth contention could invalidate coefficients faster on CPU.
- The planner uses ten-minute windows. That is appropriate for the evaluated workload shifts but may react too slowly to autoscaling events or too quickly for transitions that cause substantial page migration.
- The paper optimizes cluster throughput and demonstrates cases where P95 TTFT worsens. Production adoption needs explicit SLO constraints.
- Code availability is not established from the arXiv page, so the implementation claims are not yet independently reproducible.

The result should therefore be read as architectural evidence, not as a portable 4–14× performance promise.

---

## 6. Recommended prototype for an EPYC inference stack

The highest-value experiment is a two-socket role-elastic server that preserves socket-local state across phase changes.

### Runtime structure

1. Keep a symmetric prefill/decode worker resident per NUMA domain.
2. Preload both phases’ packed weights and kernels where capacity allows.
3. Maintain a global block directory, but record physical NUMA ownership per copy.
4. Emit branch lifecycle events from the agent gateway.
5. Fetch remote KV during queue slack, with bandwidth admission control.
6. Change the prefill/decode core split only when the predicted gain exceeds transition cost and an SLO guard remains satisfied.

### Measurements

| Category | Required measurements |
|---|---|
| Phase service | uncached tokens/s, decode tokens/s, batch occupancy, accepted speculative tokens |
| Memory | local and remote DRAM bytes, CXL bytes, KV restore bytes, duplicate capacity |
| Transition | drain time, repinning time, page faults/migrations, packed-weight warmup |
| Cache | live-reference hit rate, zero-reference eviction bytes, recomputation avoided |
| User impact | mean and P95/P99 TTFT, TPOT, end-to-end latency, deadline misses |
| Stability | switches/hour, reverted switches, model-prediction error, hysteresis margin |

Compare four configurations:

1. fixed phase/core partition with LRU;
2. elastic partition with LRU;
3. fixed partition with branch-aware eviction and asynchronous fetch;
4. fully elastic, branch-aware, queue-overlapped design.

That factorial structure will show whether the gain comes from role elasticity, better KV semantics, or simply hiding transfers. It also prevents a planner improvement from masking a regression in remote-memory traffic.

## Bottom line

Evolving Infra moves disaggregated inference from **static phase ownership with opportunistic caching** toward **elastic compute roles over durable, semantically managed state**.

For CPU inference, the lesson is not to reproduce its GPU stack literally. It is to make three separations explicit:

- compute role from worker lifetime;
- KV ownership from compute placement;
- cache value from access recency.

Once those boundaries exist, prefill/decode allocation can become a stable feedback-control problem rather than a deployment-time choice. The next serious CPU serving architecture should make role switching cheap, preserve NUMA-aware state, use agent lifecycle as cache-liveness information, and spend queueing slack on data placement before execution begins.