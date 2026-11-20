# Arrays, Strings and Hashing

> Arrays and hash tables are the default tools for storing and finding data; knowing how they behave in memory tells you when they are fast and when they quietly are not.

## Overview

Most programs spend most of their time reading and writing sequences and looking things up by key. Arrays give you ordered, contiguous storage with constant-time indexing. Strings are arrays of characters with extra rules. Hash tables give you near constant-time lookup by key at the price of order and some memory.

These structures are so common that engineers stop thinking about them. That is where trouble starts. A dynamic array that is resized in a loop, a string rebuilt one character at a time, or a dictionary keyed on attacker-controlled input can each turn a healthy service into a slow one.

This topic covers how they work, the common techniques built on them (two pointers, sliding windows, prefix sums, frequency counting) and the failure modes that appear in production.

## Core Concepts

### Arrays

An array stores elements in adjacent memory. Because every element has the same size, the address of element `i` is `base + i * size`. That makes indexing `O(1)`.

| Operation | Static array | Dynamic array |
| --- | --- | --- |
| Index read/write | O(1) | O(1) |
| Append | not applicable | O(1) amortized |
| Insert/delete in middle | O(n) shift | O(n) shift |
| Insert/delete at front | O(n) shift | O(n) shift |
| Search unsorted | O(n) | O(n) |
| Search sorted | O(log n) | O(log n) |

A dynamic array (Python `list`, Java `ArrayList`, C++ `std::vector`, Go slice, JavaScript `Array` in typical use) keeps spare capacity and grows when full. Growth is by a multiplicative factor, which is why appends are amortized constant. The exact factor differs by implementation.

Contiguity gives arrays their best property: cache locality. Reading elements in order loads neighbors into cache together, so sequential scans are fast in practice, often faster than the same loop over a pointer-based structure of equal big-O.

### Strings

A string is a sequence of characters, but "character" is subtle.

- Many languages store strings as bytes in an encoding such as UTF-8, where one visible character can take one to four bytes, and a visible character can be several code points (for example, a letter plus a combining accent).
- Python `str` indexes by Unicode code point. JavaScript strings index by UTF-16 code unit. Go strings are byte sequences, and `rune` is a code point.
- In Java, Python, JavaScript and Go, strings are immutable. Concatenation creates a new string.

Because of immutability, building a string with repeated `+=` can copy the growing string each time. Some runtimes optimize the common case, but you should not rely on it.

```python
# Quadratic in the worst case on runtimes without the optimization
s = ""
for part in parts:
    s += part

# Linear: collect, then join once
s = "".join(parts)
```

```typescript
const out: string[] = [];
for (const part of parts) out.push(part);
const s = out.join("");
```

### Hash tables

A hash table maps keys to values. It applies a hash function to the key to pick a bucket, then handles the case where two keys land in the same bucket (a collision).

```
key "alice" --hash--> 1234567 --mod 8--> bucket 7
key "bob"   --hash--> 9876543 --mod 8--> bucket 7   (collision)

bucket 7: [("alice", ...), ("bob", ...)]    chaining
```

Two common collision strategies:

- **Chaining**: each bucket holds a small list of entries.
- **Open addressing**: on collision, probe other slots in the same array until an empty one is found.

The **load factor** is entries divided by buckets. As it rises, collisions rise, so tables resize and rehash when it crosses a threshold. A resize is `O(n)`, amortized away over many inserts, just as with dynamic arrays.

| Operation | Average | Worst case |
| --- | --- | --- |
| Lookup | O(1) | O(n) |
| Insert | O(1) amortized | O(n) |
| Delete | O(1) | O(n) |
| Iterate | O(n + capacity) | same |

Requirements on keys: they must be hashable and equal keys must hash equally. Mutable keys are dangerous. If a key changes after insertion, its hash changes and the entry becomes unreachable. This is why Python forbids lists as dictionary keys.

Hash sets are hash tables that store keys only. Use them for membership tests and deduplication.

### Core techniques

**Frequency counting.** Count occurrences with a hash map.

```python
from collections import Counter

def is_anagram(a: str, b: str) -> bool:
    return Counter(a) == Counter(b)   # O(n + m) time, O(k) space for k distinct characters
```

**Two sum with a hash map.** Find indices of two numbers adding to a target.

```python
def two_sum(nums: list[int], target: int) -> tuple[int, int] | None:
    seen: dict[int, int] = {}          # value -> index
    for i, x in enumerate(nums):
        j = seen.get(target - x)
        if j is not None:
            return j, i
        seen[x] = i
    return None
```

One pass, `O(n)` expected time, `O(n)` space. Inserting after the check ensures an element is not paired with itself.

**Two pointers.** On a sorted array, move pointers inward from both ends.

```python
def pair_with_sum_sorted(a: list[int], target: int) -> tuple[int, int] | None:
    lo, hi = 0, len(a) - 1
    while lo < hi:
        s = a[lo] + a[hi]
        if s == target:
            return lo, hi
        if s < target:
            lo += 1
        else:
            hi -= 1
    return None
```

This runs in `O(n)` time and `O(1)` space, relying on sorted order.

**Sliding window.** Maintain a window over a sequence while it grows and shrinks.

