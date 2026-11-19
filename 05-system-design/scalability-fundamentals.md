# Scalability Fundamentals

> Scalability is a system's ability to handle growth in load by adding resources without a matching growth in cost, latency, or complexity; achieving it means finding and removing the next bottleneck, not adding servers blindly.

## Overview

A system is scalable if it can keep meeting its requirements as load grows, and if adding resources helps in a predictable way. Load can mean requests per second, data volume, number of users, or number of concurrent connections. These grow differently and break different things.

Scalability is not a property you add at the end. It is a result of design choices: where state lives, how work is divided, and which operations are allowed to be slow. It is also not always needed. Many systems never outgrow a well-tuned single database. Designing for scale that never arrives is a cost, not a safeguard.

The engineering skill is measurement. Find the current bottleneck, fix it, and expect the next one to appear. Scaling is a loop, not a destination.

## Core Concepts

### Performance, latency, throughput

- **Latency**: time for one request. Measured as a distribution, not an average.
- **Throughput**: work completed per unit time.
- **Scalability**: how those hold as load or resources change.

A system with fast single requests can still scale badly if it serializes on a lock. A system can have high throughput and poor tail latency. Report percentiles such as p50, p95, and p99. Averages hide the slow requests that users notice, and in fan-out systems the slowest call dominates the whole response.

### Vertical and horizontal scaling

| Approach | Meaning | Strength | Limit |
|---|---|---|---|
| Vertical (scale up) | Bigger machine | Simple; no code change | Hardware ceiling; cost grows faster; single point of failure |
| Horizontal (scale out) | More machines | No fixed ceiling; fault tolerance | Needs distribution, coordination, and statelessness |

Vertical scaling is often the right first step. It is cheap in engineering time. Horizontal scaling should come when you reach a real limit or need redundancy.

### Stateless services

A stateless service keeps no per-user state between requests in its own memory or disk. Any instance can serve any request. This makes horizontal scaling easy: add instances behind a load balancer.

State must live somewhere. Move it to a database, cache, or object store, or into a token the client carries. Session data kept in one server's memory forces "sticky" routing and makes that server a fragile point.

### Where bottlenecks live

Typical order in which things become limiting:

1. Application code (inefficient queries, N+1 calls, blocking I/O).
2. The database: CPU, memory, disk I/O, connections, lock contention.
3. A shared resource: a single queue, a lock, a hot key.
4. Network bandwidth or a downstream dependency.
5. Operational limits: deploy time, on-call load, team coordination.

Fix in measured order. Adding application servers does nothing if the database is saturated; it makes it worse by adding more connections.

### Techniques

**Caching.** Avoid repeated work. See [Load Balancing, Caching and CDN](./load-balancing-caching-and-cdn.md).

**Read replicas.** Spread reads across copies of the data. See [Replication and Sharding](../06-database-engineering/replication-and-sharding.md).

**Partitioning (sharding).** Split data by key across nodes so each handles a subset. Powerful, but it complicates queries, transactions, and rebalancing.

**Asynchronous processing.** Accept the request, queue the heavy work, respond later. Smooths bursts and protects the front end. Cost: results are not immediate.

**Batching.** Group many small operations into fewer large ones. Reduces per-operation overhead. Cost: added latency for the first item in a batch.

**Denormalization and precomputation.** Compute expensive results ahead of time. Cost: storage, and keeping copies fresh.

**Backpressure and load shedding.** When overloaded, limit or reject work early instead of failing everywhere. See [Reliability and Availability](./reliability-and-availability.md).

### Amdahl and the serial fraction

If part of the work is inherently serial, extra parallel resources cannot speed up that part. Amdahl's law states that speedup is limited by the serial fraction. Practically: a global lock, a single-leader write path, or a shared counter caps throughput regardless of node count. Look for the serial part first.

### Hot spots and skew

Load is rarely uniform. A celebrity account, a popular product, or a single time bucket concentrates traffic on one partition. Adding nodes does not help the hot one. Mitigate with caching of hot items, key splitting (adding a suffix to spread writes), or special handling for known heavy keys.

