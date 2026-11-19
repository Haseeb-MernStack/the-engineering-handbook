# How Computers Work

> A working model of the machine, from CPU to network, lets you predict why software is slow, why it breaks, and what AI-generated code is really asking the hardware to do.

## Overview

Every program you write ends up as instructions executed by a processor, reading and writing data held in memory, storage, and over a network. Languages, frameworks, and cloud platforms hide this. They do not remove it. When a service slows down under load, when a process is killed for using too much memory, or when a "simple" loop takes minutes, the cause is almost always in the layers underneath the abstraction.

You do not need to design a CPU. You need a mental model accurate enough to estimate cost. Which operations are cheap? Which are expensive by orders of magnitude? Where does data live, and how far must it travel?

A developer who writes code treats the machine as a black box that runs it. An engineer who owns outcomes knows roughly what the box does, so they can predict behavior before it reaches production and diagnose it quickly when it surprises them.

This topic covers the processor, the memory hierarchy, storage, the operating system, and the network. It stays at the level of ideas that stay true across hardware generations.

## Core Concepts

### The processor

A CPU repeatedly fetches an instruction, decodes it, and executes it. Instructions are simple: add two numbers, load a value from memory, compare, jump to another instruction. Everything a program does is built from these.

Key properties:

- **Clock rate** is how many cycles per second the processor runs. It is no longer the main source of speed gains.
- **Cores** let a processor run several instruction streams at once. A single-threaded program uses one core regardless of how many exist.
- **Pipelining and out-of-order execution** let the CPU overlap work and guess ahead. A wrong guess about a branch wastes work already started. Predictable code runs faster than unpredictable code that does the same logical work.
- **Registers** are the tiny, fastest storage inside the core. Compilers decide what lives there.

### The memory hierarchy

Memory is organized as a pyramid. Small and fast sits at the top. Large and slow sits at the bottom.

```
        registers          (inside the core, fewest bytes, fastest)
        L1 / L2 / L3 cache (on chip, kilobytes to megabytes)
        main memory (RAM)  (gigabytes, much slower than cache)
        SSD / disk         (terabytes, slower again, survives power loss)
        network storage    (effectively unlimited, slowest, can fail independently)
```

Each step down is slower by roughly an order of magnitude or more, though exact figures vary by hardware. The exact numbers matter less than the ratios. A memory access that misses every cache can cost as much time as many hundreds of simple arithmetic operations. A network round trip within a data center costs far more again. A trip across continents costs more still, and the speed of light sets a floor you cannot optimize away.

Two ideas make caches work:

- **Temporal locality**: data used recently will probably be used again soon.
- **Spatial locality**: data near recently used data will probably be used soon. Memory is moved in fixed-size blocks called cache lines, so reading one element brings its neighbors along for free.

This is why iterating over a contiguous array is usually faster than following pointers through a linked list, even when both are O(n). See [Complexity Analysis](../03-data-structures-algorithms/complexity-analysis.md) for why big-O alone does not decide speed.

### Storage

Storage is persistent. RAM is not. Data you have not written to durable storage is lost on a crash or restart.

- Sequential access is much faster than random access on most devices.
- Writes are not durable until the operating system and the device confirm it. A successful `write` call usually means "the OS accepted it", not "it is on disk". Durability needs an explicit flush (for example `fsync`) and a device that honors it.
- Storage fails. Disks wear out, filesystems fill up, and volumes become read-only.

### The operating system

The operating system shares the machine among programs. It provides:

- **Processes**: isolated programs with their own address space.
- **Threads**: units of execution inside a process that share its memory.
- **Virtual memory**: each process sees a private, contiguous address space. The OS maps it to physical RAM and may move pages to disk (swap) when RAM is short.
- **System calls**: the controlled interface for files, sockets, and memory. Crossing from your code into the kernel has a cost, so many tiny calls are slower than a few large ones.
- **Scheduling**: the OS decides which thread runs on which core and for how long. A thread can be paused at any instruction. This is the root of concurrency bugs; see [Concurrency and Parallelism](../02-programming-concepts/concurrency-and-parallelism.md).

### The network

A network is a slow, unreliable, shared channel between machines. Treat these as facts:

- Latency (time for one message) and bandwidth (bytes per second) are different things. A fast link can still have high latency.
- Messages can be delayed, duplicated, reordered, or lost.
- A call that does not return is not necessarily a call that failed. The remote side may have done the work.
- Connections have setup costs: name resolution, TCP handshake, and TLS negotiation all happen before the first useful byte.

These facts drive most of distributed system design. See [System Design](../05-system-design/README.md).

### From source to execution

Code reaches the CPU by one of a few routes:

| Route | What happens | Typical consequence |
|---|---|---|
| Compiled ahead of time | Source becomes machine code before running | Fast start, fast steady state, less runtime flexibility |
| Interpreted | A program reads and executes your source step by step | Easier to change, usually slower per operation |
| JIT compiled | A runtime compiles hot code while the program runs | Warm-up period, then fast; behavior depends on the runtime |
| Bytecode on a VM | Source compiles to an intermediate form run by a virtual machine | Portability, with a runtime to operate and tune |

Knowing which applies to your stack tells you where startup time, memory overhead, and tuning knobs come from.

## Trade-offs

Every layer trades something.

