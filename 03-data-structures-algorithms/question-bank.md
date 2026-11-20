# Question Bank: Data Structures and Algorithms

> Questions for interviews, design reviews and self-testing. Each has a model answer behind a toggle, so you can commit to your own answer first.

## How to use this page

- **Self-testing.** Answer aloud or in writing before opening the toggle. Compare for gaps, not for matching wording.
- **Interviewing.** Pick questions by level and use *Red flags* as a rubric. Follow up with "what would change your answer?" Good candidates name the conditions under which their choice is wrong.
- **Design review.** The *Applied* and *Review* questions work as prompts for code and design reviews.

Difficulty: **E** entry, **M** mid-level, **S** senior.

## Fundamentals

### 1. (E) Why can a loop that calls `list.contains(x)` inside another loop over a list become a production problem?

<details>
<summary>Model answer</summary>

Each `contains` on a list scans linearly, so the nested pair is `O(n * m)`, quadratic when both are similar in size. It works on test data and slows sharply as data grows.

**Strong answers:**
- Name the cost, not just "it is slow".
- Fix it by building a hash set from one list first, giving `O(n + m)` on average.
- Mention the extra memory for the set, and that the fix assumes items are hashable and equality semantics match.

**Red flags:** Says "use a faster computer", or cannot say why the cost grows.
</details>

### 2. (E) What does amortized `O(1)` append mean for a dynamic array?

<details>
<summary>Model answer</summary>

Most appends write into spare capacity at constant cost. Occasionally the array is full, and it allocates a larger block and copies every element, costing `O(n)`. If capacity grows geometrically, those expensive copies are rare enough that the average cost per append over a long sequence is constant.

**Strong answers:**
- Distinguish amortized cost from worst-case cost of a single operation.
- Note that a single append can be slow, which matters for latency-sensitive paths.
- Mention pre-sizing when the final size is known.

**Red flags:** Claims every append is `O(1)`.
</details>

### 3. (M) Why might a binary search over a small array lose to a linear scan?

<details>
<summary>Model answer</summary>

For small `n`, constants dominate. A linear scan over contiguous memory is cache-friendly and branch-predictable. Binary search has fewer comparisons but more unpredictable branches. The crossover depends on hardware, element size and language.

**Strong answers:**
- Say that asymptotics describe growth, not speed at a particular size.
- Recommend measuring with realistic data rather than asserting a threshold.

**Red flags:** Quotes a specific cutoff as universal.
</details>

## Hashing, strings and collections

### 4. (M) A hash map's lookups slow down sharply under certain request patterns. What could be happening?

<details>
<summary>Model answer</summary>

Many keys may be landing in the same buckets. Causes include a weak hash function, keys with shared structure that the hash handles poorly, a map that is overloaded because it never resizes, or attacker-chosen keys crafted to collide. Worst-case lookup then degrades toward linear in the colliding bucket, or logarithmic where the implementation converts long chains to trees.

**Strong answers:**
- Propose checks: key distribution, load factor, bucket length, a profile of lookup time.
- Mention mitigations: randomized or keyed hashing, input limits, a better hash, resizing.

**Red flags:** Assumes hash maps are always `O(1)`.
</details>

### 5. (M) Why is a mutable object a dangerous hash key?

<details>
<summary>Model answer</summary>

The entry is stored in a bucket chosen from the key's hash at insertion time. If the key changes in a way that alters its hash or equality, later lookups search the wrong bucket or fail equality, so the entry becomes unreachable though still present. This causes silent data loss and leaks.

**Strong answers:**
- Prefer immutable keys, or copy keys on insertion.
- Note this applies to any structure that organizes by key, including sets and sorted trees.
</details>

### 6. (E) A string of length 5 is stored as 8 bytes. How is that possible, and why does it matter?

<details>
<summary>Model answer</summary>

Characters are not bytes. In common encodings such as UTF-8, one visible character can take multiple bytes, and one user-perceived character can be multiple code points. Length, slicing and truncation can mean bytes, code points or grapheme clusters depending on the language and API.