```python
def longest_unique_substring(s: str) -> int:
    last: dict[str, int] = {}
    start = best = 0
    for i, ch in enumerate(s):
        if ch in last and last[ch] >= start:
            start = last[ch] + 1
        last[ch] = i
        best = max(best, i - start + 1)
    return best
```

Each character is processed once, so the time is `O(n)`.

**Prefix sums.** Precompute running totals so any range sum is `O(1)`.

```python
def build_prefix(a: list[int]) -> list[int]:
    p = [0]
    for x in a:
        p.append(p[-1] + x)
    return p

def range_sum(p: list[int], lo: int, hi: int) -> int:   # sum of a[lo:hi]
    return p[hi] - p[lo]
```

## Trade-offs

| Choice | Gains | Costs |
| --- | --- | --- |
| Array over hash map | Cache-friendly, compact, ordered, simple | Lookup by value is `O(n)` unless sorted |
| Hash map over sorted array | Near `O(1)` lookup | Extra memory, no range queries, order not guaranteed (or insertion order only) |
| Sorted array with binary search | Compact, range queries, `O(log n)` lookup | `O(n)` inserts |
| Preallocate capacity | Avoids repeated growth copies | Wasted memory if overestimated |
| Prefix sums | `O(1)` range queries | `O(n)` space; stale if data changes |
| Interning or canonicalizing strings | Fast equality, less memory | Lookup cost, lifetime management |

For small collections, a linear scan over a short array often beats a hash lookup, because hashing has overhead and the array fits in cache. Where the crossover lies depends on key type and runtime. Measure it if it matters.

## Failure Modes

- **Hash flooding.** If an attacker can choose keys and the hash function is predictable, they can force many keys into one bucket and push each operation toward `O(n)`. A request carrying thousands of colliding form fields or JSON keys can then burn CPU. This is a documented class of denial-of-service attack. Modern runtimes mitigate it with randomized hashing (Python randomizes string hashes per process by default) or collision-resistant tables, but you should still cap the number of keys you accept from untrusted input.
- **Mutating while iterating.** Adding or removing entries during iteration can skip elements or raise errors (Python raises `RuntimeError` when a dict changes size during iteration). Iterate over a copy or collect changes first.
- **Quadratic string building.** Repeated concatenation in a loop, or removing from the front of a list in a loop (`list.pop(0)` is `O(n)`). Use `join`, a deque, or an index.
- **Off-by-one errors.** Slices, window bounds and binary search ranges are the usual sources. Decide whether bounds are inclusive or exclusive once, then keep to it. Test with empty, one-element and two-element inputs.
- **Encoding mistakes.** Truncating a UTF-8 string at a byte limit can split a character. Comparing visually identical strings that use different Unicode normalization forms returns false. Normalize before comparing user-visible text, and count what the requirement says: bytes, code points or visible characters.
- **Floating-point keys.** `0.1 + 0.2` does not equal `0.3` in binary floating point, so such values make unreliable keys. Use integers (for example, cents) or decimals.
- **Unbounded maps used as caches.** A dictionary that only grows is a memory leak with a friendly name. Bound size and define eviction.
- **Shared aliasing.** In Python, `[[0] * n] * m` creates `m` references to one inner list. Use `[[0] * n for _ in range(m)]`.
- **Memory overhead.** Hash tables of small objects can use several times the memory of the raw data because of object headers, pointers and spare capacity. Estimate before loading millions of entries.

Detection: profile with production-shaped and hostile-shaped inputs, set limits on request size and key count, and monitor memory growth over time.

## In Practice

### Decision checklist

1. Do you need lookup by key, or only sequential access? If sequential, use an array.
2. Do you need order or range queries? If so, prefer a sorted structure over a hash table.
3. Can keys come from outside your trust boundary? If so, limit counts and rely on a runtime with hash randomization.
4. Is the collection small and hot? Measure a plain array against a map.
5. Do you need deduplication only? Use a set.
6. Will the collection grow without limit? Bound it.

### Verifying AI-generated code

- Check that membership tests use a set or dict, not a list, when run repeatedly.
- Check slice and index bounds on empty and single-element inputs.
- Confirm that string handling matches the requirement for non-ASCII text.
- Confirm that dictionary keys are immutable and hashable, and that floats are not used as identity.
- Read each library call for hidden linear work: `insert(0, x)`, `pop(0)`, `remove`, `index`, `count`, slicing and concatenation.
- Run it against adversarial inputs: all duplicates, all distinct, sorted, reverse sorted and very large.

## Further Reading

- Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest and Clifford Stein, *Introduction to Algorithms*, chapters on hash tables and elementary data structures.
- Robert Sedgewick and Kevin Wayne, *Algorithms*, sections on symbol tables and hashing.
- Steven S. Skiena, *The Algorithm Design Manual*, section on dictionaries and hashing.
- Scott A. Crosby and Dan S. Wallach, "Denial of Service via Algorithmic Complexity Attacks" (USENIX Security 2003).
- The Unicode Consortium, *The Unicode Standard* and Unicode Standard Annex 15 on normalization forms.

Previous: [Complexity Analysis](./complexity-analysis.md). Next: [Linked Lists, Stacks and Queues](./linked-lists-stacks-and-queues.md).
