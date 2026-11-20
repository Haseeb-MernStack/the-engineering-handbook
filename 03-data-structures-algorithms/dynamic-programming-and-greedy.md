# Dynamic Programming and Greedy Algorithms

> Dynamic programming and greedy methods solve optimization problems by exploiting structure; the engineering skill is recognizing when that structure exists and proving, or at least testing, that your shortcut is correct.

## Overview

Many problems ask for the best of something: the cheapest route, the largest set of compatible jobs, the fewest coins. Brute force tries every candidate and is usually exponential. Two families of techniques avoid that by using structure in the problem.

**Dynamic programming (DP)** applies when a problem breaks into overlapping subproblems and the best answer to the whole is built from best answers to the parts. You solve each subproblem once and reuse the result.

**Greedy algorithms** apply when a locally best choice, made repeatedly and never revisited, still yields a globally best answer. They are simpler and faster than DP, but they are only correct for problems with the right structure.

Outside interviews, these ideas show up in diffing tools, spell checkers, query optimizers, scheduling, routing, resource allocation, compression and text layout. You will rarely implement them from scratch, but you will need to recognize them, reason about their cost, and review code that uses them.

## Core Concepts

### The two properties DP needs

1. **Optimal substructure.** An optimal solution contains optimal solutions to its subproblems.
2. **Overlapping subproblems.** The same subproblems recur, so caching pays off. Without overlap, you have divide and conquer (as in merge sort), not DP.

### Two implementation styles

| Style | How it works | Strength | Cost |
| --- | --- | --- | --- |
| Top-down (memoization) | Write the natural recursion, cache results by argument | Easy to derive; computes only reachable states | Recursion depth limits; call overhead |
| Bottom-up (tabulation) | Fill a table in dependency order | No recursion; easier to shrink memory | You must get the evaluation order right |

### A worked example: minimum coins

Given coin denominations and a target amount, find the fewest coins that sum to it. Greedy (always take the largest coin) works for some coin systems but fails for others. With coins `{1, 3, 4}` and target `6`, greedy picks `4 + 1 + 1` (three coins), but the optimum is `3 + 3` (two coins).

DP defines `best[a]` as the fewest coins to make amount `a`:

```python
def min_coins(coins: list[int], target: int) -> int:
    INF = float("inf")
    best = [0] + [INF] * target
    for amount in range(1, target + 1):
        for coin in coins:
            if coin <= amount and best[amount - coin] + 1 < best[amount]:
                best[amount] = best[amount - coin] + 1
    return -1 if best[target] == INF else best[target]

assert min_coins([1, 3, 4], 6) == 2
assert min_coins([2], 3) == -1
```

Time is `O(target * len(coins))`. Space is `O(target)`. Note that this is pseudo-polynomial: it is polynomial in the numeric value of `target`, not in the number of bits needed to write it. A target of a billion makes this impractical.

### Designing a DP solution

1. **Define the state.** What minimal information identifies a subproblem? (Here, the remaining amount.)
2. **Write the recurrence.** How does a state's answer derive from smaller states?
3. **Set base cases.** What is the answer for the smallest states?
4. **Choose the order.** Which states must be computed before others?
5. **Identify the answer.** Which state, or combination of states, is the final result?
6. **Optimize space** if needed, by keeping only the rows or entries still required.

### Classic DP families

| Family | Example | Typical state |
| --- | --- | --- |
| Sequence alignment | Edit distance, diff | `(i, j)` positions in two sequences |
| Knapsack | Budgeted selection | `(item index, remaining capacity)` |
| Partition and path counting | Grid paths, coin ways | Position or remaining amount |
| Interval | Matrix chain ordering | `(left, right)` bounds |
| Shortest paths | Bellman-Ford, Floyd-Warshall | Node and step count or node pair |

### Memoization in practice

```python
from functools import lru_cache

def edit_distance(a: str, b: str) -> int:
    @lru_cache(maxsize=None)
    def go(i: int, j: int) -> int:
        if i == len(a):
            return len(b) - j
        if j == len(b):
            return len(a) - i
        if a[i] == b[j]:
            return go(i + 1, j + 1)
        return 1 + min(go(i + 1, j),      # delete from a
                       go(i, j + 1),      # insert into a
                       go(i + 1, j + 1))  # substitute
    return go(0, 0)

assert edit_distance("kitten", "sitting") == 3
```

This has `(len(a)+1) * (len(b)+1)` states and constant work per state. The recursion depth is bounded by `len(a) + len(b)`, so very long inputs can overflow the call stack in languages with limited recursion. Bottom-up avoids that.

### Greedy algorithms

A greedy algorithm builds a solution step by step, always taking the choice that looks best now. It is correct when two properties hold:

- **Greedy-choice property.** Some optimal solution includes the greedy choice.
- **Optimal substructure.** After making the choice, the remainder is the same kind of problem.

A standard correct example is **interval scheduling**: given jobs with start and end times, select the maximum number of non-overlapping jobs. Sort by earliest finish time, then repeatedly take the next job that starts after the last chosen one ends.

