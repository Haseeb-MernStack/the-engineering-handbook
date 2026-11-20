# Choosing a Data Structure

> Choosing a data structure means matching your access patterns, data size and consistency needs to the guarantees of a structure, then confirming the choice with measurements.

## Overview

Every earlier topic in this section describes one family of structures. In real work you face the opposite question: you have a requirement, and you must pick. The right structure depends less on the data itself than on how you will use it: which operations are frequent, which are rare, how large the data gets, and what must be ordered, unique or concurrent.

A wrong choice rarely fails on day one. It fails when data grows, traffic rises or a new access pattern appears. This topic gives a repeatable method for choosing, a reference table of defaults, and the situations where the textbook answer is not the practical one.

## Core Concepts

### Start from operations, not structures

Write down the operations the code will perform and estimate their frequency.

| Question | Why it matters |
| --- | --- |
| What are the main reads: by key, by position, by range, by prefix, by nearest value? | Each favors a different structure |
| What are the writes: append, insert in the middle, update, delete? | Cost of mutation differs sharply |
| How large can `n` get, and how fast? | Decides whether asymptotics matter |
| Is order required (insertion, sorted, priority)? | Hash tables give none by default |
| Must items be unique? | Sets and keyed maps enforce it |
| Is it accessed by many threads or processes? | Needs safe sharing or partitioning |
| Must it survive restarts or exceed memory? | Moves you to storage engines and indexes |

### Reference table of defaults

| Need | Default choice | Typical cost | Watch out for |
| --- | --- | --- | --- |
| Ordered sequence, index access, append | Dynamic array | `O(1)` index, amortized `O(1)` append | `O(n)` middle insert |
| Lookup by key, no order | Hash map | Average `O(1)` | Worst-case collisions, no ordering, memory overhead |
| Membership and uniqueness | Hash set | Average `O(1)` | Same as hash map |
| Sorted order, range queries, floor/ceiling | Balanced tree or sorted array | `O(log n)` | Higher constant factors than hashing |
| Always get the min or max | Binary heap | `O(log n)` push and pop, `O(1)` peek | No fast search or arbitrary removal |
| FIFO processing | Queue (ring buffer or deque) | `O(1)` | Unbounded growth |
| LIFO, undo, depth-first traversal | Stack | `O(1)` | Depth limits |
| Prefix search, autocomplete | Trie or sorted array with binary search | Proportional to key length | Memory use of tries |
| Relationships, paths, dependencies | Graph (adjacency list for sparse) | Depends on algorithm | Dense graphs may suit matrices |
| Approximate membership at huge scale | Bloom filter | Constant time, small memory | False positives; no deletion in the basic form |
| Disjoint grouping, connectivity | Union-find | Near-constant amortized | No split operation |
| Persistent ordered data on disk | B-tree index | `O(log n)` page reads | Write amplification, page splits |

### Constants and memory layout

Asymptotic notation hides constants, and constants often decide the winner at realistic sizes.

- **Contiguous memory is fast.** Arrays traverse cache lines efficiently. Linked structures chase pointers and miss cache. For small and medium `n`, a plain array with linear search can beat a hash map or tree.
- **Per-item overhead matters.** Hash maps and trees store extra metadata per entry. A million small entries can cost many times the payload size.
- **Allocation matters.** Structures that allocate per element put pressure on the allocator and, in managed languages, the garbage collector.

### Worked example: a rate limiter

Requirement: allow at most 100 requests per minute per user, with millions of users.

1. Lookup is by user ID, so a hash map keyed by user fits.
2. Per user, you need counts over a sliding window. Options: a queue of timestamps (exact, memory proportional to limit), or fixed buckets with counters (approximate, constant memory).
3. Memory: millions of users times 100 timestamps is large. Fixed buckets reduce it.
4. Cleanup: idle users must expire, or the map grows forever. Add an eviction policy or time-to-live.
5. Concurrency: many threads update the same user entry. Use per-key atomic operations or shard the map.
6. Scale beyond one process: an in-process map no longer works. Move state to a shared store, and reason about its failure behavior. See [Service Resilience](../07-backend-engineering/service-resilience.md).

This choice involved four structures and three trade-offs (accuracy, memory, sharing), not one lookup of "best data structure".

## Trade-offs

