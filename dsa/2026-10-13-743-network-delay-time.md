# DSA Interview Walkthrough — LeetCode 743: Network Delay Time

**Difficulty:** Medium  
**Pattern:** Weighted Directed Graph · Shortest Paths · Dijkstra’s Algorithm · Min-Heap  
**Core lesson:** When a signal spreads along weighted directed edges and we need the time when *every* node has received it, first find the **shortest arrival time to each node**. Dijkstra’s algorithm does that efficiently because all travel times are nonnegative. The final answer is the **maximum of those shortest times**, not the sum of edge weights.

## 1. Interviewer: Problem statement

**Interviewer:** You have `n` network nodes labeled `1` through `n`. Each entry `[u, v, w]` in `times` represents a directed edge: a signal takes `w` units of time to travel from `u` to `v`. A signal starts at node `k`. Return the minimum time until **all nodes** receive the signal. If some node is unreachable, return `-1`.

**Example:**

```text
times = [[2,1,1], [2,3,1], [3,4,1]]
n = 4
k = 2

2 ──1──> 1
│
1
↓
3 ──1──> 4

Answer = 2
```

**Candidate:** Are the edges directed? Can there be multiple possible routes to a node? Are weights nonnegative? Does the signal propagate simultaneously along all outgoing edges?

**Interviewer:** Yes. The standard problem uses positive travel times. Signals can travel concurrently, and we care about the earliest arrival at each node.

**Candidate:** So the time to reach a node is the length of its *shortest path* from `k`, and the network finishes when the *last* node receives the signal?

**Interviewer:** Exactly.

## 2. Recognize the problem pattern

Look for **directed edges with weights**, **earliest arrival**, **minimum travel time**, and **one starting node**. Those clues indicate a **single-source shortest-path** problem.

The key distinction is that signals do not have to travel sequentially. If one branch takes 3 units and another takes 7, both can progress at the same time. The network delay is therefore **7**, not `3 + 7`.

Let `dist[v]` be the shortest travel time from `k` to node `v`. Then:

```text
answer = max(dist[1], dist[2], ..., dist[n])
```

If any distance is infinity, the answer is `-1`.

**Interviewer:** Why not use ordinary BFS?

**Candidate:** BFS minimizes the *number of edges*. Here edges have different travel times, so the fewest-edge route may be slower. We need a weighted shortest-path algorithm.

## 3. Brute force: enumerate all simple paths

A straightforward approach is to explore every path from the starting node, recording the smallest total weight seen for each destination.

Because every edge weight is positive, a shortest path never needs to revisit a node: a cycle only increases its total cost. We can therefore enumerate **simple paths** with DFS and a visited set.

```python
def brute(times, n, k):
    graph = [[] for _ in range(n + 1)]
    for u, v, w in times:
        graph[u].append((v, w))

    dist = [float("inf")] * (n + 1)

    def dfs(u, cost, seen):
        dist[u] = min(dist[u], cost)

        for v, w in graph[u]:
            if v not in seen:
                seen.add(v)
                dfs(v, cost + w, seen)
                seen.remove(v)

    dfs(k, 0, {k})
    ans = max(dist[1:])
    return -1 if ans == float("inf") else ans
```

**Time:** Exponential in the number of vertices; a complete directed graph can have factorially many simple paths, giving an `O(n!)` worst-case bound up to polynomial factors.  
**Auxiliary space:** `O(n + E)` for the adjacency list, distances, visited set, and recursion stack, where `E = len(times)`.

**Interviewer:** What is the waste?

**Candidate:** Many paths repeatedly revisit the same intermediate nodes with different costs. I want to retain the best known cost for each node and propagate improvements instead of enumerating every possible route.

## 4. First optimization: Bellman–Ford relaxation

The shortest-path problem has an important local rule called **edge relaxation**.

For an edge `u → v` of weight `w`, if we already know a route to `u` costing `dist[u]`, then we have a candidate route to `v` costing `dist[u] + w`.

Update:

