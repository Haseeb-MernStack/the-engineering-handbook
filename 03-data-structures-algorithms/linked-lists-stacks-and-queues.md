# Linked Lists, Stacks and Queues

> Linked lists, stacks and queues define how items are ordered and removed; the right choice depends on the access pattern, and on memory behavior as much as on big-O.

## Overview

These structures describe sequences with restricted or specialized access. A linked list connects nodes by references. A stack removes the most recently added item first. A queue removes the oldest first. Stacks and queues are abstract behaviors, and each can be built on an array or on a linked list.

They appear everywhere in real systems: the call stack, undo history, expression parsing, breadth-first search, task schedulers, message brokers, rate limiters and buffers between a fast producer and a slow consumer.

The textbook case for linked lists, constant-time insertion in the middle, rarely wins on modern hardware. This topic explains why, and where linked structures still earn their place. It also covers the failure modes that matter in production, chiefly unbounded queues and unbounded recursion.

## Core Concepts

### Linked lists

A singly linked list node holds a value and a reference to the next node. A doubly linked list also holds a reference to the previous node.

```
head
 |
[ 3 | * ]-->[ 7 | * ]-->[ 9 | * ]-->null
```

| Operation | Singly linked | Doubly linked | Dynamic array |
| --- | --- | --- | --- |
| Access by index | O(n) | O(n) | O(1) |
| Insert/delete at head | O(1) | O(1) | O(n) |
| Insert/delete at tail | O(n), or O(1) with tail pointer for insert | O(1) | O(1) amortized for append |
| Insert/delete after a known node | O(1) | O(1) | O(n) |
| Delete a given node (have only the node) | O(n) | O(1) | O(n) |
| Search | O(n) | O(n) | O(n) |

The advantage is real only when you already hold a reference to the position. Finding that position costs `O(n)`.

Reversing a singly linked list in place:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class Node:
    val: int
    next: Optional["Node"] = None

def reverse(head: Optional[Node]) -> Optional[Node]:
    prev = None
    cur = head
    while cur is not None:
        nxt = cur.next      # save the rest
        cur.next = prev     # flip the link
        prev, cur = cur, nxt
    return prev
```

Time `O(n)`, extra space `O(1)`.

Detecting a cycle with two pointers moving at different speeds (Floyd's method):

```python
def has_cycle(head: Optional[Node]) -> bool:
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

If a cycle exists, the fast pointer eventually catches the slow one. Without a cycle, the fast pointer reaches the end.

### Stacks

A stack is last in, first out (LIFO). Core operations are `push`, `pop` and `peek`, each `O(1)`. In most languages, a dynamic array is the natural implementation: push and pop at the end.

Classic use: matching brackets.

```python
def balanced(s: str) -> bool:
    pairs = {")": "(", "]": "[", "}": "{"}
    stack: list[str] = []
    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False
    return not stack
```

Stacks also model depth-first traversal, backtracking, undo and evaluation of postfix expressions. Recursion uses the call stack implicitly. Replacing recursion with an explicit stack avoids stack overflow on deep inputs.

### Queues and deques

A queue is first in, first out (FIFO). Core operations are `enqueue` and `dequeue`.

A naive array-based queue that removes from the front costs `O(n)` per dequeue, because remaining elements shift. Use a structure designed for it:

- a **ring buffer** (circular array) with head and tail indices, giving `O(1)` for both ends and a fixed capacity;
- a **linked list** with head and tail pointers;
- a library **deque** (double-ended queue), such as Python's `collections.deque` or Java's `ArrayDeque`.

```python
from collections import deque

def bfs_order(graph: dict[int, list[int]], start: int) -> list[int]:
    seen = {start}
    order = []
    q = deque([start])
    while q:
        node = q.popleft()          # O(1); list.pop(0) would be O(n)
        order.append(node)
        for nb in graph.get(node, []):
            if nb not in seen:
                seen.add(nb)
                q.append(nb)
    return order
```

A **ring buffer** is worth knowing because it needs no allocation after setup:

```
capacity 5, head=2, count=3 (items A, B, C)

index:  0   1   2   3   4
       [ ] [ ] [A] [B] [C]
next write goes to (head + count) % 5 = 0, so the buffer wraps around.
```

### Priority queues and monotonic structures

A **priority queue** removes the item with the highest priority rather than the oldest. A binary heap implements it with `O(log n)` insert and remove, and `O(1)` peek. It is used for schedulers, top-k problems and shortest-path algorithms.

```python
import heapq

def k_largest(nums: list[int], k: int) -> list[int]:
    return heapq.nlargest(k, nums)    # roughly O(n log k)
```

A **monotonic stack** keeps its contents sorted as you push, popping items that can no longer be the answer. It solves "next greater element" style problems in `O(n)` total, since each element is pushed and popped at most once.

```python
def next_greater(nums: list[int]) -> list[int]:
    res = [-1] * len(nums)
    stack: list[int] = []              # indices with decreasing values
    for i, x in enumerate(nums):
        while stack and nums[stack[-1]] < x:
            res[stack.pop()] = x
        stack.append(i)
    return res
```

## Trade-offs

