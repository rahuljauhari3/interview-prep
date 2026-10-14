# DSA Interview Walkthrough — LeetCode 1192: Critical Connections in a Network

**Difficulty:** Hard  
**Pattern:** Undirected Graph · DFS · Tarjan's Bridge Algorithm · Discovery Time / Low-Link Values  
**Core lesson:** An edge is a bridge if removing it disconnects a previously connected part of the graph. Rather than removing every edge and rerunning graph traversal, one DFS can identify all bridges by checking whether a child's subtree has any alternate path back to an ancestor.

## 1. Interviewer: Problem statement

**Interviewer:** You have `n` servers numbered `0` through `n-1`, connected by undirected links. A connection is **critical** if removing it disconnects some servers that were previously connected. Return all critical connections, in any order.

For the standard problem, the network is connected, and each undirected edge appears once.

~~~text
n = 4
connections = [[0,1], [1,2], [2,0], [1,3]]

    0
   / \
  1---2
  |
  3

Answer: [[1,3]]
~~~

**Candidate:** Are links undirected? Can the network contain cycles? Are duplicate links possible?

**Interviewer:** Undirected, yes to cycles, and no duplicate links under the problem constraints.

**Candidate:** If I remove the link between 1 and 3, server 3 becomes isolated. But removing an edge of the triangle still leaves another route, so those edges are not critical.

**Interviewer:** Correct. Start with a simple solution, then optimize it.

## 2. Recognize the problem pattern

The important clues are **undirected graph**, **remove one edge**, and **does connectivity change?**

This is the definition of a **bridge** in graph theory.

A useful first observation:

> An undirected edge is a bridge exactly when it does not belong to any cycle.

Why? If the edge is part of a cycle, the rest of the cycle provides another route between its endpoints. If there is no alternative route, removing the edge separates the endpoints.

This suggests DFS, because DFS naturally organizes edges into a tree and reveals whether a subtree has a back edge to an ancestor.

The special technique is **Tarjan's bridge-finding algorithm**, using discovery times and low-link values.

## 3. Brute force: remove each edge and check connectivity

For every connection, temporarily omit it, rebuild the graph, and count connected components. If the number of components increases, that edge is critical.

~~~python
def brute(n, connections):
    def components(skip):
        graph = [[] for _ in range(n)]
        for i, (u, v) in enumerate(connections):
            if i == skip:
                continue
            graph[u].append(v)
            graph[v].append(u)

        seen = set()
        count = 0

        for start in range(n):
            if start in seen:
                continue

            count += 1
            stack = [start]
            seen.add(start)

            while stack:
                u = stack.pop()
                for v in graph[u]:
                    if v not in seen:
                        seen.add(v)
                        stack.append(v)

        return count

    original = components(-1)
    ans = []

    for i, edge in enumerate(connections):
        if components(i) > original:
            ans.append(edge)

    return ans
~~~

Let `V = n` and `E = len(connections)`.

**Time:** `O(E(V+E))`, because each of E removals rebuilds and traverses a graph.  
**Auxiliary space:** `O(V+E)` for the temporary adjacency list, visited set, and DFS stack.

**Interviewer:** What is the repeated work?

**Candidate:** I traverse almost the same graph E times. A single DFS should be able to tell me whether each tree edge has an alternate route around it.

## 4. First optimization: build a DFS tree

Run DFS from a starting vertex. Every edge used to discover a new vertex becomes a **tree edge**. An edge from a vertex to an already discovered ancestor is a **back edge**.

Consider this graph:

~~~text
    0
   / \
  1---2
  |
  3
 / \
4---5
~~~

There are two cycles: `0-1-2-0` and `3-4-5-3`. The link `1-3` is the only bridge.

When DFS enters the subtree rooted at 3 through edge `1-3`, the question becomes:

> Can any vertex in the subtree of 3 reach vertex 1 or an ancestor of 1 without using edge 1-3?

If yes, there is an alternate route and the edge is not a bridge. If no, removing it disconnects that entire subtree.

We need a compact way to answer this for every DFS tree edge.

