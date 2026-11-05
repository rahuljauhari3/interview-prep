# AI inference systems briefing — November 5, 2026

## Executive take

The most consequential new systems work since yesterday’s CPU–GPU scheduling briefing is **ROCm InferaSim**, an AMD framework that replaces exhaustive serving-configuration benchmarking with a small number of measured anchors, analytical scaling laws, trace-driven simulation, and automated search.

[ROCm InferaSim: Cut LLM Serving Trial Costs and Deploy the Best Configuration](https://arxiv.org/abs/2611.04032), submitted November 2, addresses a problem that increasingly dominates inference deployment: the fastest engine, precision, parallel layout, scheduler, speculative decoder, KV policy, and prefill/decode topology cannot be chosen independently. Exhaustively measuring the paper’s candidate space would require roughly 25,000 GPU-hours.

The framework measures each distinct **kernel set** once—on at most one node—and transfers that measurement to other batch sizes, context lengths, parallel widths, and traffic patterns that execute the same kernels. Eleven anchors took 9.1 GPU-hours and covered 226 fixed-length configurations that required 204 GPU-hours to measure directly. Across 172 filtered configurations, median TPOT error is 8.1%; agentic-workload throughput is within 8%; and 159 disaggregated deployments are predicted within 5.5% throughput error despite none being used as an anchor.

The important insight is broader than ROCm or GPUs: **measure the implementation-specific constant once, then model only the work that actually scales**. Pure roofline models fail because kernel launches, synchronization, sampling, scheduling, and graph overhead do not shrink when more devices or cores are added.

For CPU inference, this is a strong blueprint for replacing brute-force ZenDNN/oneDNN tuning with a topology-aware model calibrated by a small number of whole-step measurements.

**Assessment:** genuinely important deployment methodology with unusually broad validation. It is currently demonstrated only on MI355X and depends on manually declaring which configurations share a kernel set; it should shortlist configurations for hardware confirmation, not replace final measurement.

## 1. What changed: the profiling unit becomes a kernel set

Serving configuration choices fall into two categories.

| Choice type | Examples | Measurement implication |
|---|---|---|
| Anchor-defining | Model checkpoint, GPU type, engine, precision, attention/MoE backend, graph-capture mode, speculative method | Can change which kernels execute; requires a new anchor |
| Transferable | Batch and concurrency, prompt/context length, TP/EP/PP width, attention-DP width, replica count, arrival pattern, host KV caching | Changes work per device while preserving the kernel set; predicted from an existing anchor |

An anchor is a short harvest of complete engine steps, not a collection of isolated operator microbenchmarks. It contains:

- a decode sweep across graph-captured batch sizes;
- a prefill sweep over five prompt lengths;
- a fitted prefill model (T(n)=F+c_1n+c_2n^2);
- optional multi-prompt packing probes;
- an execution signature covering engine version, real-weight loading, prefix-cache state, and captured shapes.

This choice is important. Operator-level profiling becomes stale whenever fusion, graph capture, scheduling, or kernel selection changes. Whole-step anchors absorb those effects while the analytical model determines how they change with workload size and parallelism.

## 2. The fixed per-step term is the core modeling insight

A naïve predictor often assumes that doubling tensor-parallel width halves step time:

[
s_{2w}(b)approxrac{s_w(b)}{2}
]

That incorrectly divides work that does not scale: launch overhead, scheduler work, sampling, synchronization, graph replay, and small serial kernels.

ROCm InferaSim instead separates the fixed term (f):

[
s_{w'}(b)
=
f+
left(s_w(b)-fight)ho_{wightarrow w'}(b)
]

where (s_w(b)) is the anchored step time at width (w), batch (b), and (ho) scales only the divisible remainder.

On a DeepSeek-V4-Pro TP4-to-TP8 transfer:

- naïvely halving the TP4 step produces 44.1% median error;
- subtracting a purely analytical predicted drop produces 18.5% error;
- preserving the measured fixed term reduces median error to 0.9%.

This is directly relevant to CPU scaling. Increasing threads, CCDs, or sockets does not divide:

- executor dispatch;
- thread-pool wakeup;
- barrier and reduction setup;
- sampling and token bookkeeping;
- allocator work;
- small elementwise operators;
- cache and NUMA coordination;
- request admission and batching.

A CPU step model should therefore be:

[
T(p,b,L)
=
T_{mathrm{fixed}}(b)
+
T_{mathrm{GEMM}}(p,b)
+
T_{mathrm{attention}}(p,b,L)
+
T_{mathrm{memory}}(p,b,L)
+
T_{mathrm{NUMA}}(p)
+
T_{mathrm{collective}}(p)
]

and scale only the terms affected by the chosen resource width (p).

## 3. Reduce–measure–restore makes multi-node search practical

Large deployments may not fit on one node, but measuring every multi-node candidate defeats the purpose. ROCm InferaSim reduces the target to a single-node equivalent, measures or anchors that reduced form, then analytically restores:

- wider tensor parallelism;
- additional pipeline stages;
- cross-node communication;
- disaggregated prefill and decode pools;
- KV-cache transfer;
- host-memory KV placement;
- replica count and queueing.

The reported validation includes:

- 90 TP8 deployments predicted from TP4 anchors, with 9.7% median throughput error;
- 24 two-node pipeline deployments;
- 159 prefill/decode-disaggregated deployments across vLLM+NIXL, SGLang+MoRI, and ATOM+Mooncake;
- no disaggregated deployment used as an anchor.

Across those 159 disaggregated layouts, median throughput error is 5.5% and interactivity error is 5.3%. TTFT is harder: median error is 16.9%, reflecting host processing, tokenization, queueing, and KV handoff that are not captured by GPU step time alone.

That distinction matters. Throughput can be modeled accurately while admission latency remains wrong. A deployment search must maintain separate confidence bounds for TTFT, TPOT, ITL, and throughput rather than claim one universal error figure.

## 4. Trace replay turns step prediction into serving prediction

A fast step model is not enough for agentic traffic. Multi-turn agents pause between turns, reuse long prefixes, create dependencies between requests, and change batching opportunities.

ROCm InferaSim embeds its cost model in a discrete-event simulator that represents:

- request arrivals;
- continuous batching;
- queues;
- prefix reuse;
- session think time;
- prefill and decode phases;
- disaggregated KV movement;
- replicas and routing.

On the evaluated multi-turn coding workload, predicted throughput is within 8%. For p99 latency across 88 sufficiently large runs, median errors are:

| Metric | Median p99 prediction error |
|---|---:|
| TPOT | 10.7% |
| End-to-end latency | 9.8% |
| ITL | 13.8% |

Repeated physical runs themselves vary by 1–4% on those p99 metrics, so the simulator is not at noise-floor accuracy, but it is useful for pruning a large search space.

The correct operating model is:

1. predict thousands of candidates cheaply;
2. reject infeasible memory and SLO combinations;
3. return a short list;
4. confirm that short list on hardware.

The paper’s five-candidate short list contains the measured best deployment in 21 of 23 feasible model/tier slots and a configuration within 12% of the best in 22 slots.

## 5. Why this matters for ZenDNN and EPYC

CPU inference tuning has an equally large combinatorial space:

- model format and quantization;
- ZenDNN versus oneDNN kernels;
- ISA path and packed-weight format;
- thread count and affinity;
- CCD, socket, and NUMA placement;
- inter-op versus intra-op parallelism;
- batch and prompt bucket;
- KV precision and layout;
- weight replication versus sharding;
- persistent versus general execution plan;
- replica count and routing;
- CPU-only versus CPU/GPU collaboration.

Benchmarking every combination is expensive and often misleading because engine startup, packing, warmup, allocator state, frequency, and NUMA first-touch contaminate comparisons.

A CPU version of the anchor idea should define a kernel-set signature such as:

[
K=
H(
	ext{model revision},
	ext{quantization},
	ext{backend version},
	ext{ISA},
	ext{fusion plan},
	ext{packed layout},
	ext{attention backend},
	ext{execution graph}
)
]

Thread count, batch, context length, and socket count may be transferable only while this signature remains unchanged. If a larger batch selects a different GEMM kernel or wider threading changes the reduction implementation, the configuration needs another anchor.

### CPU anchor contents

A useful CPU harvest would record:

- batch-one and batched decode-step latency;
- prefill latency across several prompt lengths;
- packed-weight preparation and cold-start cost;
- local and remote DRAM bytes;
- memory bandwidth and LLC-miss rate;
- fixed executor, barrier, sampling, and allocator time;
- kernel IDs and shape buckets;
- frequency and power state;
- NUMA topology and first-touch placement;
- p50/p99 step variability.

Unlike a GPU anchor, it should be topology-specific. “Two sockets” is not enough; the anchor should include memory policy, thread affinity, NPS mode, SMT state, and weight/KV placement.

## 6. A practical EPYC transfer law

For scaling within one socket, separate compute, bandwidth, and fixed overhead:

[
T_{p'} =
F+
C_pcdotho_C(p,p')
+
M_pcdotho_M(p,p')
]

where:

- (F) is non-scaling runtime cost;
- (C_p) is compute-sensitive work;
- (M_p) is memory-sensitive work;
- (ho_C) is measured compute scaling;
- (ho_M) is bounded by sustainable DRAM bandwidth.

For cross-socket scaling, add an explicit topology term:

[
T_{2s}
=
F+
C_{1s}ho_C+
M_{1s}ho_M+
T_{mathrm{remote}}+
T_{mathrm{reduce}}
]

Do not infer the second-socket result from core count alone. Measure at least one half-width or cross-socket anchor so the model learns whether the workload is limited by DRAM channels, remote pages, barriers, or reductions.

The most useful output is not a single “optimal” configuration. It is a Pareto frontier over:

- p99 TTFT;
- p99 TPOT;
- throughput;
- energy per token;
- DRAM capacity;
- node count;
- cost.

## 7. Recommended validation workflow

Build a CPU inference planner in four layers.

### Layer 1: correctness and feasibility

Reject configurations that violate:

- model and KV capacity;
- socket-local memory budgets;
- supported kernel shapes;
- latency hard limits;
- precision/accuracy requirements;
- topology and collective constraints.

### Layer 2: anchor transfer

Use complete-step anchors to predict:

- unseen batches;
- prompt and context buckets;
- thread and CCD widths;
- socket counts;
- replica counts.

Flag extrapolations beyond one measured doubling or outside the anchor’s shape range.

### Layer 3: traffic simulation

Replay representative traces with:

- arrival bursts;
- multi-turn prefix reuse;
- cancellations;
- think time;
- mixed prompt/output distributions;
- tenant skew;
- cache eviction;
- load-balancer policy.

### Layer 4: hardware confirmation

Measure the top five candidates and update the model with the result. The planner should expose prediction intervals and refuse to rank candidates whose expected difference is smaller than model uncertainty.

## 8. Evidence quality and caveats

### Genuinely important

- The paper distinguishes kernel-changing choices from work-scaling choices instead of applying one generic roofline model.
- Whole-step anchors retain fixed framework and scheduling costs that analytical models usually miss.
- Evaluation spans vLLM, SGLang, and AMD’s ATOM engine, plus fixed-length, agentic, pipeline, and disaggregated serving.
- Tail latency is evaluated rather than reporting only mean throughput.
- The system uses prediction to shortlist candidates and checks those selections against measured deployments.
- Negative boundaries are explicit: anchors are rejected across GPU types or engines and flagged outside the measured range.

### Treat cautiously

- Every hardware measurement is on AMD Instinct MI355X. The method is intended to transfer to other accelerators after new anchors, but that portability is not experimentally demonstrated.
- All authors are from AMD, so independent reproduction is important.
- The framework trusts configuration metadata to decide whether two deployments share a kernel set; it does not verify equivalence from kernel traces.
- A new engine build or kernel-library change requires a new harvest.
- Model error is reported after filtering unstable or invalid runs; operators should inspect filter criteria before adopting headline accuracy.
- Median error can hide bad outliers. Capacity planning near an SLO boundary requires conservative confidence intervals.
- Disaggregated TTFT error is materially higher than throughput error.
- The search result is near-optimal only within the candidate set covered by anchors; it cannot discover an unmodeled engine, kernel, or serving architecture.
- No public implementation repository is linked on the arXiv page at the time of this briefing.

## Secondary development: Kinference moves serving policy into a Kubernetes control plane

The vLLM team published [Kinference](https://vllm.ai/blog/introducing-kinference) on November 4 as an open-source control plane for vLLM on Kubernetes, covering token-rate limiting, tenant usage and cost tracking, load balancing, and inference policies.

This is strategically aligned with ROCm InferaSim: one system predicts which deployment should run; the other provides a fleet-level place to enforce routing and policy. However, Kinference is an initial project announcement rather than a rigorous performance study. Treat it as an integration signal, not evidence of a new scheduling optimum.

The useful architectural direction is a closed loop:

[
	ext{metrics and traces}
ightarrow
	ext{performance model}
ightarrow
	ext{candidate plan}
ightarrow
	ext{control-plane rollout}
ightarrow
	ext{measured correction}
]

For CPU fleets, this would allow topology-aware replicas, batch policies, and quantization variants to be selected from workload evidence rather than static deployment manifests.

## Reading priority

1. **[ROCm InferaSim paper](https://arxiv.org/abs/2611.04032)** — Read §3.1 for anchors and transfer laws, §3.2–§3.5 for reduce–measure–restore and simulation, §5 for validation, and Appendices D–E for search quality and failure boundaries.
2. **[Kinference announcement](https://vllm.ai/blog/introducing-kinference)** — Read as control-plane context, not as a benchmark.
3. **[Vidur](https://www.usenix.org/conference/atc24/presentation/agrawal)** — Useful contrast: operator-profile simulation versus InferaSim’s whole-step measured anchors.
4. **[Yesterday’s CPU–GPU transfer-hiding paper](https://arxiv.org/abs/2611.01393)** — A natural target for anchor-based planning because its static schedule fails when memory pressure changes runtime timing.

## Bottom line

Yesterday’s briefing showed that a static CPU–GPU overlap schedule can collapse when measured timing no longer matches runtime conditions. ROCm InferaSim addresses the broader version of that problem: serving configuration should be searched using a model grounded in the actual engine’s measured step behavior, then validated on a small hardware shortlist.

For CPU inference engineering, the most transferable lesson is to stop treating every batch, context length, thread count, and topology as an unrelated benchmark. Group configurations by the kernels and execution graph they actually run. Measure each group sparsely, preserve fixed runtime costs, model only the work that scales, and replay real traffic before selecting a deployment.

That approach will not eliminate benchmarking. It can make benchmarking targeted—turning thousands of brute-force trials into a handful of anchors and a short, defensible list of configurations to verify.
