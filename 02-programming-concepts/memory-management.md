# Memory Management

> Memory management is how a program obtains, uses and releases memory; the strategy a language chooses shapes its safety, speed and the kinds of bugs you will meet.

## Overview

Every running program needs memory for its code, its local variables and its data. Someone must decide when memory is handed out and when it is taken back. Get it wrong in one direction and you leak memory until the process dies. Get it wrong in the other and you use memory that was already given to something else, which corrupts data or opens security holes.

Languages differ in who carries this responsibility. In C, the programmer does. In Java, Go and Python, a runtime does most of it. In Rust, the compiler enforces ownership rules so that the programmer's choices are checked before the program runs.

You do not need to implement an allocator to benefit from this knowledge. You need a working model of where values live, how long they live, and what the runtime does about them, so you can explain a memory spike, a latency pause or a crash. See [How Computers Work](../01-engineering-fundamentals/how-computers-work.md) for the underlying hardware picture.

## Core Concepts

### Stack and heap

- **The stack** holds data for function calls: local variables, arguments and return addresses. Allocation is a pointer adjustment, and memory is reclaimed automatically when the function returns. Sizes must usually be known in advance, and the stack is limited in size.
- **The heap** holds data whose lifetime or size is not tied to a single call. Allocation is more expensive, and something must decide when it is no longer needed.

```
 high addresses
 +----------------+
 |     stack      |  grows with calls, freed on return
 +----------------+
 |       ...      |
 +----------------+
 |      heap      |  allocated and freed on demand
 +----------------+
 | static / code  |
 +----------------+
 low addresses
```

The exact layout varies by system. The stack/heap distinction is the useful part.

### Manual management

In C, `malloc` obtains heap memory and `free` returns it. The programmer tracks ownership.

```c
char *buf = malloc(n);
if (!buf) return -1;
/* ... use buf ... */
free(buf);
buf = NULL;
```

This gives full control and predictable timing, at the price of a family of bugs: leaks (never freed), use-after-free, double free, buffer overflows and uninitialized reads. Many serious security vulnerabilities in large C and C++ codebases are of these kinds.

### Reference counting

Each object keeps a count of references to it. When the count reaches zero, the object is freed immediately. Used by Swift (ARC), CPython (alongside a cycle collector) and C++ `shared_ptr`.

- Strength: reclamation is prompt and predictable.
- Weakness: cycles (A refers to B, B refers to A) never reach zero unless broken with weak references or a cycle detector. Updating counts also costs time, and in multithreaded code those updates must be atomic.

### Tracing garbage collection

A collector periodically finds objects reachable from roots (stacks, globals) and reclaims everything else. Java, Go, C#, JavaScript and many others use some form of this.

Common ideas:

- **Generational collection**: most objects die young, so collect the young region often and the old region rarely.
- **Concurrent and incremental collectors**: do most work alongside the program to shorten pauses.
- **Compaction**: move live objects together to reduce fragmentation.

Collectors differ greatly between runtimes and versions. Consult your runtime's documentation rather than assuming.

### Ownership and borrowing

Rust assigns each value a single owner. When the owner goes out of scope, the value is dropped. Others may borrow it, subject to rules the compiler checks: either many shared reads or one exclusive write at a time.

```rust
fn main() {
    let s = String::from("hello");
    let t = s;            // ownership moves to t
    // println!("{}", s); // compile error: s was moved
    println!("{}", t);
}   // t dropped here, memory freed
```

This removes use-after-free and data races in safe code without a collector, at the cost of a learning curve and sometimes restructuring code to satisfy the borrow checker.

### Arenas and pools

An arena allocates many objects from one region and frees them all at once. A pool reuses fixed-size objects. Both reduce per-object overhead and fragmentation and suit request-scoped or frame-scoped work.

### Locality and layout

Memory access speed depends on caches. Data stored contiguously (an array of structs or of numbers) is typically much faster to scan than data scattered by pointers (a linked list of boxed objects). This matters as much as the choice of algorithm for large inputs. See [Complexity Analysis](../03-data-structures-algorithms/complexity-analysis.md).

## Trade-offs

