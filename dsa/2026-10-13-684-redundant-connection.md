# DSA Interview Walkthrough — LeetCode 684: Redundant Connection

**Difficulty:** Medium  
**Pattern:** Undirected Graph · Cycle Detection · Union-Find / DSU  
**Core lesson:** In an undirected graph, a new edge creates a cycle exactly when its endpoints are already connected.

## 1. Interviewer prompt

A graph started as a tree on nodes `1..n`, then one extra undirected edge was added. Return the edge that can be removed to make the graph a tree again. If several answers are possible, return the one appearing last in the input.

```text
edges = [[1,2], [1,3], [2,3]]
answer = [2,3]
```

## 2. Recognize the pattern

A tree has one unique path between any two vertices. Adding an edge `(u,v)` creates a cycle iff `u` and `v` were already connected.

So process edges in order and ask only one question each time:

> Are `u` and `v` already in the same connected component?

That is exactly what Disjoint Set Union is designed to answer.

## 3. Brute force with DFS

Keep an adjacency list of accepted edges. Before inserting `(u,v)`, run DFS/BFS to test whether `v` is already reachable from `u`.

```python
def brute(edges):
    n = len(edges)
    graph = [[] for _ in range(n + 1)]

    def connected(start, target):
        stack = [start]
        seen = set()

        while stack:
            node = stack.pop()
            if node == target:
                return True
            if node in seen:
                continue
            seen.add(node)
            stack.extend(graph[node])

        return False

    for u, v in edges:
        if connected(u, v):
            return [u, v]
        graph[u].append(v)
        graph[v].append(u)

    return []
```

Worst-case time is `O(n^2)` because each new edge may trigger a traversal of the growing graph.

## 4. Union-Find idea

DSU stores each connected component as a parent-pointer tree.

Operations:
- `find(x)` returns the component representative.
- `union(a,b)` merges two components.

If `find(u) == find(v)`, the edge closes a cycle and is redundant.

## 5. Path compression + union by size

Path compression flattens parent chains during `find`.

```python
def find(x):
    while parent[x] != x:
        parent[x] = parent[parent[x]]
        x = parent[x]
    return x
```

Union by size attaches the smaller component beneath the larger one.

Together, DSU operations take near-constant amortized time.

## 6. Clean final Python solution

```python
class Solution:
    def findRedundantConnection(self, edges: list[list[int]]) -> list[int]:
        n = len(edges)
        parent = list(range(n + 1))
        size = [1] * (n + 1)

        def find(x):
            while parent[x] != x:
                parent[x] = parent[parent[x]]
                x = parent[x]
            return x

        for u, v in edges:
            a, b = find(u), find(v)

            if a == b:
                return [u, v]

            if size[a] < size[b]:
                a, b = b, a

            parent[b] = a
            size[a] += size[b]

        return []
```

No lambda functions are used.

## 7. Complexity

- Time: `O(n α(n))` amortized.
- Auxiliary space: `O(n)`.

`α(n)` is the inverse Ackermann function and is effectively constant for practical inputs.

## 8. Correctness

After every accepted edge, DSU components match graph connectivity. If a new edge joins different roots, it connects two separate components and cannot create a cycle. If both endpoints already share a root, a path between them already exists, so adding the edge creates a cycle. Processing edges in order returns the last edge that completes the unique cycle.

## 9. Edge cases and common mistakes

- Triangle graph.
- Long cycle.
- Node labels start at 1, so arrays need size `n+1`.

Common mistakes:
- checking immediate parents instead of roots,
- unioning before testing whether roots match,
- connecting arbitrary nodes instead of component roots,
- using DSU unchanged for directed-cycle problems.

## 10. Interviewer follow-ups

**Return the whole cycle?** DSU only detects closure. Recover the existing path with DFS/BFS and add the redundant edge.

**Multiple extra edges?** Continue processing and collect every edge whose endpoints are already connected.

**Directed graph?** Use directed-cycle techniques such as DFS coloring or topological sorting.

**Union by rank vs size?** Both keep trees shallow; size tracks component cardinality, rank approximates height.

## 11. What to say aloud

> “A new undirected edge creates a cycle exactly when its endpoints are already connected. DFS before every insertion works but repeats connectivity work. Union-Find stores components directly. For each edge I find both roots; if they match, this edge closes the cycle and is the answer. Otherwise I union the roots. Path compression and union by size make the total runtime effectively linear.”

## 12. Pattern takeaway

```text
Edges arrive one by one
        ↓
Need repeated connectivity checks
        ↓
DFS each time: O(n²)
        ↓
Union-Find components
        ↓
Same root? cycle edge
        ↓
O(n α(n))
```

**Practice next:** LeetCode 547, 721, 1319, and 1584.