| Decision | Option A | Option B |
| --- | --- | --- |
| Speed vs. order | Hash map: fast lookup, no order | Tree: ordered iteration and ranges, slower lookup |
| Memory vs. speed | Sorted array: compact, binary search | Hash map: faster average lookup, more memory |
| Exactness vs. space | Exact set | Probabilistic filter: small, but with false positives |
| Simplicity vs. headroom | Plain list with linear scan | Indexed structure that handles growth |
| Mutation cost vs. read cost | Maintain an index on every write | Build or rebuild on demand |
| Shared mutable vs. partitioned | Locked shared structure | Per-shard structures with merge |

Further points:

- **Maintaining an index costs writes.** Every extra structure that speeds a read adds work and memory to every update, and a chance of inconsistency between copies.
- **Generality costs performance.** General-purpose structures handle every case acceptably. Specialized ones win in a narrow case and fail outside it.
- **Library choice affects guarantees.** Standard library structures differ in ordering, thread-safety and iteration stability between languages. Read the documentation instead of assuming.

## Failure Modes

- **Linear scan in a hot path.** A list used as a set works at 100 items and stalls at 100,000. Detect with profiling and with load tests at realistic sizes.
- **Unbounded structures.** Caches, queues, sets of seen IDs and per-user maps that never evict grow until memory runs out. Set limits and eviction.
- **Relying on unspecified order.** Code that depends on iteration order of a structure that does not promise one breaks on upgrade or under a different input.
- **Mutating while iterating.** Changing a collection during traversal causes skipped items, exceptions or undefined behavior.
- **Mutable keys.** Using a mutable object as a hash key, then changing it, loses the entry.
- **Hash flooding.** Attacker-controlled keys can force worst-case collisions in structures with predictable hashing. See [Arrays, Strings and Hashing](./arrays-strings-and-hashing.md).
- **Shared structures without synchronization.** Concurrent writes corrupt state or lose updates. See [Concurrency and Parallelism](../02-programming-concepts/concurrency-and-parallelism.md).
- **Stale derived structures.** A secondary index or cached view that is not updated with the source produces wrong answers that look valid.
- **Premature specialization.** A custom structure chosen for speculative performance, which adds bugs and is never measured.

## In Practice

### Decision method

1. List the operations and their frequencies.
2. Estimate `n` now and at the planned peak.
3. Pick the simplest structure that meets the requirements.
4. Check the costs against the largest realistic `n`.
5. Benchmark with realistic data if the choice is on a hot path or `n` is large.
6. Record the reasoning where a future maintainer will find it, such as a comment or a design note.

### Heuristics

- Prefer the standard library's default structures. They are well tested.
- For small `n`, prefer simplicity. For large `n`, check complexity and memory.
- If you need both fast lookup and ordered iteration, consider keeping a map plus a sorted structure, and accept the cost of keeping them in sync.
- If a problem looks like "find the best", ask whether a heap, a sort or a single pass is enough before reaching for a more elaborate method.
- When data no longer fits in memory, the question changes. See [Database Engineering](../06-database-engineering/README.md).

### Checklist before merging

- Is the expected maximum size written down?
- Is every collection bounded or deliberately unbounded with a reason?
- Does any code depend on ordering that the structure does not guarantee?
- Are keys immutable and well hashed?
- Is access from multiple threads safe, by design and not by luck?
- Is there a test or benchmark at realistic scale for hot paths?

### Developer vs. engineer

A developer picks the structure that makes the code compile and the demo pass. An engineer states the access pattern, projects growth, names the cost of each choice and leaves a measurement that would reveal if the assumption breaks.

### Verifying AI-generated code

Generated code defaults to lists and dictionaries and rarely considers bounds, eviction, ordering or concurrency. Review each collection it introduces: how big can it get, who removes items, what happens under concurrent access, and what the complexity is for the largest realistic input.

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*, for choosing structures at the storage and system level.
- Steven S. Skiena, *The Algorithm Design Manual*, for matching problems to structures.
- Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest and Clifford Stein, *Introduction to Algorithms*, for the guarantees of the core structures.
- Ulrich Drepper, "What Every Programmer Should Know About Memory", for why layout and cache behavior change real performance.
- Burton H. Bloom, "Space/Time Trade-offs in Hash Coding with Allowable Errors" (1970), the original Bloom filter paper.

Previous: [Dynamic Programming and Greedy Algorithms](./dynamic-programming-and-greedy.md). Back to the [section index](./README.md).
