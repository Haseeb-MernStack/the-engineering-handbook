# Caching in Services

> Caching trades freshness and simplicity for speed and reduced load, and it only works safely when you know exactly how stale an answer is allowed to be.

## Overview

A cache stores the result of expensive work so that later requests can skip it. The work might be a database query, a call to another service, a rendered response, or a computation. A cache hit is fast. A miss costs the original work plus the overhead of checking.

Caching is attractive because it is often the cheapest way to reduce latency and protect a backend. It is dangerous because it creates a second copy of the truth. Every cache needs an answer to one question: how wrong is it acceptable to be, and for how long?

Phil Karlton's well-known remark that cache invalidation is one of the hard problems in computer science stays true because the difficulty is not mechanical. It is about deciding what "stale" means for your product.

## Core Concepts

### Cache layers

```
Client -> CDN / edge -> API gateway -> Service (in-process cache)
                                          |
                                          v
                                   Shared cache (for example, Redis, Memcached)
                                          |
                                          v
                                       Database
```

| Layer | Latency | Shared across instances | Typical content |
|-------|---------|-------------------------|-----------------|
| Browser / HTTP client | Lowest | No | Static assets, cacheable GETs |
| CDN / edge | Low | Yes (per region) | Public responses, assets |
| In-process | Very low | No | Small, hot, rarely changing data |
| Distributed cache | Network round trip | Yes | Sessions, computed results, lookups |
| Database buffer/query cache | Internal | Yes | Managed by the database |

### Common patterns

**Cache-aside (lazy loading).** The application checks the cache, loads from the source on a miss, and writes the result back. It is the most common pattern and the easiest to reason about.

```ts
async function getProduct(id: string): Promise<Product> {
  const key = `product:v1:${id}`;
  const cached = await cache.get(key);
  if (cached) return JSON.parse(cached);

  const product = await db.products.findById(id);
  if (!product) return null as never; // handle not-found explicitly in real code
  await cache.set(key, JSON.stringify(product), { ttlSeconds: 300 });
  return product;
}
```

**Read-through.** The cache itself loads from the source on a miss. The application only talks to the cache.

**Write-through.** Writes go to the cache and the source together. Reads stay consistent with writes. Writes are slower.

**Write-behind (write-back).** Writes go to the cache and are flushed to the source later. Fast, but data can be lost if the cache fails before the flush.

**Refresh-ahead.** The cache refreshes hot entries before they expire to avoid miss spikes.

### Expiry and invalidation

- **TTL (time to live).** The entry expires after a fixed time. Simple and bounds staleness. The right TTL is a product decision.
- **Explicit invalidation.** Delete or update the entry when the source changes. Fresher, but you must find every code path that changes the data.
- **Versioned keys.** Include a version in the key (`product:v1:123`). Changing the version abandons old entries without deleting them.
- **Event-driven invalidation.** Consumers of change events purge affected keys. Needs reliable events. See [Queues and Background Jobs](./queues-and-background-jobs.md).

A practical approach is to combine a TTL with explicit invalidation. The TTL is a safety net for missed invalidations.

### Eviction

When the cache is full it must drop entries. Common policies are least recently used (LRU), least frequently used (LFU), and random. Know which policy your cache uses and what happens at the memory limit. A cache that errors on a full memory is very different from one that evicts.

### HTTP caching

