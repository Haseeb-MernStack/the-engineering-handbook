# Complexity Analysis

> Complexity analysis describes how the cost of an algorithm grows with input size; it predicts scaling behavior well, and it predicts absolute speed poorly.

## Overview

Every algorithm consumes time and memory, and both depend on the size of the input. Complexity analysis gives you a vocabulary for that dependence. Instead of timing one run on one machine, you describe how cost grows as the input grows: doubling, quadrupling, or barely changing.

This matters because systems rarely stay at the size you tested. A routine that handles 100 records in a blink can take hours on 10 million if its cost grows quadratically. Most performance incidents that trace back to an algorithm are not exotic. They are an `O(n^2)` loop that was fine in development and ruinous in production.

Asymptotic analysis is also an approximation. It discards constant factors, ignores the memory hierarchy and says nothing about small inputs. An engineer uses it to rule out bad designs early, then uses measurement to choose among the survivors.

## Core Concepts

### Big-O, Omega and Theta

- **O(f(n))**: an upper bound. The cost grows no faster than `f(n)`, up to a constant, for large `n`.
- **Omega(f(n))**: a lower bound. The cost grows at least as fast as `f(n)`.
- **Theta(f(n))**: a tight bound. Both of the above hold.

In everyday engineering talk, "O(n)" usually means "Theta(n) in the case I care about". Be precise when it matters. Saying a function is `O(n^2)` is true of a linear function too, but it is not informative.

### Common growth rates

| Class | Name | Typical example |
| --- | --- | --- |
| O(1) | Constant | Array index, hash lookup (average) |
| O(log n) | Logarithmic | Binary search, balanced tree lookup |
| O(n) | Linear | Scanning a list |
| O(n log n) | Linearithmic | Comparison sorting (merge sort, heap sort) |
| O(n^2) | Quadratic | Nested loops over the same collection |
| O(2^n) | Exponential | Enumerating all subsets |
| O(n!) | Factorial | Enumerating all permutations |

A rough sense of scale helps. For `n = 1,000,000`, a linear algorithm does on the order of a million steps. A quadratic one does on the order of a trillion. No hardware upgrade closes that gap.

### Best, worst, average and expected cases

- **Worst case** bounds the cost for any input. Use it when latency guarantees matter or when inputs may be adversarial.
- **Average case** assumes a distribution of inputs. It is only as good as that assumption.
- **Expected case** applies to randomized algorithms, where the average is taken over the algorithm's own random choices rather than the data.

Quicksort with a fixed pivot rule is `O(n log n)` on average and `O(n^2)` in the worst case. A hash table is `O(1)` on average and `O(n)` per operation in the worst case, when many keys collide.

### Amortized analysis

Some operations are occasionally expensive but cheap on average over a sequence. Appending to a dynamic array is the standard example. When the array is full, it allocates a larger buffer (commonly a constant multiple of the old size) and copies every element. That single append costs `O(n)`. Spread across all appends, the cost per append is `O(1)` amortized.

Amortized does not mean each call is fast. One call in the sequence can be slow, which matters for latency-sensitive code.

### Space complexity

Space counts extra memory beyond the input. Recursion consumes stack space proportional to depth. Memoization trades memory for time. State both: "O(n) time, O(1) extra space" is a complete statement; "O(n)" alone is not.

### Reading complexity from code

```python
def has_duplicate_slow(items: list[int]) -> bool:
    for i in range(len(items)):
        for j in range(i + 1, len(items)):   # about n^2 / 2 comparisons
            if items[i] == items[j]:
                return True
    return False

def has_duplicate_fast(items: list[int]) -> bool:
    seen: set[int] = set()
    for x in items:                          # n iterations
        if x in seen:                        # O(1) average
            return True
        seen.add(x)
    return False
```

The first is `O(n^2)` time and `O(1)` space. The second is `O(n)` expected time and `O(n)` space. The second spends memory to buy time, which is the most common trade in this field.

Rules of thumb for reading code:

1. Sequential blocks add; the larger term dominates.
2. Nested loops multiply, but only if the inner bound depends on the outer variable or on `n`.
3. A loop that halves the problem each step is logarithmic.
4. A recursive call count times the work per call gives the total. Draw the recursion tree if unsure.
5. Count the cost of library calls. `x in some_list`, `list.insert(0, x)`, `str += s` in a loop and slicing can each hide a linear cost.

### Recurrences

Divide-and-conquer algorithms are described by recurrences. Merge sort satisfies `T(n) = 2T(n/2) + O(n)`, which solves to `O(n log n)`: there are about `log2 n` levels, and each level does `O(n)` total work.

```
level 0:            n                 -> n work
level 1:      n/2       n/2           -> n work
level 2:   n/4  n/4  n/4  n/4         -> n work
...                                   (log2 n levels)
```

## Trade-offs

| Decision | Gains | Costs |
| --- | --- | --- |
| Use a lower-complexity algorithm | Scales to larger inputs | Often more code, more memory, harder to read |
| Precompute or cache | Faster queries | Memory, invalidation logic, stale data risk |
| Index or sort up front | Fast repeated lookups | Build cost; maintenance on every write |
| Keep the simple quadratic version | Clarity, low overhead for small `n` | Cliff when `n` grows |
| Optimize early | Avoids later rewrites | Wasted effort if the path is not hot |

