# Programming Paradigms

> A paradigm is a style of organizing programs around a central idea; knowing several lets you pick the one that fits the problem instead of the one your language defaults to.

## Overview

Every language nudges you toward a way of structuring code. Some push you to describe step-by-step commands. Some push you to compose functions. Some push you to model the world as objects that send messages. These styles are called paradigms.

Paradigms are not religions and they are not mutually exclusive. Most modern languages are multi-paradigm. Python, JavaScript, Kotlin, Rust, C# and Java all let you mix procedural, object-oriented and functional techniques. The skill is knowing which technique reduces complexity for the task in front of you.

A developer who learned one paradigm tends to apply it everywhere, producing deep class hierarchies for simple data transforms or clever function chains where a loop would be clearer. An engineer understands what each paradigm is good at, what it costs, and how the team will read the result in a year.

## Core Concepts

### Imperative and procedural

Imperative code is a sequence of statements that change state. Procedural code organizes those statements into named procedures.

```c
int sum(const int *xs, size_t n) {
    int total = 0;
    for (size_t i = 0; i < n; i++) {
        total += xs[i];
    }
    return total;
}
```

Strengths: it maps closely to how the machine works, it is easy to trace, and it is often the fastest to write and run. Weakness: shared mutable state spreads. As a program grows, it becomes hard to know which code changed a given variable.

### Object-oriented

Object-oriented programming (OOP) bundles data with the operations on it. The main ideas are:

- **Encapsulation**: hide internal state behind a small interface so it can change without breaking callers.
- **Polymorphism**: let different types respond to the same call in their own way.
- **Inheritance**: reuse behavior by extending another type. This is the most overused idea. Many practitioners prefer composition (holding another object and delegating to it) because it couples code less.

```python
class Notifier:
    def send(self, user, message): ...

class EmailNotifier(Notifier):
    def send(self, user, message): ...

class SmsNotifier(Notifier):
    def send(self, user, message): ...

def alert(notifier: Notifier, user, text):
    notifier.send(user, text)   # caller does not know which kind
```

OOP fits domains with many entities that have identity and lifecycle, and systems built around pluggable behavior. It fits poorly where data is just data and the work is a pipeline of transformations.

### Functional

Functional programming builds programs from functions that map inputs to outputs. Key ideas:

- **Pure functions**: the result depends only on the arguments, and there are no side effects. They are easy to test and safe to run in any order or in parallel.
- **Immutability**: values are not modified after creation. You build new values instead.
- **Higher-order functions**: functions that take or return functions, such as `map`, `filter` and `reduce`.
- **Expressions over statements**: computations return values rather than changing variables.

```python
def total_paid(orders):
    return sum(o.amount for o in orders if o.status == "paid")
```

Functional style shines for data transformation, parsing, rule evaluation and anything where you want reasoning without tracking hidden state. It can be awkward for code that is fundamentally about sequencing effects, such as I/O, and deeply recursive styles can hurt readability in languages without tail-call support.

### Declarative

Declarative code states what result you want, not how to compute it. SQL, HTML, CSS, regular expressions and build configuration are examples. The runtime chooses the execution plan.

```sql
SELECT customer_id, SUM(amount)
FROM orders
WHERE status = 'paid'
GROUP BY customer_id;
```

The benefit is that the engine can optimize without you changing the query. The cost is less control: when the engine picks a poor plan, you must learn how it decides and sometimes work around it.

### Event-driven and reactive

Here the flow of control is set by events: a click, a message on a queue, a file arriving. Handlers react to events. This suits user interfaces, servers handling many connections and decoupled services. It makes the order of execution harder to read from the source, because the control flow is spread across handlers and the runtime.

### Comparison

| Paradigm | Central idea | Good for | Common pitfall |
| --- | --- | --- | --- |
| Imperative | Commands change state | Low-level, performance-critical code | Tangled shared state |
| Object-oriented | Objects own state and behavior | Domain models, plugin points | Deep inheritance, anemic or god objects |
| Functional | Pure functions and immutable data | Transformations, concurrency safety | Over-abstraction, allocation overhead |
| Declarative | Describe the result | Queries, configuration, UI layout | Opaque performance, hard to debug the plan |
| Event-driven | React to events | UIs, messaging, I/O-heavy servers | Hard-to-trace control flow |

