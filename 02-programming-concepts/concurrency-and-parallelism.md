# Concurrency and Parallelism

> Concurrency is structuring a program as independent tasks whose execution overlaps; parallelism is running tasks at the same instant, and getting either one wrong produces bugs that are rare, severe and hard to reproduce.

## Overview

Most useful software does several things at once: serving many users, waiting on a database while handling another request, using multiple CPU cores. Doing this correctly is one of the hardest areas of programming, because correctness now depends on the timing of events you do not control.

Two terms are often confused:

- **Concurrency** is about structure: dealing with many things at once. Tasks can interleave on a single core.
- **Parallelism** is about execution: doing many things at the same time, which needs multiple cores or machines.

A program can be concurrent without being parallel (an event loop on one thread) and parallel without much concurrency in its design (a data-parallel loop split across cores).

A developer tends to add threads until it looks faster. An engineer first asks what the bottleneck is, whether it is waiting (I/O-bound) or computing (CPU-bound), and what shared state the new design creates. For distributed concerns beyond one process, see [Distributed Systems](../13-distributed-systems/README.md).

## Core Concepts

### I/O-bound vs. CPU-bound

| Workload | Limited by | Helps | Does not help |
| --- | --- | --- | --- |
| I/O-bound | Waiting on network, disk, other services | Async I/O, many threads or tasks | More cores alone |
| CPU-bound | Computation | Parallelism across cores, better algorithms | More concurrent waiting tasks |

Identify which you have before choosing a model.

### Processes, threads and tasks

- A **process** has its own memory space. Isolation is strong; communication needs explicit channels.
- A **thread** shares memory with other threads in the process. Communication is cheap and dangerous.
- A **task** (coroutine, goroutine, async function, fiber) is a lightweight unit scheduled by a runtime on top of threads. Many tasks can share a few threads.

Some runtimes restrict parallelism in a single process. CPython, for example, has traditionally used a global interpreter lock so that only one thread runs Python bytecode at a time. This is a detail of one implementation and is evolving, so check your version's documentation. Threads still help I/O-bound Python code; processes are the usual route for CPU-bound work.

### Shared state and data races

A **data race** occurs when two threads access the same memory, at least one writes, and nothing orders the accesses. The result is undefined or timing-dependent.

```python
counter = 0

def increment():
    global counter
    for _ in range(100_000):
        counter += 1    # read, add, write: three steps that can interleave
```

Two threads running this can lose updates, because `counter += 1` is not atomic.

A **race condition** is the broader category: the outcome depends on event ordering. A check-then-act sequence is the classic form: "if the file does not exist, create it" can fail when another process creates it in between.

### Synchronization tools

- **Mutex (lock)**: only one thread holds it; others wait. Protects a critical section.
- **Read-write lock**: many readers or one writer.
- **Atomic operations**: indivisible updates to a single value.
- **Condition variables and semaphores**: coordinate waiting and limit concurrency.
- **Channels and queues**: pass data between tasks instead of sharing it.

```go
var mu sync.Mutex
var count int

func inc() {
    mu.Lock()
    defer mu.Unlock()
    count++
}
```

### Models of concurrency

- **Shared memory with locks**: familiar, flexible, easy to get wrong.
- **Message passing**: tasks own their data and communicate by sending messages. The Go proverb is to share memory by communicating. The actor model is a related approach.
- **Async/await with an event loop**: one thread handles many waiting tasks. Code is cooperative: a task yields at `await`. A long computation without yielding blocks everyone.
- **Immutability**: data that never changes can be shared freely.
- **Data parallelism**: split a large collection across workers and combine results.

### Deadlock, livelock and starvation

A **deadlock** occurs when tasks wait on each other forever. Four conditions are required: mutual exclusion, hold-and-wait, no preemption and circular wait. Breaking any one prevents it. The usual practical fix is a global lock ordering.

```
Thread A: lock(X) ... lock(Y)
Thread B: lock(Y) ... lock(X)     // can deadlock
```

**Livelock**: tasks keep reacting to each other without progress. **Starvation**: a task never gets the resource because others always win.

### Memory models

Compilers and CPUs reorder operations for speed. Languages define a memory model that says when one thread's writes are guaranteed visible to another. Without proper synchronization, a thread may never see an update or may see updates out of order. The practical rule: use the language's synchronization primitives, and avoid clever lock-free code unless you have strong reasons and expertise.

## Trade-offs

