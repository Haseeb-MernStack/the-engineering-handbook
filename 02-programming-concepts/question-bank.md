# Question Bank: Programming Concepts

> Questions for interviews, design reviews and self-testing. Each has a model answer behind a toggle, so you can commit to your own answer first.

## How to use this page

- **Self-testing.** Answer aloud or in writing before opening the toggle. Compare for gaps, not for matching wording.
- **Interviewing.** Pick questions by level and use *Red flags* as a rubric. Follow up with "what would change your answer?"
- **Design review.** The *Applied* questions work as prompts for code review.

Difficulty: **E** entry, **M** mid-level, **S** senior.

## Paradigms and design

### 1. (E) Why do many teams prefer composition over inheritance?

<details>
<summary>Model answer</summary>

Inheritance couples a subclass to the internals of its parent, so a change in the parent can break many children, and deep hierarchies are hard to follow. Composition builds behavior by combining smaller parts that can be swapped, and keeps dependencies explicit. Inheritance still fits real "is-a" relationships with stable base types.

**Strong answers:**
- Give a concrete example of a fragile base class.
- Avoid absolutes; state when inheritance is reasonable.

**Red flags:** Says inheritance is always wrong.
</details>

### 2. (M) What is a pure function and why does it matter?

<details>
<summary>Model answer</summary>

A pure function returns the same output for the same input and has no side effects: it does not read or write hidden state, perform I/O or mutate its arguments. Pure functions are easy to test, reason about, cache and run in parallel. Real systems need side effects, so a common design keeps the core logic pure and pushes I/O to the edges.

**Strong answers:**
- Mention the cost: passing state explicitly can be verbose.
</details>

## Type systems

### 3. (E) What is the difference between static and strong typing?

<details>
<summary>Model answer</summary>

They are separate dimensions. Static typing checks types before the program runs, dynamic typing checks at runtime. Strong typing means the language resists implicit conversions between unrelated types, while weak typing allows many implicit conversions. A language can be dynamic and strong, or static and weak. Definitions vary, so state which you mean.

**Red flags:** Treats the two as the same thing.
</details>

### 4. (M) The code type-checks, but a runtime error shows an unexpected field in a request. What happened?

<details>
<summary>Model answer</summary>

Types describe what the compiler knows, not what arrives over the wire. Data from outside the process, such as JSON, forms and database rows, must be validated at the boundary. A type assertion or cast only tells the compiler to trust the value. Use a schema validator or parser that checks shape and constraints at runtime, then work with the validated type inside.

**Strong answers:**
- Distinguish compile-time guarantees from runtime guarantees.
- Parse, do not just check: return a typed value after validation.
</details>

### 5. (M) How can types help make invalid states unrepresentable?

<details>
<summary>Model answer</summary>

Model the domain so that illegal combinations cannot be built. For example, use a sum type or tagged union with `Pending`, `Paid` and `Failed` states, each carrying only its relevant data, rather than a record with a status string and several optional fields. Use distinct types for identifiers or units so they cannot be mixed up. Then the compiler rejects whole categories of bugs.

**Strong answers:**
- Mention the cost: more types and more ceremony.
</details>

## Memory

### 6. (M) How can a garbage-collected program leak memory?

<details>
<summary>Model answer</summary>

Garbage collection frees objects that are unreachable. A leak occurs when objects remain reachable but are no longer needed. Typical causes: caches or maps that only grow, listeners or callbacks never unregistered, long-lived collections holding references, and global state. Find them with heap snapshots, comparing growth over time and following references to a root.

**Strong answers:**
- Bound caches and define eviction.
- Mention that growth over time with stable load is the signal.
</details>

### 7. (S) Latency spikes appear every few minutes in a managed-runtime service. What do you investigate?

<details>
<summary>Model answer</summary>

Check whether the spikes line up with garbage collection pauses using runtime GC logs or metrics. Then look at allocation rate, heap size and object lifetimes. Reduce allocation in hot paths, reuse objects where it helps, size the heap appropriately, and consider collector settings or alternatives. Confirm by correlating pause times with request latency.

**Strong answers:**
- Measure before tuning.
- Note that tuning has trade-offs between throughput, pause time and memory.
</details>

## Concurrency

### 8. (E) What is a race condition?

<details>
<summary>Model answer</summary>