**Strong answers:**
- Warn that truncating by bytes can split a character and corrupt text.
- Mention that database and API limits may count bytes while the UI counts characters.
</details>

## Stacks, queues, trees and graphs

### 7. (M) How would you design a work queue so it does not take down a service under overload?

<details>
<summary>Model answer</summary>

Bound it. An unbounded queue turns overload into memory exhaustion and long delays. Decide what happens at the bound: reject new work with a clear signal, apply backpressure to producers, shed lower-priority work, or drop the oldest. Also consider timeouts on queued items, since work that is too old to be useful should not be processed.

**Strong answers:**
- Connect queue size to latency: queued items wait, so a deep queue means slow responses.
- Mention monitoring queue depth and age.

**Red flags:** "Just add more memory" or "use an unbounded queue so we never lose work".
</details>

### 8. (M) When would you choose breadth-first search over depth-first search?

<details>
<summary>Model answer</summary>

Use BFS to find the shortest path in an unweighted graph, or to explore by distance from a start. Use DFS for exhaustive exploration, topological ordering, cycle detection and when memory for a wide frontier is a concern. BFS uses memory proportional to the widest level, and DFS to the depth.

**Strong answers:**
- Mention that recursive DFS can overflow the call stack on deep graphs.
- Mention tracking visited nodes to avoid infinite loops on cyclic graphs.
</details>

### 9. (S) Dijkstra's algorithm returns a wrong route on your data. What do you check first?

<details>
<summary>Model answer</summary>

Check for negative edge weights. Dijkstra's algorithm assumes non-negative weights. With negative weights, use Bellman-Ford, which also detects negative cycles. Also check for data issues such as unit mismatches, floating-point accumulation, and a bad priority-queue comparator.

**Strong answers:**
- State the invariant that fails.
- Ask whether a negative cycle makes the "shortest path" undefined.
</details>

### 10. (S) Why do databases use B-trees instead of binary search trees?

<details>
<summary>Model answer</summary>

Storage is read in pages, and a disk or network read is far more expensive than comparing keys in memory. B-trees pack many keys per node, so the tree is shallow and a lookup touches few pages. Binary trees would need many more reads for the same data. B-trees also keep ordered keys for range scans.

**Strong answers:**
- Mention the cost side: page splits and write amplification.
- Connect to the [Database Engineering](../06-database-engineering/README.md) section.
</details>

## Sorting and searching

### 11. (E) What is a stable sort, and when does it matter?

<details>
<summary>Model answer</summary>

A stable sort keeps items with equal keys in their original order. It matters when sorting by multiple criteria in passes, for example sort by name, then stable sort by department, which yields department order with names ordered inside each group. It also gives deterministic, repeatable output.

**Red flags:** Cannot give an example, or assumes every library sort is stable.
</details>

### 12. (M) What are the common bugs in a binary search implementation?

<details>
<summary>Model answer</summary>

Off-by-one errors in the bounds, integer overflow when computing the midpoint from two large indexes (compute `lo + (hi - lo) / 2`), infinite loops from not shrinking the range, and wrong handling of duplicates when you need the first or last match.

**Strong answers:**
- State the loop invariant.
- Test with empty input, one element, absent values, duplicates and boundaries.
</details>

### 13. (M) You need the 10 largest items out of 100 million. Do you sort?

<details>
<summary>Model answer</summary>

Not a full sort. Keep a min-heap of size 10 and scan once, replacing the smallest when a larger item appears. This is `O(n log k)` time and `O(k)` memory, against `O(n log n)` and full storage for a sort. A selection algorithm such as quickselect can also work on in-memory data.

**Strong answers:**
- Mention streaming and memory limits.
- Mention ties and how to handle them.
</details>

## Dynamic programming and greedy

### 14. (M) How do you decide between a greedy and a DP approach?

<details>
<summary>Model answer</summary>

