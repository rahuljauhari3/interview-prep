# 1. Design a URL Shortener (Medium)

## Interviewer prompt
“Design a service that turns a long URL into a short link and redirects visitors who open that link. Explain how it scales, handles custom aliases and expiration, and measures clicks.”

## Clarifying questions and assumed answers
- What is the read/write ratio? Assume 100 million new links and 10 billion redirects per month, roughly 100:1.
- How long do links live? Default indefinitely; optional expiration and user deletion.
- Must a short code be predictable? No. Avoid easy enumeration where practical, but do not claim the code is a security boundary.
- Do we need custom aliases? Yes, unique within a branded domain.
- What is the latency goal? Redirect p99 below 100 ms from a nearby region, excluding destination-site latency.
- Are click analytics exact? Eventual, with approximate real-time dashboards and a later reconciled count.
- Are malicious destinations in scope? Basic abuse detection, rate limits, and takedown.

## Candidate walkthrough

### 1. Requirements and scale
The write path creates, retrieves, expires, and deletes mappings. The read path resolves a code and issues an HTTP redirect. A successful creation must make the mapping durable and soon readable; a deleted or expired mapping must cease redirecting. Availability matters most on reads, but deletion correctness matters enough that cache invalidation needs an explicit design.

At 100 million writes per month, the average is about 39 writes/s; assuming a 10x busy period gives about 400/s. Ten billion redirects per month is about 3,860/s on average and perhaps 40,000/s at peak. If a mapping plus index overhead is roughly 300 bytes, a year of new mappings is about 360 GB before replicas, indexes, and compaction. Analytics is much larger: ten billion events per month at 100 bytes is about 1 TB/month before replication and aggregation. These are assumptions to validate with an interviewer, not exact forecasts.

### 2. API and data model
POST /v1/links accepts {long_url, custom_alias?, expires_at?, idempotency_key?} and returns {id, short_url, created_at}. GET /{code} redirects or returns 404/410. GET /v1/links/{id} returns owner-visible metadata. DELETE /v1/links/{id} tombstones the link. GET /v1/links/{id}/analytics returns aggregate clicks with an as_of timestamp.

Link(code PK, destination, owner_id, created_at, expires_at, state, version, domain) is the primary record. A separate unique index on (domain, custom_alias) can be incorporated into the key namespace. Analytics events carry code, timestamp, coarse geography, referrer category, and a unique event ID; avoid retaining raw IPs unnecessarily.

### 3. Architecture and read path
A regional edge or load balancer routes GET /code to stateless redirect services. Each service checks an in-process cache, then a distributed cache, then a sharded durable key-value store. It validates state and expiration, returns a 302/307 with Location, and asynchronously emits a click event to a log. A 302 is easier to change or revoke than a cacheable 301. Give the redirect response conservative cache headers when revocation matters.

The write API authenticates, validates URL scheme and destination length, checks abuse policy, allocates a code, and performs a conditional insert in the durable store. Only after commit does it return the URL. Custom aliases use conditional insertion on (domain, alias), so two concurrent callers cannot both win. An idempotency key maps repeated client requests to one creation result.

### 4. Code generation and collision handling
With a base62 alphabet, seven characters provide 62^7, about 3.5 trillion possible values. For random codes, a collision becomes increasingly likely over a large lifetime; conditional insert and retry are mandatory. Eight characters provide considerably more headroom. Another design allocates globally unique numeric IDs and base62 encodes them: no collisions, but easy enumeration and dependence on an ID allocator. I would choose random eight-character codes plus conditional insert for simplicity and add length as capacity grows. Reserve offensive words and route patterns; alias uniqueness is scoped by domain.

### 5. Storage, cache, and consistency
Partition durable data by a hash of (domain, code), not by creation time. Replicate across availability zones. A popular code may become a hot key; the edge and two cache tiers absorb it, while negative caching of nonexistent codes uses a short TTL to resist random-code scans. On deletion or destination change, write the authoritative record first, publish invalidation, and keep cache TTL modest. A race or lost invalidation can leave stale redirects for the TTL, so define and measure a revocation bound. For urgent takedowns, an edge-distributed deny list is checked before cached mappings. A local cache can be very short-lived if the bound is strict.

Do not cache expired mappings beyond expires_at. Account for clock skew by treating the store timestamp as authoritative and including expiration in cached values. Multi-region writes can be routed to a home region or use globally coordinated uniqueness; reads can use replicated data with documented lag. If read-after-write is essential, route the new link briefly to the write region or populate a cache after commit.

### 6. Analytics pipeline
Redirects should not wait for analytics. Append click events to a durable queue or log with code-based partitioning, then stream aggregate into minute/hour buckets and eventually reconcile into durable analytical storage. The queue and consumers are at-least-once, so event IDs and deduplication or idempotent bucket updates are needed for a count with a clear accuracy target. Sampling is an explicit alternative at very high volume, but it changes the product promise. Rate-limit obvious bot traffic and distinguish human versus total clicks.

### 7. Bottlenecks, failures, and observability
A flash crowd on one link stresses cache fill and a single DB key: use request coalescing, edge caching, and replicated cache capacity. If the cache fails, shed abusive requests, bound DB concurrency, and continue serving uncached links within capacity. If the DB is unavailable, existing cached links can still work briefly, while writes fail clearly; don't claim all links remain available. A queue outage should not block redirects, but buffer bounded events and expose loss explicitly. Track redirect success and p50/p95/p99 latency by region, cache hit ratio, DB read load, invalidation age, expired-link redirects, code-collision retries, queue lag, analytics discrepancy, and abuse/takedown latency. Trace a sampled redirect through cache and store without logging full destination URLs indiscriminately.

## Concise final design
Stateless write and redirect services sit behind regional routing. Random eight-character base62 codes are inserted conditionally into a sharded, replicated key-value store. Edge and application caches serve the read-heavy path. Deletion writes a tombstone and invalidates caches, with a deny list for urgent blocks. An asynchronous event log feeds analytics. Latency and revocation bounds drive cache TTLs and monitoring.

## Interviewer follow-ups with strong sample answers
**Q: What if a celebrity link gets a million requests per second?** A: Put cacheable redirect resolution near users, prewarm the popular key, coalesce misses, and cap origin fan-in. The destination site's capacity is outside our redirect SLA.

**Q: Why not use 301?** A: A permanent redirect can be cached by browsers or intermediaries beyond our control. If links can expire, change, or be removed, 302 or 307 plus explicit cache policy preserves operational control.

**Q: How do you avoid duplicate custom aliases?** A: A conditional write on the (domain, alias) key is the arbitration point; a cache or pre-check alone is insufficient.

**Q: What does “delete immediately” mean?** A: Define a measurable maximum stale redirect interval. Commit a tombstone, invalidate caches, check an urgent deny list ahead of cached entries, and test propagation. Absolute immediate deletion from every browser cache cannot be promised.

**Q: What happens if analytics consumers replay events?** A: Give events IDs and deduplicate within a bounded window or make aggregation idempotent. State the residual error and reconcile totals from the durable log.

## Common pitfalls
Assuming a random code never collides; using a read-then-write uniqueness check; putting analytics on the redirect critical path; promising immediate revocation while issuing long-lived 301s; forgetting expiration, abuse, hot keys, and cache failure.