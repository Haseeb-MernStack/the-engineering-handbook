# Layered and Hexagonal Architecture

> Layered and hexagonal architectures are two ways of arranging code inside one deployable unit so that business rules stay independent of frameworks, databases and delivery mechanisms.

## Overview

Every application has business rules and a lot of machinery around them: HTTP handlers, database drivers, message consumers, schedulers. The central question of in-process architecture is which of these may know about which. Get the direction wrong and your pricing rules end up importing a web framework. Get it right and you can change the database, the transport or the UI without rewriting the rules.

Two families of answers dominate. The classic **layered** (n-tier) architecture stacks responsibilities and lets each layer call the one below. The **hexagonal** family, also called ports and adapters, with close relatives known as onion and clean architecture, puts the business rules in the center and makes everything else an adapter that depends inward.

The two are not rivals so much as points on a spectrum. Many healthy systems are layered with one important correction: dependencies on infrastructure are inverted.

## Core Concepts

### Classic layered architecture

```
+---------------------------+
|       Presentation        |  HTTP handlers, CLI, UI
+---------------------------+
|       Application /       |  use cases, orchestration
|        Service layer      |
+---------------------------+
|         Domain            |  entities, rules
+---------------------------+
|     Data access /         |  repositories, ORM, SQL
|     Infrastructure        |
+---------------------------+
   dependencies point down
```

Each layer depends only on the layer directly below (strict layering) or on any layer below (relaxed layering). The appeal is simplicity. Everyone understands "controller calls service calls repository".

The weak point is the bottom. In the naive form the domain layer depends on the data layer. The business rules therefore know about tables, ORM annotations and query languages. Testing a rule requires a database, or a heavy mock of one.

### Hexagonal architecture (ports and adapters)

Alistair Cockburn described the pattern to keep application logic isolated from external actors. The application core defines **ports**: interfaces that describe what it offers and what it needs. **Adapters** translate between a port and a concrete technology.

```
          driving side                            driven side
   (who calls the application)           (what the application calls)

  [HTTP adapter] --\                          /--> [SQL adapter]
  [CLI adapter]  ---+--> (inbound port)       |
  [Test harness] --/          |               +--> [Email adapter]
                         +----v----+          |
                         |  Core   |----------+--> [Payment adapter]
                         | (domain |  (outbound ports)
                         |  + use  |
                         |  cases) |
                         +---------+
        all arrows point toward the core's interfaces
```

- **Inbound (driving) ports** are the use cases: `PlaceOrder`, `CancelSubscription`. Adapters such as an HTTP controller call them.
- **Outbound (driven) ports** are what the core needs: `OrderRepository`, `PaymentGateway`, `Clock`. The core defines the interface. Adapters implement it.

The decisive rule is that source-code dependencies point inward. The core imports no framework, driver or SDK.

### Onion and clean architecture

These variants add named concentric rings: entities at the center, use cases around them, interface adapters next, frameworks and drivers outside. The rule is the same: a ring may depend on inner rings, never on outer ones. The differences between hexagonal, onion and clean are mostly vocabulary and emphasis. Do not spend review time arguing which label applies.

### A concrete shape

```
src/
  domain/            # entities, value objects, domain services. No imports from outside.
    order.py
  application/       # use cases and the ports they need
    place_order.py
    ports.py         # OrderRepository, PaymentGateway (interfaces)
  adapters/
    inbound/
      http_api.py    # calls use cases
    outbound/
      postgres_orders.py
      stripe_payments.py
  main.py            # composition root: wires adapters to ports
```

```python
# application/ports.py
from typing import Protocol
from domain.order import Order

class OrderRepository(Protocol):
    def get(self, order_id: str) -> Order: ...
    def save(self, order: Order) -> None: ...

# application/place_order.py
class PlaceOrder:
    def __init__(self, orders: OrderRepository, payments: "PaymentGateway"):
        self._orders, self._payments = orders, payments

    def execute(self, order_id: str) -> None:
        order = self._orders.get(order_id)
        order.confirm()                       # domain rule
        self._payments.charge(order.customer_id, order.total_cents)
        self._orders.save(order)
```

The **composition root** (`main.py`) is the one place that knows every concrete class. It builds adapters and passes them in. Dependency injection frameworks are optional. Constructor arguments are enough.

### Where does each kind of logic live

| Logic | Location | Example |
|---|---|---|
| Business invariant | Domain | An order cannot ship before it is paid. |
| Workflow orchestration | Application layer | Load order, charge, save, publish event. |
| Input shape and format | Inbound adapter | Parse JSON, map to a command, map errors to status codes. |
| Persistence mapping | Outbound adapter | Map `Order` to rows. |
| Cross-cutting | Edges or decorators | Authentication, logging, metrics. |

## Trade-offs

