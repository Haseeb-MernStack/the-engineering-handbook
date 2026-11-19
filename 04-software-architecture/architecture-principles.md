# Architecture Principles

> Coupling, cohesion, separation of concerns and the SOLID principles are the vocabulary for judging whether a design will stay cheap to change, which is the main thing architecture is for.

## Overview

Architecture is the set of decisions that are expensive to reverse. Most of those decisions are about boundaries: what belongs together, what must stay apart, and what each part is allowed to know about the others. Frameworks, languages and cloud vendors change. Boundaries tend to outlive them.

The principles in this chapter are not laws. They are heuristics that compress decades of experience with systems that became hard to change. Each one answers the same question from a different angle: when a requirement changes, how many places must you touch, and how sure can you be that you broke nothing?

A developer applies these principles as a style checklist after the code works. An engineer uses them earlier, as a way to predict where change will land and to place boundaries so that likely changes stay local.

## Core Concepts

### Coupling

Coupling measures how much one module depends on another. High coupling means a change in one forces changes in the other. Coupling comes in kinds, and they are not equally harmful.

| Kind | Example | Cost |
|---|---|---|
| Data coupling | Module A passes a plain value to B | Low. The contract is small and explicit. |
| Interface coupling | A calls B through a published interface | Low to moderate. Changes need versioning. |
| Shared-schema coupling | A and B read and write the same table | High. Either team can break the other silently. |
| Temporal coupling | B only works if A ran first | High. The order is invisible in the code. |
| Content coupling | A reaches into B's internals | Very high. Nothing is safe to change. |

Coupling is not removed, only chosen. Two modules that cooperate must share something. The goal is to make the shared thing small, explicit and stable.

### Cohesion

Cohesion measures how strongly the contents of a module belong together. A cohesive module has one reason to exist and its parts change together. A low-cohesion module is a junk drawer: `Utils`, `Helpers`, `Manager`, `Common`.

The two ideas pull in one direction. High cohesion inside modules and low coupling between them is the target. If you must cross a boundary to finish most changes, the boundary is in the wrong place.

### Separation of concerns

A concern is a distinct reason for code to change: how data is validated, how it is stored, how it is presented, how it is authorized. Separation of concerns means each concern lives in one place, so a change to one does not ripple into the others.

A useful test: describe the module in one sentence without the word "and". If you cannot, it probably mixes concerns.

### The SOLID principles

SOLID is five object-oriented design heuristics. They apply beyond classes to any unit with a public contract, including modules and services.

**Single Responsibility.** A unit should have one reason to change. "Responsibility" means a stakeholder or axis of change, not "does one thing". An invoice class that formats PDFs and calculates tax has two reasons to change: the design team and the tax rules.

**Open/Closed.** A unit should be open for extension but closed for modification. New behavior should arrive by adding code, not by editing tested code. This is usually achieved with a stable abstraction and several implementations.

**Liskov Substitution.** Anything that claims to be a subtype must be usable wherever its parent is expected, without surprising the caller. A `ReadOnlyList` that throws on `add` breaks the contract of `List`. The test is behavioral, not syntactic.

**Interface Segregation.** Clients should not depend on methods they do not use. Prefer several small interfaces over one wide one. A wide interface forces every client to recompile, redeploy or re-test when any part changes.

**Dependency Inversion.** High-level policy should not depend on low-level detail. Both should depend on abstractions, and the abstraction should be owned by the high-level side. Business code defines a `PaymentGateway` interface. The Stripe adapter implements it. The dependency arrow points toward the business logic.

```python
from typing import Protocol

class PaymentGateway(Protocol):
    def charge(self, customer_id: str, cents: int) -> str: ...

class CheckoutService:
    def __init__(self, gateway: PaymentGateway):
        self._gateway = gateway

    def checkout(self, customer_id: str, cents: int) -> str:
        if cents <= 0:
            raise ValueError("amount must be positive")
        return self._gateway.charge(customer_id, cents)

# A test double satisfies the same contract without any network call.
class FakeGateway:
    def charge(self, customer_id: str, cents: int) -> str:
        return "fake-charge-1"
```

`CheckoutService` states what it needs. It does not import a vendor SDK. Swapping vendors, or testing, touches no business code.

### Other durable principles

- **Information hiding.** Each module hides a design decision likely to change. This is the original reason for modularity.
- **Least knowledge (Law of Demeter).** Talk to your direct collaborators, not to their collaborators. `order.customer.address.city` couples you to three types.
- **DRY, correctly understood.** Do not duplicate knowledge. Two pieces of code that look alike but change for different reasons are not duplication.
- **YAGNI.** Do not build for speculative requirements. Keep the design easy to extend, but do not extend it yet.

## Trade-offs

Every principle has a price.

