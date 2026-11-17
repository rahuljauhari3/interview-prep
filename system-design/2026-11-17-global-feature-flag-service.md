# Design a Global Feature Flag Service

**Date:** 2026-11-17  
**Difficulty:** Medium–Hard  
**Primary themes:** databases, caching, configuration distribution, consistency, high availability, deterministic rollout, and operational safety

> **Interview time versus this walkthrough:** In a 45-minute interview, spend roughly 5 minutes on requirements, 2 minutes on entities, 5 minutes on interfaces, 10–15 minutes on the high-level design, and 10–15 minutes on deep dives. This document is intentionally more detailed than a live answer. Use the “say aloud” passages as the compact interview version and the surrounding material to learn why each choice is made.
>
> **Framework note:** The structure follows the public [Hello Interview System Design Delivery Framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery): prioritize requirements, identify core entities, define the interface, build the simplest complete design one operation at a time, and then harden the bottlenecks and non-functional requirements. It also follows the public [Hello Interview API Design guidance](https://www.hellointerview.com/learn/system-design/core-concepts/api-design) by deriving tenant and actor identity from authentication rather than trusting IDs in request bodies. The prompt, explanations, dialogue, data model, calculations, and tradeoffs below are original.

## Interviewer prompt

> Design a global feature-flag service that lets engineering teams change application behavior without deploying new code. Teams should be able to create flags, define targeting and percentage-rollout rules, publish changes safely, and evaluate flags at very high volume with minimal latency.

A feature flag looks like a distributed boolean, but the interesting problem is not storing true or false. It is distributing versioned policy to thousands of applications, evaluating it consistently, surviving stale caches and network partitions, and preventing a bad configuration from becoming a global outage.

## Targeted clarifying questions and assumed interviewer answers

**Candidate:** Is this a multi-tenant hosted service, an internal platform, or both?

**Interviewer:** Treat it as a multi-tenant platform used by many internal product teams. A tenant is an organization, with projects and environments such as development, staging, and production.

**Candidate:** Are flags only boolean?

**Interviewer:** Support boolean, string, number, and JSON variants. A flag returns one variant after evaluating ordered rules.

**Candidate:** Where should evaluation occur: in a central service or inside application SDKs?

**Interviewer:** Production applications need very low latency and must keep evaluating during a control-plane or network outage. Prefer local SDK evaluation.

**Candidate:** Which targeting rules are required?

**Interviewer:** Exact or set membership on attributes such as region, plan, and app version; reusable segments; and deterministic percentage rollout. Complex arbitrary scripts are out of scope.

**Candidate:** What is the target scale?

**Interviewer:** 10,000 projects, 100,000 active flags, 100,000 SDK instances worldwide, and one million evaluations per second at peak. Configuration writes are under 100 per second, but a global incident may cause many teams to update flags simultaneously.

**Candidate:** What latency and propagation targets matter?

**Interviewer:** Local evaluation should have p99 latency below 1 millisecond after initialization. A normal published change should reach 99.9% of connected production SDKs within 5 seconds and all healthy instances within 30 seconds.

**Candidate:** What consistency do users expect?

**Interviewer:** An individual SDK must evaluate one complete, immutable configuration version—never half of an update. Different SDKs may temporarily be on adjacent versions during propagation. For emergency kill switches, teams want the fastest safe distribution possible, but partitions still exist.

**Candidate:** What happens when an SDK starts and cannot reach the service?

**Interviewer:** It should use its last verified snapshot from disk. If none exists, use a code-defined default. The flag can also specify whether its safe fallback is on or off.

**Candidate:** Do we need an approval workflow?

**Interviewer:** Production publishes should support optimistic concurrency, optional two-person approval, scheduled activation, audit history, and rollback. Keep the workflow simple enough for an interview.

**Candidate:** Does the service collect evaluation analytics?

**Interviewer:** Yes, but asynchronously. Teams need approximate exposure counts and debugging information. Analytics failure must never block evaluation.

**Candidate:** Must percentage rollout remain stable as traffic changes?

**Interviewer:** Yes. A user assigned to the 10% treatment should remain assigned when requests move across machines. Increasing from 10% to 20% should retain the original 10%.

**Candidate:** Should secrets be used as targeting attributes?

**Interviewer:** No. SDK contexts may include identifiers and approved attributes, but secrets and highly sensitive personal data are out of scope. Server SDKs can receive full rules; untrusted client SDKs need a restricted or proxied model.

**Candidate:** I will focus on authoring and publishing versioned flag configurations, distributing immutable snapshots to server-side SDKs, and evaluating locally with deterministic targeting. I will also cover caches, emergency propagation, analytics, safety, and failure behavior. Is that the right scope?

**Interviewer:** Yes.

---

# 1. Requirements (~5 minutes)

## 1.1 Core functional requirements

Prioritize three complete user journeys:

1. **Create and edit a flag draft.** An authorized developer defines variants, ordered targeting rules, a percentage rollout, prerequisites, and a safe fallback.
2. **Publish or roll back an environment configuration.** A production change uses optimistic concurrency, validation, optional approval, immutable versioning, audit history, and fast distribution.
3. **Evaluate a flag in an application.** An SDK evaluates locally from a verified snapshot, returns the chosen variant and reason, and asynchronously records an exposure event.

These requirements naturally separate the system into a write-heavy correctness-oriented **control plane** and a read-heavy, highly available **evaluation plane**.

**Say aloud:**

> “I am treating the product as a versioned configuration system. The three operations are edit a draft, atomically publish or roll back an environment version, and evaluate locally from a snapshot. The critical separation is a strongly consistent control plane from an availability-first evaluation plane.”

## 1.2 Quantified non-functional requirements

1. **Evaluation latency:** p99 below 1 millisecond after SDK initialization; evaluation must not make a network call.
2. **Scale:** one million evaluations per second, 100,000 connected SDK instances, 100,000 active flags, and fewer than 100 configuration writes per second.
3. **Propagation:** 99.9% of connected production SDKs receive a normal publish within 5 seconds; all healthy connected instances within 30 seconds.
4. **Availability and durability:** local evaluation remains available during service or network failure; control-plane APIs target 99.95% availability; published versions and audit records are durable across an availability-zone failure.
5. **Safety and consistency:** each SDK evaluates an atomic immutable snapshot; publish operations do not lose concurrent edits; deterministic rollouts remain sticky across processes and regions.

## 1.3 Scope

**In scope**

- Multi-tenant projects and environments
- Typed variants
- Ordered attribute rules and reusable segments
- Deterministic percentage rollout
- Draft, validation, approval, publish, rollback
- Server-side SDK snapshots and local evaluation
- Streaming invalidation plus pull fallback
- Approximate exposure analytics
- Audit logs and emergency kill-switch behavior

**Out of scope**

- Arbitrary user-provided code in rules
- Full experimentation statistics or causal analysis
- Secret management
- Mobile-app release management
- General-purpose dynamic configuration beyond flag variants
- Highly sensitive targeting attributes
- Complete client-side rule delivery to untrusted browsers

## 1.4 Consistency and availability contract

| Concern | Contract |
|---|---|
| Draft edits | Strong consistency per flag draft using a revision or ETag. Concurrent stale writes fail instead of overwriting. |
| Publish | Linearizable per environment. One transaction creates an immutable ConfigVersion and advances the environment’s active_version. |
| SDK snapshot | Atomic and immutable. The SDK switches from version N to N+1 only after downloading, validating, and checksum-verifying the complete snapshot. |
| Cross-instance visibility | Eventual consistency. Healthy connected instances target 5-second propagation; a partitioned SDK may stay on its last valid snapshot. |
| Evaluation | Availability over freshness. A stale verified snapshot is preferred to a network dependency or malformed new configuration. |
| Percentage assignment | Deterministic for the tuple project, environment, flag, salt, and targeting key. |
| Analytics | At-least-once delivery with deduplication or approximate aggregation; delayed or lost analytics never changes a flag result. |
| Emergency change | Same atomic version model, distributed through a priority channel. “Emergency” reduces queueing but cannot defeat a network partition. |

This is not globally linearizable evaluation. Requiring every evaluation worldwide to observe the latest write would put a remote quorum on the hot path, violating the sub-millisecond and outage-survival requirements.

**Say aloud:**

> “I will make publishing strongly consistent and evaluation highly available. Each SDK sees a complete version, but two SDKs may temporarily see adjacent versions. During a partition the SDK serves the last verified snapshot, because stale configuration is safer than a remote dependency or a partial update.”

## 1.5 Scale estimates that change design decisions

Hello Interview’s delivery guidance recommends doing arithmetic when it affects the design, rather than calculating numbers only to conclude that the system is large.

### Central evaluation is infeasible

At one million evaluations per second, even a small 300-byte request and response pair produces roughly:

\[
1{,}000{,}000 \times 300\ bytes = 300\ MB/s
\]

That excludes TLS, headers, retries, and cross-region latency. More importantly, every application decision would depend on a network service. This directly selects **local SDK evaluation**.

### Snapshot distribution bandwidth

Assume an average compressed environment snapshot is 2 MB and 100,000 SDK instances are connected. Sending a full snapshot to everyone after each publish would transfer:

\[
2\ MB \times 100{,}000 = 200\ TB
\]

per globally relevant update. In reality, SDKs subscribe only to the project and environment they use, but large tenants can still have thousands of instances. We therefore need:

- subscription scoping,
- a CDN or regional snapshot cache,
- a tiny invalidation message followed by pull,
- conditional GETs and delta support as an optimization,
- coalescing when multiple versions publish quickly.

### Stream connection scale

100,000 long-lived connections are manageable, but not on one process. If one stream gateway handles 10,000 connections with headroom, we need at least 10 gateways globally, then multiply for regions, deployments, and failure capacity. This selects a horizontally scaled regional streaming tier rather than database polling from every SDK.

### Analytics volume

At one million evaluations per second, logging one event per evaluation produces 86.4 billion events per day. Even at 100 bytes per compact event, that is 8.64 TB/day before replication. Exact raw logging is expensive and unnecessary for most flags. SDK-side aggregation and sampling materially change the analytics design.

**Say aloud:**

> “The one-million-per-second figure rules out central evaluation. The bandwidth math also rules out pushing full snapshots over every stream. I will push small version notifications, let SDKs pull scoped snapshots through regional caches, and aggregate analytics inside the SDK.”

---

# 2. Core entities (~2 minutes)

Identify the minimum nouns before expanding the schema.

| Entity | Purpose | Essential fields |
|---|---|---|
| **Project** | Tenant-owned namespace | project_id, organization_id, name |
| **Environment** | Independent configuration boundary | environment_id, project_id, name, active_version, approval_policy |
| **Flag** | Stable logical key and metadata | flag_id, project_id, key, type, archived, salt |
| **FlagDraft** | Mutable proposed definition | flag_id, environment_id, revision, variants, rules, fallback, updated_by |
| **Segment** | Reusable audience definition | segment_id, environment_id, revision, included_keys, excluded_keys, rules |
| **ConfigVersion** | Immutable published snapshot | environment_id, version, blob_uri, checksum, schema_version, created_at |
| **PublishRequest** | Validation and approval state | publish_id, base_version, proposed_hash, state, approvals |
| **SDKInstance** | Optional connection and diagnostics identity | instance_id, project_id, environment_id, sdk_version, last_seen_version |
| **ExposureAggregate** | Batched analytics record | interval, flag_id, version, variant, reason, count |

The important distinction is between a mutable draft and an immutable published version. Runtime SDKs never read draft rows.

**Say aloud:**

> “Flag is the stable identity, FlagDraft is mutable authoring state, and ConfigVersion is an immutable runtime snapshot. That separation lets the authoring schema evolve without exposing partially edited rules to applications.”

---

# 3. API or system interface (~5 minutes)

The management API uses REST because it maps cleanly to resources and is not on the evaluation hot path. The server SDK exposes an in-process library call. Identity and organization membership come from the authenticated principal; request bodies do not supply a trusted actor or tenant ID.

## 3.1 Create a flag draft

~~~http
POST /v1/projects/proj_checkout/environments/production/flags
Authorization: Bearer <management-token>
Idempotency-Key: create-new-checkout-v1
Content-Type: application/json
~~~

~~~json
{
  "key": "new_checkout",
  "type": "STRING",
  "variants": {
    "control": "legacy",
    "treatment": "one_page"
  },
  "safe_fallback_variant": "control",
  "rules": [
    {
      "rule_id": "enterprise-us",
      "conditions": [
        {"attribute": "plan", "operator": "IN", "values": ["enterprise"]},
        {"attribute": "region", "operator": "IN", "values": ["US", "CA"]}
      ],
      "serve": {"variant": "treatment"}
    }
  ],
  "default_serve": {
    "rollout": [
      {"variant": "treatment", "weight": 1000},
      {"variant": "control", "weight": 9000}
    ],
    "bucket_by": "user_id"
  }
}
~~~

Weights use a fixed 10,000-bucket space. A weight of 1,000 means 10%.

~~~json
{
  "flag_id": "flag_7F3",
  "key": "new_checkout",
  "draft_revision": 1,
  "environment_active_version": 182,
  "validation": {"state": "VALID"}
}
~~~

## 3.2 Update a draft with optimistic concurrency

~~~http
PUT /v1/projects/proj_checkout/environments/production/flags/new_checkout/draft
Authorization: Bearer <management-token>
If-Match: "draft-revision-1"
Content-Type: application/json
~~~

The body contains the complete proposed definition. Returning **412 Precondition Failed** for an old revision prevents two editors from silently overwriting each other.

## 3.3 Publish a configuration

~~~http
POST /v1/projects/proj_checkout/environments/production/publishes
Authorization: Bearer <management-token>
Idempotency-Key: publish-checkout-20pct
Content-Type: application/json
~~~

~~~json
{
  "base_version": 182,
  "flag_keys": ["new_checkout"],
  "change_ticket": "CHG-4812",
  "mode": "NORMAL",
  "scheduled_for": null
}
~~~

~~~json
{
  "publish_id": "pub_91A",
  "state": "AWAITING_APPROVAL",
  "base_version": 182,
  "proposed_version": 183,
  "required_approvals": 2,
  "current_approvals": 1
}
~~~

After approval:

~~~http
POST /v1/publishes/pub_91A/approvals
Authorization: Bearer <management-token>
Content-Type: application/json
~~~

~~~json
{
  "decision": "APPROVE"
}
~~~

A successful activation returns:

~~~json
{
  "publish_id": "pub_91A",
  "state": "ACTIVE",
  "environment_version": 183,
  "checksum": "sha256:4f6d...",
  "propagation": {
    "connected_instances_seen": 98640,
    "connected_instances_total": 99102
  }
}
~~~

Rollback creates a new immutable version based on an older snapshot rather than moving the pointer backward:

~~~http
POST /v1/projects/proj_checkout/environments/production/rollbacks
Authorization: Bearer <management-token>
Idempotency-Key: rollback-incident-771
Content-Type: application/json
~~~

~~~json
{
  "restore_from_version": 182,
  "mode": "EMERGENCY",
  "reason": "Checkout error rate exceeded 5%"
}
~~~

This may produce version 184 whose content matches 182. Monotonic version numbers make ordering, caches, and audits simpler.

## 3.4 SDK initialization and local evaluation

The application initializes once:

~~~python
client = FeatureFlagClient(
    sdk_key=os.environ["FEATURE_FLAG_SDK_KEY"],
    bootstrap="last_verified",
    stream_updates=True
)
~~~

The hot-path call is in-process:

~~~python
decision = client.evaluate(
    flag_key="new_checkout",
    context={
        "user_id": "user_2841",
        "region": "US",
        "plan": "enterprise",
        "app_version": "7.4.1"
    },
    default="legacy"
)
~~~

Result:

~~~json
{
  "value": "one_page",
  "variant": "treatment",
  "reason": "RULE_MATCH",
  "rule_id": "enterprise-us",
  "config_version": 183
}
~~~

The SDK key identifies project, environment, and permissions. It is not included in each evaluation context.

## 3.5 Snapshot and update protocol

SDK snapshot fetch:

~~~http
GET /sdk/v1/environments/env_prod/snapshot
Authorization: SDK-Key <server-sdk-key>
If-None-Match: "version-182-sha256-a33c"
Accept-Encoding: zstd, gzip
~~~

Responses:

- **304 Not Modified** if current
- **200 OK** with immutable snapshot, version, checksum, signature, and cache headers
- **410 Gone** if the SDK/schema is too old and must upgrade

The long-lived update stream carries only metadata:

~~~text
event: config.version
data: {"environment_id":"env_prod","version":183,"checksum":"sha256:4f6d...","priority":"NORMAL"}
~~~

The SDK then fetches the snapshot through its nearest regional endpoint.

**Say aloud:**

> “Management uses revisioned REST resources. Runtime evaluation is not an RPC—it is an in-process function. Streams carry version notifications rather than entire configurations, and the SDK pulls an immutable snapshot with ETag and checksum validation.”

---

# 4. Data flow

A configuration service benefits from showing two separate flows.

## Publish flow

1. Developer edits a draft using a revision precondition.
2. Publish service loads the base ConfigVersion plus selected drafts.
3. Validator checks types, rule operators, segment references, prerequisites, limits, and cycles.
4. Approval policy is evaluated.
5. Snapshot compiler produces a canonical immutable document.
6. Object storage receives the blob; checksum and signature are computed.
7. One database transaction inserts ConfigVersion metadata, advances Environment.active_version, and writes an audit/outbox record.
8. Outbox publisher emits a version event.
9. Regional stream gateways notify subscribed SDKs.
10. SDKs fetch, verify, atomically swap, and acknowledge the version.

## Evaluation flow

1. Application calls the SDK with flag key, context, and code default.
2. SDK reads the current snapshot pointer without locking the evaluation path.
3. It locates the flag, evaluates prerequisites and ordered rules, and selects a variant.
4. For a rollout, it hashes the flag salt and targeting key into a stable bucket.
5. It returns value, variant, reason, and version.
6. It updates an in-memory exposure counter.
7. A background thread periodically flushes aggregated analytics; failures are dropped or retried within a bounded buffer.

---

# 5. High-level design (~10–15 minutes)

Build the simplest complete working system one operation at a time.

## 5.1 Compact architecture

~~~mermaid
flowchart TB
    M["Management clients"] --> CP["Control plane"]
    CP --> DB[("Metadata DB")]
    CP --> OBJ[("Immutable snapshot store")]
    CP --> BUS[("Version event log")]

    BUS --> SG["Regional stream gateways"]
    OBJ --> CDN["Regional cache / CDN"]

    SG --> SDK["Application SDKs"]
    CDN --> SDK
    SDK --> APP["Local evaluation"]
    SDK --> ANA[("Analytics ingest")]
~~~

The control plane is not on the evaluation hot path. Applications continue evaluating from their local immutable snapshot even if the database, event log, stream gateway, or Internet connection is unavailable.

## 5.2 Operation 1: create and edit a draft

### Baseline components

- API gateway for authentication, authorization, rate limits, and idempotency
- Flag service for resource validation
- Relational metadata database
- Audit log

### Why relational storage?

The write rate is small, while correctness requires:

- unique flag keys within a project,
- transactions,
- optimistic concurrency,
- approvals and audit relationships,
- referential checks for segments and prerequisites.

A relational database is a strong default. Runtime evaluation does not query it.

### Request trace

1. Gateway authenticates the user and derives organization, role, and allowed projects.
2. Flag service validates the flag type, variant values, rule operators, attribute allowlist, and rollout weights summing to 10,000.
3. The database transaction:
   - inserts or updates FlagDraft,
   - increments draft_revision,
   - stores the idempotency result,
   - writes an AuditEntry.
4. The API returns the draft revision and validation status.

### Relevant fields introduced beside the component

**Flag**

- flag_id
- project_id
- key; unique within project
- type
- salt
- archived_at

**FlagDraft**

- environment_id + flag_id
- revision
- enabled
- variants_json
- ordered_rules_json
- default_serve_json
- safe_fallback_variant
- updated_by
- updated_at

**AuditEntry**

- audit_id
- actor_id
- organization_id
- resource_type and resource_id
- action
- before_hash and after_hash
- reason
- created_at

The salt is generated once when the flag is created. If changing a rollout percentage also changed the salt, users would be randomly reassigned. Stable salt preserves stickiness.

**Interviewer:** Why store rule JSON rather than normalize every condition?

**Candidate:** Authoring reads and writes a flag definition as one document, rule shapes may evolve, and write QPS is low. JSON in a transactional row simplifies atomic revisioning. I would separately index metadata used for search, such as project, environment, key, state, and updated_at. The compiled runtime snapshot removes database concerns from evaluation.

**Say aloud:**

> “The authoring database optimizes for correct revisioned writes, not runtime reads. A flag definition can be a JSON document inside a relational row, while keys, revisions, and workflow metadata remain indexed columns.”

## 5.3 Operation 2: publish and distribute a version

### Snapshot compiler

The compiler transforms authoring objects into a deterministic runtime format:

- only fields required for evaluation,
- flags keyed by stable flag key,
- segments expanded or referenced by compact ID,
- typed operators converted to runtime opcodes,
- prerequisite graph topologically ordered,
- canonical ordering for stable checksums,
- schema_version for SDK compatibility.

A snapshot might look like:

~~~json
{
  "environment_id": "env_prod",
  "version": 183,
  "schema_version": 4,
  "flags": {
    "new_checkout": {
      "type": "STRING",
      "salt": "e741...",
      "fallback": "control",
      "variants": {
        "control": "legacy",
        "treatment": "one_page"
      },
      "rules": [
        {
          "id": "enterprise-us",
          "all": [
            ["plan", "IN", ["enterprise"]],
            ["region", "IN", ["US", "CA"]]
          ],
          "serve": ["VARIANT", "treatment"]
        }
      ],
      "default": [
        "ROLLOUT",
        "user_id",
        [["treatment", 1000], ["control", 9000]]
      ]
    }
  }
}
~~~

### Validation before publish

Reject a publish if:

- variants do not match the declared type,
- a rule references an unknown segment or attribute,
- rollout weights do not total 10,000,
- prerequisite dependencies form a cycle,
- a flag references itself,
- the snapshot exceeds project limits,
- the target SDK compatibility floor cannot understand the schema,
- base_version is no longer current.

### Atomic publish trace

1. Publish service reads Environment.active_version = 182.
2. It verifies request.base_version = 182.
3. It compiles and validates the proposed version.
4. It writes the blob to immutable object storage under a content-addressed key.
5. In one database transaction it:
   - inserts ConfigVersion 183 with blob URI and checksum,
   - conditionally changes active_version from 182 to 183,
   - marks PublishRequest active,
   - inserts AuditEntry and OutboxEvent.
6. If the conditional update affects zero rows, another publish won; return 409 and require rebase.
7. Outbox publisher sends the version event to a durable event log.
8. Regional distributors fan the notification to only the environment’s subscribers.

Writing the object before the transaction can leave an unreferenced blob if the transaction fails. That is harmless and later garbage-collected. Advancing the database pointer before the blob is durable would be unsafe.

### Distribution strategy

Do not push a multi-megabyte snapshot down every persistent stream. Push:

- environment ID
- monotonically increasing version
- checksum
- priority

SDKs independently fetch through a CDN/regional cache. The immutable snapshot URL can include version and checksum, making it safe to cache aggressively.

**Pull fallback:** An SDK also polls periodically with jitter—for example every 60 seconds. If the stream is broken or an event is missed, polling converges it.

**Reconnect:** On reconnect the SDK sends its current version. The gateway replies with the latest version rather than replaying every intermediate publish. Evaluation needs current state, not an event history.

**Coalescing:** If versions 183, 184, and 185 publish faster than an SDK can download, it may skip directly from 182 to 185. ConfigVersion remains auditable, but runtime need not apply obsolete intermediate states.

### Atomic SDK swap

The SDK:

1. downloads to a temporary buffer or file,
2. verifies signature, checksum, environment, schema compatibility, and limits,
3. parses the complete snapshot,
4. constructs immutable indexes,
5. atomically swaps one pointer from old snapshot to new snapshot,
6. persists the verified snapshot to disk,
7. acknowledges the version.

Concurrent evaluations that already captured the old pointer finish on the old version. New evaluations see the new pointer. No request sees half old and half new.

**Interviewer:** Why do you need both the stream and polling?

**Candidate:** The stream gives low propagation latency; polling repairs missed events and avoids making the stream service the sole correctness mechanism. The snapshot source of truth is immutable storage plus the active version, while notifications are an optimization.

## 5.4 Operation 3: evaluate locally

### Evaluation algorithm

For a flag key:

1. Read the current immutable snapshot pointer.
2. If the flag is absent, return the caller’s code default with reason **FLAG_NOT_FOUND**.
3. If disabled, return the configured off or fallback variant.
4. Evaluate prerequisites with cycle protection already guaranteed by publish validation.
5. Evaluate ordered rules:
   - missing attributes make a condition false unless the operator explicitly handles absence,
   - all conditions in a rule must match,
   - first matching rule wins.
6. If the rule serves a fixed variant, return it.
7. If it serves a rollout:
   - read the declared bucket_by attribute,
   - compute a stable bucket,
   - map the bucket into cumulative weight ranges.
8. If no rule matches, evaluate default_serve.
9. Return value, variant, reason, rule ID, and config version.
10. Increment a local analytics counter.

### Deterministic bucketing

Use a stable, cross-language specified hash:

\[
bucket = H(project\_id \;||\; environment\_id \;||\; flag\_salt \;||\; targeting\_key) \bmod 10{,}000
\]

Suppose treatment owns buckets [0, 1000) for 10%. Increasing treatment to 20% changes the interval to [0, 2000), preserving the original 10%.

The hash specification must define:

- UTF-8 encoding,
- field separators or length prefixes,
- hash algorithm and version,
- unsigned integer conversion,
- modulo behavior,
- test vectors shared by every SDK language.

Using each language’s native hash is incorrect because results may differ across processes or versions.

### Runtime indexes and caching

The entire scoped snapshot lives in memory, so evaluation is an exact-key dictionary lookup plus a short rule walk. Additional indexes include:

- flag key → compiled flag
- segment ID → compiled segment
- prerequisite flag ID → compiled flag
- optional pre-parsed semantic versions and CIDR sets

Do not add a network cache such as Redis to the evaluation path. The SDK’s immutable memory snapshot is the cache and runtime database.

### Fallback hierarchy

1. Current verified in-memory snapshot
2. Last verified disk snapshot during restart or network loss
3. Code-supplied default if no snapshot or flag
4. Optional per-flag safe fallback encoded in a valid snapshot

The code default is mandatory because the application, not the remote platform, ultimately owns safe behavior when initialization fails.

**Say aloud:**

> “Evaluation is a pure local function over an immutable snapshot. Ordered rules are compiled before publish; percentage rollout uses a versioned cross-language hash and fixed bucket space. The SDK returns a reason and version for debugging and records analytics off the hot path.”

---

# 6. Deep dives (~10–15 minutes)

## Deep dive 1: propagation, caches, and emergency changes

### Where the baseline breaks

Polling alone cannot meet a 5-second propagation target without 100,000 SDKs hammering the service. Pushing full snapshots over streams creates huge fanout bandwidth and slow consumers. Relying only on streams risks permanent staleness after a missed event. CDN caches can also serve an older active pointer if cache keys are mutable.

### Plausible approaches

| Approach | Strength | Failure or cost |
|---|---|---|
| Central evaluation RPC | Immediate logical freshness | Violates latency and availability; one million RPS |
| Frequent polling | Simple and self-healing | Load spikes, slow emergency propagation, synchronized polling |
| Full snapshot push | Fast in theory | Massive fanout bandwidth, slow-client backpressure |
| Version push plus immutable pull | Small notifications, CDN-friendly, self-healing | More moving parts and temporary version skew |

### Selection

Use **version notification plus immutable pull**, with jittered polling as repair.

- Event log partitions by environment ID to preserve publish order.
- Regional stream gateways maintain environment subscription groups.
- Notification contains no rules or sensitive attributes.
- SDK pulls versioned immutable content through regional caches.
- SDK ignores any notification whose version is not greater than its current version.
- A periodic pull of the active-version manifest repairs missed notifications.

### Avoiding cache inconsistency

Cache immutable URLs:

~~~text
/snapshots/env_prod/183/sha256-4f6d.snapshot
~~~

These can have a long TTL because their content never changes. The small active-version manifest has a short TTL, must-revalidate semantics, or bypasses shared caching. The stream usually tells the SDK the exact immutable URL, avoiding a stale manifest on the fast path.

### Emergency mode

An emergency publish:

- bypasses batching/coalescing delays in the event pipeline,
- uses reserved high-priority partitions and gateway capacity,
- tells SDKs to fetch immediately instead of waiting for normal jitter,
- increases retry urgency within a bounded budget,
- exposes live acknowledgement coverage,
- can optionally send a compact signed override for a very small class of predeclared kill switches.

The last option is dangerous. A separate override protocol can violate normal version semantics. If used, restrict it to predefined boolean kill switches, sign it, expire it, record it in audit history, and merge it deterministically above the snapshot. The simpler baseline uses the normal immutable-version system with priority delivery.

### Backpressure and slow SDKs

A stream gateway should not queue every missed version per client. It stores only the latest known version for each subscribed environment. If a client cannot keep up, disconnect it; on reconnect it learns the latest version.

### Failure modes

| Failure | Behavior |
|---|---|
| Event log delayed | SDKs remain on valid old snapshot; polling eventually converges |
| Stream gateway outage | SDK reconnects elsewhere; jittered polling continues |
| CDN/regional cache outage | Fetch origin with rate limiting or retain old snapshot |
| Snapshot checksum mismatch | Reject new snapshot, keep old, alert security/operations |
| Bad but syntactically valid rule | Roll back by publishing a new monotonic version |
| Network partition | Continue evaluating last verified version; surface staleness telemetry |

### Observability

Track:

- publish-to-event-log latency
- event-to-gateway latency
- gateway-to-SDK notification latency
- snapshot fetch latency and cache hit rate
- SDK version acknowledgement distribution
- percentage on latest, previous, and older versions
- stale age by SDK version and region
- checksum or signature failures
- emergency propagation coverage over time

Do not claim “all clients updated in five seconds” without defining the denominator. Disconnected and dead processes cannot acknowledge. Report both connected coverage and last-seen fleet coverage.

**Interviewer:** Can you guarantee a kill switch reaches every application in five seconds?

**Candidate:** No distributed system can guarantee that across partitions or suspended processes. I can guarantee priority processing inside healthy infrastructure, measure connected-client coverage, and make stale behavior safe. If the business needs a hard external cutoff, enforce it at a centralized boundary such as an API gateway in addition to the application flag.

**Say aloud:**

> “Notifications accelerate convergence; immutable snapshots are the truth; polling repairs missed notifications. Emergency mode gets reserved delivery capacity, but I will not promise to defeat partitions. For truly hard shutdowns, place an independent control at a centralized enforcement point.”

## Deep dive 2: deterministic targeting, rule correctness, and security

### Where the baseline breaks

A naïve percentage rollout using random numbers changes on every request. A language-native hash assigns different variants across SDKs. Reordering rules can unexpectedly alter behavior. Cyclic prerequisites can recurse forever. Shipping all rules to browser clients leaks business logic and segment definitions.

### Stable rollout design

Use a fixed bucket range and stable hash. The rollout interval representation matters:

~~~text
treatment: [0, 1000)
control:   [1000, 10000)
~~~

When increasing treatment to 2,000 buckets, only users in [1000, 2000) move. Decreasing reverses the newest slice.

For multiple experiments using the same user, include flag salt so assignments are independent. For mutually exclusive experiments, introduce an explicit shared experiment namespace rather than accidentally sharing salt.

### Rule semantics

Specify semantics rather than leaving them to SDK authors:

- ordered first-match wins,
- all conditions inside a rule are AND by default,
- missing attribute normally means no match,
- numeric comparison never coerces arbitrary strings,
- semantic-version comparison uses a defined parser,
- time comparisons use UTC,
- set membership uses exact normalized values,
- regex availability is restricted to safe engines and length limits,
- segment include/exclude lists take defined precedence,
- prerequisite values are evaluated from the same snapshot.

The compiler emits a compatibility schema and every SDK passes a shared conformance suite. Golden test vectors include edge cases for Unicode, hashing, versions, missing attributes, and nested prerequisites.

### Rule complexity limits

Protect application latency:

- maximum flags per environment,
- maximum rules and conditions per flag,
- maximum prerequisite depth,
- maximum segment size,
- maximum regex length and execution cost,
- snapshot byte limit.

Reject pathological configuration at publish time, not while an application request is waiting.

### Server versus client SDKs

**Server SDK:** Trusted environment; may receive full targeting rules. The SDK key is a secret and can fetch server snapshots.

**Browser/mobile SDK:** Untrusted and inspectable. Options:

1. Ship only flags explicitly marked client-visible and remove sensitive rules.
2. Evaluate on a tenant-owned edge proxy and return decisions.
3. Precompute coarse public segments without user lists.
4. Never embed a server SDK key.

The chosen baseline supports full local evaluation for server SDKs and filtered snapshots for approved client-visible flags.

### Attribute privacy

The evaluation context stays inside the application process. The service does not need user attributes for local evaluation. Exposure analytics should use:

- project and environment
- flag and variant
- rule/reason category
- configuration version
- coarse SDK metadata
- optional pseudonymous sampling ID under policy

Avoid uploading full contexts. Tenant-defined attribute names can themselves reveal sensitive concepts, so management logs and analytics require access control and retention limits.

### Configuration integrity

- Snapshots are checksummed and signed.
- SDK keys are scoped to one project/environment and rotate.
- Management actions use RBAC and optional two-person approval.
- Audit entries are append-only and exported to durable storage.
- The compiler uses deterministic serialization before signing.
- SDK rejects unknown signature keys unless they arrive through a trusted key-rotation chain.

**Interviewer:** What happens if two SDK implementations disagree on a rule?

**Candidate:** That is a platform correctness incident. I would specify semantics centrally, compile to a narrow intermediate representation, version the schema and hash algorithm, publish golden test vectors, and require every SDK release to pass the same conformance suite before it is supported.

## Deep dive 3: publish safety, rollback, analytics, and multi-region reliability

### Where the baseline breaks

A configuration service can distribute mistakes faster than code deployment. Two operators may publish from the same base version. A database transaction and event publication can diverge. Exact per-evaluation analytics can overwhelm ingest. A regional database failure must not stop evaluation.

### Publish concurrency and approval

Every PublishRequest names base_version. Activation uses a compare-and-swap:

~~~sql
UPDATE environment
SET active_version = 183
WHERE environment_id = 'env_prod'
  AND active_version = 182;
~~~

If zero rows change, the publish is stale. The user rebases and reviews the diff against the new active version.

For protected production environments:

- author cannot provide the second approval,
- approval binds to proposed_hash, so later edits invalidate it,
- scheduled activation rechecks base_version at execution time,
- service accounts have narrowly scoped roles,
- emergency publish can reduce required approval only through an audited break-glass role.

### Transactional outbox

The database transaction that advances active_version also inserts an OutboxEvent. A relay publishes outbox rows to the version log and marks them delivered. If it crashes after publishing but before marking, it publishes again. Consumers deduplicate by environment_id + version.

This avoids a dangerous dual write:

1. database commit succeeds,
2. process crashes before publishing,
3. SDKs never learn about the version.

At-least-once event publication is acceptable because version events are idempotent.

### Rollback strategy

Do not mutate ConfigVersion 183 or point active_version backward. Compile version 184 from the known-good content of 182. Benefits:

- version numbers remain monotonic,
- streams and caches never need reverse-order logic,
- rollback actor and reason are explicit,
- analytics can distinguish original and restored periods,
- history is append-only.

### Analytics pipeline

The SDK aggregates counts in memory by:

- time bucket
- flag ID
- variant
- evaluation reason
- config version
- optional sampled context dimensions

It flushes compressed batches every few seconds with a bounded disk or memory buffer. The ingest service authenticates SDK scope and writes to a partitioned event log. Stream processors aggregate into an analytical store.

Analytics semantics can be approximate:

- retries may duplicate a batch,
- SDK crash may lose the latest buffer,
- sampling reduces rare-event precision,
- offline SDKs upload later.

Include batch_id and sequence number for deduplication when exactness is worth the cost. Product decisions must not assume exposure dashboards are a transactional ledger.

### Multi-region control plane

Use a primary write region for each organization or project, with synchronous multi-AZ replication and asynchronous cross-region replicas. Management traffic routes to the owner region. This avoids cross-region consensus for a write rate below 100 per second.

Published snapshots replicate to regional object storage or a global object store. Version events replicate to regional consumers. If the owner region fails:

- evaluation continues locally everywhere,
- publishing pauses briefly or fails over to a promoted region,
- the promoted region uses fencing tokens/epochs so the old writer cannot resume and create split brain,
- new versions continue monotonically.

The evaluation-plane SLO should be much stronger than the management-plane SLO because cached snapshots isolate applications from control-plane failure.

### Disaster recovery

- Metadata database: point-in-time recovery and cross-region replica
- Snapshot store: versioning, checksums, cross-region replication
- Audit log: append-only export with retention
- Event log: replayable from outbox and ConfigVersion table
- Stream state: disposable; clients reconnect and compare versions
- Analytics: loss-tolerant by contract, with replicated raw batches where required

### SLOs and alerts

**Control plane**

- publish success and conflict rates
- validation latency
- approval age
- outbox oldest-unpublished age
- snapshot compile and replication latency

**Evaluation plane**

- SDK initialization success
- local evaluation latency by SDK language/version
- fallback/default rate
- snapshot age
- connected stream count and reconnect rate
- version propagation percentiles

**Safety**

- emergency publishes
- rapid publish/rollback sequences
- change failure rate
- percentage of evaluations on fallback
- rule evaluation errors
- unknown flag lookups

**Interviewer:** Would you make the control-plane database multi-primary?

**Candidate:** Not initially. Writes are tiny compared with evaluations, and correctness plus auditability matter more than write locality. A single owner region per project with multi-AZ durability is simpler. Evaluation is already regional and offline-capable. I would add carefully fenced ownership transfer before true multi-primary writes.

**Say aloud:**

> “Publishing is a compare-and-swap from one active version to the next, with a transactional outbox for distribution. Rollback creates a new monotonic version. SDK analytics are aggregated and failure-isolated, and a control-plane regional outage does not stop local evaluations.”

---

# 7. Final design, requirements check, follow-ups, and pitfalls

## 7.1 Concise final design

The design separates a correctness-focused control plane from an availability-focused local evaluation plane.

- Developers edit revisioned FlagDrafts in a relational database.
- Publish service validates rules, approval, compatibility, rollout weights, and dependency cycles.
- Snapshot compiler writes a canonical immutable signed ConfigVersion to object storage.
- A transaction conditionally advances Environment.active_version and writes audit plus outbox state.
- A durable event log fans small version notifications to regional stream gateways.
- SDKs pull scoped immutable snapshots through regional caches, verify them, build indexes, and atomically swap one pointer.
- Evaluations are in-process dictionary lookups and compiled rule walks with deterministic 10,000-bucket rollout.
- Last verified snapshots and code defaults keep applications working through outages.
- Jittered polling repairs missed stream events.
- SDKs batch or sample exposure analytics so observability never blocks evaluation.
- Rollback publishes a new monotonic version based on prior content.

## 7.2 Requirements check

| Requirement | Design mechanism |
|---|---|
| Create and edit flag drafts | Relational metadata DB, typed validation, draft revisions, ETags |
| Publish and roll back | Immutable ConfigVersion, compare-and-swap activation, approval, audit, monotonic rollback |
| Evaluate flags | In-process SDK, compiled rules, atomic snapshot pointer |
| p99 below 1 ms | No network call, memory indexes, bounded rule complexity |
| One million evaluations/s | Work distributed across application processes |
| 5-second connected propagation | Version event log and regional stream gateways |
| Outage survival | Last verified memory/disk snapshot and code default |
| Atomic configuration | Whole-snapshot verification and pointer swap |
| Sticky percentage rollout | Stable salt, specified cross-language hash, fixed bucket ranges |
| Safe concurrent publishing | base_version precondition and conditional update |
| Scalable analytics | SDK aggregation, sampling, asynchronous ingest |
| Auditable operation | Immutable versions, append-only audit entries, publish metadata |

## 7.3 Realistic interviewer follow-up questions with strong sample answers

### 1. Why not store every flag in Redis and have applications read it directly?

**Strong answer:**

Redis can serve exact-key reads quickly, but one million network calls per second still add latency, cost, and a hard dependency. It also makes multi-flag atomicity difficult: an SDK could read some flags before an update and others after. An immutable local snapshot gives sub-millisecond evaluation, outage survival, and a consistent version boundary. Redis could help the control plane or regional manifests, but not the evaluation hot path.

### 2. How would you support a flag that depends on another flag?

**Strong answer:**

Model prerequisites explicitly in the compiled snapshot. Publish validation detects cycles and limits depth. Evaluation uses the same snapshot for both flags and memoizes prerequisite decisions within one evaluation call. If a prerequisite does not return the required variant, serve the dependent flag’s configured fallback with reason **PREREQUISITE_FAILED**.

### 3. Can different requests for the same user receive different variants?

**Strong answer:**

They should not if every SDK uses the same snapshot, targeting key, salt, and hash specification. They can differ temporarily if SDKs are on adjacent config versions or the application supplies inconsistent contexts. Return config version and reason with decisions, provide hash test vectors, and expose SDK-version diagnostics to debug this.

### 4. How would you support scheduled rollouts?

**Strong answer:**

A scheduler owns leases on pending PublishRequests. At activation time it revalidates approval, proposed hash, permissions, and base_version, then runs the normal publish transaction. If base_version changed, it pauses for rebase instead of blindly overwriting newer configuration. Scheduling does not create a second activation path.

### 5. How would you make a gradual rollout automatic?

**Strong answer:**

Represent the rollout plan as a sequence such as 1%, 5%, 20%, 50%, 100%, with wait periods and guardrail queries. An orchestration service publishes each step as a new version after checking error and latency metrics. It pauses or publishes a rollback version on guardrail failure. The flag service remains the versioned actuation layer; monitoring owns health evidence.

### 6. What if a snapshot is too large for mobile clients?

**Strong answer:**

Do not ship the server snapshot. Produce a filtered client snapshot containing only client-visible flags and non-sensitive rules, scoped further by application. Use background refresh, compression, and version deltas if necessary. If targeting depends on confidential data, evaluate on a trusted edge or application backend.

### 7. Would you distribute deltas instead of full snapshots?

**Strong answer:**

Full immutable snapshots are the correctness baseline because they are easy to verify and recover. Deltas reduce bandwidth for large environments but require a base-version check, ordered application, checksum verification of the reconstructed result, and fallback to a full snapshot when any delta is missing. I would add deltas only after measuring snapshot bandwidth as a bottleneck.

### 8. How do you prevent a bad rule from taking down every SDK?

**Strong answer:**

Compile rules into a constrained versioned intermediate representation, enforce complexity and size limits, run shared conformance tests, canary the snapshot to internal SDKs, and validate checksums and schema before swap. SDK parsing happens off the evaluation thread. If parsing fails, it retains the old snapshot and reports the rejected version.

### 9. What is the behavior if a user lacks the bucket_by attribute?

**Strong answer:**

Make it explicit in the rule contract. The rollout cannot assign that user, so evaluation continues to the next rule or the configured fallback. Do not silently use a random value because that destroys stickiness. The returned reason should identify a missing targeting key.

### 10. How would you support extremely large allowlists?

**Strong answer:**

Do not embed millions of identifiers in every snapshot. Options include compiling a probabilistic membership filter with an exact confirmation service, assigning users to a durable segment attribute upstream, or using a server-side edge evaluator. Which one I choose depends on false-positive tolerance and latency. For ordinary segments, compact sets in the snapshot are simpler.

### 11. How do you expire a temporary flag?

**Strong answer:**

Store owner, purpose, created_at, and optional expires_at metadata. Send warnings before expiry and surface stale flags in dashboards. Automatic runtime deletion is risky, so expiry normally blocks new publishes or creates a review task. A separate policy may automatically turn off flags whose safe fallback is explicitly declared and approved.

### 12. How would you test the distribution system?

**Strong answer:**

Use deterministic SDK conformance tests, publish-to-ack latency tests across regions, missed-event and reconnect tests, corrupted-snapshot tests, clock-skew tests, stream gateway failure, CDN failure, database failover, rapid publish coalescing, and network partitions. Verify that applications keep evaluating the last valid version and that no SDK exposes a partial snapshot.

## 7.4 Common pitfalls

1. **Calling a central service for every evaluation.** This violates the latency and availability contract.
2. **Treating a flag as one database boolean.** Real systems need typed variants, rules, versions, workflow, and diagnostics.
3. **Pushing full snapshots over every stream.** Push a version notification and pull immutable content through caches.
4. **Using polling alone without jitter.** It propagates slowly or creates synchronized load spikes.
5. **Using only streams.** Missed events must be repaired by version comparison and pull.
6. **Mutating published versions.** Immutable versions make caching, audit, and rollback reliable.
7. **Moving the active pointer backward during rollback.** Publish a new monotonic version from old content.
8. **Using random rollout assignment.** It changes on every evaluation.
9. **Using a language-native hash.** Cross-language results will disagree.
10. **Changing the flag salt with the percentage.** Existing users will be reshuffled.
11. **Applying flags independently during an update.** The SDK must atomically swap a whole snapshot.
12. **Claiming instant global consistency.** Network partitions and suspended processes make this impossible.
13. **Logging every evaluation synchronously.** Analytics must be batched, sampled, and failure-isolated.
14. **Sending sensitive segments to browsers.** Client snapshots need strict filtering or trusted proxy evaluation.
15. **Skipping complexity limits.** A pathological rule can damage application latency.
16. **Ignoring concurrent publishes.** Use base-version compare-and-swap, not last-write-wins.
17. **Making the event bus the source of truth.** Immutable snapshots and the active version are authoritative; events accelerate convergence.
18. **Promising that a feature flag is a hard safety control.** For non-bypassable shutdowns, add enforcement at a centralized boundary.

## Closing interview summary

**Candidate:**

> “The central decision is local evaluation from immutable versioned snapshots. Authors edit revisioned drafts, and a strongly consistent publish transaction advances one environment version while writing audit and outbox state. Regional streams notify SDKs of a new version; SDKs pull through caches, verify the complete snapshot, and atomically swap it. Evaluation is a pure in-process rule walk with deterministic bucketing, while analytics runs asynchronously. This gives sub-millisecond decisions and outage survival without hiding the fact that global propagation is eventually consistent.”