## Trade-offs

**Mutable state vs. immutability.** Mutation is cheap in memory and time and matches hardware. It makes reasoning, testing and concurrency harder. Immutability makes reasoning easy and sharing safe, but creating new values costs allocations. Persistent data structures reduce that cost but do not remove it.

**Inheritance vs. composition.** Inheritance gives quick reuse but ties a subclass to the internals of its parent. Changes in the parent can break children (the fragile base class problem). Composition costs a little more boilerplate for delegation and buys flexibility.

**Abstraction vs. directness.** Abstractions hide detail and let you change implementations. Each layer also hides behavior from the reader and adds indirection to debug. An abstraction with one implementation and no foreseeable second is often a cost with no benefit.

**Consistency vs. fit.** Using one style across a codebase makes it predictable. Forcing a style on a problem it does not fit makes the code worse. Prefer the conventions of the language and the existing codebase unless there is a clear reason.

**Team familiarity.** A technique only the author understands is a liability. The best paradigm is partly the one the team can read and maintain.

## Failure Modes

- **The god object.** One class accumulates unrelated responsibilities until every change touches it. Symptoms: huge files, many collaborators, frequent merge conflicts. Prevent by splitting on responsibilities and reviewing class size and dependency count.
- **Inheritance hierarchy lock-in.** A tree of subclasses cannot express a new requirement that crosses branches. Detect it when you start adding flags or overriding methods to do nothing. Refactor toward composition early.
- **Hidden mutation.** A function mutates an argument or a shared object, and a caller elsewhere sees surprising changes. A classic Python example is a mutable default argument:

  ```python
  def add(item, bucket=[]):   # one list shared across all calls
      bucket.append(item)
      return bucket
  ```

  Use `None` as the default and create the list inside the function.
- **Purity in name only.** A function labeled pure reads the clock, a global or the network. Tests become flaky. Pass such dependencies in as parameters.
- **Clever chains.** Long chains of higher-order calls that nobody can step through in a debugger. If you need a comment to explain a chain, a named intermediate variable or loop is often clearer.
- **Callback tangles.** Event-driven code where ordering assumptions are implicit leads to race conditions that appear only under load. Log event sequences and make ordering requirements explicit.

## In Practice

### Choosing a style

1. Is the work a transformation of data into other data? Lean functional.
2. Does the domain have entities with identity, state over time and rules about their changes? Lean object-oriented, with small interfaces.
3. Is the work a query over structured data? Use the declarative tool built for it.
4. Is the work I/O driven by external events? Use the event model your platform provides.
5. Is performance dominated by memory layout or tight loops? Use straightforward imperative code and measure.

### Habits that carry across paradigms

- Keep functions small and focused on one job.
- Separate computation from side effects. Put decisions in pure functions and push I/O to the edges.
- Make state explicit. Know who owns each piece of mutable data.
- Prefer composition and small interfaces.
- Name things for what they mean in the domain.

### Developer vs. engineer

A developer reaches for the pattern they know. An engineer asks what makes this particular code easy to change and verify, and picks accordingly. They also accept the local convention even if they would have chosen differently, because consistency is itself a property worth protecting.

### Verifying AI-generated code

Generated code often mixes styles without regard for your codebase, or applies a pattern by rote (a factory for a single class, an interface for one implementation). Check that it follows the local conventions, that it does not mutate inputs unexpectedly, and that any abstraction it introduces earns its place. Ask whether you would have written that structure on purpose.

## Further Reading

- Harold Abelson and Gerald Jay Sussman, *Structure and Interpretation of Computer Programs*.
- Erich Gamma, Richard Helm, Ralph Johnson and John Vlissides, *Design Patterns: Elements of Reusable Object-Oriented Software*.
- Peter Van Roy and Seif Haridi, *Concepts, Techniques, and Models of Computer Programming*.
- Joshua Bloch, *Effective Java* (the items on composition over inheritance and immutability apply beyond Java).
- Michael Feathers, *Working Effectively with Legacy Code*.

Next: [Type Systems](./type-systems.md). Back to the [section overview](./README.md).