| Choice | Gains | Costs |
| --- | --- | --- |
| Linked list over array | Cheap insert/delete at a known node; stable references to nodes | No indexing; one allocation per node; poor cache locality; pointer overhead |
| Array-backed stack/queue | Compact, cache-friendly, fast in practice | Occasional resize copy; ring buffers have fixed capacity unless grown |
| Linked-list queue | No resize; steady per-operation cost | Allocation per item; garbage collection or free-list pressure |
| Doubly linked list | Delete a known node in `O(1)` (useful for LRU caches) | Extra pointer per node |
| Bounded queue | Memory and latency are capped; applies backpressure | Producers must block, drop or fail when full |
| Unbounded queue | Producers never block | Memory grows without limit; latency grows silently |

Linked lists are slow in practice for a reason. Each node is a separate allocation, so traversal chases pointers across memory, and every step may miss the cache. A dynamic array that shifts elements with a block copy often beats a linked list for insertion in the middle until the collection is large. Measure for your sizes.

Where linked lists do fit: LRU caches (a hash map pointing into a doubly linked list gives `O(1)` lookup, move-to-front and eviction), intrusive lists inside kernels and allocators, and lock-free structures where nodes are swapped by pointer. Many general application codes need none of these.

## Failure Modes

- **Unbounded queues.** A queue with no limit between a fast producer and a slow consumer grows until the process runs out of memory, or latency becomes so large that results arrive after nobody wants them. Bound queue length, decide what happens at the limit (block, reject, drop oldest) and alert on depth and age of the oldest item.
- **Stack overflow from deep recursion.** Recursive traversal of a long linked list, a degenerate tree or a hostile nested input (deeply nested JSON, for example) exhausts the stack. Cap the depth of untrusted input, or convert to an explicit stack.
- **Pop from empty.** `pop()` on an empty stack, `peek` on an empty queue, or `head.next` on a null head. Handle the empty case explicitly and test it.
- **Lost nodes during pointer surgery.** Overwriting `cur.next` before saving the old value drops the rest of the list. Order the assignments and test with zero, one and two nodes.
- **Cycles from bugs.** A list that accidentally loops makes traversal infinite. Guard loops that walk untrusted or complex structures, and add cycle checks in tests.
- **Memory leaks via retained references.** A stack or queue that "removes" items by moving an index but leaves references in the array keeps objects alive. Clear slots on removal. Likewise, a cache's linked list must be unlinked on eviction.
- **Quadratic front removal.** `list.pop(0)` or `shift()` in a loop. Use a deque or an index.
- **Ordering assumptions.** A queue guarantees order only within itself. Multiple consumers, retries and partitions can reorder processing. Do not assume end-to-end FIFO unless the system says so.
- **Concurrency races.** Plain stacks and queues are not safe for concurrent writers. Use a thread-safe structure or a lock, and test under load.

Detection: expose queue depth and oldest-item age as metrics, test with empty and single-element cases, fuzz with deeply nested inputs, and run a heap profiler if memory grows over time.

## In Practice

### Decision checklist

1. Do you need removal from one end only (stack) or both (deque)? Pick the structure whose operations match.
2. Do you need to remove from the middle given only a node reference? A doubly linked list may fit, as in an LRU cache.
3. Do you need order by priority, not arrival? Use a heap.
4. Is the producer faster than the consumer at times? Bound the queue.
5. Is depth unbounded? Use an explicit stack.
6. Default to the language's array-backed or deque type unless profiling shows a reason to do otherwise.

### Worked example: bounded work queue

```typescript
class BoundedQueue<T> {
  private buf: (T | undefined)[];
  private head = 0;
  private size = 0;

  constructor(private readonly capacity: number) {
    this.buf = new Array(capacity);
  }

  offer(item: T): boolean {              // false means "full": caller decides
    if (this.size === this.capacity) return false;
    this.buf[(this.head + this.size) % this.capacity] = item;
    this.size++;
    return true;
  }

  poll(): T | undefined {
    if (this.size === 0) return undefined;
    const item = this.buf[this.head];
    this.buf[this.head] = undefined;     // release the reference
    this.head = (this.head + 1) % this.capacity;
    this.size--;
    return item;
  }
}
```

Both operations are `O(1)`. The `offer` result forces the caller to handle a full queue rather than ignoring it.

### Verifying AI-generated code

- Does it use `pop(0)`, `shift()` or `insert(0, x)` where a deque belongs?
- Does a hand-written linked list handle empty, single-node and head or tail removal?
- Is recursion used on input that could be deep?
- Is the queue bounded, and what happens when it fills?
- Are references cleared when items are removed?
- Is the structure shared across threads without protection?

## Further Reading

- Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest and Clifford Stein, *Introduction to Algorithms*, chapters on elementary data structures and heapsort.
- Robert Sedgewick and Kevin Wayne, *Algorithms*, section on bags, queues and stacks.
- Donald E. Knuth, *The Art of Computer Programming*, Volume 1, section on information structures.
- Ulrich Drepper, "What Every Programmer Should Know About Memory" (2007), for the cache effects behind pointer chasing.
- Martin Kleppmann, *Designing Data-Intensive Applications*, for queues, backpressure and ordering in distributed systems.

Previous: [Arrays, Strings and Hashing](./arrays-strings-and-hashing.md). Next: [Trees and Graphs](./trees-and-graphs.md).
