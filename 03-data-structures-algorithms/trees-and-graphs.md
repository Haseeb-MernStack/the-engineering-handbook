# Trees and Graphs

> Trees and graphs model hierarchy and relationships; most real problems (dependencies, routes, permissions, indexes) reduce to a handful of traversal and shortest-path patterns.

## Overview

A graph is a set of nodes (vertices) connected by edges. A tree is a connected graph with no cycles, usually with a designated root. File systems, the DOM, organization charts, database indexes and parse trees are trees. Road maps, social networks, package dependencies, build systems and service call maps are graphs.

Recognizing that a problem is a graph problem is half the work. "Which service must start before this one?" is topological ordering. "What is the fewest hops between two accounts?" is shortest path. "Are these two machines in the same network?" is connectivity. Standard algorithms exist for each.

This topic covers representations, traversals, binary search trees and balance, heaps, tries, and the core graph algorithms, then the production failures that come with them: deep recursion, cycles, unbalanced trees and graphs too large for memory.

## Core Concepts

### Trees

A **binary tree** gives each node at most two children. A **binary search tree** (BST) keeps keys ordered: everything in the left subtree is smaller, everything in the right is larger.

```
        8
       / \
      3   10
     / \    \
    1   6    14
```

Searching a BST follows one path from the root, so the cost is proportional to the tree's height `h`. A balanced tree has `h` of about `log2 n`. An unbalanced one can reach `h = n`: inserting keys in sorted order into a plain BST produces a linked list.

Self-balancing trees (AVL trees, red-black trees) maintain height `O(log n)` using rotations, giving `O(log n)` search, insert and delete in the worst case. **B-trees** hold many keys per node, which suits disk and page-based storage; they underlie many database and file system indexes.

### Traversals

Binary tree traversals visit each node once, `O(n)` time.

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class TreeNode:
    val: int
    left: Optional["TreeNode"] = None
    right: Optional["TreeNode"] = None

def inorder(root: Optional[TreeNode]) -> list[int]:
    out: list[int] = []
    stack: list[TreeNode] = []
    cur = root
    while cur or stack:
        while cur:
            stack.append(cur)
            cur = cur.left
        cur = stack.pop()
        out.append(cur.val)       # visit
        cur = cur.right
    return out
```

In-order traversal of a BST yields keys in sorted order. The iterative form uses an explicit stack, so depth does not depend on the call stack.

| Traversal | Order | Common use |
| --- | --- | --- |
| Pre-order | node, left, right | Copying or serializing a tree |
| In-order | left, node, right | Sorted output from a BST |
| Post-order | left, right, node | Deleting a tree; computing sizes from children |
| Level-order | by depth, using a queue | Shortest path in unweighted trees; layered output |

### Heaps

A **binary heap** is a tree stored in an array where each parent is no larger (min-heap) than its children. It supports peek in `O(1)` and insert and remove-min in `O(log n)`. Building a heap from `n` items takes `O(n)`. It is the standard priority queue.

### Tries

A **trie** (prefix tree) stores strings by character path. Lookup and insert take time proportional to the key length, independent of how many keys are stored. Tries suit autocomplete and prefix matching, at the cost of memory, since each node may hold many child pointers.

### Graph representations

| Representation | Space | Edge check | List neighbors | Best for |
| --- | --- | --- | --- | --- |
| Adjacency list | O(V + E) | O(degree) | O(degree) | Sparse graphs (most real graphs) |
| Adjacency matrix | O(V^2) | O(1) | O(V) | Dense graphs, small V |
| Edge list | O(E) | O(E) | O(E) | Sorting edges; simple input formats |

Graphs may be directed or undirected, weighted or unweighted, cyclic or acyclic. A directed acyclic graph (DAG) models dependencies.

### Graph traversal

**Breadth-first search (BFS)** explores by distance from the start, using a queue. On an unweighted graph it finds the fewest-edge path.

```python
from collections import deque

def shortest_hops(graph: dict[str, list[str]], src: str, dst: str) -> int:
    if src == dst:
        return 0
    dist = {src: 0}
    q = deque([src])
    while q:
        u = q.popleft()
        for v in graph.get(u, []):
            if v not in dist:
                dist[v] = dist[u] + 1
                if v == dst:
                    return dist[v]
                q.append(v)
    return -1   # unreachable
```

Time `O(V + E)`, space `O(V)`.

**Depth-first search (DFS)** follows a path as deep as possible before backtracking, using a stack or recursion. It supports cycle detection, topological sorting and finding connected components.

**Topological sort** orders a DAG so every edge points forward. Kahn's algorithm repeatedly removes nodes with no remaining incoming edges. If some nodes cannot be removed, there is a cycle.

```python
from collections import deque

def topo_order(deps: dict[str, list[str]]) -> list[str] | None:
    # deps[a] lists nodes that depend on a (edge a -> b)
    nodes = set(deps) | {v for vs in deps.values() for v in vs}
    indeg = {n: 0 for n in nodes}
    for vs in deps.values():
        for v in vs:
            indeg[v] += 1
    q = deque(n for n, d in indeg.items() if d == 0)
    order: list[str] = []
    while q:
        u = q.popleft()
        order.append(u)
        for v in deps.get(u, []):
            indeg[v] -= 1
            if indeg[v] == 0:
                q.append(v)
    return order if len(order) == len(nodes) else None   # None means a cycle
```

### Shortest paths

| Algorithm | Handles | Time (typical implementation) |
| --- | --- | --- |
| BFS | Unweighted edges | O(V + E) |
| Dijkstra | Non-negative weights | O((V + E) log V) with a binary heap |
| Bellman-Ford | Negative weights, detects negative cycles | O(V * E) |
| A* | Non-negative weights with a heuristic | Depends on heuristic quality |

Dijkstra with a heap:

```python
import heapq

