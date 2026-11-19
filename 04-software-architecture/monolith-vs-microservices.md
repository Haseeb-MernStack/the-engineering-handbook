# Monolith vs. Microservices

> The choice between one deployable unit and many is a trade between simplicity of operation and independence of teams, and it should be made on organizational and operational evidence rather than fashion.

## Overview

A **monolith** is deployed as a single unit. All modules run in one process and are released together. **Microservices** split a system into independently deployable services, each owning a narrow capability and its own data, communicating over a network.

Neither is a quality level. Both are ways of drawing boundaries. A monolith with strict internal boundaries is often a better design than a set of services that share a database and must deploy in lockstep. The latter is sometimes called a distributed monolith: all the costs of distribution and none of the independence.

The deeper question is what you are optimizing. Microservices mainly address **organizational scaling**: letting many teams change and release without coordinating. They address technical scaling only secondarily. A system with a handful of engineers rarely has the problem microservices solve.

## Core Concepts

### The deployment spectrum

| Style | Description | Deploy unit |
|---|---|---|
| Big ball of mud monolith | No internal structure | One |
| Modular monolith | One deployable, strict module boundaries, each module owns its data | One |
| Service-based (coarse services) | A few large services, often shared database | Few |
| Microservices | Many small services, each owning its data | Many |
| Functions / nano-services | Very fine-grained, event or request triggered | Very many |

The modular monolith is the underused middle. Modules communicate through explicit interfaces, not by reading each other's tables. If a module later needs extraction, the boundary already exists.

### What a service boundary means

A well-formed microservice:

- Owns its data. No other service reads or writes its tables directly.
- Exposes a contract (API or events) and versions it.
- Can be deployed, rolled back and scaled without coordinating with others.
- Maps to a business capability and, ideally, to one team.

If any of these fail, you pay the network cost without getting the independence.

### What changes when you cross the network

In-process calls are fast, reliable and transactional. Network calls are none of these. Splitting a system converts function calls into remote calls and brings the classic problems.

| In a monolith | Across services |
|---|---|
| A call returns or throws | A call may time out with unknown outcome |
| One ACID transaction | Sagas, compensation, eventual consistency |
| One stack trace | Distributed traces across many logs |
| Refactor with an IDE | Coordinated, versioned contract changes |
| One deploy | Many pipelines, version skew between services |
| Local test | Integration or contract tests, test environments |

See the [Distributed Systems](../13-distributed-systems/README.md) section for the underlying theory of partial failure, consistency and consensus.

### Conway's Law

Organizations design systems that mirror their communication structure. This is an observation, not a rule you can opt out of. If three teams must agree to change one service, that service's boundary does not match the organization. Teams and services should be shaped together. This is sometimes used deliberately, as the "inverse Conway maneuver": choose the team structure that produces the architecture you want.

### Communication styles

- **Synchronous request/response** (HTTP, gRPC). Simple to reason about. Couples availability: if B is down, A's request fails.
- **Asynchronous messaging or events.** Decouples in time. Adds eventual consistency and more complex failure handling. See [Event-Driven Architecture](./event-driven-architecture.md).

A long synchronous call chain multiplies failure probability and latency. Five services each available 99.9 percent of the time yield a chain available roughly 99.5 percent of the time, assuming independent failures. The exact number matters less than the direction: chains erode availability.

## Trade-offs

| Dimension | Monolith | Microservices |
|---|---|---|
| Initial development speed | Fast | Slower. Infrastructure and contracts up front. |
| Operational complexity | Low | High. Orchestration, service discovery, observability, many pipelines. |
| Independent team releases | Hard at scale | A primary benefit |
| Independent scaling | Scale the whole unit | Scale hot services separately |
| Technology freedom | One stack | Per-service choice, at the cost of fragmentation |
| Fault isolation | One bug can take down all | Failures can be contained, if designed for |
| Data consistency | Simple transactions | Eventual consistency, sagas |
| Refactoring across boundaries | Easy and safe | Slow and risky |
| Debugging | One process | Distributed tracing required |
| Cost | Lower baseline | Higher baseline: more infrastructure, more engineers on platform work |

Honest cautions:

- **Microservices move complexity, they do not remove it.** The complexity of a large domain remains. It shifts from code structure to the interactions between services, where it is harder to see and test.
- **Independent scaling is often overstated.** Most systems scale well by running more copies of a monolith behind a load balancer. The database is usually the bottleneck either way.
- **Team size is the strongest signal.** Small teams carry the full cost of distribution with little of the benefit.
- **The debate is contested.** Several well-known companies have publicly described consolidating services back into larger units. Others operate thousands of services successfully. The difference lies in tooling, discipline and organizational size, not in a universal truth.