```text
dist[v] = min(dist[v], dist[u] + w)
```

Bellman–Ford repeatedly applies this rule to every edge. A shortest simple path has at most `n - 1` edges, so `n - 1` full passes suffice when there is no reachable negative cycle.

```python
def bellman_ford(times, n, k):
    dist = [float("inf")] * (n + 1)
    dist[k] = 0

    for _ in range(n - 1):
        changed = False

        for u, v, w in times:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                changed = True

        if not changed:
            break

    ans = max(dist[1:])
    return -1 if ans == float("inf") else ans
```

**Time:** `O(nE)`.  
**Auxiliary space:** `O(n)`.

This is a major improvement over enumerating paths, and it works even with negative edge weights if there are no reachable negative cycles. But our weights are positive, so we can do better.

## 5. The key greedy insight behind Dijkstra’s algorithm

Imagine all nodes initially have tentative arrival time infinity, except the source, which has time zero.

We repeatedly choose the **unprocessed node with the smallest tentative arrival time**.

Why is that safe?

Suppose node `u` currently has the smallest tentative distance `d`. Any alternative route that reaches `u` through some still-unprocessed node must first reach that other node. Its tentative distance is at least `d`, and traversing additional nonnegative-weight edges cannot reduce the cost below `d`.

Therefore `d` is already the true shortest distance to `u`. We can finalize it and use its outgoing edges to improve neighboring distances.

This is the greedy foundation of **Dijkstra’s algorithm**.

**Interviewer:** How do we efficiently select the smallest tentative distance?

**Candidate:** Use a **min-heap**, which returns the smallest `(distance, node)` pair. In Python, `heapq` implements a min-heap.

## 6. Derive Dijkstra step by step

Maintain an adjacency list, a distance array, and a min-heap.

**Initialization:** Set `dist[k] = 0`, all other distances to infinity, and push `(0, k)` into the heap.

**Main loop:**

1. Pop `(d, u)`, the smallest available tentative arrival.
2. If `d > dist[u]`, skip this entry: a better route was discovered after it was pushed.
3. For every edge `u → v` with weight `w`, calculate `new_dist = d + w`.
4. If `new_dist < dist[v]`, update `dist[v]` and push `(new_dist, v)`.

**Finish:** If any node still has infinite distance, return `-1`; otherwise return the largest distance.

### Why do stale heap entries appear?

Python's `heapq` has no built-in decrease-key operation. Instead of modifying an existing entry, we push the improved distance. The older, worse entry remains in the heap and is ignored when popped.

For example, we might push `(5, 3)`, later discover a route of length 3 and push `(3, 3)`. The `(3, 3)` entry is processed first. When `(5, 3)` eventually appears, `5 > dist[3]` identifies it as stale.

## 7. Detailed dry run

Consider a slightly richer example:

```text
n = 5
k = 1
times = [
    [1,2,2], [1,3,5],
    [2,3,1], [2,4,4],
    [3,4,1], [3,5,8],
    [4,5,3]
]
```

Initial distances: `[0, ∞, ∞, ∞, ∞]` for nodes 1 through 5.

| Heap pop | What happens | Best distances after processing (1–5) |
|---|---|---|
| `(0,1)` | Discover node 2 at 2 and node 3 at 5 | `[0,2,5,∞,∞]` |
| `(2,2)` | Improve node 3 to 3; discover node 4 at 6 | `[0,2,3,6,∞]` |
| `(3,3)` | Improve node 4 to 4; discover node 5 at 11 | `[0,2,3,4,11]` |
| `(4,4)` | Improve node 5 to 7 | `[0,2,3,4,7]` |
| `(5,3)` | Stale: 5 is worse than `dist[3]=3` | Unchanged |
| `(6,4)` | Stale: 6 is worse than `dist[4]=4` | Unchanged |
| `(7,5)` | Node 5 has no outgoing edges | Unchanged |
| `(11,5)` | Stale: 11 is worse than `dist[5]=7` | Unchanged |

The shortest arrival times are:

