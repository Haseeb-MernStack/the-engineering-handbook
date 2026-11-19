# Error Handling

> Error handling is the discipline of detecting, representing, reporting and recovering from things going wrong; production reliability is largely decided by how well it is done.

## Overview

Code runs on the happy path in demos and on every other path in production. Files are missing, networks time out, inputs are malformed, disks fill up, dependencies return nonsense. A program that handles only success is a prototype.

Error handling has four jobs:

1. **Detect** that something went wrong, as close to the cause as possible.
2. **Represent** the failure in a form that carries enough information to act on.
3. **Decide** who is able to handle it: the current code, the caller or a human.
4. **Report** it so that operators can find and fix the cause.

A developer often treats errors as noise to silence. An engineer treats them as part of the interface: a function's failure behavior is as much a contract as its return value.

## Core Concepts

### Kinds of failure

| Kind | Examples | Typical response |
| --- | --- | --- |
| Expected, recoverable | User input invalid, file not found, timeout | Handle locally or return a clear error; retry if safe |
| Programmer error | Null dereference, violated invariant, index out of range | Fail fast, fix the code; do not "recover" silently |
| Environmental, transient | Network blip, dependency overloaded | Retry with limits and backoff |
| Environmental, persistent | Disk full, misconfiguration, expired credential | Alert a human; degrade gracefully |
| Fatal | Out of memory, corrupt state | Stop safely; restart supervised |

Separating these is the foundation. Retrying a programmer error just repeats it. Crashing on a transient timeout wastes availability.

### Mechanisms

**Return codes** make failure explicit but are easy to ignore (C).

**Exceptions** separate the error path from the normal path and unwind the stack to a handler.

```python
def load_config(path):
    try:
        with open(path) as f:
            return parse(f.read())
    except FileNotFoundError as e:
        raise ConfigError(f"config not found: {path}") from e
```

Exceptions keep happy-path code readable, but any call may throw, so control flow is invisible at the call site. Checked exceptions (Java) make this visible; many teams found them burdensome, and opinion is divided.

**Result or error-value types** make failure part of the type, so the compiler forces the caller to deal with it.

```rust
fn parse_port(s: &str) -> Result<u16, std::num::ParseIntError> {
    s.trim().parse::<u16>()
}

match parse_port(input) {
    Ok(p) => start(p),
    Err(e) => eprintln!("bad port: {e}"),
}
```

**Multiple return values** are the Go convention:

```go
f, err := os.Open(path)
if err != nil {
    return fmt.Errorf("open config %q: %w", path, err)
}
defer f.Close()
```

None of these is universally best. Follow the idiom of your language and codebase, and be consistent.

### Fail fast

Detect invalid state early and stop, rather than carrying bad data deep into the system where the failure becomes a mystery. Validate inputs at boundaries. Use assertions for conditions that must never happen, understanding that some languages disable assertions in production builds.

### Context and wrapping

An error that says only "not found" is nearly useless. Add context as it travels upward: what operation, on which resource, with which identifiers. Preserve the original cause.

```
failed to process order 8123: charge card: payment gateway timeout after 5s
```

Do not log secrets or personal data in error messages.

### Cleanup

Resources must be released on every path. Use constructs that guarantee cleanup: `finally`, `with`, `defer`, RAII.

### Retries, timeouts and idempotency

Transient failures are normal in distributed systems. A safe retry policy needs:

- A **timeout** on every call, so a hung dependency cannot hang you.
- A **limit** on attempts.
- **Exponential backoff with jitter**, so clients do not retry in lockstep.
- **Idempotency**: repeating the operation must not cause duplicate effects, such as double charges. Use idempotency keys or naturally idempotent operations.

### Graceful degradation and circuit breaking

If a non-essential dependency fails, serve a reduced experience instead of failing entirely: show cached recommendations, skip the avatar. A circuit breaker stops calling a dependency that is clearly failing, giving it time to recover and protecting your resources.

### Errors for different audiences

- **Users** need an actionable message without internals.
- **Callers** need a stable error category or code they can branch on.
- **Operators** need detailed logs, identifiers and stack traces to diagnose.

Mixing these leaks internals to users or starves operators of detail.

## Trade-offs

| Decision | Benefit | Cost |
| --- | --- | --- |
| Exceptions | Clean happy path, automatic propagation | Hidden control flow, easy to catch too broadly |
| Result types | Failures visible and checked | Verbose, wrapping and conversion boilerplate |
| Fail fast | Bugs found early and near cause | Crashes visible to users; needs supervision and restarts |
| Catch and continue | Higher availability for the request | May mask corruption; hides bugs |
| Retries | Masks transient faults | Amplify load on a struggling dependency; duplicate effects without idempotency |
| Detailed errors | Faster diagnosis | Risk of exposing sensitive information |
| Graceful degradation | Better user experience in partial outage | More code paths to build and test |