- **Speed vs. capacity.** Fast memory is small and costly. Large storage is cheap and slow. Caching moves data up the pyramid at the cost of memory and the risk of stale data.
- **Latency vs. throughput.** Batching improves throughput by amortizing fixed costs. It increases the latency of each individual item, since items wait for the batch to fill.
- **Abstraction vs. control.** A high-level language lets you move faster and make fewer memory errors. It costs you direct control of layout, allocation, and timing. Low-level control costs development time and invites a class of bugs.
- **Durability vs. speed.** Flushing every write to disk is safe and slow. Buffering is fast and loses data on a crash. Systems choose a point on this line deliberately, or by accident.
- **Locality vs. flexibility.** Contiguous, flat data layouts are fast to scan. Pointer-rich structures are easier to modify. Choose based on the access pattern.
- **Local vs. remote.** Calling a function in the same process is cheap and reliable. Calling a service adds latency, failure modes, and operational burden. Splitting a system has a price paid on every call.

## Failure Modes

**Out of memory.** A process grows until the OS or a container limit kills it. Causes include unbounded caches, loading a whole file or result set into memory, and leaks. Detect it with memory metrics over time and alerts on restarts. Prevent it with bounds on every queue and cache, and by streaming large data.

**Resource exhaustion that is not memory.** File descriptors, sockets, threads, and disk space all have limits. A leak or a burst hits the limit and everything that needs the resource fails at once. Monitor usage against limits, not just raw usage.

**Hidden I/O in hot paths.** A function that looks cheap performs a disk read or a network call. Called in a loop, it multiplies. The classic case is the N+1 query: one query to get a list, then one more per item. Detect it with request tracing and query counts per request.

**Timeouts and retries that amplify load.** A slow dependency causes callers to time out and retry. Retries add load to the already struggling dependency, which slows it further. Prevent it with bounded retries, backoff with jitter, and circuit breakers.

**Assuming a write is durable.** The application confirms a save, the machine loses power, and the data is gone. Verify what your storage layer guarantees and test crash recovery, not just the happy path.

**Clock assumptions.** Wall-clock time can jump backward or differ between machines. Using it to order events or measure durations causes subtle bugs. Use a monotonic clock for durations and avoid relying on cross-machine timestamps for ordering.

**Cache cold starts and stampedes.** After a restart, caches are empty, and every request goes to the backing store at once. Plan for warm-up, and coalesce duplicate requests for the same key.

**Unexpected scale of data.** Code that is fine for a thousand rows pauses the system at a hundred million. The failure appears only with production-sized data. Test with realistic volumes.

## In Practice

### Estimate before you build

You can reason about cost with orders of magnitude. For any operation ask: does it touch registers and cache, RAM, disk, or the network? Count how many times it runs per request, and how many requests arrive per second. Multiply. If the result is surprising, measure.

```python
# Worked estimate: is this endpoint safe?
# For each of 500 orders, we fetch the customer from the database.
orders = repo.list_orders(limit=500)          # 1 query
for order in orders:
    order.customer = repo.get_customer(order.customer_id)  # 500 queries
```

If one query takes a few milliseconds, 500 of them take seconds, in serial, on every request. The fix is one batched query:

```python
ids = {o.customer_id for o in orders}
customers = repo.get_customers(ids)           # 1 query
by_id = {c.id: c for c in customers}
for order in orders:
    order.customer = by_id[order.customer_id]
```

The logic is the same. The cost differs by orders of magnitude because the number of network round trips changed.

### Measure, then change

Intuition finds suspects. Measurement convicts them. Use a profiler or tracing to see where time goes. Change one thing at a time. Re-measure. Do not optimize code that is not on the critical path.

### Checklist: understanding a slow or failing system

- Which resource is saturated: CPU, memory, disk, network, or a limit such as connections?
- Is the time spent computing, or waiting on I/O?
- Is the work repeated unnecessarily (N+1, no caching, recomputation)?
- Does the problem scale with data size, request rate, or both?
- What changed recently: code, data volume, traffic shape, configuration?
- What do metrics, logs, and traces show, as opposed to what you assume?

### Developer vs. engineer

| Situation | Developer who writes code | Engineer who owns outcomes |
|---|---|---|
| Slow page | Adds a cache because it feels right | Profiles, finds the repeated query, fixes the cause, then decides whether caching is still needed |
| Memory errors | Increases the memory limit | Finds what grows, bounds it, and adds an alert |
| Saving data | Calls `save()` and moves on | Knows what "saved" guarantees and tests a crash |
| New dependency | Imports it | Considers its latency, failure behavior, and resource use |

### When AI generates code

Generated code is often correct in logic and careless about cost. Check:

- Loops that call a database, file, or network inside the body.
- Reading entire files or result sets into memory when streaming would do.
- Missing timeouts on network calls.
- Missing limits on caches, queues, and retries.
- Assumptions that a write is durable or that a call cannot partially fail.

You cannot spot these without the mental model above. This is why fundamentals matter more when code is cheap to produce: the ability to judge cost and failure is the scarce skill.

## Further Reading

- *Computer Systems: A Programmer's Perspective*, Randal E. Bryant and David R. O'Hallaron. A thorough tour of how programs map to hardware.
- *Operating Systems: Three Easy Pieces*, Remzi and Andrea Arpaci-Dusseau. A freely available text on processes, memory, and persistence.
- *Computer Architecture: A Quantitative Approach*, John L. Hennessy and David A. Patterson. The standard reference on performance and the memory hierarchy.
- *Systems Performance*, Brendan Gregg. A practical method for finding bottlenecks.
- "What Every Programmer Should Know About Memory", Ulrich Drepper. A detailed paper on caches and main memory.

Back to the section index: [Engineering Fundamentals](./README.md)