```text
node 1: 0
node 2: 2
node 3: 3  (1 → 2 → 3)
node 4: 4  (1 → 2 → 3 → 4)
node 5: 7  (1 → 2 → 3 → 4 → 5)

Answer = max(0, 2, 3, 4, 7) = 7
```

**Interviewer:** Why not return 4, the number of edges on the path to node 5?

**Candidate:** The edge weights represent travel time. That path takes `2 + 1 + 1 + 3 = 7` units.

## 8. Clean final LeetCode solution — Python

```python
import heapq

class Solution:
    def networkDelayTime(self, times: list[list[int]], n: int, k: int) -> int:
        graph = [[] for _ in range(n + 1)]
        for u, v, w in times:
            graph[u].append((v, w))

        dist = [float("inf")] * (n + 1)
        dist[k] = 0
        heap = [(0, k)]

        while heap:
            d, u = heapq.heappop(heap)
            if d > dist[u]:
                continue

            for v, w in graph[u]:
                nd = d + w
                if nd < dist[v]:
                    dist[v] = nd
                    heapq.heappush(heap, (nd, v))

        ans = max(dist[1:])
        return -1 if ans == float("inf") else ans
```

The code is concise, uses no lambda functions, handles directed edges correctly, and does not modify the input.

## 9. Complexity analysis

Let `V = n` and `E = len(times)`.

**Graph construction:** `O(V + E)` time and space.

**Dijkstra:** Every successful relaxation may push a heap entry. With lazy deletion, there are `O(E + 1)` heap entries overall, and each push/pop costs `O(log(E + 1))`. Processing outgoing edges across non-stale pops costs `O(E)`.

**Total time:** `O(V + E log(E + 1))` for a general input with edges, often written `O((V + E) log V)` for a simple graph.  
**Auxiliary space:** `O(V + E)` for the graph, distance array, and heap.

This is the standard practical choice for single-source shortest paths with nonnegative weights.

## 10. Correctness proof

**Invariant:** `dist[v]` is always the length of some discovered path from `k` to `v`, or infinity if no path has yet been found. It never increases.

**Base case:** `dist[k] = 0` is the shortest distance to the source. Other nodes have not been reached.

**Greedy step:** Suppose `(d,u)` is the smallest non-stale heap entry. Assume there were a shorter path to `u` than `d`. On that path, consider the first vertex not yet finalized and the finalized vertex immediately before it. Relaxing their connecting edge would have placed a tentative distance no larger than the true path prefix into the heap. Since all weights are nonnegative, that tentative distance would be smaller than `d`, contradicting the choice of `u` as the minimum. Therefore `d` is the shortest possible distance to `u`.

Relaxing all outgoing edges preserves the invariant and exposes candidate shortest paths to neighbors. By induction, every reachable node eventually receives its true shortest distance. Unreachable nodes remain at infinity.

The signal reaches every node as soon as its shortest path has elapsed, and all edges can transmit concurrently. Therefore the time until all nodes receive the signal is the maximum finite shortest-path distance. If any node is unreachable, the required answer is `-1`.

## 11. Edge cases and common mistakes

**Single node, no edges:** Return `0`; the source has already received the signal.

**Unreachable node:** Return `-1`, even if every other node is reachable.

**Different routes to the same node:** Keep the minimum total weight, not the first route discovered.

**Directed edges:** An edge `u → v` does **not** imply `v → u`.

**Cycles:** Positive-weight cycles do not cause infinite processing; only strictly improving distances are pushed.

**Mistake — mark a node visited when first pushed:** A better route may be discovered before it is popped. Finalization happens when the minimum non-stale entry is popped.

**Mistake — use a FIFO queue for arbitrary positive weights:** BFS is correct for unit-weight edges, not general weighted edges.

**Mistake — add all shortest distances:** Signals spread concurrently, so take the maximum shortest distance.

**Mistake — use the largest direct edge weight:** A shortest path can involve multiple edges; the delay depends on full routes.

