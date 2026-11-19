# Software Architecture

## Overview

Architecture is the set of structural decisions that are expensive to change: how a system is divided, how the parts communicate, and where responsibilities live. Good architecture lets a team change the system safely and cheaply for years. Bad architecture makes every change slow and risky, however clean the individual lines of code are.

For an engineer, architecture is mostly about trade-offs and communication. There is rarely a correct answer, only choices with different costs. This section teaches you to name the forces, compare options honestly, and record decisions so others can follow them. This matters more as AI tools generate larger parts of the code: a tool can produce a working service, but it does not know your constraints, your team, or what you will regret in two years.

## What you will learn

- The principles that guide structural decisions and how to apply them without dogma.
- How layered and hexagonal designs isolate business rules from infrastructure.
- When a monolith is the right choice and when services are worth their cost.
- How events decouple components, and the failure modes they introduce.
- How to use domain-driven design to find boundaries and model complex rules.
- How to record decisions in lightweight, reviewable documents.

## Topics

| Topic | What it covers |
|---|---|
| [Architecture Principles](./architecture-principles.md) | Core principles, quality attributes, and reasoning about trade-offs |
| [Layered and Hexagonal Architecture](./layered-and-hexagonal-architecture.md) | Separating business logic from infrastructure with layers, ports, and adapters |
| [Monolith vs. Microservices](./monolith-vs-microservices.md) | Deployment and organizational trade-offs, and the modular monolith |
| [Event-Driven Architecture](./event-driven-architecture.md) | Events, brokers, idempotency, outbox, and choreography versus orchestration |
| [Domain-Driven Design](./domain-driven-design.md) | Ubiquitous language, bounded contexts, aggregates, and context mapping |
| [Architecture Decision Records](./architecture-decision-records.md) | Capturing significant decisions and their consequences |

## Prerequisites

- Comfort reading and writing code in at least one general-purpose language.
- Familiarity with the ideas in [The Engineering Handbook](../README.md) fundamentals, in particular thinking in systems and managing complexity.
- Basic knowledge of HTTP APIs and relational databases.

## What good looks like

- [ ] You can state the top three quality attributes for your system and what you traded away for them.
- [ ] Business rules can be tested without starting a database or a web server.
- [ ] Module and service boundaries follow business capabilities, not technical layers.
- [ ] You can explain why your system is, or is not, split into services.
- [ ] Asynchronous consumers are idempotent and have a documented ordering and retry policy.
- [ ] Significant decisions are written down with alternatives and costs.
- [ ] Decisions include a note on how hard they are to reverse.
- [ ] You review generated designs against your own constraints before adopting them.
- [ ] Architecture is revisited when measurements or team structure change, not only at the start.

Back to [The Engineering Handbook](../README.md).
