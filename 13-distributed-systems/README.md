# Distributed Systems

## Overview

You build a distributed system the moment your code calls another service, a networked database, a queue, or a third-party API. The network can drop, delay, duplicate, or reorder messages. Machines fail independently. Clocks disagree. From the outside, a slow component and a dead one look identical. Most of the surprising production behavior in modern software comes from these facts.

This section gives you the vocabulary and the reasoning tools to work with them. The aim is not to make you design a consensus protocol. It is to let you ask the right questions when a call times out, when two replicas disagree, or when a retry doubles a charge. This is where the difference between writing code and owning outcomes becomes visible. It also matters more as generated code spreads: tools readily produce a network call and rarely the timeout, idempotency key, or fallback that makes it safe.

## What you will learn

- Why partial failure, unreliable networks, and the absence of a shared clock shape every design.
- How logical clocks and ordering let you reason about causality without trusting wall-clock time.
- The consistency models, what CAP and its refinements actually say, and what each choice costs.
- How consensus protocols provide agreement, and what that costs in latency and availability.
- How replication and partitioning scale a system, and how they fail.
- Why messaging delivers at least once, and how idempotency makes that safe.
- How to bound failure with timeouts, retries, circuit breakers, bulkheads, and designed degradation.

## Topics

Suggested reading order:

| Topic | What it covers |
|-------|----------------|
| [Fundamentals and Fallacies](./fundamentals-and-fallacies.md) | Partial failure, the fallacies of distributed computing, system and fault models |
| [Time, Clocks, and Ordering](./time-clocks-and-ordering.md) | Physical clock limits, logical and vector clocks, causality, ordering guarantees |
| [Consistency Models and CAP](./consistency-models-and-cap.md) | Linearizability, sequential and causal consistency, eventual consistency, CAP and its trade-offs |
| [Consensus](./consensus.md) | Agreement among unreliable nodes, quorums, leader election, replicated logs, fencing |
| [Replication and Partitioning](./replication-and-partitioning.md) | Leader and leaderless replication, lag anomalies, sharding strategies, rebalancing, hot spots |
| [Messaging and Idempotency](./messaging-and-idempotency.md) | Queues and logs, delivery semantics, idempotency keys, outbox, sagas, backpressure |
| [Failure Modes and Resilience Patterns](./failure-modes-and-resilience-patterns.md) | Cascading failure, timeouts, retries, breakers, bulkheads, load shedding, testing failure |

## Prerequisites

- Comfort with networked programming: HTTP, sockets or RPC, and the idea of a request and response.
- Basic database knowledge, including transactions. See [Database Engineering](../06-database-engineering/README.md).
- Familiarity with how a backend service is structured. See [Backend Engineering](../07-backend-engineering/README.md).

## What good looks like

- [ ] Every remote call you write has a timeout, and you can say what the caller does when it fires.
- [ ] You retry only idempotent operations or operations carrying idempotency keys, with backoff, jitter, and a cap.
- [ ] You never use wall-clock timestamps alone to decide ordering or correctness-critical expiry.
- [ ] You can state which consistency guarantee each piece of data in your system needs, and what the system does during a partition.
- [ ] You use fencing tokens, not just leases, when a stale actor could corrupt shared state.
- [ ] You know the replication lag of your read replicas and which queries may tolerate it.
- [ ] Your message consumers are idempotent, bounded in retries, and have a dead-letter path.
- [ ] You have written down the fallback behavior for each dependency and have tested it by injecting failure.
- [ ] After an incident, you add the missing limit, isolation, or alert, not only a fix for the single trigger.

Back to [The Engineering Handbook](../README.md)