For HTTP APIs, standard headers do much of the work.

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=60
ETag: "v42"
```

A client can revalidate with `If-None-Match: "v42"`. If unchanged, the server returns `304 Not Modified` with no body. Use `Cache-Control: private` for per-user responses and `no-store` for sensitive data. Use `Vary` when the response depends on a request header, or shared caches may serve the wrong variant. The semantics are in RFC 9111.

### Negative caching

Cache "not found" results too, with a short TTL. Otherwise requests for missing keys always hit the database.

## Trade-offs

- **Freshness versus speed.** Longer TTLs mean more hits and more staleness. Shorter TTLs mean less staleness and more load on the source.
- **Complexity.** Each cache adds a place where data can be wrong, a component to operate, and behavior that differs between local development and production.
- **Consistency.** Cache-aside has a race: a reader loads old data, a writer updates and invalidates, then the reader writes the old data back. The entry stays stale until its TTL. Closing this fully is hard. Most systems accept a bounded window.
- **In-process versus distributed.** In-process is fastest and has no network dependency. Each instance holds a different copy and invalidation does not reach them. Distributed caches are consistent across instances and add a network hop and a failure dependency.
- **Write-behind durability.** Faster writes cost a risk of losing acknowledged data.
- **Memory cost.** Caching everything is expensive. Caching the wrong things yields a low hit rate and wasted memory.
- **Serialization overhead.** Large objects cost CPU to serialize and bytes to move. A cache hit can be slower than a well-indexed query for small rows.

## Failure Modes

- **Cache stampede (thundering herd).** A popular key expires and hundreds of requests miss at once, all hitting the database. Mitigate with request coalescing (one loader per key, others wait), jittered TTLs so keys do not expire together, and refresh-ahead or stale-while-revalidate.
- **Cold start.** After a deploy or cache restart, the cache is empty and the database takes the full load. It may fall over. Mitigate with warming, gradual traffic shifting, and load shedding.
- **Serving stale or wrong data.** An invalidation path was missed, so users see old prices or permissions. Detect by comparing sampled cache values with the source. Prevent with TTLs as a backstop and by centralizing writes.
- **Cache poisoning and key collisions.** Keys that omit a tenant or user identifier return one customer's data to another. Prevent with strict key construction in one helper, and include every input that affects the result.
- **Caching errors.** A transient failure is cached and served for the whole TTL. Do not cache failures, or cache them very briefly.
- **Hot keys.** One key receives a disproportionate share of traffic and overloads a single cache node. Mitigate with local replicas of the hot key or key splitting.
- **Cache as a hard dependency.** The service cannot run if the cache is down, and the database cannot handle the fallback load. Decide the behavior up front: fail open to the source with rate limiting, serve stale, or fail fast. See [Service Resilience](./service-resilience.md).
- **Unbounded cardinality.** Keys include arbitrary user input (search strings, timestamps), so the hit rate is near zero and memory fills. Monitor hit rate and key count.
- **Dual-write inconsistency.** Writing to the database and cache separately, with a failure between them, leaves them different. Prefer invalidating on write over updating, because deletion is idempotent and simpler to get right.

## In Practice

### Decision checklist

Before adding a cache, answer:

- What is the measured problem (latency, load, cost)? Is a missing index or a bad query the real cause?
- What is the read-to-write ratio? Caches help most for read-heavy data.
- How stale may the data be, and who agreed to that?
- What is the key, and does it include every input that changes the result (tenant, locale, permissions)?
- What happens on a miss storm and on a cache outage?
- How will you measure hit rate, evictions, and latency?
- Is the cached data sensitive? Is it encrypted or access controlled appropriately?

### Worked example: protecting a hot endpoint

A product page endpoint reads from the database at 2,000 requests per second during a sale, and the database is near its limit. Product data changes a few times a day.

1. Measure first. Confirm the endpoint is read-heavy and the query is already indexed.
2. Add a CDN-friendly header (`Cache-Control: public, max-age=30`) for anonymous traffic. Accept 30 seconds of staleness for price and description.
3. Add a distributed cache-aside layer with a TTL of a few minutes plus a randomized jitter.
4. Coalesce concurrent loads for the same key in each instance.
5. Invalidate the key when an admin edits the product.
6. Show inventory separately and uncached or with a very short TTL, because it changes fast and affects checkout correctness.
7. Add dashboards for hit rate and origin request rate, and an alert if the hit rate falls sharply.

### Developer vs. engineer

A developer adds a cache call when the page is slow and sees it get faster. An engineer first finds why it was slow, then states the staleness contract, designs the key, plans for stampedes and outages, and instruments the hit rate so the cache's value is known and its failure is visible.

### When AI generates caching code

Check that:

- Keys include all varying inputs, especially tenant and user.
- Every cached entry has a TTL.
- Failures and nulls are handled deliberately.
- The code does not assume the cache is always available.
- Invalidation exists on every write path, not only the one the model saw.
- Sensitive or per-user responses are not marked publicly cacheable.

## Further Reading

- RFC 9111, HTTP Caching (IETF).
- Martin Kleppmann, Designing Data-Intensive Applications (chapters on replication, consistency, and derived data).
- Redis and Memcached official documentation on eviction policies and expiry.
- Betsy Beyer et al. (eds.), Site Reliability Engineering, Google (chapters on handling overload and cascading failures).
- Nishtala et al., "Scaling Memcache at Facebook" (NSDI 2013).