```python
def max_non_overlapping(jobs: list[tuple[int, int]]) -> int:
    count, last_end = 0, float("-inf")
    for start, end in sorted(jobs, key=lambda j: j[1]):
        if start >= last_end:
            count += 1
            last_end = end
    return count

assert max_non_overlapping([(1, 3), (2, 4), (3, 5)]) == 2
```

Other well-known greedy results include Dijkstra's shortest paths (with non-negative weights), Kruskal's and Prim's minimum spanning tree algorithms, and Huffman coding.

The usual proof technique is an **exchange argument**: take any optimal solution and show you can swap in the greedy choice without making it worse.

## Trade-offs

| Choice | What you gain | What it costs |
| --- | --- | --- |
| Greedy | Simple code, usually `O(n log n)` or better, low memory | Correct only for problems with the right structure; wrong answers look plausible |
| DP | Exact optimum for a broad class of problems | Memory for the table, pseudo-polynomial time in some problems, harder to derive |
| Brute force | Obviously correct, easy to write | Exponential time; usable only for tiny inputs |
| Approximation or heuristic | Fast answers on hard problems | No optimality guarantee; needs a measured error bound |

Key points:

- **Memory is the common DP bottleneck.** A table over two large dimensions can be huge. Rolling arrays reduce space, at the cost of losing the ability to reconstruct the solution unless you track choices separately.
- **Reconstruction costs extra.** Computing the optimal value is often easier than recovering the optimal choices. Storing back-pointers adds memory.
- **Exactness may not be worth it.** For NP-hard problems such as general knapsack at large scale or the travelling salesman problem, production systems often use heuristics or solvers and accept near-optimal answers.

## Failure Modes

- **A greedy shortcut that is wrong.** It passes small examples and fails on adversarial ones. The coin example above is typical. Detect it by comparing against brute force on many random small inputs.
- **Hidden state omitted from the DP.** If the answer depends on something the state does not capture, results are silently wrong. Symptom: the answer changes with evaluation order.
- **Memoization key errors.** Caching on a mutable object, or on arguments that do not fully identify the subproblem, returns stale or incorrect values.
- **Recursion depth.** Deep top-down recursion raises stack overflows or recursion errors on large input. Prefer tabulation or an explicit stack.
- **Pseudo-polynomial blow-up.** An algorithm that is `O(n * W)` looks fast until `W` is a large numeric value such as a price in minor currency units or a byte count.
- **Unbounded caches.** A memoization cache that lives for the process lifetime and is keyed on user input grows without limit.
- **Integer overflow and sentinel misuse.** Using a large integer as infinity, then adding to it, can overflow in fixed-width languages. Floating-point infinity avoids that, at the cost of type mixing.
- **Floating-point ties in greedy choices.** Comparing floats for "best" can produce unstable or incorrect choices. Use integer units where possible.

## In Practice

### How to decide

1. Can you state the problem as "optimize over choices"? If not, neither technique applies.
2. Can a local choice be proven safe (exchange argument) or refuted with a counterexample? Try to break the greedy idea first. It takes minutes.
3. If greedy fails, look for a state definition that captures all that matters about the past. If the state space is small enough to enumerate, DP is viable.
4. Estimate the table size before coding: number of states times bytes per state.
5. If the table is too big, consider space reduction, pruning, approximation or a different formulation.

### Review checklist

- Is the state definition written down in a comment or doc, not only implied by code?
- Are the base cases and evaluation order correct?
- Is there a randomized test against brute force on small inputs?
- Are the time and memory bounds stated in terms of the real input sizes, including large numeric values?
- Is recursion depth bounded?
- Is any cache bounded and keyed correctly?

### Developer vs. engineer

A developer who finds a plausible greedy rule ships it when the sample cases pass. An engineer asks what property makes it correct, tries to build a counterexample, and writes a brute-force comparison test. The engineer also checks the size of the table against real data before committing to DP.

### Verifying AI-generated solutions

Generated DP code often has the right shape and a wrong detail: an off-by-one in a base case, a missing state dimension, or a loop order that reads values not yet computed. Generated greedy solutions are frequently stated confidently without justification. Ask for the recurrence or the exchange argument, check it yourself, and run a randomized comparison against a simple exhaustive solution.

## Further Reading

- Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest and Clifford Stein, *Introduction to Algorithms*, chapters on dynamic programming and greedy algorithms.
- Jon Kleinberg and Eva Tardos, *Algorithm Design*, for greedy exchange arguments and DP formulations.
- Steven S. Skiena, *The Algorithm Design Manual*, for guidance on recognizing problem types.
- Richard Bellman, *Dynamic Programming*, the original treatment of the method.
- Sanjoy Dasgupta, Christos Papadimitriou and Umesh Vazirani, *Algorithms*, for a concise treatment of both techniques.

Previous: [Sorting and Searching](./sorting-and-searching.md). Next: [Choosing a Data Structure](./choosing-a-data-structure.md).
