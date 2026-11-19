# Backend Engineering

## Overview

The backend is where business rules, data and trust live. Clients can be replaced, reloaded or attacked, but the backend is the system of record and the last line of defense. Mistakes here are expensive: lost data, double charges, outages that affect every user at once.

Backend work is also where the gap between writing code and owning outcomes is widest. An endpoint that returns the right JSON in a demo is easy. An endpoint that stays correct under retries, concurrent writers, slow dependencies, partial failure and malicious input is engineering. This section covers the design decisions and operational habits that make services dependable.

## What you will learn

- How to design APIs that are consistent, evolvable and hard to misuse.
- How to separate authentication from authorization and apply both correctly.
- How to use caching without serving wrong data or creating new failure modes.
- How to move work out of the request path safely with queues and background jobs.
- How to keep a service useful when its dependencies or callers misbehave.
- How to instrument services so failures can be detected and explained.

## Topics

| Topic | What it covers |
|---|---|
| [API Design](./api-design.md) | Resource modeling, contracts, errors, pagination, idempotency, versioning |
| [Authentication and Authorization](./authentication-and-authorization.md) | Identity, sessions and tokens, access control models, common mistakes |
| [Caching in Services](./caching-in-services.md) | Cache placement, invalidation, consistency, stampedes |
| [Queues and Background Jobs](./queues-and-background-jobs.md) | Delivery guarantees, idempotent consumers, dead-letter queues, the outbox pattern |
| [Service Resilience](./service-resilience.md) | Timeouts, retries, backoff, circuit breakers, bulkheads, rate limiting |
| [Observability in Services](./observability-in-services.md) | Structured logs, metrics, traces, cardinality, health checks |

## Prerequisites

- Comfort with at least one backend language and a relational database.
- Basic knowledge of HTTP and how a request moves from client to server.
- The ideas in [Engineering Fundamentals](../01-engineering-fundamentals/README.md), especially ownership and thinking in systems.

## What good looks like

- [ ] API contracts are written down and reviewed before implementation.
- [ ] Every write endpoint that can be retried is safe to retry.
- [ ] Authorization is checked on the server for every request, not assumed from the client.
- [ ] Every outbound call has a timeout, and retries use backoff with jitter.
- [ ] Each cache entry has a known owner, expiry and invalidation story.
- [ ] Background jobs are idempotent, have a dead-letter path, and someone is alerted when it fills.
- [ ] Logs are structured and carry a request or trace id.
- [ ] Dashboards show rate, errors and latency per route and per dependency.
- [ ] Failure behavior has been tested, not just reasoned about.
- [ ] AI-generated backend code is reviewed for the same properties as hand-written code.

Back to [The Engineering Handbook](../README.md).