## Trade-offs

| Technique | What it buys | What it costs |
|---|---|---|
| Scale up | Zero application change | Ceiling, price, single point of failure |
| Scale out | Capacity and redundancy | Distributed-system complexity |
| Caching | Lower latency and database load | Staleness, invalidation bugs, cold-start risk |
| Read replicas | More read capacity | Replication lag; stale reads |
| Sharding | Write and storage scale | Cross-shard queries, resharding pain, operational load |
| Async queues | Burst absorption, decoupling | Eventual results, duplicate handling, extra infrastructure |
| Denormalization | Fast reads | Write complexity, risk of inconsistent copies |

The unifying cost: each technique moves complexity somewhere. Choose the one whose cost your team can afford to operate.

## Failure Modes

**Scaling the wrong layer.** Doubling app servers while the database is the bottleneck. Connections multiply and the database gets slower. Detect with per-tier utilization and saturation metrics before changing anything.

**Premature distribution.** Sharding a database that fits comfortably on one machine. The team now pays the operational cost of a distributed system with no benefit.

**Connection exhaustion.** Many instances each hold a pool of connections; the database hits its limit. Use bounded pools, a connection proxy, and capacity-plan connections as instances x pool size.

**Thundering herd.** A popular cache entry expires and thousands of requests hit the database at once. Mitigate with request coalescing, staggered expiry, and refreshing before expiry.

**Retry amplification.** Under load, clients retry, which adds load, which causes more failures. Use backoff with jitter, retry budgets, and server-side shedding.

**Unbounded queues.** A queue that grows without limit hides overload until memory or latency collapses. Bound queues and alert on age of oldest item.

**Hot partition.** One shard at 100 percent while others idle. Monitor per-shard load, not only aggregate.

**Tail latency in fan-out.** A request calls 50 services; the slowest dominates. Use timeouts, hedged requests carefully, and reduce fan-out.

**Untested scaling.** The design assumes horizontal scaling but nobody ran load tests. The first real surge reveals a global lock. Load test with realistic data and traffic mix.

## In Practice

### A scaling loop

1. Define the target: load, latency percentile, and cost ceiling.
2. Measure the current system under realistic load.
3. Identify the saturated resource (the bottleneck).
4. Apply the cheapest effective change.
5. Re-measure. Repeat.

### Heuristics

- Prefer the simplest change that removes the measured bottleneck.
- Make services stateless early; it is cheap and keeps options open.
- Know your single points of failure and your single points of contention.
- Use percentiles, not averages.
- Plan capacity with headroom, since systems degrade sharply near saturation.
- Test at multiples of expected load to see how failure begins.

### Worked example: a slow product page

Symptoms: page latency rises during sales. Measurement shows the database CPU at its limit, dominated by one query that loads product details and inventory for each request.

Options: (a) bigger database, (b) cache product details, (c) read replica, (d) shard.

Reasoning: product details change rarely and are read constantly, so caching gives the biggest gain at low cost. Inventory changes often and must be accurate at purchase, so keep that read from the primary or use a short expiry with a final check at checkout. Sharding is unjustified. The team caches details, re-measures, and sets an alert on cache hit ratio and database CPU.

### Developer vs. engineer

A developer sees a slow page and adds servers or a cache wherever it seems plausible. An engineer finds the saturated resource first, chooses a fix that matches the access pattern, and verifies with a load test.

### Verifying AI-generated designs

Generated designs often add sharding, caches, and queues by reflex. Ask:

- Which measured or estimated bottleneck does each addition address?
- Is the service actually stateless, or does it hold session data in memory?
- Are connection counts, queue bounds, and timeouts specified?
- Does the cache have an invalidation and stampede plan?
- Were hot keys considered?

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*, chapters on scalability and partitioning.
- Brendan Gregg, *Systems Performance*.
- Jeff Dean and Luiz Barroso, "The Tail at Scale," *Communications of the ACM* (2013).
- Michael Nygard, *Release It!*.
- Neil Gunther, *Guerrilla Capacity Planning*, for scalability models.