| Principle | What it buys | What it costs |
|---|---|---|
| Low coupling via interfaces | Replaceability, testability | Indirection. More types to read. Harder to follow a call in a debugger. |
| High cohesion via small units | Local change, clear ownership | Many small files. A feature spans several of them. |
| Open/Closed via abstraction | Extension without edits | You must guess the axis of variation. A wrong guess adds abstraction nobody uses. |
| DRY via shared code | One place to fix bugs | Shared code couples its consumers. A change for one caller risks the others. |
| Dependency inversion | Business logic independent of vendors | Adapters to write and maintain. |

Two tensions come up constantly.

**DRY versus coupling.** Extracting shared code creates a dependency. Across services or teams, a small amount of duplication is often cheaper than a shared library that every team must coordinate to change. Inside one module, the balance usually favors extraction. The rule of thumb many teams use is to wait for the third occurrence before abstracting, because by then you can see what actually varies.

**Abstraction versus clarity.** An interface with one implementation and no test double is often speculative. It adds a hop for the reader and buys nothing yet. Introduce the abstraction when there is a second implementation, a test need, or a real boundary to protect. This is contested: some teams prefer interfaces at all module boundaries by policy, accepting the cost for consistency.

## Failure Modes

**The big ball of mud.** No visible structure. Every module imports every other. Changes cause distant breakage. It rarely starts that way. It arrives through hundreds of small expedient shortcuts. Detection: dependency graphs with cycles, files that change together in version-control history regardless of folder.

**Shotgun surgery.** One logical change requires small edits in many files. This signals low cohesion: a single concern is scattered. Detection: look at commits. If most feature commits touch more than a handful of unrelated directories, concerns are split badly.

**Divergent change.** The opposite: one file changes for many unrelated reasons. This signals a module with multiple responsibilities. Detection: a file with a very high change frequency and many different authors and ticket types.

**Speculative generality.** Abstractions built for requirements that never came. Symptoms: interfaces with one implementation, configuration options nobody sets, plugin systems with no plugins. The cost is cognitive load paid forever.

**Leaky abstraction.** The interface promises to hide a detail, but callers must know it anyway. An ORM that hides SQL until a slow query forces everyone to read the generated SQL. Detection: callers repeatedly working around the abstraction.

**Circular dependencies.** A depends on B depends on A. The modules cannot be built, tested or reasoned about separately. Prevention: enforce dependency direction in CI with an architecture test or a linter that fails on forbidden imports.

**God object.** One class that knows or does too much and attracts every new feature because it is already there.

## In Practice

### Make the rules executable

Architecture documents decay. Rules enforced by tools do not. Declare the allowed dependency directions and fail the build when they are violated.

```python
# Illustrative architecture test: domain code must not import infrastructure.
import pathlib, re

def test_domain_does_not_import_infrastructure():
    offenders = []
    for path in pathlib.Path("src/domain").rglob("*.py"):
        text = path.read_text()
        if re.search(r"^\s*(from|import)\s+src\.infrastructure", text, re.M):
            offenders.append(str(path))
    assert not offenders, f"Domain imports infrastructure: {offenders}"
```

Mature ecosystems have dedicated tools for this. Prefer them over hand-rolled scripts once the rules grow.

### A design review checklist

1. What is the most likely requirement change in the next year? How many modules does it touch?
2. Which dependencies point the wrong way, from stable policy to volatile detail?
3. Can each module be tested without starting the rest of the system?
4. Is there any shared mutable state or shared table between modules?
5. For each abstraction: what is the second implementation, or the test that needs it?
6. Which decisions here are hard to reverse? Have they been written down?

### Developer versus engineer

A developer sees a duplicated block and extracts it immediately, because duplication is "bad". An engineer asks whether the two blocks change for the same reason, who owns each caller, and whether the new shared dependency crosses a team boundary. The developer optimizes for the current diff. The engineer optimizes for the cost of the next ten changes.

### Verifying AI-generated designs and code

AI assistants produce plausible structure quickly. They also tend to add layers, interfaces and factories by pattern rather than by need, and to copy a common shape even when your constraints differ. When reviewing generated code:

- Check the dependency direction. Does the generated domain code import a framework or SDK?
- Look for abstractions with a single implementation and no stated reason.
- Check whether the generated module reaches into another module's internals or tables.
- Ask for the change scenario. If the assistant cannot say what the abstraction makes easier, delete it.

You remain accountable for the boundaries. The tool does not know your team size, your rate of change or your deployment constraints.

## Further Reading

- *A Philosophy of Software Design*, John Ousterhout. Deep modules, information hiding and complexity as the core enemy.
- *Clean Architecture*, Robert C. Martin. The dependency rule and the SOLID principles from their main proponent.
- "On the Criteria To Be Used in Decomposing Systems into Modules", D. L. Parnas (1972). The original argument for information hiding.
- *Refactoring*, Martin Fowler. A catalogue of code smells including shotgun surgery and divergent change.
- *Structured Design*, Edward Yourdon and Larry Constantine. Origin of the coupling and cohesion vocabulary.

Related: [Layered and Hexagonal Architecture](./layered-and-hexagonal-architecture.md), [Domain-Driven Design](./domain-driven-design.md), [Section index](./README.md).
