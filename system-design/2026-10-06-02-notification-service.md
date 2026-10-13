# 2. Design a Large-Scale Notification Service (Medium/Hard)

## Interviewer prompt
“Design a service that sends transactional and product notifications through push, email, and SMS. Support preferences, retries, scheduled messages, and high-volume fan-out.”

## Clarifying questions and assumed answers
- Are payment/security alerts higher priority than marketing? Yes, with different quotas and latency targets.
- Delivery guarantee? At least once to provider APIs, with best-effort deduplication; exactly-once appearance in a user's inbox is not generally possible across providers.
- What counts as delivered? A provider acceptance and an actual device/email delivery are different states.
- Can users opt out? Yes for optional messages; mandatory security/transactional cases follow policy.
- Do we need ordering? Per user and notification type where meaningful, not global ordering.
- Scale? 200 million users, 500 million notification intents per day, and 10x spikes.

## Candidate walkthrough

### 1. Requirements and scale
Producers submit a notification intent with recipient, template, data, priority, channel policy, and idempotency key. The service checks preferences and consent, chooses channels, renders localized content, schedules or dispatches it, retries transient failures, and exposes status. A product goal might be 99% of high-priority intents handed to a provider within 30 seconds, subject to provider health. That is not an end-user delivery SLA.

Five hundred million/day averages about 5,800 intents/s; a 10x spike is roughly 58,000/s. At two channel attempts per intent, peak provider operations could be roughly 116,000/s before retries, far beyond many provider quotas. We must buffer, prioritize, and throttle per provider and tenant. If each intent record is 500 bytes, raw intent storage is 250 GB/day before indexes and retention; compact or tier old events.

### 2. API and data model
POST /v1/notifications takes {recipient_id, template_id, template_version?, variables, priority, channel_policy, scheduled_at?, idempotency_key}. Return 202 with notification_id and status. GET /v1/notifications/{id} returns state and per-channel attempts. PUT /v1/users/{id}/preferences updates consent, quiet hours, and channel choices.

NotificationIntent(id, tenant_id, recipient_id, category, template_version, variables_ref, priority, scheduled_at, state, created_at, idempotency_key) holds the logical request. DeliveryAttempt(id, intent_id, channel, provider, attempt_no, state, provider_message_id, next_retry_at, error_code) records each outbound attempt. Preference(user_id, category, channel, enabled, quiet_hours, version) and DeviceToken(user_id, token, platform, state) support routing. Store sensitive variables minimally and encrypt at rest.

### 3. Architecture
Producer -> authenticated ingestion API -> durable intent store plus outbox event -> priority queues -> policy/renderer workers -> channel-specific queues -> provider adapters -> email/SMS/push providers. Provider callbacks enter a separate webhook handler, which validates signatures and updates attempt states. A status API reads durable state, while metrics and analytics consume events asynchronously.

Use an outbox row committed in the same transaction as the intent so a crash after accepting a request cannot silently lose dispatch. A relay publishes outbox records and marks them sent. Duplicate publication is expected; workers check an idempotency ledger. If the durable queue is the source of truth, the API should not acknowledge until its append is durable and the corresponding status record is recoverable.

### 4. Preferences, scheduling, and fan-out
Resolve the applicable policy at dispatch time so recent opt-outs take effect. For scheduled work, persist scheduled_at and index or bucket due messages; a scheduler moves due IDs into queues in bounded batches. Quiet hours depend on the user's timezone and daylight saving rules; define exceptions for urgent categories. Large campaigns should be represented as a campaign plus a recipient cursor, with fan-out workers generating bounded batches, not one gigantic producer request. Reserve capacity for high-priority traffic so marketing bursts cannot starve security alerts.

Template rendering is versioned and locale-aware. Validate variables at ingestion, enforce content size, and avoid putting secret personal data into logs or queue payloads. A stable notification ID lets the user's in-app inbox deduplicate independently from external provider behavior.

### 5. Retry and delivery semantics
Use exponential backoff with jitter for timeouts, 429s, and 5xx responses, respecting Retry-After. Treat invalid addresses, revoked device tokens, and policy refusals as permanent failures. Retry within an explicit deadline and attempt cap; then move the intent to a dead-letter queue for inspection and possible controlled replay. A timeout after provider acceptance creates ambiguity: retry may produce a duplicate. Pass a provider idempotency key when available; otherwise log the ambiguity and accept at-least-once semantics. Do not say “exactly once” without a provider-side deduplication mechanism.

A provider's HTTP 200 means accepted, not delivered. Track accepted, delivered, bounced, opened, and failed separately, and expect callbacks to arrive late or out of order. State transitions should be monotonic where possible, with event time and provider sequence metadata.

### 6. Capacity, failure modes, and bottlenecks
Partition queues by priority and perhaps tenant or recipient hash. For per-user ordering, serialize only the subset requiring it, since a global queue partition would choke throughput. Apply per-provider token buckets and per-tenant quotas. If one provider is degraded, circuit-break its adapter and route to an approved backup if category policy permits; avoid sending duplicate messages through both providers on ambiguous timeout. If preference storage is unavailable, fail closed for marketing and define an explicit policy for critical alerts. If the queue backs up, expose age-of-oldest-message and protect urgent capacity. Callback replay must be idempotent.

### 7. Observability
Measure time from intent acceptance to provider acceptance by priority/channel, queue age and depth, retry rates by error, throttling, DLQ count, invalid token rate, cost per channel, provider acceptance/delivery/bounce rates, and opt-out violations. Trace by notification ID across intent, policy decision, attempt, and callback. Audit why a notification was sent, suppressed, or retried, while restricting payload visibility and retention.

## Concise final design
A durable ingestion/outbox path feeds priority queues. Workers evaluate current consent, render versioned templates, and dispatch through throttled provider adapters. Per-attempt state and idempotency keys make retries understandable. Scheduled fan-out runs in bounded batches, provider callbacks update delivery status, and a dead-letter queue handles exhausted retries. Capacity isolation protects urgent alerts.

## Interviewer follow-ups with strong sample answers
**Q: A provider times out after accepting an SMS. Retry?** A: The outcome is ambiguous. Query status or use a provider idempotency token if supported; otherwise a retry trades possible duplicates for possible loss. State the policy by notification category.

**Q: A user opts out while a campaign is queued.** A: Check preferences at dispatch, not only at campaign creation; cancel or suppress unsent work and record that decision.

**Q: How do you protect payment alerts during a sale?** A: Separate priority queues and worker pools, reserve provider quota, and shed or defer marketing before urgent traffic.

**Q: How do you schedule millions for 9 a.m. local time?** A: Persist due time in UTC computed from each recipient's timezone, shard due buckets, and drain with quotas. Recompute future schedules if timezone rules or user settings change under a defined policy.

**Q: Can you guarantee exactly-once delivery?** A: Not end to end across network failures and external providers. Provide idempotent intake and best-effort provider deduplication, document at-least-once attempts, and distinguish acceptance from delivery.

## Common pitfalls
Calling provider acceptance “delivery”; retrying permanent errors; omitting consent checks at send time; putting all traffic in one queue; assuming callbacks arrive in order; claiming exactly-once semantics without explaining the external boundary.