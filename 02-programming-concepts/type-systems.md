# Type Systems

> A type system classifies values and rules out meaningless operations; used well, it turns a class of runtime failures into errors you see before the code ships.

## Overview

A type is a set of values together with the operations allowed on them. A type system is the set of rules a language uses to assign types to expressions and to reject programs that misuse them. Adding an integer to a string, calling a method that does not exist, passing a user where an order is expected: types exist to catch these mistakes.

Type systems vary widely, and the vocabulary is often misused. This topic separates the main axes: when checking happens (static or dynamic), how strict conversions are (strong or weak), how types are matched (nominal or structural), and how much the programmer must write (explicit or inferred).

For an engineer, types are a design tool as much as a safety net. They document intent, guide refactors and encode rules of the domain so invalid states cannot be constructed. They also have limits. A type checker proves some properties, not that your program does what users need.

## Core Concepts

### Static vs. dynamic typing

- **Static typing**: types are checked before the program runs (at compile time or in a separate checker). Examples: Java, Go, Rust, TypeScript.
- **Dynamic typing**: types are attached to values and checked as the program runs. Examples: Python, Ruby, JavaScript.

Many languages blend these. Python has optional type annotations checked by external tools. TypeScript adds a static layer over JavaScript, but types are erased at runtime, so data arriving from outside is not checked unless you validate it.

### Strong vs. weak typing

These terms are loosely defined, and practitioners disagree on them. A common usage: a strongly typed language refuses to silently convert between unrelated types, while a weakly typed one coerces freely.

```javascript
"5" + 1      // "51"   (JavaScript coerces the number to a string)
"5" - 1      // 4      (and here it coerces the string to a number)
```

```python
"5" + 1      # TypeError in Python: no silent coercion
```

Static and strong are independent. C is statically typed yet allows unsafe casts. Python is dynamically typed yet refuses many implicit conversions.

### Nominal vs. structural

- **Nominal**: two types are compatible only if declared related by name (Java classes and interfaces).
- **Structural**: two types are compatible if their shape matches (TypeScript object types, Go interfaces).

```go
type Reader interface{ Read(p []byte) (int, error) }
// Any type with a matching Read method satisfies Reader, no declaration needed.
```

Structural typing reduces ceremony. Nominal typing prevents accidental compatibility between types that merely look alike.

### Type inference

Inference lets the compiler deduce types so you need not write them everywhere.

```rust
let names = vec!["ada", "grace"];          // inferred as Vec<&str>
let lengths: Vec<usize> = names.iter().map(|s| s.len()).collect();
```

Annotate public interfaces explicitly and let inference handle local details.

### Generics (parametric polymorphism)

Generics let one definition work over many types while keeping type safety.

```java
static <T> T firstOrDefault(List<T> xs, T fallback) {
    return xs.isEmpty() ? fallback : xs.get(0);
}
```

### Algebraic data types and making illegal states unrepresentable

Sum types (tagged unions) model "one of these alternatives". Product types (records, tuples) model "all of these fields". Together they let you encode rules in the structure.

```typescript
type Payment =
  | { kind: "card"; last4: string }
  | { kind: "bank"; iban: string }
  | { kind: "cash" };

function describe(p: Payment): string {
  switch (p.kind) {
    case "card": return `card ending ${p.last4}`;
    case "bank": return `bank ${p.iban}`;
    case "cash": return "cash";
  }
}
```

If you add a new variant, a checker with exhaustiveness checking flags every switch that forgets it. Compare with a record holding `cardLast4`, `iban` and a string `type`, where nothing stops `type: "cash"` carrying an IBAN.

### Null and optional values

