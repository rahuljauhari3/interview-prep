# AI inference systems briefing — November 19, 2026

## Executive assessment

The strongest new result since yesterday is not a kernel or cache-layout optimization. It is a production dispatch design that treats inference capacity as a moving, partially observable ceiling.

Meta’s new [Cap-and-Track](https://arxiv.org/abs/2611.24891) paper reports that a refusal-driven traffic shaper served one classifier workload with 43% fewer H100s than its queued fixed-rate predecessor. The more general contribution is a control boundary: the saturated inference resource makes a fast admission decision, while an upstream dispatcher learns current capacity solely from refusals and successful probes.

That design transfers directly to CPU inference fleets. CPU capacity shifts with request shape, NUMA locality, memory-bandwidth contention, frequency throttling, model mix, and colocated work. A fixed requests-per-second limit—or a predictor trained on yesterday’s input/output distribution—will become stale. The backend is often the only component that can observe whether another request will violate its batch, memory, or latency envelope.

A second paper, [KV-LTM](https://arxiv.org/abs/2611.21998), proposes retaining per-session KV state as a long-term prefill accelerator. It is conceptually useful and unusually explicit about storage costs, but it contains no implementation or measured performance results. It belongs on the research agenda, not in a production plan yet.

## 1. Cap-and-Track: let the saturated resource expose its admission ceiling

**Importance: High for large-scale inference scheduling and fleet efficiency**

[Cap-and-Track: Optimizing LLM Inference Request Dispatch Across Shifting Capacity](https://arxiv.org/abs/2611.24891), submitted November 18, addresses asynchronous inference workloads whose true service demand is not known at admission. Output length varies, request sizes drift, fleets resize, and shared provider capacity can disappear during backfill or failures. Under those conditions, a centrally configured dispatch rate is either too conservative or eventually overloads the backend.

The system separates three responsibilities:

| Component | Responsibility | Important property |
|---|---|---|
| Durable priority queue | Hold work outside execution | Absorbs overload without occupying scarce inference resources |
| Cap | Fail-fast admission check at each saturated replica | Refuses before a request waits in a long engine queue |
| Track | Stateful upstream controller | Probes upward after success and backs off after refusals |

The backend refusal is the capacity signal. Track does not require predicted generation length, an explicit service-time model, or a fleet-wide rate that operators must continually retune.

That is a meaningful departure from conventional rate limiting. A token-based predictor may be useful when request cost is known, but generative output length is unknown and the realized service rate changes with batching, cache hits, speculative acceptance, and hardware state. Cap-and-Track instead closes the loop around observed admission feasibility.

### Reported evidence

The paper is unusually valuable because it includes concurrent production evidence rather than only trace simulation:

- In a same-traffic production A/B, per-replica Cap plus Track served a classifier on **43% fewer H100s** than the queued fixed-rate predecessor.
- In steady-capacity real-engine replay, Track ran within **1% of the goodput** of a fixed rate deliberately tuned above measured capacity, while producing about **one quarter of that configuration’s refusal share**.
- The deployed shaper carries more than **300 million online requests** and hundreds of terabytes of media per day.
- During one backfill event, it handled **1.1 billion requests in 24 hours**.

The tradeoff is explicit: lower-priority work waited longer during backfill peaks. This is not free capacity. The system converts spare deadline slack into fleet savings, so fleet sizing becomes a policy question about how long low-priority jobs may wait.

### Why it matters for CPU and EPYC inference

A CPU serving replica can define its Cap around the resource that actually determines its service envelope. Depending on the workload, that might be:

- estimated socket-local KV and workspace capacity;
- active decode sequences per NUMA node;
- memory-bandwidth headroom;
- the number of runnable thread teams before barrier and cache contention dominate;
- an upper bound on predicted p99 queueing plus service time;
- admission to a model-specific batch whose shape still has positive marginal throughput.

The critical implementation detail is to refuse early. Once excess work enters the engine, it consumes queue memory, complicates cancellation, distorts continuous batching, and makes observed latency a mixture of queue delay and execution capacity. Keep only enough queued work inside the runtime to form efficient batches; keep the durable backlog outside.

For a heterogeneous CPU fleet, Track should normally operate per capacity class or replica pool rather than globally. A dual-socket EPYC node with local weight replicas, a CXL-backed node, and a smaller single-socket node do not share one stable service ceiling. Their refusal streams should drive separate controllers, with routing decisions layered above them.

### A practical prototype

A useful CPU implementation would expose a structured refusal rather than a generic overload error:

```text
reason: LOCAL_KV_CAPACITY | MEMORY_BW | BATCH_SHAPE | DEADLINE_RISK
retry_after_hint: optional
capacity_epoch: monotonically increasing
replica_class: hardware and model configuration
```

Track should not need these fields to remain stable—the binary accept/refuse signal is the control primitive—but the metadata helps diagnose oscillation and partition controllers correctly.

Evaluate it against:

1. fixed request-rate dispatch;
2. fixed token-rate dispatch using predicted output length;
3. queue-length-based autoscaling;
4. refusal-driven Track.

Replay workload shifts in prompt length, output length, prefix-hit rate, model mix, and memory tier. Measure goodput under an SLO, refusal fraction, p50/p99 external queue delay, engine-resident queue depth, controller convergence time, and oscillation after capacity loss or restoration.

For CPU inference, also record DRAM bandwidth, remote-NUMA traffic, effective frequency, and batch-shape distribution. A controller that maximizes accepted requests while forcing remote memory or sustained downclocking may reduce actual goodput.

### Assessment and limitations

This is a substantive systems contribution. The production scale, concurrent A/B, explicit control mechanism, and discussion of operational tradeoffs put it well above a promotional autoscaling claim.

The applicability boundary matters:

- The reported production workloads are asynchronous post and live-video understanding, not interactive token streaming.
- Durable queuing assumes some requests have meaningful deadline slack.
- Refusal feedback can lag or oscillate when replicas differ sharply or capacity changes faster than the controller reacts.
- A 43% GPU reduction from one classifier workload should not be generalized as an expected saving for autoregressive LLM serving.

For interactive generation, the design is most applicable before a session begins or when routing resumable background work. Once streaming starts, migration and preemption costs require a different control loop.

**Read:** [Cap-and-Track: Optimizing LLM Inference Request Dispatch Across Shifting Capacity](https://arxiv.org/abs/2611.24891)

## 2. KV-LTM: persistent KV as stored execution state

**Importance: Medium as a research direction; low as current evidence**

[KV-LTM: Persistent Per-Session KV Caches as Long-Term Memory for Personal Assistants](https://arxiv.org/abs/2611.21998), announced November 18, proposes storing each session’s raw text together with the native KV state produced by a fixed reader model. On a later query, the service restores selected sessions and appends the question, avoiding repeated prefill of the stored text.

The paper makes two sound architectural distinctions:

- **Text remains the memory of record.** If cached state is invalid, missing, or incompatible, the system can reconstruct it.
- **KV is an acceleration artifact.** It trades storage and transfer for avoided prefill; it is not a replacement for retrieval or indexing.

This is a natural extension of prefix caching across much longer time scales, but the economics are severe. The paper estimates roughly **31–328 KB per token** for BF16 full-attention GQA and latent-cached MLA configurations. At its declared 30,000-token-per-day scenario, that becomes approximately **0.34–3.6 TB per user-year**, versus about 49 MB for raw text. A pinned hybrid KDA+MLA case is smaller per token but also carries a fixed recurrent state per session.

### Practical implications

The proposal is useful mainly because it forces several deployment contracts to become explicit:

1. Cache identity must include the exact model, tokenizer, prompt template, positions, RoPE configuration, attention mask, precision, and relevant runtime state.
2. Model upgrades invalidate or require migration of stored native state.
3. Storage tier and link bandwidth determine whether loading KV is cheaper than recomputing prefill.
4. Retention should be based on expected future reuse value, not simply session age.
5. Encryption, integrity, deletion, and tenant isolation requirements apply to activation state as well as text.

For CPU deployments, persistent KV could be more plausible for small, stable reader models when state is restored from local NVMe into DRAM and repeated prefills are expensive. It is less attractive when model versions turn over rapidly, sessions are rarely revisited, or stored state approaches hundreds of kilobytes per token.

The right decision rule is approximately:

[
P(	ext{reuse}) 	imes C_{	ext{avoided prefill}}
>
C_{	ext{write}} + C_{	ext{retention}} + P(	ext{reuse}) 	imes C_{	ext{restore}}
]

That calculation should be performed per model architecture, session class, and storage tier. It should also price invalidation: a cache that is usually discarded at the next model rollout has little long-term value.

### Assessment and limitations

KV-LTM is a detailed design proposal, not a demonstrated inference system. It provides analytical sizing and an evaluation protocol but reports no implementation, measured restore latency, production reuse distribution, or end-to-end quality result. The authors explicitly leave those outcomes open.

Read it for the state contract and cost model. Do not cite it as evidence that persistent per-user KV is practical at scale.

**Read:** [KV-LTM: Persistent Per-Session KV Caches as Long-Term Memory for Personal Assistants](https://arxiv.org/abs/2611.21998)

## 3. What did not clear the importance bar

[HyCoInfer](https://arxiv.org/abs/2611.22086) splits prefill onto cloud GPUs and decode across cloud GPUs and an edge FPGA, combining pipelined KV transfer, decoder-side precomputation, and residual-vector-quantized KV compression. It reports up to 4.11× cloud-only vLLM goodput at 500 Mbps using two V100 GPUs and one Alveo U280.

The design is interesting, but the comparison is too topology-specific to treat as a general serving advance. V100s, one FPGA, 100–1,000 Mbps links, and compressed cross-WAN KV transfer create a narrow operating point. The transferable lesson—include network state and heterogeneous decode capacity in placement—is already established. This is worth watching if cloud-edge FPGA deployment is relevant, but it is not a priority read for general CPU inference optimization.

[Measuring and Charging for Retained Context](https://arxiv.org/abs/2611.24071) reports that prefix-cache hits can reduce repeated-prefix compute cost by more than two orders of magnitude at 65,536 tokens. Its main contribution is an accounting and pricing framework, not a new inference mechanism. The measurement reinforces the economic importance of prefix locality, but it does not change the runtime design recommendations from the recent KV working-set and cache-semantics briefings.

## Recommended engineering actions

1. **Move backlog outside the engine.** Keep only enough engine-resident work to sustain batching; use a durable external priority queue for overload.
2. **Implement fail-fast, replica-local admission.** Base Cap on the actual limiting resource—local memory, bandwidth, batch shape, or deadline risk—not a fleet-wide request count.
3. **Use refusal feedback to tune dispatch.** Compare its stability and goodput with fixed request-rate and predicted-token-rate control under abrupt workload and capacity changes.
4. **Partition controllers by hardware and model class.** Do not make unlike NUMA, memory-tier, or ISA configurations share one learned ceiling.
5. **Treat serialized KV as versioned executable state.** If exploring persistence, bind state to the complete model/runtime contract and always retain canonical text for reconstruction.
6. **Require measured reuse economics.** Do not retain long-lived KV without real session-revisit distributions, restore bandwidth measurements, model-rollout cadence, and deletion costs.

## Bottom line

Yesterday’s briefing focused on representing and moving inference state correctly. Today’s strongest result sits one level above that: dispatch should learn available capacity from the resource that owns admission, rather than depend on a stale global rate or an output-length predictor.

For a CPU inference fleet, Cap-and-Track is worth prototyping. NUMA effects, bandwidth saturation, shape-dependent kernels, and model mix make capacity just as variable as it is on GPU fleets. A fast local refusal combined with a stateful upstream controller can turn those dynamics into a usable feedback signal while preserving efficient batches and keeping overload out of the execution engine.

KV-LTM is a useful longer-term thought experiment about activation state as durable storage, but it has not yet earned deployment confidence. Its main immediate value is to formalize cache identity, invalidation, and reuse economics.