Try to break the greedy idea with a counterexample or prove it with an exchange argument. If a local choice can be shown safe, greedy is simpler and faster. If it fails, look for a state that captures all that matters about the past and see whether the state space is small enough for DP.

**Strong answers:**
- Give a counterexample such as coins `{1, 3, 4}` with target `6`.
- Test against brute force on small inputs.

**Red flags:** Chooses greedy because it passes the sample cases.
</details>

### 15. (S) A DP solution is correct but uses too much memory. What are your options?

<details>
<summary>Model answer</summary>

Keep only the rows or entries still needed, which often reduces a 2D table to one or two rows. Prune unreachable states, use a more compact encoding, or reformulate the state. Note the cost: dropping earlier rows prevents reconstructing the solution unless you store choices separately or recompute. If the problem is NP-hard at scale, consider approximation.

**Strong answers:**
- Estimate states times bytes first.
- Mention pseudo-polynomial behavior when a numeric value drives table size.
</details>

## Applied and review

### 16. (M) In a code review, what do you check on a new in-memory cache?

<details>
<summary>Model answer</summary>

- A size or memory bound, and an eviction policy.
- Expiry, and what happens to stale entries.
- Key design: immutable, and including everything that affects the value.
- Thread safety and behavior under concurrent misses, including duplicate loads.
- Behavior across multiple instances and restarts.
- Metrics for hit rate, size and evictions.

**Red flags:** Reviews only whether the code returns the right value.
</details>

### 17. (S) A page that was fast last quarter is now slow. The data doubled. How do you determine whether it is an algorithm problem?

<details>
<summary>Model answer</summary>

Profile with realistic data at several sizes and see how time grows. Doubling the data and seeing time roughly quadruple points to quadratic behavior. Locate the hot path with a profiler, inspect the data structures on it, and check for scans in loops, repeated sorting, or lookups that moved from an index to a scan. Confirm a fix by remeasuring.

**Strong answers:**
- Measure first, then change code.
- Consider that the cause may be I/O or queries rather than in-process algorithms. See [Performance and Scalability](../12-performance-scalability/README.md).
</details>

## AI-era verification

### 18. (M) An assistant generates a solution to a scheduling problem and claims it is optimal. What do you do?

<details>
<summary>Model answer</summary>

Ask for the reasoning: the recurrence, or the exchange argument for a greedy method. Then verify independently. Write a brute-force solver for small inputs, generate random cases and compare results. Try edge cases: empty input, ties, a single item and extreme values. Check complexity against realistic input sizes.

**Strong answers:**
- Treat confidence of the claim as no evidence of correctness.
- Keep the comparison test in the repository.

**Red flags:** Accepts the answer because the sample output matches.
</details>

### 19. (S) What would convince you that a generated algorithm is safe to put on a hot path?

<details>
<summary>Model answer</summary>

Stated complexity that you have checked yourself, a benchmark at realistic and worst-case sizes, tests including adversarial inputs, bounds on memory and recursion depth, and monitoring in production that would reveal a regression. Also an owner who understands the code well enough to maintain it.

**Strong answers:**
- Include "who can debug this at 3 a.m." as a criterion.
</details>

## Scoring and follow-up

For self-testing, mark each question as *clear*, *partial* or *gap*. Reread the matching topic for each gap:

| Questions | Topic |
| --- | --- |
| 1 to 3 | [Complexity Analysis](./complexity-analysis.md) |
| 4 to 6 | [Arrays, Strings and Hashing](./arrays-strings-and-hashing.md) |
| 7 to 10 | [Linked Lists, Stacks and Queues](./linked-lists-stacks-and-queues.md) and [Trees and Graphs](./trees-and-graphs.md) |
| 11 to 13 | [Sorting and Searching](./sorting-and-searching.md) |
| 14 to 15 | [Dynamic Programming and Greedy Algorithms](./dynamic-programming-and-greedy.md) |
| 16 to 19 | [Choosing a Data Structure](./choosing-a-data-structure.md) |

Then check yourself with the [Self-Assessment](./self-assessment.md). Back to the [section index](./README.md).
