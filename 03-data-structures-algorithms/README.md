# Data Structures and Algorithms

## Overview

Data structures decide how data is organized in memory and on disk. Algorithms decide how you process it. Together they set the upper limit on how fast, how large and how predictable your software can be. Hardware and frameworks can hide a poor choice for a while, but they cannot remove it.

An engineer uses this material as a working vocabulary, not as a set of puzzles. It lets you predict how code behaves as data grows, explain why a query or loop became slow, choose between a hash map, a tree and a sorted array on evidence, and review generated code for hidden quadratic behavior or unbounded growth.

## What you will learn

- How to reason about time and space cost, and when asymptotics mislead.
- How arrays, strings and hash tables work, and how they fail.
- How linked structures, stacks and queues behave, and where each fits.
- How trees and graphs model hierarchy and relationships, and how to traverse them.
- How sorting and searching work, and how to avoid comparator and boundary bugs.
- How dynamic programming and greedy methods exploit problem structure.
- How to choose a data structure from the access pattern, not from habit.

## Topics

| Topic | What it covers |
| --- | --- |
| [Complexity Analysis](./complexity-analysis.md) | Big-O, amortized cost, space, constants and measuring instead of guessing. |
| [Arrays, Strings and Hashing](./arrays-strings-and-hashing.md) | Contiguous storage, dynamic arrays, string behavior, hash tables and collisions. |
| [Linked Lists, Stacks and Queues](./linked-lists-stacks-and-queues.md) | Pointer-based structures, LIFO and FIFO, deques, ring buffers and bounded queues. |
| [Trees and Graphs](./trees-and-graphs.md) | Hierarchies, balanced trees, heaps, traversal, shortest paths and cycle detection. |
| [Sorting and Searching](./sorting-and-searching.md) | Sorting properties, stability, binary search and comparator correctness. |
| [Dynamic Programming and Greedy Algorithms](./dynamic-programming-and-greedy.md) | Optimal substructure, memoization, tabulation, exchange arguments and when greedy fails. |
| [Choosing a Data Structure](./choosing-a-data-structure.md) | A method for matching access patterns to structures, with reference defaults. |

## Prerequisites

- Comfort writing small programs in at least one language.
- The material in [Engineering Fundamentals](../01-engineering-fundamentals/README.md) and [Programming Concepts](../02-programming-concepts/README.md), especially [Memory Management](../02-programming-concepts/memory-management.md).

## What good looks like

- You state the expected size of your data and the cost of your main operations before choosing a structure.
- You can explain why an algorithm with a better asymptotic bound may lose at small sizes.
- You avoid scans over lists inside loops over lists, and you can spot them in review.
- You bound every queue, cache and collection, and decide what is evicted.
- You treat attacker-controlled keys as a risk for hash structures.
- You use the standard library sort and search, and you test comparators and boundary cases.
- You try to break a greedy idea with a counterexample before trusting it.
- You benchmark hot paths with realistic data instead of relying on intuition.
- You can read generated algorithmic code and say what its complexity is and where it fails.

## Where this connects

- Storage-level structures such as B-trees and indexes: [Database Engineering](../06-database-engineering/README.md).
- Measuring and fixing slow code: [Performance and Scalability](../12-performance-scalability/README.md).
- Designing systems around data growth: [System Design](../05-system-design/README.md).

Back to [The Engineering Handbook](../README.md).
