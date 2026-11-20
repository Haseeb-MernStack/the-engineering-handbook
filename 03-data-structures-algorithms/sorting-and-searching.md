# Sorting and Searching

> Sorting and searching are solved problems; the engineering work is choosing the right library routine, understanding its guarantees, and keeping the ordering rules correct.

## Overview

Sorting arranges items by a key. Searching finds an item or its position. Together they sit under indexes, reports, merges, deduplication, scheduling and ranking. Binary search on sorted data turns a linear scan into a logarithmic one, which makes sorting a worthwhile investment whenever you search repeatedly.

In application code you almost never write a sort. You call the standard library, which is tuned, tested and usually a hybrid algorithm. What you do own is the comparator, the choice of key, the stability requirement and the behavior at scale. Most sorting bugs in production are comparator bugs, not algorithm bugs.

This topic covers the main algorithms and their properties, binary search and its pitfalls, and the engineering judgment about when a full sort is the wrong tool.

## Core Concepts

### Properties of sorting algorithms

- **Comparison-based**: orders items by comparing pairs. No comparison sort can beat `O(n log n)` comparisons in the worst case.
- **Stable**: items with equal keys keep their original relative order. Stability lets you sort by one field, then another.
- **In-place**: uses `O(1)` or `O(log n)` extra memory.
- **Adaptive**: runs faster on partly sorted input.

### Common algorithms

| Algorithm | Best | Average | Worst | Extra space | Stable |
| --- | --- | --- | --- | --- | --- |
| Insertion sort | O(n) | O(n^2) | O(n^2) | O(1) | Yes |
| Selection sort | O(n^2) | O(n^2) | O(n^2) | O(1) | No (typical form) |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quicksort | O(n log n) | O(n log n) | O(n^2) | O(log n) | No (typical form) |
| Heapsort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting sort | O(n + k) | O(n + k) | O(n + k) | O(k) | Yes |
| Radix sort | O(d(n + k)) | O(d(n + k)) | O(d(n + k)) | O(n + k) | Yes |

Counting and radix sorts are not comparison sorts. They apply to integers or fixed-width keys in a bounded range (`k` is the range of values, `d` the number of digits), and can beat `n log n` when those assumptions hold.

**Insertion sort** is fast on tiny or nearly sorted input.

**Merge sort** splits, sorts each half and merges. Its cost is predictable and it is stable, and its merge step works well on data streamed from disk.

```python
def merge_sort(a: list[int]) -> list[int]:
    if len(a) <= 1:
        return a
    mid = len(a) // 2
    left, right = merge_sort(a[:mid]), merge_sort(a[mid:])
    out: list[int] = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:          # <= keeps the sort stable
            out.append(left[i]); i += 1
        else:
            out.append(right[j]); j += 1
    out.extend(left[i:])
    out.extend(right[j:])
    return out
```

**Quicksort** partitions around a pivot and recurses. It is fast in practice because of cache-friendly sequential access and small constants. A poor pivot rule causes worst-case behavior. Production implementations choose pivots carefully, randomize, or switch to heapsort when recursion gets too deep (the introsort approach).

**What libraries actually do.** Python's `sort` and `sorted` use Timsort, a stable hybrid of merge and insertion sort that exploits existing runs in the data. Java sorts objects with a stable merge-based algorithm and primitives with a dual-pivot quicksort. C++ `std::sort` is typically introsort, while `std::stable_sort` is stable. Details vary by version and platform, so rely on the documented guarantees (stability, worst-case bound) rather than the internals.

### Sorting with keys and comparators

```python
people = [("Ada", 36), ("Grace", 45), ("Alan", 36)]

# Sort by age, then name. Tuple keys compare element by element.
people.sort(key=lambda p: (p[1], p[0]))

# Sort by age descending, then name ascending: negate the numeric key.
people.sort(key=lambda p: (-p[1], p[0]))
```

```typescript
const byAgeThenName = (a: Person, b: Person) =>
  a.age - b.age || a.name.localeCompare(b.name);
people.sort(byAgeThenName);
```

A valid comparator defines a consistent total order: if `a < b` and `b < c` then `a < c`, and `compare(a, b)` is the opposite sign of `compare(b, a)`. Violating this gives undefined or inconsistent results, and some runtimes raise an error.

Note the JavaScript trap: `Array.prototype.sort()` without a comparator converts elements to strings, so `[10, 9, 1].sort()` gives `[1, 10, 9]`. Always pass a numeric comparator.

### Binary search

Binary search finds a value in a sorted random-access array in `O(log n)` by halving the range.

```python
def lower_bound(a: list[int], target: int) -> int:
    """Index of the first element >= target, or len(a) if none."""
    lo, hi = 0, len(a)                 # search in [lo, hi)
    while lo < hi:
        mid = lo + (hi - lo) // 2      # avoids overflow in fixed-width languages
        if a[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return lo

def contains(a: list[int], target: int) -> bool:
    i = lower_bound(a, target)
    return i < len(a) and a[i] == target
```

Using half-open ranges `[lo, hi)` and a "first position where the condition holds" formulation removes most off-by-one errors. Python's `bisect` module provides `bisect_left` (equivalent to `lower_bound`) and `bisect_right`.

Binary search applies beyond arrays. Any monotonic yes/no condition can be searched. "What is the smallest capacity such that all jobs finish in time?" is a binary search over the answer.

### Selection and top-k

You often need only the smallest `k` items or the median, not a full sort.