## 5. The special concept: discovery time and low-link

For every vertex `u`, maintain:

**`tin[u]` (discovery time):** The order in which DFS first visits `u`.

**`low[u]` (low-link value):** The smallest discovery time reachable from `u` or its DFS descendants by traveling down zero or more DFS-tree edges and then using at most one back edge.

When visiting `u`, initialize:

~~~text
tin[u] = low[u] = current_time
~~~

For each neighbor `v`:

**Case A — `v` is the parent:** Ignore the undirected edge back to the parent. It is the same tree edge we used to enter `u`, not an alternate path.

**Case B — `v` is unvisited:** Explore `v`, then propagate what its subtree learned:

~~~text
low[u] = min(low[u], low[v])
~~~

**Case C — `v` is already visited:** This edge can provide an alternate route to an ancestor:

~~~text
low[u] = min(low[u], tin[v])
~~~

For a DFS tree edge `u -> v`, the decisive test is:

~~~text
if low[v] > tin[u]:
    (u, v) is a bridge
~~~

**Why strictly greater?** If `low[v] <= tin[u]`, the subtree of `v` can reach `u` or an ancestor through a back edge. The tree edge is part of a cycle and can be removed without separating the subtree. If `low[v] > tin[u]`, no such alternate connection exists.

**Interviewer:** Why compare `low[v]` with `tin[u]`, not `low[u]`?

**Candidate:** The question is whether the *child's subtree* can reach the parent or above without the tree edge. `low[u]` might be small because of a completely different branch, which does not rescue the child.

## 6. Detailed dry run

Use:

~~~text
n = 6
connections = [
    [0,1], [1,2], [2,0],
    [1,3],
    [3,4], [4,5], [5,3]
]
~~~

Assume DFS visits vertices in this order: `0 -> 1 -> 2`, backtracks, then `1 -> 3 -> 4 -> 5`.

| Vertex | `tin` | Final `low` | Reason |
|---|---:|---:|---|
| 0 | 0 | 0 | DFS root |
| 1 | 1 | 0 | Its child 2 can reach ancestor 0 |
| 2 | 2 | 0 | Back edge 2 → 0 |
| 3 | 3 | 3 | Its subtree cannot reach vertex 1 or above |
| 4 | 4 | 3 | Its child 5 can reach ancestor 3 |
| 5 | 5 | 3 | Back edge 5 → 3 |

Check each tree edge when returning from its child:

- `1 -> 2`: `low[2] = 0 <= tin[1] = 1`. Not a bridge.
- `4 -> 5`: `low[5] = 3 <= tin[4] = 4`. Not a bridge.
- `3 -> 4`: `low[4] = 3 <= tin[3] = 3`. Not a bridge.
- `1 -> 3`: `low[3] = 3 > tin[1] = 1`. **Bridge.**
- `0 -> 1`: `low[1] = 0 <= tin[0] = 0`. Not a bridge.

**Answer:** `[[1,3]]`.

The important distinction is that a back edge inside the `3-4-5` cycle lowers those vertices' low-link values to 3, **not** to 1. That cycle does not provide a route back to the left side of the graph.

## 7. From recursive DFS to a practical Python solution

The textbook version uses recursive DFS: record `tin` and `low` on entry, recurse into unvisited neighbors, and evaluate a tree edge when the recursive call returns.

That is elegant, but this problem allows up to 100,000 vertices. A long chain can exceed Python's recursion limit. Raising the recursion limit very high may still risk exhausting the interpreter's native call stack.

For a robust Python implementation, simulate DFS with an explicit stack. Each stack frame stores:

~~~text
(node, parent, next_neighbor_index)
~~~

The frame remains on the stack while we explore its children. When all neighbors have been processed, popping the frame is equivalent to **returning from a recursive DFS call**. That is the moment to propagate `low[child]` to its parent and check the bridge condition.

## 8. Clean final LeetCode solution — Python