## Failure Modes

**Distributed monolith.** Services must be deployed together, share a database, or call each other in tight chains. Detection: release notes that say "deploy A, then B, then C in this order"; a schema change that requires editing several services.

**Shared database.** Two services read the same tables. Either can break the other with a migration, and ownership of the data is unclear. Prevention: one owner per table. Others use the owner's API or consume its events.

**Cascading failure.** A slow dependency exhausts caller threads or connections, and the failure spreads upstream. Prevention: timeouts on every remote call, bounded queues, circuit breakers, bulkheads, load shedding.

**Retry storms.** Callers retry a struggling service in unison and finish it off. Prevention: exponential backoff with jitter, retry budgets, and idempotent operations so retries are safe.

**Version skew.** Service A deploys a new response shape while B still expects the old one. Prevention: backward-compatible changes (add fields, never repurpose), consumer-driven contract tests, expand-then-contract rollouts.

**Chatty services.** A single user request fans out into dozens of calls, producing high latency and fragile behavior. Detection: traces with very wide or deep call trees. Prevention: coarser contracts, data denormalization, composing at an edge layer.

**Premature decomposition.** Boundaries guessed before the domain is understood turn out wrong, and every change crosses three services. Wrong boundaries are costlier in microservices than in a monolith because moving them means moving data and contracts, not moving files.

**Operational overload.** A small team spends most of its time on pipelines, clusters and dashboards instead of product work.

**Monolith failure modes.** Do not romanticize the other side. A monolith's characteristic failures are build and test times that grow, merge conflicts, a release train that every team must join, one memory leak taking everything down, and a codebase no one dares to touch because boundaries were never enforced.

## In Practice

### A decision checklist

Lean toward a (modular) monolith when most of these are true:

- Fewer than a few teams work on the product.
- The domain is still being discovered and boundaries are uncertain.
- You do not have mature automated deployment, monitoring and tracing.
- Strong transactional consistency across features is needed.

Consider extracting a service when specific evidence appears:

- A component has a clearly different scaling or availability profile.
- A team is blocked repeatedly by another team's release cycle.
- A component needs a different runtime, compliance boundary or security isolation.
- The boundary has been stable for a long time inside the monolith.

### Prerequisites for microservices

Before you start, confirm you can answer yes to most of these:

1. Can you deploy any service to production automatically, many times a day, in minutes?
2. Do you have centralized logs, metrics and distributed tracing with correlation IDs?
3. Is there a standard way to handle service discovery, configuration and secrets?
4. Is there a contract testing or versioning policy?
5. Does each team have on-call ownership of its services?

### Extracting a service safely

1. Make the module boundary real inside the monolith first: no cross-module table access, communication through an interface.
2. Give the module its own schema or tables, still in the same database.
3. Route calls through a remote-capable interface, still in process.
4. Run the new service alongside the old code, and move traffic gradually (the strangler fig approach, described by Martin Fowler).
5. Migrate data ownership. Remove the old code only after the new path is proven.

### Developer versus engineer

A developer proposes microservices because they are modern, or monolith because they are simple, and argues from preference. An engineer states the problem (for example, "team A waits two weeks for team B's release"), lists what each option costs the organization, and picks the cheapest one that solves it, with a recorded trigger for revisiting. See [Architecture Decision Records](./architecture-decision-records.md).

### Verifying AI-generated designs

AI tools readily produce a ten-service diagram with a message broker for a product with one team and no users yet. Ask: which team owns each service, what is the failure mode when each call times out, and what does it cost to run? If the answers are vague, collapse the design to a modular monolith. Also check that generated service code sets timeouts and handles retries, since defaults are often missing.

## Further Reading

- *Building Microservices*, Sam Newman. Practical treatment of boundaries, data ownership and operations, including candid discussion of when not to use them.
- *Monolith to Microservices*, Sam Newman. Migration patterns, including the strangler fig.
- "Microservices", James Lewis and Martin Fowler. A widely cited article defining the style; available on martinfowler.com.
- *Designing Data-Intensive Applications*, Martin Kleppmann. Background on distributed data, consistency and failure.
- "How Do Committees Invent?", Melvin Conway (1968). The origin of Conway's Law.

Related: [Event-Driven Architecture](./event-driven-architecture.md), [Domain-Driven Design](./domain-driven-design.md), [Section index](./README.md).