| Choice | Benefits | Costs |
|---|---|---|
| Plain layered | Fast to start. Familiar. Little ceremony. | Rules leak into controllers or SQL. Domain is tied to the persistence model. Tests need a database. |
| Hexagonal | Fast, database-free tests of the core. Replaceable infrastructure. Clear place for each kind of logic. | More files and mapping code. Indirection to follow. Easy to over-apply. |
| Full clean architecture | Strongest isolation | Highest ceremony. Separate models per ring means a lot of copying between near-identical structures. |

Specific costs to weigh:

- **Mapping overhead.** Keeping a domain model separate from the persistence model means translating between them. For a simple create-read-update-delete service, this is pure cost.
- **The "swap the database" claim is weaker than advertised.** Teams rarely change database vendors. The more realistic gains are testability and keeping rules readable. Justify the architecture by those, not by a hypothetical migration.
- **Performance.** Hiding persistence behind a repository can discourage set-based queries and produce N+1 patterns. The abstraction must still allow efficient access.
- **Contested ground.** Some engineers argue that thin CRUD services gain nothing from ports and adapters. Others use it everywhere for uniformity. Match the ceremony to the complexity of the domain, not to fashion.

## Failure Modes

**Anemic domain model.** Entities are bags of getters and setters. All logic lives in service classes. You have paid for layers and receive procedural code with extra steps. Detection: domain classes with no behavior, and services full of `if` statements over entity fields.

**Leaky persistence.** ORM entities are used directly as domain objects and API responses. A column rename changes the public API. Lazy loading triggers queries in the presentation layer. Detection: ORM annotations in domain code, queries firing during JSON serialization.

**Pass-through layers.** A service method that only calls the repository with the same arguments. The layer adds noise without value. Detection: methods whose body is one delegating line, repeated across a whole layer.

**Layer skipping and cycles.** Controllers query the database directly "just this once". Over time the layers become decoration. Prevention: executable dependency rules in the build.

**The fat use case.** Without a rich domain, use cases absorb all rules and grow into long procedural scripts. Business rules are then untestable in isolation.

**Port design that mirrors the technology.** An outbound port named `SqlOrderQuery` with methods that take SQL fragments is not an abstraction. The port should speak the language of the domain: `find_unshipped_orders(customer)`.

**Transaction boundaries in the wrong place.** If each repository call opens its own transaction, a use case that spans several is not atomic. Decide who owns the transaction. Usually the use case defines the unit of work and the adapter implements it.

**Over-abstraction early.** A two-endpoint prototype gets six layers and four mapping classes. It ships late and no one can find the logic.

## In Practice

### Choosing a starting point

1. Is there real domain logic: invariants, calculations, state machines? If not, a simple layered or even a thin script is appropriate.
2. Will the code live for years and be changed by many people? If yes, invest in boundaries.
3. Are external dependencies slow or costly to run in tests (payments, email, third-party APIs)? Ports pay for themselves quickly.
4. Start with the dependency rule only: the domain imports nothing from outside. Add more structure when pain appears.

### A refactoring path from layered to hexagonal

1. Identify one outbound dependency the domain currently imports, such as the ORM session.
2. Define an interface in the application or domain code that expresses what the use case needs.
3. Implement it in an adapter that wraps the existing code. Behavior does not change.
4. Inject the adapter through the constructor. Delete the old import.
5. Write a fast test using an in-memory fake of the interface.
6. Repeat for the next dependency. Stop when the pain stops.

### Testing strategy

| Test type | What it exercises | Speed |
|---|---|---|
| Domain unit tests | Entities and rules, no I/O | Very fast |
| Use case tests with fakes | Orchestration against in-memory ports | Fast |
| Adapter integration tests | Real database or real HTTP against the adapter alone | Slower |
| A few end-to-end tests | Wiring and configuration | Slowest |

A fake should be tested against the same contract as the real adapter so that the two do not drift. Run the same repository test suite against both.

### Developer versus engineer

A developer places code where the framework tutorial put it, usually in the controller. An engineer asks what must survive a framework change and puts exactly that behind the dependency rule. The engineer also knows when not to: a small internal tool does not need a ring diagram.

### Verifying AI-generated code

Generated code often defaults to the framework's idiom: business logic in route handlers, ORM models passed everywhere. Or it overshoots and creates `IUserServiceImpl`-style layers for a three-field form. Check three things: does the domain import infrastructure, does each layer do real work, and can the main use case run in a test without a database?

## Further Reading

- "Hexagonal Architecture" (Ports and Adapters), Alistair Cockburn. The original article is on his website.
- *Clean Architecture*, Robert C. Martin. The dependency rule and concentric rings.
- *Patterns of Enterprise Application Architecture*, Martin Fowler. Layering, repositories, unit of work, domain model versus transaction script.
- *Domain-Driven Design*, Eric Evans. The layered architecture and the domain model it protects.
- "The Onion Architecture", Jeffrey Palermo. A series of blog posts that introduced the onion naming.

Related: [Architecture Principles](./architecture-principles.md), [Domain-Driven Design](./domain-driven-design.md), [Section index](./README.md).