~~~python
class Solution:
    def criticalConnections(self, n: int, connections: list[list[int]]) -> list[list[int]]:
        graph = [[] for _ in range(n)]
        for u, v in connections:
            graph[u].append(v)
            graph[v].append(u)

        tin = [-1] * n
        low = [0] * n
        time = 0
        ans = []

        for root in range(n):
            if tin[root] != -1:
                continue

            stack = [(root, -1, 0)]

            while stack:
                u, parent, i = stack[-1]

                if tin[u] == -1:
                    tin[u] = low[u] = time
                    time += 1

                if i == len(graph[u]):
                    stack.pop()
                    if parent != -1:
                        low[parent] = min(low[parent], low[u])
                        if low[u] > tin[parent]:
                            ans.append([parent, u])
                    continue

                v = graph[u][i]
                stack[-1] = (u, parent, i + 1)

                if v == parent:
                    continue
                if tin[v] == -1:
                    stack.append((v, u, 0))
                else:
                    low[u] = min(low[u], tin[v])

        return ans
~~~

This is an iterative version of Tarjan's bridge algorithm. It uses **no lambda functions**, avoids recursion-depth problems, and handles disconnected graphs as well, although the original problem guarantees connectivity.

The `v == parent` check relies on the problem's guarantee that there are no duplicate undirected edges. If parallel edges are allowed, track an **edge ID** and skip only the exact edge used to enter the current vertex.

## 9. Complexity analysis

Each vertex is discovered once. Every undirected edge appears twice in adjacency lists and is examined at most twice. Each DFS frame is pushed once, updated as its neighbors are processed, and popped once.

**Time:** `O(V+E)`.  
**Auxiliary space:** `O(V+E)` for the adjacency list, `tin`, `low`, and the explicit DFS stack. The answer can contain up to E edges.

Compared with `O(E(V+E))` brute force, this is a major improvement: we obtain **every bridge in one traversal**.

## 10. Correctness proof

Consider a DFS tree edge `(u,v)`, where `u` is the parent of `v`.

In an undirected DFS tree, an edge from the subtree rooted at `v` to a vertex outside that subtree must connect to an ancestor. There cannot be a cross edge to an unrelated completed subtree: DFS would have discovered that endpoint while exploring the first subtree.

By construction, `low[v]` is the earliest discovery time reachable from the subtree of `v` using tree edges and possibly a back edge.

**If `low[v] <= tin[u]`:** Some vertex in `v`'s subtree can reach `u` or an ancestor of `u` without traversing tree edge `(u,v)` in reverse. This forms an alternate route, so deleting `(u,v)` does not disconnect the graph. The edge is not a bridge.

**If `low[v] > tin[u]`:** No vertex in the subtree can reach `u` or an ancestor through another edge. The tree edge `(u,v)` is the only connection from that subtree to the rest of the DFS tree. Removing it disconnects the subtree, so it is a bridge.

The algorithm computes the same low-link recurrence as recursive DFS, because it updates the parent's low value only after the child's frame finishes. It tests every DFS tree edge exactly once. A non-tree edge belongs to a cycle formed with the DFS tree, so it cannot be a bridge.

Therefore, the algorithm returns exactly all critical connections.

## 11. Edge cases and common mistakes

**A simple chain:** `0-1-2-3`. Every edge is a bridge.

**A simple cycle:** `0-1-2-0`. No edge is a bridge.

**Two cycles connected by one edge:** Only the connecting edge is a bridge.

**One vertex:** There are no edges, so the result is empty.

**Disconnected graph:** Run DFS from each unvisited vertex. A bridge is defined relative to the original component; the implementation handles this.

**Common mistakes:**

- **Treating the graph as directed.** Add both `u -> v` and `v -> u`.
- **Using `low[v] >= tin[u]` for bridges.** Equality means a back edge reaches `u`, so the correct condition is strictly `>`.
- **Updating `low[u]` with `low[v]` for an already visited neighbor.** For a back edge, use `tin[v]`; child low values are propagated only after exploring a DFS-tree child.
- **Treating the parent edge as a back edge.** That would incorrectly make tree edges appear to have an alternate path.
- **Checking a tree edge before its child finishes.** Its low-link value is not final yet.
- **Confusing bridges with articulation points.** Bridges are edges whose removal disconnects; articulation points are vertices.
- **Assuming a nested loop means quadratic time.** Across all frames, each adjacency entry is processed once.

