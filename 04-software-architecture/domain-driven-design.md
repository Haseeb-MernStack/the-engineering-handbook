# Domain-Driven Design

> Domain-driven design is an approach to complex software that puts a shared model of the business at the center, so code, conversation, and boundaries all reflect how the domain actually works.

## Overview

Most failed software projects do not fail on technology. They fail because the code models the wrong thing, or because developers and domain experts mean different things by the same word. A "customer" to sales is a lead. To billing it is an account. To support it is a person with a problem. One class named `Customer` that serves all three becomes a tangle.

Domain-driven design (DDD), introduced by Eric Evans, addresses this. It has two parts:

- **Strategic design**: how to divide a large domain into parts and how those parts relate. This is the more valuable part for most teams.
- **Tactical design**: patterns for modeling inside one part, such as entities, value objects, and aggregates.

DDD is not needed for every system. A simple CRUD application over a table gains little. DDD pays off where the business rules are complex, change often, and are a source of competitive value. Applying heavy tactical patterns to a trivial domain is a common and costly mistake.

## Core Concepts

### Ubiquitous language

The team builds a single vocabulary with domain experts and uses it everywhere: in conversation, tickets, tests, and code. If the business says "policy lapses," the code has `policy.lapse()`, not `policy.setStatus(3)`.

When the language in the code drifts from the language of the business, bugs hide in the gap. Renaming code to match the domain is real design work, not cosmetics.

### Bounded contexts

A bounded context is an explicit boundary within which a model and its language are consistent. Inside it, `Customer` means one thing. In another context, the same word may mean something else, and that is fine.

```
 Sales context          Billing context         Support context
 +------------+         +--------------+        +--------------+
 | Lead       |         | Account      |        | Requester    |
 | Opportunity|  --->   | Invoice      |  --->  | Ticket       |
 +------------+         +--------------+        +--------------+
        translation at the boundary (contracts, mapping)
```

Bounded contexts are about models and language, not about deployment. A context may live in a module of a monolith or in a separate service. They are, however, a good starting point for choosing service boundaries (see [Monolith vs. Microservices](./monolith-vs-microservices.md)).

### Context mapping

Contexts must integrate. The relationship between two contexts is a design decision. Common patterns:

| Pattern | Meaning |
|---|---|
| Shared kernel | Two contexts share a small, jointly owned model. Needs tight coordination. |
| Customer-supplier | Upstream supplies; downstream's needs are considered in planning. |
| Conformist | Downstream accepts upstream's model as is. |
| Anti-corruption layer | Downstream translates upstream's model so it does not leak in. |
| Open host / published language | Upstream offers a stable, documented interface for many consumers. |

An anti-corruption layer is especially useful when integrating with a legacy system or external vendor whose model you do not want to adopt. It is the same idea as the ports and adapters in [Layered and Hexagonal Architecture](./layered-and-hexagonal-architecture.md).

### Subdomains

Strategic design classifies parts of the business:

- **Core**: what differentiates the business. Invest your best people and deepest modeling here.
- **Supporting**: needed, but not differentiating. Build simply.
- **Generic**: solved problems such as authentication or payments. Buy or use existing solutions.

This classification tells you where DDD effort belongs. Spending sophisticated modeling on a generic subdomain wastes the budget.

### Tactical building blocks

**Entity.** Has an identity that persists over time. Two orders with identical contents are still different orders.

**Value object.** Defined by its attributes, immutable, no identity. `Money(10, "EUR")`, `DateRange`, `EmailAddress`. Value objects validate on creation, so invalid states cannot exist. They remove many bugs that primitives allow, such as adding euros to dollars.

**Aggregate.** A cluster of objects treated as one consistency unit, with a single root. Outside code can reference only the root. The root enforces the invariants of the whole cluster.

**Domain event.** A fact that happened in the domain. See [Event-Driven Architecture](./event-driven-architecture.md).

**Repository.** Provides collection-like access to aggregates, hiding persistence.

**Domain service.** Domain logic that does not naturally belong to one entity.

### Aggregates and invariants

The key design rule: an aggregate is a transaction boundary. Everything inside is changed atomically. Between aggregates, consistency is eventual.