- A heap of size `k` finds the top `k` in `O(n log k)` time and `O(k)` space.
- Quickselect finds the `k`th item in `O(n)` average time and `O(n^2)` worst case.

### External sorting

When data exceeds memory, sort chunks that fit, write each as a sorted run to disk and merge the runs. This is the basis of how databases and big-data tools sort large datasets.

## Trade-offs

| Choice | Gains | Costs |
| --- | --- | --- |
| Library sort | Tested, tuned, documented guarantees | Little control over internals |
| Sort then binary search | `O(log n)` repeated queries | `O(n log n)` once; stale if data changes |
| Hash lookup instead | `O(1)` average | No ordering or range queries |
| Stable sort | Predictable multi-key ordering | May use more memory than an in-place unstable sort |
| Partial sort / heap for top-k | Avoids full sort | More code; only helps when `k` is small relative to `n` |
| Counting or radix sort | Linear time on bounded keys | Needs bounded keys and extra memory |
| Keep data sorted on insert | Always ready for queries | Each insert costs `O(n)` in an array, `O(log n)` in a balanced tree |

If you search once, a linear scan costs `O(n)` and sorting first costs `O(n log n)`, so scanning wins. If you search many times, sort once. For small arrays, a linear scan is often as fast as binary search because of low overhead and cache behavior; measure to find your crossover.

## Failure Modes

- **Inconsistent comparators.** Comparators that return only `true` or `false`, ignore ties, or compare floating-point values with `NaN` break the total order. Results can be wrong or differ between runs, and some libraries raise exceptions. Test comparators with ties, equal elements, and edge values.
- **Default lexicographic sort.** Sorting numbers as strings (the JavaScript default) or sorting version strings such as "1.10" and "1.9" lexicographically gives wrong order. Parse into a sortable key.
- **Locale and Unicode surprises.** Byte order, code point order and locale-aware collation give different results. Choose deliberately, and be consistent between application and database. A mismatch between a database index collation and application sorting causes pagination bugs.
- **Unstable sorts breaking multi-key logic.** Sorting by one field and relying on a previous order for ties works only with a stable sort. Prefer a single sort with a composite key.
- **Off-by-one and infinite loops in binary search.** Updating `lo = mid` instead of `mid + 1`, or using closed and half-open ranges inconsistently, produces infinite loops or missed elements. Test with empty arrays, one element, target below the minimum, above the maximum and duplicates.
- **Midpoint overflow.** `(lo + hi) / 2` can overflow fixed-width integers. Known incidents include a long-lived bug in widely used library implementations. Use `lo + (hi - lo) / 2`.
- **Worst-case inputs.** Quicksort variants with predictable pivots degrade to quadratic time on adversarial or patterned input (all equal keys, sorted data). Prefer library sorts with a worst-case guarantee on untrusted input.
- **Sorting inside loops.** Re-sorting a list on each iteration, or on each request, multiplies cost. Sort once, or maintain a heap or ordered structure.
- **Memory spikes.** Merge sort needs an `O(n)` buffer. Sorting a large dataset in one process may double memory. Estimate peak memory or use an external sort.
- **Binary search on unsorted data.** The function returns plausible, wrong answers. Verify the sorted precondition, or assert it in tests.
- **Pagination on non-unique sort keys.** Paging by `ORDER BY created_at` when many rows share a timestamp can skip or repeat rows between pages. Add a unique tiebreaker such as the primary key.

Detection: property-based tests comparing your result with a trusted reference (`sorted`), checking that output is ordered and is a permutation of the input; fuzzing with duplicates and extreme values; load tests at production volume.

## In Practice

### Decision checklist

1. Can the library sort do it? Use it.
2. What is the ordering, exactly? Define the key, direction, tie-breaking and treatment of missing values and `NaN`.
3. Do ties need stable order? If so, use a stable sort or a composite key.
4. Do you need the whole order, or only the top `k`, the median, or one lookup?
5. Is the data bigger than memory?
6. How often will you search compared with how often the data changes?
7. Is the input untrusted? Prefer algorithms with worst-case bounds.

### Worked example: test a binary search against a reference

```python
import random
from bisect import bisect_left

def test_lower_bound() -> None:
    for _ in range(10_000):
        a = sorted(random.randint(0, 20) for _ in range(random.randint(0, 12)))
        t = random.randint(-2, 22)
        assert lower_bound(a, t) == bisect_left(a, t)
```

Random small inputs with many duplicates exercise the edge cases that hand-written examples miss.

### Verifying AI-generated code

- Is a hand-written sort used where the standard library would do?
- Does the comparator define a consistent total order, including ties?
- Is the sort stable when the logic needs it?
- Does binary search use a consistent range convention, and does it terminate on one-element input?
- Are numbers being sorted as strings?
- Is the data sorted inside a loop?
- Are duplicates, empty input and extreme values covered by tests?

## Further Reading

- Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest and Clifford Stein, *Introduction to Algorithms*, chapters on sorting and order statistics.
- Robert Sedgewick and Kevin Wayne, *Algorithms*, chapters on sorting and searching.
- Donald E. Knuth, *The Art of Computer Programming*, Volume 3: Sorting and Searching.
- Jon Bentley, *Programming Pearls*, the chapters on binary search and on sorting.
- David R. Musser, "Introspective Sorting and Selection Algorithms" (1997).
- Joshua Bloch, "Extra, Extra - Read All About It: Nearly All Binary Searches and Mergesorts are Broken" (2006).

Previous: [Trees and Graphs](./trees-and-graphs.md). Next: [Dynamic Programming and Greedy](./dynamic-programming-and-greedy.md).