## 12. Interviewer-style follow-up questions

**Q1. Why is the bridge condition `low[child] > tin[parent]` rather than `>=`?**

Equality means the child's subtree can reach the parent through a back edge. That alternate route keeps the graph connected if the tree edge is removed. Only a strictly larger low-link value proves there is no alternate route.

**Q2. What is the difference between a bridge and an articulation point?**

A bridge is a critical **edge**; an articulation point is a critical **vertex**. Both can be found using DFS discovery times and low-link values, but articulation points use different conditions and require special handling for the DFS root.

**Q3. Can Union-Find solve this efficiently?**

Union-Find efficiently answers connectivity as edges are added, but it does not directly identify every edge whose deletion disconnects the graph. Rebuilding DSU after each candidate deletion is still expensive. DFS low-link is the right static-graph technique.

**Q4. Why is every non-tree edge safe from being a bridge?**

In an undirected DFS, a non-tree edge joins a vertex to an ancestor. Together with the DFS-tree path between them, it forms a cycle. Removing one edge of a cycle leaves another route.

**Q5. How would the code change if parallel edges were allowed?**

Give every undirected edge a unique ID, store `(neighbor, edge_id)` in both adjacency lists, and skip only the incoming edge ID. A second edge to the parent must count as a valid alternate connection.

**Q6. Why not simply increase Python's recursion limit?**

It can work for many inputs, but a chain of 100,000 vertices creates very deep recursion. An explicit stack is more robust and avoids reliance on the interpreter's native call-stack capacity.

**Q7. Can this technique find all 2-edge-connected components?**

Yes. First identify all bridges. Remove those bridges conceptually, then find connected components in the remaining graph. Vertices within each such component remain connected after deleting any single edge of that component.

**Q8. What if edges are continually added and removed?**

This is a dynamic connectivity/bridge-maintenance problem. Recomputing bridges with Tarjan after each update is the simple baseline, but specialized dynamic graph data structures are needed for efficient frequent updates.

## 13. What to say aloud in the interview

> “This is an undirected graph, and a critical connection is exactly a bridge. Brute force would remove each edge and rerun DFS to see whether the number of connected components increases, costing O(E(V+E)). To optimize, I'll run one DFS and assign each vertex a discovery time and a low-link value. The low-link value tells me how far back toward the DFS root a subtree can reach through a back edge. For a tree edge from parent u to child v, if low[v] is greater than tin[u], the child's subtree has no alternate route to u or above, so the edge is a bridge. Each vertex and edge is processed a constant number of times, giving O(V+E) time and space. I'll use an explicit stack in Python to avoid recursion-depth issues on long chains.”

## 14. Pattern takeaway and practice

~~~text
Remove one edge -> does connectivity change?
                    ↓
Undirected graph bridge problem
                    ↓
Brute force: remove edge + DFS each time
                    ↓
DFS tree + back edges
                    ↓
tin[u] = discovery time
low[u] = earliest ancestor reachable from subtree
                    ↓
For each tree edge u -> v:
low[v] > tin[u] ?
                    ↓
Yes: bridge     No: alternate route exists
                    ↓
All bridges in O(V+E)
~~~

**Practice next:** LeetCode 1568 (Minimum Number of Days to Disconnect Island) for articulation-style reasoning; 684 (Redundant Connection) to contrast cycles and Union-Find; 1192 again with an explicit recursive implementation; and 1489 (Find Critical and Pseudo-Critical Edges in Minimum Spanning Tree) to distinguish graph bridges from critical MST edges.

**Transferable interview insight:** When an edge-deletion question seems to require repeatedly recomputing connectivity, ask whether DFS can summarize the alternate routes of entire subtrees. Discovery times and low-link values compress those repeated connectivity checks into one linear-time traversal.