**Mistake — include index 0 in the maximum:** Nodes are labeled 1 through n; index 0 is unused and remains infinity.

**Mistake — apply Dijkstra to negative weights:** Its greedy finalization proof relies on nonnegative edge weights.

## 12. Interviewer-style follow-up questions

**Q1. Why is Dijkstra greedy?**

Once the smallest tentative distance is popped, no route through unprocessed vertices can improve it when edge weights are nonnegative. That local choice is permanently safe.

**Q2. When would BFS be better?**

If every edge has weight 1, BFS computes shortest paths in `O(V+E)` time without a heap. If weights are only 0 or 1, **0–1 BFS** with a deque is another linear-time option.

**Q3. What if edge weights can be negative?**

Use Bellman–Ford for general single-source shortest paths, assuming no reachable negative cycle. It takes `O(VE)` time and can detect negative cycles with an additional relaxation pass.

**Q4. How do you return the actual shortest path to a node?**

Keep `parent[v]`. Whenever relaxing `u → v` improves `dist[v]`, set `parent[v] = u`. Afterward, follow parent pointers backward from the destination to the source and reverse the sequence.

**Q5. What if I need shortest paths between every pair of nodes?**

For small graphs, Floyd–Warshall is a simple `O(V³)` dynamic programming approach. Repeating Dijkstra from every source can be better for sparse graphs with nonnegative weights.

**Q6. Why use `(distance, node)` in the heap rather than just node IDs?**

The heap must prioritize **earliest tentative arrival**, not smallest node label. Python compares tuples lexicographically, so the distance determines priority.

**Q7. What if multiple edges connect the same ordered pair?**

The adjacency list can contain them all. Relaxation naturally keeps whichever route yields the smallest distance.

**Q8. Could we stop Dijkstra early?**

If we only needed the shortest path to one destination, we could stop when that destination is popped non-stale. Here we need every node's shortest arrival, so we generally continue until the heap is empty or all reachable nodes are finalized.

**Q9. Is the answer the longest path from the source?**

No. It is the **largest among shortest-path distances**. A long detour to a node does not delay the signal if a faster route exists.

## 13. What to say aloud in the interview

"This is a single-source shortest-path problem on a directed, positively weighted graph. The earliest arrival at each node is its shortest distance from the starting node, and the network delay is the maximum of those distances. Brute-force DFS could enumerate simple paths, but that is exponential. Bellman–Ford improves this to O(VE) through repeated edge relaxation. Since all weights are nonnegative, Dijkstra is more efficient. I build an adjacency list, initialize all distances to infinity except the source, and use a min-heap of distance-node pairs. Each time I pop the smallest non-stale distance, I relax its outgoing edges and push improvements. Finally, if any node is unreachable I return -1; otherwise I return the maximum shortest distance. With a heap, the solution is near O((V+E) log V) time and O(V+E) space."

## 14. Pattern takeaway and practice

```text
Directed graph + weighted travel times + earliest arrival
                         ↓
Single-source shortest paths
                         ↓
Brute force: enumerate simple paths (exponential)
                         ↓
Bellman–Ford: repeatedly relax all edges, O(VE)
                         ↓
Nonnegative weights enable greedy finalization
                         ↓
Dijkstra: min-heap + relax outgoing edges
                         ↓
Skip stale heap entries
                         ↓
Network delay = maximum shortest distance
                         ↓
Any infinity? Return -1
```

**Practice next:** LeetCode 1631 (Path With Minimum Effort) for a modified Dijkstra cost function; 1514 (Path With Maximum Probability) for max-product paths; 787 (Cheapest Flights Within K Stops) for shortest paths with an additional stop constraint; and 1368 (Minimum Cost to Make at Least One Valid Path in a Grid) for 0–1 BFS.

**Transferable interview insight:** First identify *what quantity each path minimizes*. Then ask whether the edge costs allow a greedy shortest-path algorithm. Dijkstra works when extending a path cannot make its cost smaller; when additional constraints or negative weights break that property, revisit the state definition or choose a different algorithm.