A reference that can be absent is a common source of failure. Languages respond differently: nullable by default (Java, C#, historically), explicit optional types (Rust `Option`, Swift optionals, Kotlin nullable types), or checker-enforced null checks (TypeScript with strict null checks). Tony Hoare has called his invention of the null reference a billion-dollar mistake; the lesson is to make absence visible in the type.

### Domain types

Wrapping primitives in distinct types prevents mixing values that are both just numbers.

```python
from typing import NewType
UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)

def cancel(order: OrderId) -> None: ...
cancel(UserId(7))   # flagged by a type checker
```

## Trade-offs

| Choice | Gains | Costs |
| --- | --- | --- |
| Static typing | Early error detection, tooling, refactoring safety, documentation | Up-front annotation effort, sometimes fighting the checker, slower to prototype |
| Dynamic typing | Fast iteration, flexible code, less ceremony | Errors surface at runtime, refactors rely on tests, weaker tooling |
| Strict type rules | Fewer silent bugs | More explicit conversions |
| Rich domain types | Invalid states cannot be built | More types to define and maintain |
| Gradual typing | Adopt incrementally | Gaps where `any`/untyped code leaks unchecked values |

Evidence on how much static typing reduces defects is mixed in empirical studies, and results depend on the language, team and codebase. Treat strong claims in either direction with caution. The practical benefits that most teams report are better tooling and safer large-scale refactoring.

## Failure Modes

- **False confidence from erased types.** A TypeScript interface says `age: number`, but the JSON from the network says `"42"`. The checker passed; production fails. Validate at the boundary with a runtime schema and derive the static type from it where your tools allow.
- **Escape hatches everywhere.** Widespread `any`, unchecked casts, `# type: ignore` or `unsafe` silently remove guarantees. Track and limit them, and review each one.
- **Stringly typed code.** Passing identifiers, states and currencies around as plain strings invites mix-ups. Use enums or domain types.
- **Numeric surprises.** Types do not save you from overflow, precision loss or unit errors. Floating-point money drifts; 32-bit counters wrap. Use decimal or integer minor units for money, and wider integers where counts can grow. Mixing units has destroyed real spacecraft and aircraft operations; encode units in types where it matters.
- **Null dereference.** Absent values reach code that assumed presence. Enable strict null checks and model absence explicitly.
- **Over-engineered types.** Elaborate generic hierarchies that nobody can read. If the type is harder to understand than the bug it prevents, simplify.
- **Serialization drift.** Two services each have a correct type for "Order", but the versions disagree. Types do not cross process boundaries; use versioned schemas and compatibility checks.

Detection: run the type checker in continuous integration with a strict configuration, count escape hatches over time, and test boundaries with real malformed inputs.

## In Practice

### Guidelines

1. Turn on the strictest checking your language offers for new code, and ratchet legacy code upward.
2. Put types at module boundaries first; they carry the most value there.
3. Parse, do not just validate. Convert raw input into a trusted domain type once at the edge, then pass the trusted type inward.
4. Prefer sum types over flags and optional fields that are only valid in combination.
5. Use distinct types for identifiers and units that can be confused.
6. Do not let a type error be "fixed" by a cast without understanding why it appeared.

### Worked example: parse, do not validate

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError("invalid email")

def register(email: Email) -> None:
    ...  # no need to re-check the format here
```

The check runs once, at construction. Every function taking an `Email` can rely on it.

### Developer vs. engineer

A developer sees a type error as an obstacle and silences it. An engineer reads it as information: either the code is wrong or the model of the data is wrong. Both are worth knowing.

### Verifying AI-generated code

Generated code frequently compiles because it reaches for `any`, casts or broad types such as `object` and `dict`. Check for these. Confirm that data from outside the process is validated at runtime, that nullable values are handled, and that the generated types match the real shapes your system produces rather than an invented plausible shape.

## Further Reading

- Benjamin C. Pierce, *Types and Programming Languages*.
- Yaron Minsky, Anil Madhavapeddy and Jason Hickey, *Real World OCaml* (the chapters on variants and making illegal states unrepresentable).
- Scott Wlaschin, *Domain Modeling Made Functional*.
- Alexis King, "Parse, don't validate" (essay).
- The official documentation for your language's type system, such as the TypeScript Handbook or the Python `typing` documentation.

Previous: [Programming Paradigms](./programming-paradigms.md). Next: [Memory Management](./memory-management.md).
