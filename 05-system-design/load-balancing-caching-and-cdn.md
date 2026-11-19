# Load Balancing, Caching and CDN

> Load balancers spread requests across instances, caches avoid repeating expensive work, and content delivery networks move content close to users; together they are the main tools for cheap speed and horizontal scale, and each adds its own failure modes.

## Overview

Most read-heavy systems are scaled with three building blocks. A load balancer distributes traffic so no single instance is overwhelmed and failed instances are skipped. A cache keeps the result of work so it need not be repeated. A content delivery network (CDN) is a cache distributed geographically, so users fetch content from a nearby location.

They are easy to add and easy to misuse. A cache that serves stale data can be worse than a slow correct answer. A load balancer that routes to dead instances turns a partial failure into an outage. A CDN that caches a personalized page can leak one user's data to another. The engineering work is in the rules: what is cached, for how long, who may see it, and what happens when each component fails.

## Core Concepts

### Load balancing

A load balancer sits between clients and a pool of backends.

```
Clients -> [Load balancer] -> Backend A
                          -> Backend B
                          -> Backend C
```

**Layers.**

| Layer | Sees | Typical use |
|---|---|---|
| L4 (transport) | IPs, ports, TCP/UDP | Fast, simple distribution; any protocol |
| L7 (application) | HTTP paths, headers, cookies | Routing by URL, TLS termination, header rewriting |

**Algorithms.**

- **Round robin**: each backend in turn. Simple; ignores differing load.
- **Weighted round robin**: more traffic to larger backends.
- **Least connections**: send to the backend with the fewest active requests. Better when request cost varies.
- **Hash-based (consistent hashing)**: the same key goes to the same backend. Useful for cache locality and sticky behavior.
- **Random with two choices**: pick two at random, choose the less loaded. Cheap and performs well in practice.

**Health checks.** The balancer must detect failed backends and stop sending traffic. Active checks probe an endpoint. Passive checks watch real traffic for errors. A shallow check (process is up) misses broken dependencies. A deep check (queries the database) can cause all backends to be marked down together when the database is slow. Choose deliberately.

**Other responsibilities.** TLS termination, connection reuse, rate limiting, and graceful draining of instances during deploys.

The balancer itself must not be a single point of failure. It is run redundantly, or provided as a managed, redundant service.

### Caching

A cache stores computed or fetched data for faster reuse. It works because access is skewed: a small part of the data gets most of the reads.

**Where caches live.**

- Browser and client.
- CDN or edge.
- Reverse proxy.
- Application memory (in-process).
- Shared cache service (separate process or cluster).
- Database buffer and query caches.

**Patterns.**

| Pattern | How it works | Notes |
|---|---|---|
| Cache-aside | App checks cache; on miss, loads from source and fills cache | Most common; app owns the logic |
| Read-through | Cache loads from source on miss | Simpler app code |
| Write-through | Writes go to cache and source together | Consistent; slower writes |
| Write-behind | Writes go to cache, flushed to source later | Fast; risks data loss |

```python
def get_product(pid):
    value = cache.get(f"product:{pid}")
    if value is not None:
        return value
    value = db.load_product(pid)
    cache.set(f"product:{pid}", value, ttl=300)
    return value
```

**Invalidation.** The hard part. Options:

- **TTL (time to live)**: entries expire after a set time. Simple; accepts bounded staleness.
- **Explicit invalidation**: delete or update the entry when the source changes. Fresher; easy to miss a code path.
- **Versioned keys**: include a version in the key so new data uses a new key.

**Eviction.** When full, the cache removes entries. Common policies: least recently used (LRU), least frequently used (LFU), or first in first out. Choose by access pattern.

**What to cache.** Data that is read often, expensive to compute, and tolerant of some staleness. Do not cache what must be exact at the moment of use, such as an account balance used for a transfer decision, unless you validate at the source.

### CDN

A CDN has points of presence in many locations. Users are routed to a nearby one. If it has the content it serves it directly; otherwise it fetches from your origin and keeps a copy.

Good fits: static assets (images, scripts, stylesheets), downloads, video segments, and cacheable API responses. Control behavior with HTTP headers: `Cache-Control` (max-age, s-maxage, no-store, private), `ETag`, `Vary`. Fingerprinted file names (`app.3f9a1c.js`) allow very long lifetimes, since a new version gets a new name.