Defensive checks everywhere add noise and can hide real design problems. Validate at trust boundaries and trust internal invariants, using assertions to document them.

## Failure Modes

- **Swallowed exceptions.** An empty `catch` or an ignored `err` hides the failure and produces wrong results later.

  ```python
  try:
      save(record)
  except Exception:
      pass          # data silently lost
  ```

  Catch the narrowest type you can handle, and log or re-raise otherwise.
- **Catching too broadly.** Catching everything also catches interrupts, programmer errors and out-of-memory conditions. Reserve broad handlers for the top level of a request or job.
- **Log and rethrow at every layer.** The same error is logged five times, flooding logs and making incidents harder to read. Handle once, at the layer that can act; wrap with context elsewhere.
- **Lost cause.** Wrapping an error without preserving the original removes the evidence. Chain causes.
- **Retry storms.** Every client retries immediately when a service slows, multiplying load and turning a partial failure into a total one. Use backoff with jitter, retry budgets and circuit breakers.
- **Retrying non-idempotent operations.** A timeout does not mean the request failed; it may have succeeded. A blind retry duplicates payments or messages.
- **Missing timeouts.** Default timeouts are often infinite or very long. One slow dependency exhausts threads and connections. Set explicit timeouts everywhere.
- **Error paths never tested.** The recovery code has never run, and fails the first time it is needed. Test failure injection: kill dependencies, return errors, fill disks in a test environment.
- **Partial failure leaving inconsistent state.** A multi-step operation fails midway. Use transactions, compensating actions or designs where steps can be safely repeated.
- **Leaking internals.** Stack traces or SQL in user-visible responses help attackers. Return a generic message and an error identifier; keep detail in logs.
- **Unobserved background failures.** Failures in jobs and tasks that no one watches. Emit metrics, alert on error rates and dead-letter queues.

## In Practice

### Checklist for any function that can fail

1. What can go wrong? List the failure kinds.
2. Which can this code handle, and which must it surface?
3. Does the error carry enough context to diagnose without a debugger?
4. Is every resource released on every path?
5. Is a retry safe? Is the operation idempotent?
6. What does the user see? What does the operator see?
7. Is there a test that triggers this failure?

### Worked example: calling a flaky dependency

```python
import random, time

def call_with_retry(fn, attempts=4, base=0.2, cap=5.0):
    for i in range(attempts):
        try:
            return fn(timeout=2.0)
        except TransientError:
            if i == attempts - 1:
                raise
            delay = min(cap, base * (2 ** i))
            time.sleep(random.uniform(0, delay))   # full jitter
```

Note the explicit timeout, the bounded attempts, the jittered backoff, and that only a transient error type is retried. Pair this with an idempotency key if `fn` has side effects.

### Habits

- Define an error taxonomy for your service and map it to responses consistently.
- Log errors once, with structured fields (operation, identifiers, cause).
- Alert on symptoms users feel, such as error rate and latency, not only on individual exceptions.
- Write the failure test before the fix.
- Review `catch` blocks and ignored errors in code review as carefully as logic.

### Developer vs. engineer

A developer wraps the failing line in a try and moves on. An engineer asks what this failure means for the user, the data and the on-call person, and designs the response accordingly. See also [The Engineering Mindset](../01-engineering-fundamentals/engineering-mindset.md).

### Verifying AI-generated code

Generated code often includes broad `except Exception` blocks that print and continue, invented retry logic without backoff or idempotency, missing timeouts and error messages with no context. Check each failure path: what happens, what is reported and whether the resource is cleaned up. Try to trigger the errors, rather than assuming the handler works.

## Further Reading

- Michael Nygard, *Release It!* (stability patterns: timeouts, circuit breakers, bulkheads).
- Joe Duffy, "The Error Model" (essay on the design of error handling in the Midori project).
- Steve Klabnik and Carol Nichols, *The Rust Programming Language* (the error handling chapter).
- Marc Brooker and others, Amazon Builders' Library, "Timeouts, retries and backoff with jitter".
- Robert C. Martin, *Clean Code* (chapter on error handling, with the caveat that some of its advice is debated).

Previous: [Concurrency and Parallelism](./concurrency-and-parallelism.md). Next: [Clean Code and Refactoring](./clean-code-and-refactoring.md).