A race condition is a bug where the outcome depends on the timing or ordering of concurrent operations on shared state. For example, two requests read a balance, each add to it and write it back, so one update is lost. Fix it by removing shared mutable state, using an atomic operation, a lock, or a database transaction or constraint that enforces correctness.

**Strong answers:**
- Give the lost-update example.
- Note that races are timing-dependent and often invisible in tests.
</details>

### 9. (M) How do deadlocks form, and how do you prevent them?

<details>
<summary>Model answer</summary>

A deadlock occurs when tasks each hold a resource and wait for another that is held by one of them, so none can proceed. For example, one thread locks A then B while another locks B then A. Prevent by acquiring locks in a consistent global order, holding locks for as short a time as possible, avoiding calls to unknown code while holding a lock, and using timeouts or lock-free designs where suitable.

**Strong answers:**
- Mention detection through thread dumps.
</details>

### 10. (M) A service handles each incoming request by spawning a task. Under a traffic spike it collapses. Why?

<details>
<summary>Model answer</summary>

Unbounded concurrency lets load create unlimited tasks, which exhaust memory, threads, file descriptors or downstream capacity, and each task slows the others. Bound concurrency with worker pools, semaphores or queues of limited size, apply backpressure or reject excess load, and set timeouts. Protect downstream dependencies with their own limits.

**Strong answers:**
- Describe what happens at the limit: reject, queue or shed.
- See [Service Resilience](../07-backend-engineering/service-resilience.md).
</details>

## Error handling

### 11. (E) Why is catching an exception and ignoring it a problem?

<details>
<summary>Model answer</summary>

The failure disappears while its effects remain: data may be half-written, the caller assumes success, and nobody can diagnose the cause later. Handle an error where you can do something meaningful about it. Otherwise propagate it with context, and log or report it once at a boundary.

**Red flags:** Catches the broadest exception type to "keep the program running".
</details>

### 12. (M) When is it safe to retry a failed request?

<details>
<summary>Model answer</summary>

When the operation is idempotent, or made so with an idempotency key, and the failure is likely to be transient. Use limited retries with exponential backoff and jitter, and a total time budget. Do not retry errors that will not change, such as validation failures. Retries multiply load on a struggling dependency, so combine them with timeouts and circuit breaking.

**Strong answers:**
- Mention duplicate side effects, such as double charges, as the key risk.
</details>

## Clean code and refactoring

### 13. (M) You need to change untested, messy code. How do you proceed?

<details>
<summary>Model answer</summary>

First add characterization tests that capture current behavior, even if imperfect. Then refactor in small steps, running the tests after each, and keep behavior changes separate from structural changes in commits. Find seams where you can isolate dependencies. Make the requested change once the area is safe to work in.

**Strong answers:**
- Avoid rewriting from scratch without a clear reason and a plan.
</details>

### 14. (S) When is duplication better than abstraction?

<details>
<summary>Model answer</summary>

When code looks alike by coincidence but changes for different reasons, forcing it into one abstraction couples unrelated things and produces flags and special cases. Wait until a pattern has appeared several times and the shared concept is clear. A wrong abstraction costs more to unwind than duplicated code.

**Strong answers:**
- Judge by reasons to change, not by text similarity.
</details>

## AI-era verification

### 15. (M) A generated function works on the sample input. What do you check before merging?

<details>
<summary>Model answer</summary>

Check empty, null, large and malformed inputs. Check error paths: are errors swallowed or are timeouts missing? Check concurrency safety and hidden shared state. Check resource use: unbounded growth, unclosed resources, blocking calls in async code. Check that validation matches real data. Confirm tests test behavior rather than restating the implementation.

**Strong answers:**
- Treat the sample passing as a weak signal.
</details>

## Scoring and follow-up

Mark each question as *clear*, *partial* or *gap*, then reread the matching topic:

| Questions | Topic |
| --- | --- |
| 1 to 2 | [Programming Paradigms](./programming-paradigms.md) |
| 3 to 5 | [Type Systems](./type-systems.md) |
| 6 to 7 | [Memory Management](./memory-management.md) |
| 8 to 10 | [Concurrency and Parallelism](./concurrency-and-parallelism.md) |
| 11 to 12 | [Error Handling](./error-handling.md) |
| 13 to 14 | [Clean Code and Refactoring](./clean-code-and-refactoring.md) |
| 15 | Any of the above, starting with *Failure Modes* |

Then check yourself with the [Self-Assessment](./self-assessment.md). Back to the [section index](./README.md).