Benefits: lower latency for distant users, less origin load, and absorption of traffic spikes. Many CDNs also provide TLS, request filtering, and protection against floods of traffic.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| L7 balancing | Smart routing, TLS handling | More processing; more configuration; more to secure |
| Sticky sessions | Simple stateful apps | Uneven load; lost sessions when an instance dies |
| Long TTL | High hit rate, low origin load | More stale data |
| Short TTL | Fresher data | More misses and load |
| Explicit invalidation | Freshness | Complexity; missed paths cause stale data |
| In-process cache | Fastest access | Per-instance copies diverge; memory pressure |
| Shared cache | One consistent view; large capacity | Network hop; a new dependency and failure point |
| Write-behind | Write speed | Possible data loss on cache failure |
| CDN | Latency and offload | Cost; purge delays; debugging stale content at the edge |

The central trade-off is freshness versus speed and load. State the acceptable staleness per data type as a requirement.

## Failure Modes

**Stale data served.** An update path forgets to invalidate. Users see old prices. Prevent by centralizing writes through one code path, using short TTLs as a safety net, and testing invalidation.

**Cache stampede (thundering herd).** A hot key expires; many requests miss simultaneously and hit the database. Mitigate: coalesce concurrent loads so one request refills, add random jitter to TTLs, refresh early, or serve stale while revalidating.

**Cold start.** After a restart or a new cache cluster, hit rate is zero and the database takes full load. Warm caches before shifting traffic, and ramp gradually. Know whether your database can survive with no cache.

**Cache penetration.** Requests for keys that do not exist always miss and reach the database. Cache negative results briefly, or filter with a Bloom filter.

**Hot key.** One key receives disproportionate traffic and saturates its cache node. Replicate hot keys, add a local cache layer, or split the key.

**Cache as source of truth.** Data exists only in the cache and is lost on eviction or restart. A cache should be safe to lose.

**Leaking private content.** A CDN or shared cache stores a personalized response and serves it to others, because `Vary` or `private` was missing. Mark user-specific responses `private` or `no-store`, and test with two users. This is a security incident, not only a bug.

**Health-check cascade.** Deep health checks mark every backend unhealthy when a shared dependency slows, and the balancer routes to nothing. Use shallow checks for routing and separate dependency alerts, or fail open when all backends look unhealthy.

**Uneven load from sticky routing.** A few heavy users pin to one backend. Prefer stateless services with shared session storage.

**Balancer saturation.** The balancer itself runs out of connections or bandwidth. Monitor it as a first-class component.

## In Practice

### Decision checklist for adding a cache

- [ ] What exactly is cached, and what is the key?
- [ ] What is the read-to-write ratio and the expected hit rate?
- [ ] How stale may it be, and who agreed?
- [ ] How is it invalidated on every write path?
- [ ] What happens on a miss storm and on a cold start?
- [ ] What happens if the cache is down? Does the database survive?
- [ ] Is the data safe to share across users?
- [ ] Are hit ratio, evictions, and latency monitored?

### Metrics to watch

- Cache hit ratio (overall and per key class).
- Origin request rate behind the cache.
- Evictions and memory use.
- Backend error rate and latency per instance at the balancer.
- Balancer connection counts and saturation.

### Worked example: a catalog API

The catalog is read 100 times for each write. Static images go to a CDN with fingerprinted names and a long lifetime. Catalog responses are public, so the CDN caches them for 60 seconds with stale-while-revalidate. Behind it, a shared cache holds product records with a 5-minute TTL, and the update service deletes the key on every change. Prices at checkout are read from the database, not the cache. Misses for the same key are coalesced to prevent a stampede.

### Developer vs. engineer

A developer adds a cache and sees faster responses. An engineer also decides what staleness is acceptable, handles invalidation and stampedes, and tests that the system survives when the cache is empty.

### Verifying AI-generated designs

Generated code often adds `cache.set` with a fixed TTL and stops. Check:

- Invalidation on all update paths, including background jobs.
- Cache keys that include every input that changes the output (user, locale, version).
- Behavior on cache failure: timeouts and fallback, not hanging requests.
- Whether personal data could enter a shared or edge cache.
- Stampede protection for hot keys.

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*.
- RFC 9111, "HTTP Caching" (IETF), and RFC 9110, "HTTP Semantics."
- Michael Mitzenmacher, "The Power of Two Choices in Randomized Load Balancing" (paper).
- Karger et al., "Consistent Hashing and Random Trees" (paper).
- Betsy Beyer et al. (eds.), *Site Reliability Engineering*, chapters on load balancing.
