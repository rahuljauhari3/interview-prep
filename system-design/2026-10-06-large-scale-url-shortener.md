# System Design Interview Practice — Design a Large-Scale URL Shortener

## Interviewer prompt

Design a URL shortening service like Bitly. Users submit a long URL and receive a short URL. When someone visits the short URL, they should be redirected to the original URL.

## Clarifying questions

- Do we need custom aliases? Assume yes, optionally.
- Do links expire? Support optional expiration.
- Do we need analytics? Basic click counts are useful, but redirect latency is the priority.
- What scale should we design for? Assume hundreds of millions of created links and very high redirect traffic.
- Is strong consistency required? Link creation should be correct immediately; analytics may be eventually consistent.

## Functional requirements

- Create a short URL from a long URL.
- Redirect short URL to the original URL.
- Optional custom alias.
- Optional expiration.
- Basic click analytics.

## Non-functional requirements

- Very low redirect latency.
- High availability.
- Durable mappings.
- Horizontal scalability.
- Redirect path should survive partial outages.

## Scale estimates

Suppose we create 100 million URLs per month. That is roughly 3.3 million per day, or about 40 writes/sec on average, perhaps hundreds/sec at peak.

Redirect traffic is usually much larger. If each URL receives 100 redirects over its lifetime, the read workload dominates dramatically. This means the system is read-heavy, which strongly motivates aggressive caching.

## API design

Create URL:

```http
POST /urls
```

```json
{
  "long_url": "https://example.com/really/long/path",
  "custom_alias": null,
  "expires_at": null
}
```

Response:

```json
{
  "short_url": "https://sho.rt/aZ91Kq"
}
```

Redirect:

```http
GET /aZ91Kq
```

Response: HTTP 301 or 302 redirect to the original URL.

## Data model

```text
ShortURL
--------
code PK
long_url
created_at
expires_at
owner_id nullable
```

Analytics should be separated from the redirect mapping because writes to analytics must not slow redirects.

## Short-code generation

One approach is to generate a globally unique numeric ID and Base62-encode it. Base62 uses `0-9`, `a-z`, `A-Z`. A 7-character Base62 key provides approximately `62^7`, which is trillions of possibilities.

### Option A: auto-incrementing database IDs
Easy, but a single sequence becomes a scaling bottleneck and predictable IDs may expose system growth.

### Option B: distributed ID generator
Use a Snowflake-style 64-bit ID and Base62 encode it.

Advantages:
- distributed
- collision-free if configured correctly
- no central sequence bottleneck

### Option C: random short codes
Generate random Base62 strings and retry on collisions. Simple, but collision checks add overhead at large scale.

For an interview, a distributed unique ID generator is a strong default.

## High-level architecture

```text
Clients
  |
Load Balancer
  |
API / Redirect Service
  |------> Redis / distributed cache
  |             |
  |             v
  |----------> URL Mapping Database
  |
  +------> Event Queue ------> Analytics Workers ------> Analytics Store
```

## Redirect flow

- Client requests `sho.rt/aZ91Kq`.
- Redirect service checks cache.
- If found, immediately return redirect.
- On cache miss, read mapping from the database.
- Populate cache.
- Return redirect.
- Emit an asynchronous click event to a queue.

Analytics must not be synchronous with the redirect.

## Cache strategy

Because reads dominate writes, cache hot mappings in Redis or another distributed in-memory store.

Use cache-aside:
- read cache
- on miss, read DB
- populate cache

TTL can be long for permanent links and bounded by link expiration for expiring links.

### Hot-key problem

A viral URL may receive enormous traffic concentrated on one key.

Mitigations:
- local in-process cache in redirect servers
- Redis replication
- CDN/edge caching where appropriate

## Database choice

The mapping workload is essentially key-value access:

```text
short_code -> long_url
```

A distributed key-value database such as DynamoDB, Cassandra, or a sharded relational store could work. The important interview point is not the brand name. The access pattern is simple, and horizontal partitioning by short code is straightforward.

## Partitioning

Hash the short code to determine the partition.

Why hashing?
- distributes writes and reads evenly
- avoids range hotspots caused by sequential IDs

## Consistency

After creating a short URL, the user expects it to work immediately. The mapping store should therefore provide sufficiently strong read-after-write consistency for newly created mappings, or the creation service can seed the cache immediately after persisting the record.

Analytics can be eventually consistent.

## 301 vs 302 redirects

A permanent 301 may be cached aggressively by browsers and intermediaries, reducing server traffic but also reducing visibility into subsequent clicks.

A 302 keeps more requests flowing through your service, which provides better control and analytics. Many interview designs choose 302 by default unless permanent caching is explicitly desired.

## Failure modes

### Cache outage
Fallback to the database, but protect it with rate limiting and circuit breakers so the DB is not overwhelmed.

### Database partition unavailable
Replicas can serve reads. Because redirects are critical, replication across availability zones is important.

### Queue outage
Redirects should continue. Click events may be buffered, dropped under extreme pressure, or written to a local durable log depending on analytics guarantees.

### ID generator failure
Use multiple generator nodes with unique worker IDs and health checks.

## Observability

Track:
- redirect latency p50/p95/p99
- cache hit rate
- DB read latency
- redirect error rate
- queue lag
- URL creation failure rate
- hot-key concentration
- expired-link lookup rate

## Bottlenecks

The most important bottleneck is usually the redirect read path, not URL creation. A good candidate should notice that caching and replication matter much more than optimizing a relatively modest write rate.

## Security considerations

- Validate URL schemes to prevent dangerous or malformed redirects.
- Abuse detection for phishing and malware links.
- Rate limit creation endpoints.
- Reserve administrative aliases.
- Avoid exposing sequential IDs if predictability matters.

## Interviewer follow-up questions

### How would you support custom aliases?
Attempt an atomic insert where `code = requested_alias`. If it already exists, reject the request.

### What happens when links expire?
Store `expires_at`. On lookup, verify expiration before redirecting. Cache TTL should not exceed the remaining lifetime.

### How would you handle 1 million requests/sec for one viral link?
Use multi-layer caching: CDN/edge cache if acceptable, local process cache, then distributed cache. Avoid routing every request to one Redis shard if possible.

### How would you prevent duplicate shortened links for the same URL?
Usually you do not need to. If deduplication is required, maintain a hash of normalized long URLs, but this adds complexity and raises semantic questions because users may want independent links and analytics.

### Could the database be relational?
Yes. The workload is simple enough that a sharded relational database can work well. Choosing NoSQL is not automatically better; the decision depends on operational requirements and expected scale.

## Common pitfalls

- Spending too much time on URL creation while ignoring redirect traffic.
- Sending analytics synchronously on the critical path.
- Saying “use Redis” without explaining cache miss behavior.
- Forgetting collision handling.
- Ignoring hot keys.
- Ignoring expiration semantics.
- Naming technologies without explaining the underlying access patterns.

## Concise final design

Use stateless redirect/API servers behind a load balancer. Generate globally unique IDs with a distributed ID generator and Base62 encode them into short codes. Persist `code -> long_url` mappings in a horizontally partitioned, replicated store. Use a distributed cache plus optional local/edge caching for the read-heavy redirect path. Emit click events asynchronously to a queue and process analytics separately. Replicate across availability zones and design the redirect path so cache or analytics failures do not take down link resolution.