| Strategy | Benefits | Costs |
| --- | --- | --- |
| Manual | Maximum control, no runtime overhead, predictable | Whole classes of memory-safety bugs, careful discipline needed |
| Reference counting | Prompt reclamation, simple model | Cycle leaks, counting overhead, atomic updates under threads |
| Tracing GC | Productivity, no manual frees, handles cycles | Memory overhead, possible pauses, less predictable timing |
| Ownership (Rust) | Safety without a collector, deterministic freeing | Learning curve, restructuring code, some patterns hard to express |
| Arenas | Fast, simple lifetime | All-or-nothing release, objects held longer than needed |

A garbage collector trades some memory and occasional latency for fewer bugs. For most services this is the right trade. For hard real-time systems, very latency-sensitive paths or constrained devices, the pauses or overhead may be unacceptable. Many teams find that tuning allocation behavior within a collected language is enough, and only a few need to leave it.

## Failure Modes

- **Memory leak in a collected language.** The collector cannot free what is still reachable. Typical causes: an unbounded cache or map, listeners never unregistered, a growing list of "recent" items, thread-local data on pooled threads, long-lived references to large request objects. Symptoms: memory climbs steadily, GC runs more often and more slowly, eventually out-of-memory. Detect with memory graphs over days and heap snapshots comparing two points in time. Prevent with bounded caches, eviction policies and ownership of lifecycle.
- **Use-after-free and buffer overflow.** Memory is read or written after release or beyond bounds. These are hard to reproduce and can be security vulnerabilities. Use memory-safe languages for new code where possible, and in C or C++ use sanitizers, fuzzing and bounds-checked containers in testing.
- **Cycle leaks under reference counting.** Parent and child hold strong references to each other. Use weak references on the back-pointer.
- **GC pauses hurting tail latency.** Average latency looks fine, but the slowest requests spike. Look at pause metrics, allocation rate and heap size. Reducing allocation in hot paths often helps more than flags.
- **Allocation storms.** Creating many short-lived objects per request, such as repeated string concatenation in loops, drives allocation rate. Profile before changing.
- **Out-of-memory kills.** The operating system or container limit ends the process without a graceful shutdown. Set explicit limits, know the difference between the runtime's heap limit and the container's limit, and leave headroom for non-heap memory.
- **Unbounded buffering.** Reading an entire file or response into memory when streaming would do. A test with small data passes; one large input exhausts memory.
- **Stack overflow.** Deep or unbounded recursion exhausts the stack. Convert to iteration or bound the depth.

## In Practice

### Diagnosing a memory problem

1. Confirm it. Graph memory use over time, not just at one moment.
2. Separate the kinds: heap, stack, off-heap and native memory, container limit.
3. Capture heap profiles at two times and compare what grew.
4. Find what retains the objects (the path from a root).
5. Fix the retention (bound it, release it, remove it) and verify with the same measurement.

### Habits

- Put a size limit on every cache, queue and buffer, and decide what happens when it is full.
- Stream large data instead of loading it whole.
- Release external resources (files, sockets, locks) explicitly and deterministically; do not wait for the collector. Use the language's construct for this, such as `with` in Python, `try-with-resources` in Java, `defer` in Go or RAII in C++ and Rust.

```python
with open(path, "rb") as f:
    for chunk in iter(lambda: f.read(65536), b""):
        process(chunk)
```

- Prefer contiguous data structures for large collections that are scanned often.
- Measure before optimizing. Memory profilers beat intuition.

### Developer vs. engineer

A developer restarts the service when memory grows. An engineer finds what holds the memory, fixes it and adds an alert on the trend, so the next instance is caught before it becomes an outage.

### Verifying AI-generated code

Look for unbounded collections used as caches, reading whole inputs into memory, missing resource cleanup, and loops that build large strings or lists repeatedly. In C or C++ output, check every allocation for a matching release, every index for a bound and every pointer for lifetime. Do not accept generated code that "works on the example" as safe on large or hostile inputs.

## Further Reading

- Richard Jones, Antony Hosking and Eliot Moss, *The Garbage Collection Handbook*.
- Brian Kernighan and Dennis Ritchie, *The C Programming Language*.
- Steve Klabnik and Carol Nichols, *The Rust Programming Language* (the chapter on ownership; freely available from the Rust project).
- Randal Bryant and David O'Hallaron, *Computer Systems: A Programmer's Perspective*.
- Ulrich Drepper, "What Every Programmer Should Know About Memory".

Previous: [Type Systems](./type-systems.md). Next: [Concurrency and Parallelism](./concurrency-and-parallelism.md).