```python
class Order:
    def __init__(self, order_id, customer_id):
        self.id = order_id
        self.customer_id = customer_id   # reference by id, not by object
        self._lines = []
        self.status = "OPEN"

    def add_line(self, sku, qty, unit_price):
        if self.status != "OPEN":
            raise ValueError("cannot modify a submitted order")
        if qty <= 0:
            raise ValueError("quantity must be positive")
        self._lines.append((sku, qty, unit_price))

    def submit(self):
        if not self._lines:
            raise ValueError("an order needs at least one line")
        self.status = "SUBMITTED"
```

The rules live in the model. No caller can bypass them by setting fields directly.

Guidelines for aggregates:

- Keep them small. Large aggregates cause lock contention and slow loads.
- Reference other aggregates by identity, not by holding the object.
- Change one aggregate per transaction when possible.
- Design around invariants that must be immediately true, not around data relationships.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Rich domain model | Rules in one place, testable without infrastructure | More code and design effort than simple CRUD |
| Separate bounded contexts | Models stay clean and teams stay independent | Translation code; duplicated concepts; integration work |
| Small aggregates | Low contention, fast operations | Cross-aggregate rules become eventual; needs events or sagas |
| Large aggregates | Strong consistency across more data | Contention, heavy loads, harder scaling |
| Anti-corruption layer | Protects your model from external change | Extra code to build and maintain |
| Shared kernel | Less duplication | Coupling and coordination between teams |

DDD also costs people time. It needs sustained access to domain experts. Without that access, the "model" is guesswork. Some practitioners also argue that its tactical patterns are over-applied; use them selectively.

## Failure Modes

**Anemic model.** Entities are bags of getters and setters; all logic lives in service classes. Rules are duplicated and inconsistently enforced. Detect by finding the same validation in several services. Fix by moving behavior into the model.

**One model for everything.** A single enterprise-wide `Customer` with 80 fields serves every team. Every change needs a negotiation, and every team is blocked by the others. Split by context.

**Boundaries by technical layer.** Contexts drawn as "the database team" and "the UI team" instead of by business capability. Each feature then crosses every boundary.

**Giant aggregates.** An `Account` aggregate containing every transaction ever. Loading it is slow and concurrent updates collide. Detect through lock-wait metrics and optimistic-lock failures. Shrink the aggregate and reference by ID.

**Language drift.** Code names diverge from business terms. New engineers learn two vocabularies and mistranslate. Review names in code review; ask a domain expert when unsure.

**Cargo-cult DDD.** Repositories, factories, and aggregates for a simple form-over-database app. The ceremony outweighs the value. Apply tactical patterns only where the domain is complex.

**Leaky integration.** An upstream model change breaks downstream code because no translation layer exists. Add an anti-corruption layer at important boundaries.

## In Practice

### Starting without ceremony

1. Talk to domain experts. Write down the terms they use and where they disagree.
2. Run an event-storming style workshop: list the domain events on a timeline, then the commands that cause them, then group them. Groupings often reveal context boundaries.
3. Classify subdomains as core, supporting, or generic.
4. Draw a context map showing integrations and their relationship patterns.
5. Model the core subdomain richly. Keep the rest simple.

### Checklist

- [ ] Domain terms in code match terms used by the business.
- [ ] Each bounded context has a named owner.
- [ ] Context boundaries and integration styles are documented.
- [ ] Invariants are enforced inside aggregates, not in callers.
- [ ] Value objects are used for concepts such as money, quantities, and identifiers.
- [ ] Aggregates are small and reference each other by ID.
- [ ] Generic subdomains use existing solutions.

### Developer vs. engineer

A developer builds tables and endpoints from a requirements list, mirroring the schema in the code. An engineer first asks what the business rule is, who owns it, which words are ambiguous, and which rules must always hold. The result is a smaller model that is harder to misuse.

### Verifying AI-generated designs

Generated domain models tend to be anemic data classes with a service layer, and tend to merge distinct concepts into one entity because the prompt used one word. Check that:

- The language matches your domain, not generic textbook names.
- Invariants are enforced in the model, not only in controllers.
- Aggregate boundaries match real transaction needs.
- One concept is not stretched across several contexts.
- Primitives such as `float` for money have been replaced by value objects.

## Further Reading

- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software*.
- Vaughn Vernon, *Implementing Domain-Driven Design*.
- Vaughn Vernon, *Domain-Driven Design Distilled*.
- Alberto Brandolini, *Introducing EventStorming*.
- Scott Wlaschin, *Domain Modeling Made Functional*.