### When asymptotics mislead

- **Constants matter at small n.** An `O(n)` method with a heavy constant can lose to an `O(n^2)` method with a tiny one for small inputs. This is why many production sort implementations switch to insertion sort for short ranges.
- **Memory layout dominates.** Contiguous arrays are read in cache-friendly runs. Pointer-heavy structures such as linked lists and many tree layouts jump around memory. Two algorithms with the same big-O can differ by a large factor in practice. Measure before assuming a factor.
- **Hidden factors in "O(1)".** A hash lookup must hash the key. Hashing a long string costs time proportional to its length. Treating key size as constant is usually fine, until it is not.
- **I/O dwarfs CPU.** One network or disk round trip costs far more than many in-memory operations. An `O(n)` algorithm making `n` remote calls is worse than an `O(n log n)` one making one.
- **Big-O hides the multiplier.** `O(n)` with a 50x constant is still `O(n)`.

## Failure Modes

- **The accidental quadratic.** A loop that calls `list.remove`, `in` on a list, `str.join` done by repeated concatenation, or `list.insert(0, x)` is quadratic overall. It passes tests on small data and times out in production. Detect it by profiling with inputs at realistic and 10x scale, and by plotting time against input size.
- **N+1 queries.** One query fetches `n` parent rows, then `n` more queries fetch children. The code looks linear, but each step is a network round trip. Batch or join instead.
- **Worst-case inputs.** Algorithms with a bad worst case can be attacked. Sorted input against a naive-pivot quicksort, or crafted keys against a predictable hash function, can force quadratic behavior from a service that is fast on ordinary traffic. Use algorithms with worst-case guarantees on untrusted input, or randomize.
- **Integer overflow in index math.** The classic midpoint `(lo + hi) / 2` can overflow fixed-width integers when `lo + hi` exceeds the maximum. Write `lo + (hi - lo) / 2`. Python integers do not overflow, but the same code ported to Java, C, C++ or Go does.
- **Unbounded recursion.** Depth proportional to `n` exhausts the call stack. Python's default recursion limit is on the order of a thousand frames, and other runtimes have their own limits. Convert to iteration or use an explicit stack for deep inputs.
- **Memory blowup from time optimizations.** Memoizing every intermediate result or materializing every candidate trades time for memory without bound. Cap caches and measure peak memory.
- **Benchmarks that lie.** Tiny inputs, warm caches, a single run, or data unlike production produce misleading numbers. Compare at several sizes, repeat runs, and use production-shaped data.

## In Practice

### A workflow

1. State the input size and its realistic maximum. If you do not know it, find out.
2. Estimate complexity by reading the code. Reject designs whose worst case cannot meet the requirement.
3. Write the simplest correct version.
4. Measure with a profiler on production-shaped data. Fix the measured hot spot, not the suspected one.
5. Keep a regression benchmark for any path where a complexity change would hurt.

### Quick checklist before accepting an algorithm

- What is `n`, and what is its maximum?
- What is the worst-case time, and can an attacker or an unlucky dataset trigger it?
- What is the extra space?
- Does it make remote calls inside a loop?
- Is recursion depth bounded?
- Is it correct on empty input, a single element and duplicates?

### Measuring growth empirically

```python
import time

def timed(fn, n):
    data = list(range(n))
    start = time.perf_counter()
    fn(data)
    return time.perf_counter() - start

for n in (10_000, 20_000, 40_000, 80_000):
    print(n, round(timed(has_duplicate_slow, n), 4))
# If time roughly quadruples each time n doubles, the growth is quadratic.
```

Doubling `n` and watching the ratio is a quick, honest check of growth. Note that the worst case for `has_duplicate_slow` is a list with no duplicates, which `range(n)` provides.

### Verifying AI-generated code

Generated code often favors the shortest correct-looking form. Look for:

- Membership tests against lists inside loops, where a set was needed.
- Repeated string concatenation or list copying inside loops.
- Sorting inside a loop when sorting once would do.
- Recursion without a depth bound or memoization.
- Claims in comments such as "O(n)" that the code does not support. Verify the stated complexity yourself by reading each line and each library call.

Ask for the complexity, then check it. The explanation is a hypothesis, not a proof.

## Further Reading

- Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest and Clifford Stein, *Introduction to Algorithms* (CLRS), chapters on growth of functions, recurrences and amortized analysis.
- Robert Sedgewick and Kevin Wayne, *Algorithms*, for analysis of algorithms with empirical growth tests.
- Steven S. Skiena, *The Algorithm Design Manual*, for the practical treatment of asymptotics.
- Jon Bentley, *Programming Pearls*, for performance reasoning in real programs.
- Robert E. Tarjan, "Amortized Computational Complexity" (1985), the foundational treatment of amortization.

Next: [Arrays, Strings and Hashing](./arrays-strings-and-hashing.md).
