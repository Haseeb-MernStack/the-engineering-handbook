# Self-Assessment: Data Structures and Algorithms

> Check what you can do today, not what you have read. Each item describes observable behavior. Be strict: if you could not do it in front of a colleague without looking it up, leave it unchecked.

## How to use this page

1. Work through one topic at a time. Check an item only if you can do it **unprompted** and **explain why**.
2. Count the checked items for each topic and use the score guide at the end.
3. Revisit in three to six months. The goal is a shrinking list of unchecked items, not a perfect score.

## Complexity Analysis

- [ ] I can state the time and space cost of a loop, a nested loop and a recursive call without a reference.
- [ ] I can explain amortized cost and name an operation where it applies.
- [ ] I can name a case where a lower-complexity algorithm is slower in practice, and give the reason.
- [ ] I measure with realistic data before I call code "slow" or "fast".
- [ ] I can spot hidden quadratic behavior, such as a lookup in a list inside a loop, in code review.

## Arrays, Strings and Hashing

- [ ] I can explain how a dynamic array grows and why append is amortized constant.
- [ ] I can explain how a hash table handles collisions and what happens as it fills.
- [ ] I know why mutable objects are unsafe as hash keys.
- [ ] I can explain how attacker-chosen keys can degrade a hash table, and one way to mitigate it.
- [ ] I know the difference between bytes, code points and user-perceived characters in strings, and where it causes bugs.

## Linked Lists, Stacks and Queues

- [ ] I can explain why a linked list is often slower than an array despite cheaper insertion.
- [ ] I can implement a queue with a ring buffer and say what happens when it is full.
- [ ] I can say when a queue must be bounded, and what the system should do at the bound.
- [ ] I can convert a recursive traversal into an explicit stack version.

## Trees and Graphs

- [ ] I can explain why an unbalanced binary search tree can degrade to linear time.
- [ ] I can choose between breadth-first and depth-first traversal for a given question.
- [ ] I can explain when Dijkstra's algorithm is valid and when it is not.
- [ ] I can detect a cycle in a dependency graph and explain what to do with one.
- [ ] I can explain why databases use B-trees rather than binary trees.

## Sorting and Searching

- [ ] I can explain stability and give a case where it matters.
- [ ] I can write a binary search and name its common boundary bugs.
- [ ] I can write a comparator that is consistent, and explain what goes wrong if it is not.
- [ ] I can say when a full sort is unnecessary, for example when I need only the top `k` items.

## Dynamic Programming and Greedy Algorithms

- [ ] I can state the two properties a problem needs for DP to apply.
- [ ] I can define state, recurrence and base cases for a small DP problem.
- [ ] I can build a counterexample that breaks a plausible greedy rule.
- [ ] I can estimate the memory of a DP table before writing the code.

## Choosing a Data Structure

- [ ] I list the operations, their frequency and the maximum size before I pick a structure.
- [ ] I bound every cache, queue and collection I add, and decide what gets evicted.
- [ ] I do not depend on iteration order the structure does not promise.
- [ ] I can explain why I chose a structure, and what measurement would show I was wrong.

## Reviewing generated code

- [ ] I can read AI-generated algorithmic code and state its time and space complexity.
- [ ] I check generated code for unbounded growth, boundary errors and unstated ordering assumptions.
- [ ] I write a test or comparison against a simple brute-force version for non-trivial algorithms.

## Score guide

Count the checked items **per topic**.

| Checked in the topic | Where you are | What to do |
| --- | --- | --- |
| All or all but one | Solid | Move on. Test yourself with the [question bank](./question-bank.md). |
| About half | Partial | Reread the topic, then redo the checklist. Write a small program that exercises each unchecked item. |
| Fewer than half | Gap | Start with the topic's **Overview** and **Core Concepts**, then build the worked example by hand. |

If the **Reviewing generated code** group is mostly unchecked, work through the *Failure Modes* and *In Practice* sections of each topic. They are written for this purpose.

## Related

- Practice questions: [Question Bank](./question-bank.md)
- Decision aid: [Choosing a Data Structure](./choosing-a-data-structure.md)
- Back to the [section index](./README.md)
