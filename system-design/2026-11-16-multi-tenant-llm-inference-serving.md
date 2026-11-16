# Design a Multi-Tenant LLM Inference Serving Platform

**Date:** 2026-11-16  
**Difficulty:** Senior / Staff  
**Primary themes:** AI inference serving, GPU scheduling, continuous batching, model caching, queues, multi-tenant fairness, reliability, and consistency

> **Interview time versus this walkthrough:** In a 45-minute interview, aim for roughly 5 minutes on requirements, 2 minutes on entities, 5 minutes on interfaces, 10–15 minutes on the high-level design, and 10–15 minutes on deep dives. This document is deliberately more detailed than what you would say live. Use the “say aloud” snippets as a compact interview script; use the surrounding explanation to learn the reasoning behind it.
>
> **Framework note:** The organization follows the public [Hello Interview System Design Delivery Framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery): narrow requirements, name core entities, define interfaces, build the simplest complete design one operation at a time, and then deepen the design around the non-functional requirements. The API shape also follows the public [Hello Interview API Design guidance](https://www.hellointerview.com/learn/system-design/core-concepts/api-design), particularly deriving caller identity from authentication instead of accepting it in request bodies. The problem, dialogue, numbers, architecture, and tradeoffs below are original.

## Interviewer prompt

> Design a multi-tenant platform that serves large language models for many internal product teams. Teams should be able to deploy an approved model version, issue low-latency streaming generation requests, and submit large asynchronous batch jobs. The platform should share expensive GPU capacity efficiently while isolating tenants and surviving common failures.

The phrase “serve large language models” hides several separate problems. A strong candidate does not immediately draw GPUs. First, turn ambiguity into a contract: which models, which request modes, what latency, what scale, how much isolation, and what does “accepted” mean during failure?

## Targeted clarifying questions and assumed interviewer answers

**Candidate:** Are we training models, fine-tuning them, or only serving already-produced model artifacts?

**Interviewer:** Only inference. A separate pipeline validates model artifacts and places them in object storage. Our system starts with an approved artifact.

**Candidate:** Do tenants bring arbitrary code, or choose from model formats and runtimes that the platform supports?

**Interviewer:** Use supported transformer model formats and trusted runtimes. Arbitrary tenant code is out of scope.

**Candidate:** Which request modes matter most?

**Interviewer:** First, interactive text generation with streamed tokens. Second, asynchronous batch generation from a file. Third, tenants can create and update deployments that point to an approved model version.

**Candidate:** Should one request span models or tools?

**Interviewer:** No. Each request targets one deployment. Agent orchestration, retrieval, prompt moderation, and model training are out of scope.

**Candidate:** What scale and model sizes should I design for?

**Interviewer:** Assume 5,000 tenants, about 30 active model versions, and a highly skewed distribution: five versions receive 95% of interactive traffic. Models range from 8 GB to 140 GB of weights. At peak, the fleet receives 50,000 generated tokens per second and 25,000 prompt tokens per second.

**Candidate:** What latency targets matter?

**Interviewer:** For a warm, healthy deployment, p95 time to first token should be under 1.5 seconds and p95 inter-token gap under 100 milliseconds. Batch jobs can wait minutes but must not steal interactive capacity.

**Candidate:** What availability and durability do we promise?

**Interviewer:** The control plane and request admission APIs should be 99.95% available. Once a batch job is accepted, its manifest and eventual result must survive a regional worker failure. Interactive streams may fail explicitly; do not pretend token generation is exactly once.

**Candidate:** Do tenants need hard GPU reservations, fair sharing, or both?

**Interviewer:** Both. Paid production deployments may reserve capacity. Unreserved traffic shares spare capacity according to tenant quotas and weights. A noisy tenant must not dominate the fleet.

**Candidate:** May we route across regions?

**Interviewer:** Keep interactive requests in the nearest region when possible. Fail over only to a region that already has the exact deployment version and enough capacity. Deployment configuration should converge globally within a minute.

**Candidate:** What consistency is required when a deployment changes versions?

**Interviewer:** A deployment update is strongly consistent in the control plane. New requests should switch atomically from the old version to the new one after activation. Already-admitted requests finish on the version recorded at admission.

**Candidate:** Is exact billing required?

**Interviewer:** Usage must be auditable and must not double-charge retries. A few minutes of reporting delay is acceptable.

**Candidate:** Can clients retry?

**Interviewer:** Yes. Creation and batch-submission calls accept idempotency keys. An interactive request may be retried only before any token is delivered; after that, a new request is a new generation.

**Candidate:** I will focus on deployment management, interactive streaming, and asynchronous batch inference. I will discuss scheduling, model placement, fairness, failure recovery, and usage accounting. I will leave training, arbitrary runtimes, retrieval, moderation, and agent orchestration out of scope. Does that sound right?

**Interviewer:** Yes.

---

# 1. Requirements (~5 minutes)

## 1.1 Core functional requirements

Prioritize three operations that create a complete product:

1. **Manage a deployment.** A tenant creates or updates a logical deployment that points to an approved model artifact plus inference defaults and capacity policy.
2. **Run interactive inference.** An authenticated client sends a prompt to a deployment and receives generated tokens as a stream.
3. **Run asynchronous batch inference.** A tenant submits an object-storage manifest containing many prompts, checks job state, and retrieves durable results.

The first requirement is control plane work. The other two are data plane work. That separation will become a major architectural boundary.

**Say aloud:**

> “I am reducing the product to three operations: activate a model deployment, stream one interactive generation, and process a durable batch job. That gives us a control plane and two data-plane paths with very different latency expectations.”

## 1.2 Quantified non-functional requirements

1. **Latency:** for warm deployments, p95 time to first token below 1.5 seconds and p95 inter-token gap below 100 milliseconds.
2. **Throughput:** handle 25,000 prompt tokens per second and 50,000 output tokens per second at peak, with 2× short-burst headroom.
3. **Availability and durability:** 99.95% monthly availability for control-plane and admission APIs; accepted batch manifests and committed result fragments survive a worker or availability-zone failure.
4. **Isolation and fairness:** enforce per-tenant request, token, and concurrency quotas; production reservations receive their contracted capacity; batch work cannot materially degrade interactive latency.
5. **Configuration convergence:** deployment changes are strongly consistent at the source of truth and visible in all serving regions within 60 seconds. Usage reports may be eventually consistent by up to five minutes.

## 1.3 Scope

**In scope**

- Approved model artifacts and immutable versions
- Deployment lifecycle and atomic version activation
- Streaming text generation
- Durable batch generation
- GPU placement, admission, scheduling, quotas, usage accounting
- Regional failover when the same version is ready elsewhere

**Out of scope**

- Training and fine-tuning
- Arbitrary tenant Python or custom kernels
- Prompt moderation and policy enforcement
- Retrieval-augmented generation and vector search
- Agent/tool orchestration
- Model-quality evaluation
- Client chat-history storage

## 1.4 Consistency and availability contract

This is worth stating early because “highly available AI serving” otherwise becomes vague.

| Concern | Contract |
|---|---|
| Deployment update | Linearizable at the control-plane database. A single deployment has one active revision at a time. |
| Routing configuration | Eventually consistent across regions, target under 60 seconds. A signed revision number prevents a region from serving an unknown or partially applied configuration. |
| Request-to-version binding | Immutable after admission. The request record stores deployment revision and model version. |
| Interactive stream | At-most-one active execution per request ID inside the platform. Delivery is not exactly once; a disconnect after emitted tokens can leave the client uncertain. |
| Batch processing | At-least-once task execution, exactly-once visible result per item through deterministic item IDs and conditional result writes. |
| Usage accounting | Durable, deduplicated, eventually aggregated. The immutable usage event is keyed by request or batch-item attempt outcome. |
| Regional partition | Prefer serving already-admitted work and rejecting new work when quota/configuration cannot be verified. Do not silently use a different model version. |

A useful distinction is **execution semantics versus visible outcome**. A batch item may execute twice after a lease expires, but only one result becomes authoritative. Interactive generation cannot be replayed invisibly after the client has seen tokens because sampling may produce a different continuation.

**Say aloud:**

> “I will make deployment state strongly consistent, let regional routing converge asynchronously, and bind every admitted request to an immutable model version. Batch execution can be at least once with a deduplicated visible result. A token stream is not exactly once, so after tokens are emitted I surface failure rather than silently regenerate.”

## 1.5 Scale estimates that change the design

Do not multiply numbers merely to perform arithmetic. Estimate the resource that determines architecture: tokens and GPU memory.

### Throughput capacity

Assume the mixed fleet sustains about **800 output tokens per second per GPU** after batching. Actual throughput varies sharply by model, GPU, sequence length, and quantization, so this is a planning average, not a universal constant.

\[
\frac{50{,}000\ output\ tokens/s}{800\ output\ tokens/s/GPU} \approx 63\ GPUs
\]

Add 35% for failures, fragmentation, and latency headroom:

\[
63 \times 1.35 \approx 85\ GPUs
\]

Round to roughly **100 GPU equivalents for the peak interactive output path**, before batch capacity and large-model tensor-parallel overhead. This estimate changes the design in two ways:

- We need a scheduler that batches requests; one request per worker would waste most of the accelerator.
- We need admission control based on token and memory estimates, not just request count.

### Model-weight memory

Suppose the largest artifact is 140 GB and a worker node has eight 80 GB GPUs. It fits only when sharded across at least two GPUs, and practical serving needs more space for runtime buffers and the key-value cache. Pinning every one of 30 versions in every region would require at least:

\[
30 \times 140\ GB = 4.2\ TB
\]

That is only weights, before replication and cache memory. Because five versions receive 95% of traffic, we should pin hot versions and load cold versions on demand rather than fully replicate everything.

### Key-value cache pressure

During autoregressive generation, each sequence stores attention keys and values for tokens already processed. This **KV cache** avoids recomputing the entire prefix for every next token, but its memory grows with active sequence length. If an average active request consumes 1 GB of KV cache and a worker has 200 GB available after weights and runtime buffers, it can host only about 200 such sequences even if compute is idle. That makes memory-aware scheduling essential.

**Say aloud:**

> “Request QPS is not the governing unit here. Tokens determine compute, model weights determine placement, and KV-cache bytes determine active concurrency. My scheduler and quotas will use all three.”

---

# 2. Core entities (~2 minutes)

Name the essential entities before designing a full schema.

| Entity | Purpose | Essential fields introduced now |
|---|---|---|
| **ModelVersion** | Immutable approved artifact | model_version_id, artifact_uri, checksum, runtime, weight_bytes, parallelism_profile, status |
| **Deployment** | Tenant-facing stable name and policy | deployment_id, tenant_id, active_revision, active_model_version_id, defaults, quota_policy_id, capacity_class |
| **DeploymentRevision** | Immutable activation history | deployment_id, revision, model_version_id, config, state, created_at |
| **InferenceRequest** | One interactive generation | request_id, tenant_id, deployment_id, deployment_revision, model_version_id, prompt_uri_or_hash, limits, state, region |
| **BatchJob** | Durable collection of prompts | job_id, tenant_id, deployment_revision, input_manifest_uri, output_prefix, state, counts |
| **BatchItem** | Idempotent unit of batch work | job_id, item_id, attempt, lease_until, state, result_uri |
| **UsageEvent** | Auditable accounting fact | event_id, tenant_id, request_or_item_id, input_tokens, output_tokens, model_version_id, timestamp |

Do not put prompt bodies in the hot relational database by default. Keep short-lived interactive payloads in memory and encrypted logs only under policy; store batch manifests and results in object storage. The database stores identifiers, state, and audit metadata.

**Say aloud:**

> “The stable tenant object is Deployment; ModelVersion is immutable; DeploymentRevision records atomic changes. Every admitted request copies the exact revision and model version so a later rollout cannot change in-flight behavior.”

---

# 3. API or system interface (~5 minutes)

Caller identity comes from a validated OAuth token or service API key. The body never accepts a trusted tenant ID. The gateway derives **tenant_id**, roles, plan, and allowed deployments from authentication.

## 3.1 Create or update a deployment

### Create

~~~http
POST /v1/deployments
Authorization: Bearer <service-token>
Idempotency-Key: dep-create-7f2
Content-Type: application/json
~~~

~~~json
{
  "name": "support-copilot-prod",
  "model_version_id": "mv_llama70b_2026_11_02",
  "generation_defaults": {
    "temperature": 0.2,
    "max_output_tokens": 512
  },
  "capacity": {
    "class": "RESERVED",
    "min_output_tokens_per_second": 4000
  }
}
~~~

~~~json
{
  "deployment_id": "dep_8J2K",
  "revision": 1,
  "state": "WARMING",
  "model_version_id": "mv_llama70b_2026_11_02",
  "ready_regions": []
}
~~~

Creation returns **202 Accepted** because loading and health-checking a large model can take minutes.

### Atomic activation of a new revision

~~~http
PUT /v1/deployments/dep_8J2K/revisions/2
Authorization: Bearer <service-token>
If-Match: "revision-1"
Content-Type: application/json
~~~

~~~json
{
  "model_version_id": "mv_llama70b_2026_11_12",
  "generation_defaults": {
    "temperature": 0.2,
    "max_output_tokens": 512
  },
  "activation_policy": {
    "required_ready_regions": ["us-east", "us-west"]
  }
}
~~~

The **If-Match** precondition prevents two operators from overwriting one another. The control plane first warms the requested regions, then performs a transaction that advances **active_revision** from 1 to 2.

## 3.2 Interactive streaming inference

~~~http
POST /v1/deployments/dep_8J2K/responses
Authorization: Bearer <service-token>
Idempotency-Key: infer-01HQ9ZZR
Accept: text/event-stream
Content-Type: application/json
~~~

~~~json
{
  "input": "Summarize the attached support case in three bullets.",
  "max_output_tokens": 256,
  "temperature": 0.1,
  "deadline_ms": 12000
}
~~~

The server streams Server-Sent Events:

~~~text
event: response.accepted
data: {"request_id":"req_4M9","deployment_revision":2,"model_version_id":"mv_llama70b_2026_11_12"}

event: response.delta
data: {"sequence":1,"text":"• The customer"}

event: response.delta
data: {"sequence":2,"text":" reports intermittent"}

event: response.completed
data: {"input_tokens":14,"output_tokens":87,"finish_reason":"stop"}
~~~

SSE is sufficient because application data flows from server to client after one HTTP request. It is simpler than a bidirectional WebSocket for this one-way token stream.

If a duplicate idempotency key arrives before any output was emitted, return the same request handle or attach to the same stream if supported. If output was already emitted and the original connection is gone, return a conflict with the known state rather than beginning a second generation under the same key.

Possible errors:

- **409** deployment revision changed during a conditional operation
- **429** tenant quota or fleet admission limit exceeded, with a retry hint
- **503** exact model version has no healthy capacity
- **504** request expired before admission or generation completed

## 3.3 Submit and inspect a batch job

~~~http
POST /v1/deployments/dep_8J2K/batch-jobs
Authorization: Bearer <service-token>
Idempotency-Key: nightly-cases-2026-11-16
Content-Type: application/json
~~~

~~~json
{
  "input_manifest_uri": "s3://tenant-bucket/jobs/2026-11-16.jsonl",
  "output_prefix": "s3://tenant-bucket/results/2026-11-16/",
  "max_output_tokens": 300,
  "completion_target": "PT6H"
}
~~~

~~~json
{
  "job_id": "job_71D",
  "state": "VALIDATING",
  "deployment_revision": 2,
  "model_version_id": "mv_llama70b_2026_11_12",
  "status_url": "/v1/batch-jobs/job_71D"
}
~~~

~~~http
GET /v1/batch-jobs/job_71D
Authorization: Bearer <service-token>
~~~

~~~json
{
  "job_id": "job_71D",
  "state": "RUNNING",
  "total_items": 1000000,
  "succeeded_items": 417200,
  "failed_items": 91,
  "output_manifest_uri": null
}
~~~

Cancellation is a state transition, not a promise to stop an instruction already executing:

~~~http
POST /v1/batch-jobs/job_71D/cancellation
Authorization: Bearer <service-token>
~~~

**Say aloud:**

> “The API exposes the three requirements directly. Authentication supplies tenant identity. Deployment activation is conditional and revisioned; interactive inference uses SSE; batch submission freezes the deployment revision and returns an asynchronous job.”

---

# 4. Data flow

A pipeline view is useful because interactive and batch work share GPU workers but enter through different admission paths.

## Interactive path

1. Gateway authenticates the caller, derives tenant policy, and validates the request.
2. Regional router resolves the deployment’s active revision from a signed configuration cache.
3. Admission control reserves tenant token budget, concurrency, and estimated KV-cache bytes.
4. Scheduler places the request in a queue for a compatible loaded model replica.
5. A worker performs **prefill**: it processes all prompt tokens and creates the initial KV cache.
6. The worker performs **decode** repeatedly, producing one token per active sequence per iteration.
7. Tokens flow through a stream relay to the client; usage is finalized at completion or failure.

**Prefill** is compute-heavy and parallel across prompt tokens. **Decode** is iterative and often memory-bandwidth-bound. Treating them separately helps scheduling and capacity planning.

## Batch path

1. Batch service stores a job record and validates object ownership, format, and item count.
2. Manifest expander writes deterministic items keyed by **job_id + item_id** to a durable queue.
3. Batch scheduler consumes only from a reserved batch capacity pool or spare interactive capacity.
4. Workers write result fragments to a temporary object key.
5. A conditional database write claims the authoritative result for that item.
6. A compactor produces the final output manifest after all terminal items are accounted for.
7. Usage events are aggregated asynchronously.

---

# 5. High-level design (~10–15 minutes)

Following the delivery framework, start with the simplest complete design and add one operation at a time.

## 5.1 Compact architecture

~~~mermaid
flowchart TB
    C["Tenant clients"] --> G["Global API gateway"]
    G --> CP["Control plane"]
    G --> DP["Regional inference plane"]

    CP --> DB[("Deployment DB")]
    CP --> OBJ[("Model and job object storage")]

    DP --> A["Router + admission"]
    A --> S["Interactive and batch schedulers"]
    S --> W["GPU worker pools"]
    DP --> Q[("Durable batch queue")]
    W --> Q
    W --> OBJ
    DP --> U[("Usage event log")]
~~~

The diagram deliberately shows a control plane and a regional data plane. Control-plane availability should not sit on the token hot path. A cached, signed deployment snapshot lets already-configured regions continue serving during a temporary control-plane outage.

## 5.2 Operation 1: create and activate a deployment

### Baseline components

- **API gateway:** authentication, authorization, request size limits, idempotency-key forwarding.
- **Deployment service:** validates an approved ModelVersion and writes DeploymentRevision.
- **Relational deployment database:** transactions, uniqueness, conditional revision changes.
- **Artifact registry/object storage:** immutable weights, tokenizer, runtime metadata, checksums.
- **Placement controller:** chooses regions and worker groups, asks them to load the artifact.
- **Regional configuration stream:** publishes signed revisions to routers.

### Request trace

1. The gateway derives **tenant_id** and checks the caller has deployment-admin permission.
2. Deployment service looks up ModelVersion and verifies its status is **APPROVED**.
3. In one transaction it creates Deployment revision 1 in **WARMING** state and records the idempotency key.
4. Placement controller reads the model’s weight size and parallelism profile. For a 140 GB model, it may allocate a tensor-parallel group spanning four GPUs so weights and runtime state fit with KV-cache headroom.
5. Each selected worker downloads artifacts into a node-local NVMe cache, verifies the checksum, loads weights into GPU memory, performs a known-answer health probe, and reports readiness.
6. When the activation policy is satisfied, Deployment service transactionally changes the revision to **ACTIVE** and updates Deployment.active_revision.
7. A configuration event is published. Regional routers apply only monotonically increasing, signed revision numbers.
8. The API reports ready regions.

### Relevant data model

**Deployment**

- primary key: deployment_id
- tenant_id
- name; unique within tenant
- active_revision
- quota_policy_id
- capacity_class
- created_at

**DeploymentRevision**

- primary key: deployment_id + revision
- model_version_id
- generation_defaults_json
- activation_policy_json
- state
- created_by
- created_at

**IdempotencyRecord**

- tenant_id + operation + idempotency_key
- request_hash
- resource_id
- response_status
- expires_at

If the same idempotency key arrives with a different request hash, reject it. Reusing a key must not mutate an existing deployment accidentally.

**Interviewer:** Why use a relational database rather than a key-value store?

**Candidate:** The control plane is modest in QPS, but it needs transactions across deployment revision, active pointer, authorization metadata, and idempotency. A relational database provides conditional updates and audit-friendly history. Serving routers do not query it per token request; they consume cached snapshots.

**Say aloud:**

> “I will optimize the control plane for correctness. Activation is a transaction only after workers prove readiness. The data plane receives immutable signed revisions, so it does not depend on a database lookup for every inference request.”

## 5.3 Operation 2: stream an interactive generation

### Baseline components

- **Regional API edge and stream relay**
- **Deployment routing cache**
- **Quota/admission service**
- **Per-model scheduler**
- **GPU workers**
- **Usage event log**

### Request and state trace

1. Global routing sends the request to the nearest healthy region advertised as ready for the deployment.
2. The gateway authenticates the token, derives tenant ID, and validates request limits.
3. Router reads DeploymentSnapshot from local memory:
   - deployment_id
   - active_revision
   - model_version_id
   - allowed parameters
   - quota policy
   - eligible worker pool
4. It creates InferenceRequest in **ADMITTING** state with the frozen deployment revision.
5. Admission control estimates:
   - input tokens from the tokenizer
   - maximum output tokens
   - initial KV-cache reservation
   - deadline
6. It atomically checks tenant rate, token, and concurrency buckets. If unavailable, return 429 before GPU work begins.
7. The model scheduler chooses a loaded replica with compatible runtime and enough KV-cache headroom. The request moves to **QUEUED**.
8. Worker starts prefill; state becomes **RUNNING**. When the first token is ready, the stream relay emits **response.accepted** and deltas.
9. On completion, the worker reports actual input/output tokens and finish reason.
10. Request becomes **SUCCEEDED**, quota reservation is reconciled against actual use, and an immutable UsageEvent is appended.
11. A separate aggregator materializes tenant dashboards and billing summaries.

A production implementation should avoid a database write for every token. The worker emits token chunks through an in-memory or low-latency streaming channel; state transitions and periodic progress checkpoints are much less frequent.

### Why a stream relay?

The GPU worker should not own public client sockets. A deploy or worker crash would sever both compute and network state. A stateless relay:

- terminates SSE connections,
- applies backpressure if the client reads slowly,
- buffers only a small bounded number of chunks,
- cancels generation when policy says a disconnected client should not consume capacity,
- lets workers communicate over an internal protocol optimized for token chunks.

A relay cannot manufacture exactly-once delivery. Sequence numbers help the client detect a missing or duplicate chunk, but if the relay crashes after forwarding a token and before recording that fact, uncertainty remains.

**Interviewer:** Could we store every emitted token so a client can reconnect?

**Candidate:** We could persist chunks for premium resumable streams, but that increases latency, storage, and privacy exposure. The baseline keeps small ephemeral buffers and returns final output only if policy allows. I would make resumability an explicit product contract, not imply it from SSE.

## 5.4 Operation 3: process a batch job

Interactive traffic is latency-sensitive; batch traffic is throughput-sensitive. Combining them in one FIFO is a classic failure.

### Batch components

- Batch job service
- Manifest validator and expander
- Durable partitioned queue
- Batch scheduler
- Result committer
- Manifest finalizer

### Request and state trace

1. Batch service authorizes the input and output object-storage prefixes using tenant-scoped credentials.
2. It transactionally stores BatchJob in **VALIDATING**, freezes deployment revision/model version, and records the idempotency key.
3. Validator streams the JSONL manifest rather than loading it into memory. Each line must have a unique tenant-provided item ID or receives a deterministic line-number ID.
4. Expander publishes compact queue messages:
   - job_id
   - item_id
   - prompt object offset or object URI
   - model_version_id
   - limits
5. Batch schedulers consume from separate queues and receive a configured fraction of each model pool. They may borrow idle interactive capacity but are preemptible before admission to a decode batch.
6. A worker leases an item. After generation, it writes output to a unique attempt path such as **jobs/job_71D/items/42/attempt-3.json**.
7. Result committer performs a conditional insert for **job_id + item_id**. One attempt wins; duplicates become garbage-collection candidates.
8. It atomically increments sharded completion counters or emits completion events for aggregation.
9. When every item is terminal, finalizer writes an output manifest and transitions the job to **SUCCEEDED**, **PARTIAL**, **FAILED**, or **CANCELLED**.

### Why a durable queue?

A million-item job should not be represented by one giant task held by one worker. Fine-grained queue items enable parallelism, retries, visibility timeouts, and progress. A lease/visibility timeout means a worker crash causes work to reappear. That produces at-least-once execution, so conditional result commitment is necessary.

**Say aloud:**

> “Batch and interactive requests can share runtimes but not a FIFO. I give batch work a durable queue, deterministic item IDs, leased execution, and conditional result commits. It may execute twice, but only one result is visible and billable.”

---

# 6. Deep dives (~10–15 minutes)

Choose the parts where the baseline breaks under the stated scale. Three strong deep dives are scheduling, model placement, and multi-tenant reliability/accounting.

## Deep dive 1: continuous batching and memory-aware scheduling

### Where the baseline breaks

A naïve worker accepts a fixed group of requests, waits until every sequence finishes, and then starts another group. Generation lengths vary. Short requests finish early, leaving empty batch slots while one long request keeps the batch open. Alternatively, one-request-per-GPU has great isolation but terrible throughput and cost.

The scheduler must also consider that:

- prefill cost grows with prompt length,
- decode advances one token at a time,
- active sequences consume KV-cache memory,
- long prompts can block short interactive requests,
- a request with a near deadline may be more urgent than an earlier batch job.

### Plausible approaches

| Approach | Advantages | Problems |
|---|---|---|
| One request per worker | Simple latency reasoning, strong isolation | Extremely poor GPU utilization and cost |
| Static batching | Easy to implement; good for equal-size offline inputs | Head-of-line blocking and idle slots from variable output lengths |
| Continuous batching | Adds and removes sequences between decode iterations; high utilization | More complex scheduler, memory tracking, cancellation, and fairness |
| Separate prefill and decode pools | Tune each phase independently; isolates long prompts | KV transfer and network complexity; additional failure modes |

### Selection

Use **continuous batching** within a model replica. At each decode iteration, completed or cancelled sequences leave and eligible queued sequences join if token and KV-memory budgets permit. Start with combined prefill/decode workers; split the phases only when measurements show long prefill work damages decode latency.

The scheduling hierarchy is:

1. **Admission gate:** enforce global model capacity and tenant quotas.
2. **Service class queue:** reserved interactive, shared interactive, then batch.
3. **Fair tenant selection:** weighted deficit round robin using estimated token cost.
4. **Replica selection:** compatible model/version, sufficient KV cache, low predicted completion time.
5. **Worker batch formation:** cap total prompt tokens for prefill and active KV bytes for decode.

**Weighted deficit round robin** gives each tenant credits proportional to its weight. A request spends credits based on estimated tokens, so a tenant sending huge prompts cannot look equal to one sending tiny prompts merely because both issued one request.

### Token-aware admission

A request declares maximum output tokens, but it may finish early. Reserve a conservative amount at admission, then reconcile actual usage. To stop dishonest or buggy clients from asking for enormous limits:

- hard-cap prompt and output tokens per deployment,
- charge quota on reserved tokens initially,
- release unused reservation on completion,
- use historical completion ratios for capacity prediction but never for authorization,
- reject when projected KV memory would violate a safety margin.

### Prefix caching

Many requests may share a large system prompt. A **prefix cache** stores KV blocks for an identical token prefix so workers skip repeated prefill. It can materially reduce latency and cost, but it requires careful keys:

- model version
- tokenizer version
- exact prompt-token hash
- inference parameters that affect cache correctness
- tenant or security domain

Do not share cached prefixes across tenants by default. Even if keys are hashes, cache timing and memory reuse can become a side channel. A per-tenant cache is safer; explicitly public prefixes may use a shared domain.

### Overload behavior

Do not let queues grow without bound. A queue that adds 20 seconds to a request with a 12-second deadline is not useful availability.

- Estimate start time from queued token work and worker throughput.
- Reject early with 429 or 503 when the deadline cannot be met.
- Include **Retry-After** and capacity reason.
- Shed batch borrowing before shared interactive work.
- Preserve reserved interactive capacity.
- Use bounded queues and circuit breakers during model-worker failure storms.

### Failure modes and observability

| Failure | Response | Key signal |
|---|---|---|
| Worker out of memory | Kill unhealthy replica, retry only before output, reduce admission estimate error | GPU OOM rate by model and sequence bucket |
| Long prompt causes head-of-line blocking | Separate prefill token budget; optionally chunked prefill | p95 TTFT by prompt-length bucket |
| Slow client blocks relay | Bounded relay buffer; cancel or truncate by policy | blocked-stream count and buffer age |
| Scheduler starvation | Weighted fair queue and maximum wait boost | queue age by tenant/service class |
| KV fragmentation | Paged/block KV allocator and compaction-aware placement | free versus allocatable KV bytes |

**Interviewer:** Why not just autoscale on GPU utilization?

**Candidate:** High GPU utilization can coexist with bad time-to-first-token, and low utilization may still mean KV memory is full. I would scale on queued token-seconds, deadline miss forecast, ready replica count, KV pressure, and per-model arrival rate. GPU utilization is supporting evidence, not the control signal.

**Say aloud:**

> “I choose continuous batching because sequence lengths vary. The scheduler is token- and KV-aware, not QPS-based. It uses weighted fairness across tenants, preserves reserved interactive capacity, and rejects work before its deadline becomes impossible.”

## Deep dive 2: model placement, hot caches, and cold starts

### Where the baseline breaks

The router can pick only replicas that have the exact model version loaded. Loading 140 GB from remote object storage, deserializing it, and initializing kernels can take minutes. Keeping every version loaded wastes scarce GPU memory. Random load balancing can route requests to nodes that cannot serve them.

### Plausible approaches

| Placement strategy | Advantages | Problems |
|---|---|---|
| Fully replicate every version | No cold starts, simple routing | Prohibitively expensive; GPU memory explosion |
| Load on every first request | Low idle cost | User-visible multi-minute latency and thundering herds |
| Pin only hot versions | Efficient for skewed traffic | Cold versions need explicit behavior |
| Warm pool plus predictive placement | Balances latency and cost | Forecasting and state coordination complexity |

### Selection

Use tiered model placement:

1. **Hot tier:** top models are pinned with N+1 replica headroom in each required region.
2. **Warm tier:** artifacts and compiled runtime plans live on node-local NVMe; GPU memory is allocated on demand.
3. **Cold tier:** artifact remains only in regional object storage; interactive calls return 503 or a documented warming response unless the deployment purchased cold-start tolerance.
4. **Reserved tier:** production reservations maintain a minimum ready replica count regardless of recent traffic.

The placement controller consumes:

- per-model arrival and token rates,
- deployment reservation contracts,
- weight and KV-memory requirements,
- hardware compatibility,
- fault-domain constraints,
- load time,
- observed queue delay.

It solves a constrained packing problem, but an interview does not require an optimal bin-packing algorithm. A greedy heuristic is enough: satisfy reservations, maintain fault-domain diversity, pin hot versions, then allocate warm capacity using recent and forecasted demand.

### Preventing a thundering herd

If 1,000 requests arrive for a cold model, they must not trigger 1,000 loads.

- A regional **load coordinator** owns a lease keyed by model version and hardware profile.
- One loader starts; other requests see **WARMING**.
- Only a bounded queue waits, and only if its deadlines tolerate the expected load.
- Loaded workers pass checksum and known-answer probes before becoming routable.
- Failed loads use exponential backoff and a circuit breaker.

### Routing and consistency

Each regional router maintains a table:

- deployment_id → active revision
- revision → model version
- model version → ready replica set
- replica → current queue/KV capacity and heartbeat epoch

Deployment activation publishes an immutable revision. Replica readiness is ephemeral and may change every second. Keep these two state types separate: configuration comes from the control plane, while readiness comes from regional health streams.

A router never substitutes another model version because “something similar is available.” During propagation, a request contains or is assigned a deployment revision. If the region does not know that revision, it refreshes configuration or rejects; it does not guess.

### Safe rollouts

For revision 2:

1. Create immutable revision in **WARMING**.
2. Load it into required regions.
3. Run synthetic correctness and latency probes.
4. Transactionally advance active_revision.
5. Route new requests to revision 2.
6. Let revision-1 requests drain.
7. Keep old replicas for a rollback window.
8. Roll back by advancing to a new revision that points to the prior ModelVersion; do not rewrite history.

### Failure modes and observability

| Failure | Design response | Key signal |
|---|---|---|
| Artifact corruption | Checksum validation and signed registry metadata | checksum failures |
| Region loses a model pool | Route to another ready region only if policy and capacity allow | ready replicas by version/region |
| Placement oscillation | Minimum residency time and hysteresis | loads/evictions per model-hour |
| Hot model evicts another tenant repeatedly | reservation floor and eviction budgets | cold-start rate by tenant |
| Worker heartbeat is stale | Remove from ready set using epoch/lease | heartbeat age and false-positive removals |

**Interviewer:** Would you use consistent hashing to route requests?

**Candidate:** Consistent hashing can improve prefix-cache locality, but pure hashing ignores queue length and KV pressure. I would use bounded-load hashing: hash deployment and prefix to a small candidate set, then pick the healthiest candidate. That keeps locality without sending traffic to an overloaded replica.

**Say aloud:**

> “Because weights are huge and traffic is skewed, I pin hot and reserved versions, keep warm artifacts on local NVMe, and coordinate cold loads with one lease. Configuration revisions are durable; replica readiness is ephemeral. Routing requires both the exact revision and a healthy loaded replica.”

## Deep dive 3: fairness, failure semantics, and auditable usage

### Where the baseline breaks

A single shared queue lets a tenant flood the fleet. Automatic retries can duplicate expensive generations. A worker may emit tokens and crash before recording completion. A batch lease may expire while the original attempt is still running. If billing consumes mutable request rows, late updates and retries can double-charge or lose usage.

### Fairness and isolation

Enforce limits at multiple levels:

- **Gateway:** requests per second and payload size
- **Admission:** input tokens per minute, reserved output tokens, concurrent sequences, batch backlog
- **Scheduler:** weighted token fairness and service-class priority
- **Worker:** maximum sequence length, KV allocation, runtime sandbox boundaries
- **Accounting:** budget alerts and hard spending caps

Separate quota dimensions because request count alone is misleading. Ten 100,000-token prompts are not equivalent to ten 20-token prompts.

Reservations are enforced by capacity partitions or scheduler guarantees. Completely dedicated GPU pools offer strongest isolation but lower utilization. The selected design uses logical reservations on shared pools with a small dedicated escape pool for premium workloads. Batch may borrow unused reserved slots only when it can be preempted before starting a decode iteration; do not kill active interactive work just to reclaim a borrowed slot.

### Interactive failure semantics

Track explicit states:

~~~mermaid
stateDiagram-v2
    [*] --> ADMITTING
    ADMITTING --> QUEUED
    QUEUED --> RUNNING
    RUNNING --> SUCCEEDED
    ADMITTING --> REJECTED
    QUEUED --> EXPIRED
    RUNNING --> FAILED_PARTIAL
    RUNNING --> CANCELLED
~~~

Rules:

- Before the first token, the platform may retry once on another compatible worker under the same request ID.
- After the first token, a worker failure yields **FAILED_PARTIAL** with the last known sequence number.
- Do not bill reserved output tokens; bill actual successfully computed tokens according to the product contract.
- If the client disconnects, cancel by default after a short grace period unless it requested server-side completion.
- A duplicate idempotency key returns the existing request state; it never starts a second execution.

This is honest at-least-once network reality presented as an unambiguous API contract.

### Batch exactly-once visible outcome

For each batch item:

1. Consumer obtains a lease and increments attempt.
2. Worker writes attempt output to a unique immutable object.
3. Committer executes a conditional insert:
   - key: job_id + item_id
   - condition: no authoritative result exists
4. Winner appends one UsageEvent with deterministic event ID.
5. Losers observe the existing result and delete their temporary output later.

A worker may finish after its lease expires. Conditional commitment prevents it from overwriting the winner.

### Usage outbox

When an interactive request reaches a terminal state, write its terminal row and a **usage outbox row** in one database transaction. A publisher moves outbox records to the usage log and marks them published. If it crashes between publishing and marking, it republishes. The downstream consumer deduplicates by UsageEvent.event_id.

This **transactional outbox** bridges a database state change and a message log without requiring a distributed transaction. Delivery is at least once; materialized billing is exactly once per event ID.

For high-throughput token events, do not update the relational row per token. Workers maintain local counters and send a signed terminal usage summary. Periodic checkpoints limit loss for long generations. Compare GPU runtime telemetry with billed usage to detect discrepancies.

### Regional failure

- Deployment metadata is replicated across regions through the global control plane.
- Each region has independent admission, queues, relays, and GPU pools.
- Interactive DNS/anycast routes new connections to a healthy ready region.
- In-flight SSE streams fail when their region fails; the API surfaces partial status.
- Batch job manifests and result objects use multi-AZ durable storage. Queue tasks can be reconstructed from the manifest and authoritative item table.
- Only one region owns a batch-job scheduling lease at a time. A new region takes over after lease expiry and continues from uncommitted items.
- Cross-region fallback requires the exact deployment revision and available quota. Otherwise reject rather than violate correctness.

### Backpressure and retry storms

Clients receive structured errors with retry hints. Internal retries use:

- exponential backoff with jitter,
- a small retry budget per request,
- circuit breakers by model/version/region,
- dead-letter queues for batch items with deterministic failures,
- no retry for validation, quota, or unsupported-parameter errors.

A retry budget prevents a 10% failure rate from turning into 100% additional load.

### Observability and SLOs

Measure by model, version, region, tenant service class, and prompt/output bucket, while avoiding raw prompt labels.

**Golden signals**

- request rate and generated tokens per second
- admission rejection rate and reason
- queue delay, time to first token, inter-token gap, total latency
- success, failure-before-output, and partial-stream rates
- worker GPU utilization, high-bandwidth-memory use, KV allocatable bytes
- batch oldest-item age and completion throughput
- model load time, cache hit rate, eviction rate
- usage event lag and deduplication count

**SLO examples**

- 99% of warm reserved interactive requests begin streaming within 1.5 seconds monthly.
- 99.95% of valid admission API calls receive a non-5xx response.
- 99.9% of accepted batch items reach a terminal result within the job’s completion target, excluding tenant data errors.
- 99.999% of committed usage events appear in reporting within five minutes.

Use trace IDs from gateway through scheduler, worker, stream relay, and usage event. Sample ordinary traces, but retain all errors, OOMs, and deadline misses. Never place prompt text or generated text in metric labels.

**Interviewer:** How do you prove one tenant is not harming another?

**Candidate:** I would expose queue delay and deadline-miss rate by service class and anonymized tenant bucket, run controlled noisy-neighbor load tests, and alert when shared-pool latency diverges from reserved-pool latency. Scheduler fairness metrics should show allocated token-seconds versus each tenant’s entitlement over a rolling window.

**Say aloud:**

> “Reliability is a contract, not a retry loop. I allow one transparent retry only before output. Batch tasks may execute more than once but conditionally commit one result. Quotas exist at gateway, admission, scheduler, and worker layers, and usage comes from deduplicated immutable events.”

---

# 7. Final design, checks, follow-ups, and pitfalls

## 7.1 Concise final design

The platform separates a strongly consistent **control plane** from independent regional **inference planes**.

- The control plane stores immutable ModelVersions and DeploymentRevisions in a relational database.
- Placement controllers warm exact model versions from object storage, using hot, warm, cold, and reserved tiers.
- Activation advances a deployment’s active revision only after readiness policy is met.
- Regional routers consume signed configuration snapshots and combine them with ephemeral replica health.
- Interactive requests are authenticated, frozen to a revision, admitted against multi-dimensional quota, scheduled with weighted token fairness, and streamed through SSE relays.
- GPU workers use continuous batching and memory-aware KV-cache allocation.
- Batch jobs expand durable manifests into leased queue items and use conditional result commits for exactly-once visible output.
- Usage is captured through immutable, deduplicated events and aggregated asynchronously.
- Regions fail independently; new traffic can use another region only when the exact deployment revision is ready there. Partial interactive streams fail explicitly.

## 7.2 Requirements check

| Requirement | Design element |
|---|---|
| Manage deployments | Revisioned control plane, artifact registry, placement controller, atomic activation |
| Stream interactive generation | Regional router, admission, continuous scheduler, GPU worker, SSE relay |
| Run durable batch jobs | Object manifests, durable queue, leased items, conditional result commits |
| p95 TTFT < 1.5 s for warm models | Pinned replicas, local routing cache, bounded queues, token-aware admission, prefix caching |
| p95 inter-token gap < 100 ms | Continuous batching, decode priority, KV-memory tracking |
| 50k output tokens/s peak | Token-based capacity estimate, roughly 100 interactive GPU equivalents with headroom |
| 99.95% API availability | Multi-AZ stateless services, cached config, independent regional data planes |
| Tenant isolation | Multi-dimensional quotas, weighted fairness, reservations, separate batch service class |
| Strong rollout semantics | Linearizable active revision and immutable request binding |
| Auditable no-double-charge usage | Transactional outbox, deterministic event IDs, downstream deduplication |

## 7.3 Realistic interviewer follow-up questions with strong sample answers

### 1. How would speculative decoding change the design?

**Strong answer:**

Speculative decoding uses a smaller draft model to propose several tokens and the target model to verify them. It can reduce target-model decode steps when acceptance is high. The scheduler now needs paired draft and target capacity, compatible tokenizer/version metadata, and metrics for acceptance rate and end-to-end speedup. I would enable it per deployment only after measuring the extra memory and coordination cost. Billing remains based on delivered target-model tokens, while internal capacity accounting includes draft work.

### 2. What if a tenant needs strict data residency?

**Strong answer:**

Add allowed regions to the deployment and tenant policy. Artifact copies, prompts, KV caches, logs, batch objects, and usage details stay within that boundary. Global configuration may carry non-sensitive identifiers, but routing must never fail over outside the residency set. If no allowed region has the exact version, return unavailable rather than violate policy.

### 3. How do you handle model versions requiring different GPU types?

**Strong answer:**

ModelVersion includes a hardware/runtime compatibility profile: minimum memory, supported accelerator architecture, tensor-parallel degree, quantization, and compiled engine version. Placement filters eligible node pools before scoring capacity. A deployment cannot become active in a required region until a compatible group passes its probe.

### 4. Why not put Kafka directly between the interactive gateway and GPUs?

**Strong answer:**

A durable log is excellent for batch and usage events, but the interactive path needs low queue latency, cancellation, deadline-aware scheduling, and streaming affinity. A regional in-memory scheduler with replicated or reconstructable admission state is a better fit. If the scheduler fails before output, the gateway can retry once. Durability for every interactive prompt is a product decision with privacy and latency cost, not automatically beneficial.

### 5. How do you estimate whether a request will fit KV memory?

**Strong answer:**

Use model-profile coefficients derived from runtime measurements, multiplied by prompt plus maximum generated tokens and adjusted for parallelism and cache block rounding. Admission uses a conservative bound; the worker’s paged KV allocator is the final authority. Track estimate error by model and sequence bucket, update coefficients offline, and keep a safety margin to avoid OOM cascades.

### 6. What happens if the control-plane database is unavailable?

**Strong answer:**

Deployment mutations stop, but regional inference continues from last-known signed snapshots until their policy-defined expiry. Routers do not invent revisions. If a snapshot becomes too old to satisfy a tenant’s safety policy, admission fails closed for that deployment. This isolates the serving SLO from a temporary control-plane outage without accepting unbounded configuration staleness.

### 7. How would you support canary rollouts?

**Strong answer:**

Make a rollout revision reference both stable and candidate versions with a deterministic traffic policy. Hash tenant-defined session key or request ID into a bucket so retries remain on the same version. Record the selected model version on admission. Compare latency, error, safety, and product-quality metrics, then atomically promote or disable the candidate. Never let an in-flight request switch versions.

### 8. How would you reduce cost during quiet periods?

**Strong answer:**

Preserve contractual reserved replicas, then scale shared replicas using queued token-seconds and load forecasts. Evict cold GPU weights to local NVMe before removing nodes. Batch work fills otherwise idle capacity. Use minimum residency and hysteresis to prevent load/evict oscillation, and compare saved GPU-minutes with added cold-start latency.

### 9. Can the client resume a broken stream?

**Strong answer:**

Not in the baseline. Sequence numbers detect gaps, but deterministic replay is not guaranteed under sampling. For a resumable premium mode, I would persist token chunks and generation checkpoints, keep the worker running briefly after disconnect, and let the relay replay acknowledged chunks. A regional or worker failure may still require returning a partial result unless the inference runtime supports a portable KV checkpoint.

### 10. Where would caching help, and where would it be dangerous?

**Strong answer:**

Cache immutable artifacts in regional storage, compiled plans on local NVMe, deployment snapshots in router memory, and exact prompt prefixes as tenant-scoped KV blocks. Full-response caching is safe only when the request is deterministic, policy permits content retention, and the key covers model version and every generation parameter. Cross-tenant prompt caching risks privacy leakage and timing side channels, so it is disabled by default.

## 7.4 Common pitfalls

1. **Starting with GPUs before defining the contract.** The important distinctions—interactive versus batch, strong revision semantics, partial-stream failure—must come first.
2. **Sizing by request QPS alone.** Tokens, weight memory, KV memory, and model mix determine capacity.
3. **Claiming exactly-once token delivery.** A network disconnect after emitted output creates ambiguity. State the limitation.
4. **Using one FIFO for all work.** Batch requests will destroy interactive latency.
5. **Loading every model everywhere.** The memory arithmetic makes that untenable; exploit skew with hot/warm/cold tiers.
6. **Letting the router choose any “equivalent” model.** Correctness requires the exact deployment revision and model version.
7. **Ignoring tenant identity in the scheduler.** Gateway rate limits alone do not prevent token or KV-memory monopolization.
8. **Retrying after output without telling the client.** Sampling may produce a different continuation and double cost.
9. **Writing every token to the primary database.** Stream through relays; persist coarse state and terminal usage events.
10. **Treating GPU utilization as the only autoscaling metric.** Queue token-seconds, deadline risk, ready replicas, and KV pressure matter more.
11. **Confusing execution with commitment.** Batch items can execute at least once while producing one authoritative result.
12. **Skipping rollout and rollback semantics.** Immutable revisions and drain windows make updates explainable and safe.
13. **Logging prompts in metrics.** This creates privacy exposure and explosive cardinality.
14. **Presenting every optimization at once.** Build a complete baseline first, then earn complexity by showing where it breaks.

## Closing interview summary

**Candidate:**

> “The core design separates a correct, revisioned control plane from low-latency regional inference planes. Deployment activation is atomic only after exact model versions are ready. Interactive requests freeze that revision, pass token- and KV-aware admission, and run through a fair continuous-batching scheduler with explicit partial-stream failure semantics. Batch work uses durable leased tasks and conditional result commits. Hot and reserved models stay pinned, colder models use coordinated warming, and usage is recorded as deduplicated immutable events. The design meets the three functional requirements while making latency, fairness, consistency, and failure behavior explicit.”