| Choice | Benefits | Costs |
| --- | --- | --- |
| Threads with shared memory | Simple to start, uses all cores | Races, deadlocks, hard debugging |
| Async/await | Efficient for many I/O waits, few threads | Blocking calls stall the loop; "colored function" split between sync and async code; harder stack traces |
| Message passing | Less shared state, clearer ownership | Message overhead, design of protocols, backpressure to manage |
| Processes | Isolation, fault containment | Memory duplication, serialization cost, slower communication |
| Coarse locks | Simple, fewer deadlocks | Contention limits scaling |
| Fine-grained locks | Better scaling | Complexity, higher deadlock risk |
| Lock-free structures | No blocking | Very hard to get right, subtle memory-model issues |

More concurrency is not always faster. Adding workers adds scheduling and synchronization overhead, and past some point contention dominates. Amdahl's law states that the serial portion of a task limits the speedup you can achieve from parallelism.

## Failure Modes

- **Lost updates and torn reads.** Unsynchronized shared state gives wrong totals, stale values or corrupted structures. Often appears only under load. Detect with race detectors (such as Go's race detector or ThreadSanitizer for C, C++ and Rust unsafe code) and stress tests. Prevent by minimizing shared mutable state.
- **Deadlock.** The service hangs with no error and low CPU. Thread dumps show threads waiting on locks. Prevent with consistent lock ordering, timeouts on lock acquisition, and holding locks briefly.
- **Holding locks across slow calls.** A lock held during a network call turns one slow dependency into a global stall.
- **Blocking the event loop.** A synchronous call or heavy computation in async code freezes all tasks. Latency rises for everyone. Offload to a worker pool.
- **Unbounded concurrency.** Spawning a task per item for a million items exhausts memory, file descriptors or downstream capacity. Use bounded worker pools and semaphores.
- **Missing backpressure.** Producers outpace consumers and queues grow until memory runs out. Use bounded queues and decide whether to block, drop or reject.
- **Lost errors in tasks.** A background task fails and nobody observes it. Always collect results, propagate errors and log failures from tasks.
- **Thundering herd and retry storms.** Many tasks wake or retry at once. Add jitter and limits.
- **Flaky tests.** Tests that pass most of the time often hide a race. Do not "fix" them with sleeps; find the missing synchronization.
- **Check-then-act across processes.** Local locks do not protect shared external resources. Use database constraints, transactions or atomic operations at the shared resource.

## In Practice

### Decision checklist

1. Is the work I/O-bound or CPU-bound? Measure.
2. Do you need concurrency at all? A simple sequential program that meets the requirement is easier to get right.
3. Can tasks avoid sharing state? Prefer passing immutable data or messages.
4. If state must be shared, who owns it, and which lock or mechanism protects it? Write it down in a comment.
5. Is concurrency bounded, and what happens when capacity is reached?
6. How do errors, cancellation and timeouts propagate?
7. How will you test it, and which tool will check for races?

### Worked example: bounded concurrent fetching

```python
import asyncio

async def fetch_all(urls, fetch, limit=10):
    sem = asyncio.Semaphore(limit)

    async def one(u):
        async with sem:
            return await fetch(u)

    return await asyncio.gather(*(one(u) for u in urls), return_exceptions=True)
```

The semaphore bounds load on the downstream service. `return_exceptions=True` keeps one failure from hiding the others; the caller must still inspect the results for exceptions.

### Habits

- Prefer higher-level constructs (thread pools, channels, structured concurrency) to raw threads and locks.
- Keep critical sections small and free of I/O.
- Always set timeouts on waits.
- Make operations idempotent so retries are safe.
- Run race detectors in continuous integration where available.

### Developer vs. engineer

A developer adds a lock where a test failed. An engineer asks what invariant the lock protects, whether the shared state should exist at all and how the design behaves under cancellation and overload.

### Verifying AI-generated code

Generated concurrent code often looks tidy and is subtly wrong. Check for shared variables modified without protection, locks held during I/O, missing timeouts, unbounded task creation, swallowed exceptions in tasks, blocking calls inside async functions and tests that rely on `sleep`. Run it under a race detector and under load, not only on a single happy-path call.

## Further Reading

- Brian Goetz et al., *Java Concurrency in Practice*.
- Maurice Herlihy and Nir Shavit, *The Art of Multiprocessor Programming*.
- Tony Hoare, *Communicating Sequential Processes*.
- Rob Pike, "Concurrency Is Not Parallelism" (talk).
- Martin Kleppmann, *Designing Data-Intensive Applications* (for concurrency across processes and machines).

Previous: [Memory Management](./memory-management.md). Next: [Error Handling](./error-handling.md).