def dijkstra(graph: dict[str, list[tuple[str, int]]], src: str) -> dict[str, int]:
    dist = {src: 0}
    heap = [(0, src)]
    while heap:
        d, u = heapq.heappop(heap)
        if d > dist.get(u, float("inf")):
            continue                      # stale entry
        for v, w in graph.get(u, []):
            nd = d + w
            if nd < dist.get(v, float("inf")):
                dist[v] = nd
                heapq.heappush(heap, (nd, v))
    return dist
```

Dijkstra is incorrect with negative edge weights. This is a common mistake when weights represent "gains" or adjustments.

### Union-Find

A **disjoint set** (union-find) structure tracks connected components under merges. With path compression and union by rank, each operation is effectively constant time (the inverse Ackermann function, which grows more slowly than any practical input). It is the basis of Kruskal's minimum spanning tree algorithm.

## Trade-offs

| Choice | Gains | Costs |
| --- | --- | --- |
| Plain BST | Simple | Degenerates to `O(n)` height on ordered input |
| Self-balancing tree | Guaranteed `O(log n)`, ordered iteration, range queries | Complex to implement; more work per write |
| Hash map instead of tree | Faster average lookup | No ordering or range queries |
| B-tree | Few page reads per lookup; good for disk | Complex; node splits and merges on writes |
| Adjacency list | Compact for sparse graphs | Edge existence is not `O(1)` |
| Adjacency matrix | `O(1)` edge check | `O(V^2)` memory |
| BFS | Shortest hops; bounded memory per level | Queue can grow wide |
| DFS | Low memory on narrow, deep graphs | Deep recursion; does not find shortest paths |
| Pointer-based trees | Flexible shape | Cache-unfriendly; allocation per node |
| Array-based heap | Compact; cache-friendly | Fixed relationship between index and parent |

For in-memory ordered lookups, a sorted array with binary search or a library ordered map is often a better choice than a hand-built tree. Writing your own balanced tree is rarely justified outside education or a database engine.

## Failure Modes

- **Stack overflow on deep structures.** Recursive DFS on a graph with a long chain, or recursive traversal of a skewed tree or deeply nested user input, exceeds the call stack. Use an explicit stack, limit nesting accepted from untrusted sources and test with a 100,000-deep chain.
- **Infinite loops on cycles.** Traversals that do not track visited nodes loop forever on cyclic graphs. Dependency graphs from real configuration often contain accidental cycles. Track visited state and report cycles with a clear error naming the nodes involved.
- **Unbalanced trees in production.** A plain BST fed sorted, or nearly sorted, keys (timestamps, auto-increment IDs) performs like a list. Use a balanced structure or a hash map.
- **Wrong algorithm for the weights.** Dijkstra with negative edges silently returns wrong answers. BFS on weighted edges returns the fewest hops, not the cheapest route. Check the edge semantics first.
- **Graphs that do not fit in memory.** Social or transaction graphs may have billions of edges. In-memory algorithms fail on them. Consider streaming, partitioning, a graph database or approximate methods, and estimate memory before designing.
- **Combinatorial explosion.** Enumerating all paths in a graph is exponential. "Find all routes" often means "find the best route" in disguise; ask which.
- **Heap with stale entries.** Dijkstra implementations that push duplicates need the stale-entry check shown above, or they redo work.
- **Hidden recursion cost in "nested" data.** Permission systems with group-in-group membership can become slow or loop if cycles are allowed. Enforce acyclicity at write time and cache expanded results with invalidation.
- **Integer overflow in path sums.** Summing weights in fixed-width integers may wrap in languages with overflow. Use wider types and set a sentinel for infinity that cannot overflow when added to.

Detection: property tests that generate random graphs including cycles, self-loops, disconnected nodes and empty graphs; depth and size limits on untrusted input; profiling traversals on production-scale data.

## In Practice

### Decision checklist

1. Is the data hierarchical (tree) or relational (graph)? Can cycles occur?
2. Are edges weighted? Can weights be negative?
3. What question is asked: reachability, shortest path, ordering, components, or minimum spanning tree?
4. Sparse or dense? Pick the representation accordingly.
5. How large can it get, and does it fit in memory?
6. Is recursion depth bounded? If not, iterate.
7. Do you need ordered or range queries? If so, choose an ordered structure over a hash map.

### Worked example: build order with cycle detection

Given packages and their dependencies, `topo_order` above returns a valid build order or `None` when a cycle exists. In a build tool, return the cycle members in the error message. A bare "cycle detected" costs an engineer an hour.

### Verifying AI-generated code

- Does traversal mark nodes visited before enqueueing or pushing, so nodes are not queued many times?
- Is visited state kept across components when scanning a disconnected graph?
- Are weights non-negative where Dijkstra is used?
- Is recursion used on input that can be deep?
- Does the code handle empty graphs, a single node, self-loops and duplicate edges?
- Is a BST claimed to be balanced without any balancing logic?
- Are the stated complexities consistent with the data structure used for the queue or heap?

## Further Reading

- Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest and Clifford Stein, *Introduction to Algorithms*, chapters on trees, graph algorithms and shortest paths.
- Robert Sedgewick and Kevin Wayne, *Algorithms*, chapters on search trees and graphs.
- Steven S. Skiena, *The Algorithm Design Manual*, the graph problem catalog.
- Jon Kleinberg and Eva Tardos, *Algorithm Design*, for graph algorithms and their reasoning.
- Rudolf Bayer and Edward McCreight, "Organization and Maintenance of Large Ordered Indexes" (1972), the original B-tree paper.

Previous: [Linked Lists, Stacks and Queues](./linked-lists-stacks-and-queues.md). Next: [Sorting and Searching](./sorting-and-searching.md).
