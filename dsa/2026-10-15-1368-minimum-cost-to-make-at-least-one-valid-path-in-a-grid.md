# DSA Interview Walkthrough — LeetCode 1368: Minimum Cost to Make at Least One Valid Path in a Grid

**Difficulty:** Hard  
**Pattern:** Graph shortest paths · Dijkstra · 0-1 BFS · Deque  
**Core lesson:** When every edge costs **0 or 1**, a deque can replace Dijkstra's heap. Put free moves at the front and paid moves at the back to obtain linear-time shortest paths.

## 1. Interviewer: Problem statement

**Interviewer:** You have an m × n grid. Each cell contains an arrow: **1 = right, 2 = left, 3 = down, 4 = up**. Starting at the top-left cell, reach the bottom-right by following arrows. You can change a cell's arrow for a cost of 1. Find the minimum number of changes needed to create at least one valid path.

**Candidate:** Can an arrow point outside the grid? Do we need to make every cell lead to the destination?

**Interviewer:** Arrows may point outside, and only one valid route is required. A cell can be changed at most once.

Example:

~~~text
grid = [[1, 2],
        [4, 3]]
answer = 1
~~~

Follow (0,0) → (0,1) for free, then change the arrow at (0,1) from left to down. You reach (1,1) with one change.

## 2. Recognize the pattern

The clues are **minimum cost**, **moving between grid cells**, and **different costs for different moves**. This suggests a weighted graph shortest-path problem.

Treat each cell as a vertex. From (r,c), create directed edges to its in-bounds neighbors. The edge cost is:

~~~text
0 if the move follows grid[r][c]'s arrow
1 otherwise
~~~

The cost depends on the **source** cell. Moving A → B and B → A may have different costs.

**Interviewer:** Why is this graph model valid when a cell may only be changed once?

**Candidate:** An optimal route can be chosen without repeating cells, since removing cycles never increases a nonnegative path cost. On a simple route, each visited cell has only one outgoing move, so a mismatched arrow requires exactly one change.

The special clue is that **all edge weights are 0 or 1**. This is precisely the setting for **0-1 BFS**.

## 3. Brute force: enumerate all simple routes

Try every route from the start to the destination. Maintain visited cells to avoid cycles, and add 0 or 1 for each chosen move.

~~~python
def brute(grid):
    m, n = len(grid), len(grid[0])
    dirs = ((0, 1), (0, -1), (1, 0), (-1, 0))
    seen = set()
    best = float("inf")

    def dfs(r, c, cost):
        nonlocal best
        if cost >= best:
            return
        if (r, c) == (m - 1, n - 1):
            best = cost
            return

        seen.add((r, c))
        for d, (dr, dc) in enumerate(dirs, 1):
            nr, nc = r + dr, c + dc
            if 0 <= nr < m and 0 <= nc < n and (nr, nc) not in seen:
                dfs(nr, nc, cost + (grid[r][c] != d))
        seen.remove((r, c))

    dfs(0, 0, 0)
    return best
~~~

Let V = mn. A simple path visits at most V vertices, with at most four choices per step. **O(4^V)** is a loose worst-case time bound; pruning does not eliminate exponential growth. The visited set and recursion stack use **O(V)** auxiliary space.

**Interviewer:** What's being recomputed?

**Candidate:** Different routes reach the same cell at different costs. Instead of exploring every complete route, I should maintain the cheapest cost to each cell.

## 4. First optimization: Dijkstra

Define **dist[r][c]** as the minimum known cost to reach a cell. Initialize the start at 0 and all other distances at infinity.

Dijkstra uses a min-heap to process the currently cheapest cell. For every neighboring direction, calculate:

~~~text
new_cost = dist[r][c] + (grid[r][c] != direction)
~~~

If this improves the neighbor's distance, update it and push it into the heap. Skip stale heap entries.

There are V = mn vertices and E ≤ 4mn directed edges, so Dijkstra takes **O(mn log(mn)) time** and **O(mn) space**.

**Interviewer:** Can you remove the logarithmic factor?

**Candidate:** Yes. A heap handles arbitrary nonnegative edge weights, but ours are only 0 and 1. A deque is enough.

## 5. The key optimization: 0-1 BFS

Ordinary BFS uses a FIFO queue because all edges have equal weight. Dijkstra uses a priority queue because edge costs vary.

In **0-1 BFS**, use a double-ended queue:

- If an edge costs **0**, put the improved neighbor at the **front**.
- If an edge costs **1**, put the improved neighbor at the **back**.

Why? Processing a cell at distance d can generate only candidates at distance d or d+1. Front insertion prioritizes the d candidates; back insertion delays the d+1 candidates. This preserves the distance ordering needed by shortest-path algorithms.

Maintain a distance matrix and relax only when **new_cost < dist[neighbor]**. Store the candidate cost in each deque entry and skip stale entries if a cheaper path has since been found.

**Important:** Do not mark a cell permanently visited when first enqueuing it. Its distance may improve before processing.

## 6. Detailed dry run

Consider:

~~~text
grid:
1 1 3
3 2 2
1 1 4
~~~

The arrows create this free route:

~~~text
(0,0) → (0,1) → (0,2)
                    ↓
(1,0) ← (1,1) ← (1,2)
  ↓
(2,0) → (2,1) → (2,2)
~~~

Each move follows its source cell's arrow, so every edge costs 0. Zero-cost neighbors go to the deque's front, and the destination receives **distance 0**. No solution can cost less.

For a case requiring a change:

~~~text
grid = [[1,2],
        [4,3]]
~~~

| Processed cell | Cost | Relevant update |
|---|---:|---|
| (0,0) | 0 | Right gets cost 0; down gets cost 1 |
| (0,1) | 0 | Down to destination gets cost 1 |
| (1,0) | 1 | Cannot improve destination |
| (1,1) | 1 | Return **1** |

Equal-cost entries may be processed in a different order without changing the result.

## 7. Clean final LeetCode solution — Python

~~~python
from collections import deque

class Solution:
    def minCost(self, grid: list[list[int]]) -> int:
        m, n = len(grid), len(grid[0])
        dirs = ((0, 1), (0, -1), (1, 0), (-1, 0))
        dist = [[float("inf")] * n for _ in range(m)]
        dist[0][0] = 0
        q = deque([(0, 0, 0)])

        while q:
            cost, r, c = q.popleft()
            if cost != dist[r][c]:
                continue
            if (r, c) == (m - 1, n - 1):
                return cost

            for d, (dr, dc) in enumerate(dirs, 1):
                nr, nc = r + dr, c + dc
                if not (0 <= nr < m and 0 <= nc < n):
                    continue

                weight = int(grid[r][c] != d)
                new_cost = cost + weight
                if new_cost >= dist[nr][nc]:
                    continue

                dist[nr][nc] = new_cost
                if weight == 0:
                    q.appendleft((new_cost, nr, nc))
                else:
                    q.append((new_cost, nr, nc))
~~~

No lambda functions, no heap, and no input modification. A route can always be created by changing arrows, so the function returns a minimum cost.

## 8. Complexity analysis

The graph contains V = mn cells and E ≤ 4mn directed edges. In 0-1 BFS, deque ordering finalizes shortest distances; each edge is considered a constant number of times, and stale entries are skipped.

**Time:** **O(V+E) = O(mn)**.  
**Auxiliary space:** **O(V) = O(mn)** for the distance matrix and deque.

| Approach | Time | Space |
|---|---|---|
| Exhaustive DFS | Exponential | O(mn) |
| Dijkstra with heap | O(mn log(mn)) | O(mn) |
| **0-1 BFS** | **O(mn)** | **O(mn)** |

## 9. Correctness proof

**Graph equivalence:** On a simple path, each cell is left exactly once. A move that follows its arrow needs no change; any other move needs one change. Thus the path's total edge weight equals its required modifications. Any successful arrow configuration yields a route to the target, and cycles can be removed without increasing cost. Therefore the original minimum equals the graph's shortest-path distance.

**Deque invariant:** When processing a non-stale entry at distance d, a zero-weight relaxation produces a candidate at d and goes to the front, while a one-weight relaxation produces d+1 and goes to the back. The deque therefore processes useful entries in nondecreasing distance order, like Dijkstra.

When a cell is removed from the front with its current best distance, no undiscovered route can be cheaper: such a route would pass through an earlier cheaper entry. The cell's distance is final. When the destination is finalized, its distance is the shortest path and hence the minimum arrow-change cost.

## 10. Edge cases and common mistakes

**Single cell:** [[4]] → 0. Start and destination coincide.

**One row of right arrows:** Cost 0.

**Out-of-bounds arrows:** Ignore moves outside the grid. Changing the arrow can still allow an in-bounds move.

**Free cycles:** Equal-distance relaxations are rejected, preventing infinite cycling.

**Multiple optimal routes:** Only the minimum cost is required.

Common mistakes: using ordinary BFS despite unequal weights; charging for the destination cell's arrow rather than the source's; pushing both cost types to the same deque end; marking visited on enqueue; forgetting stale-entry checks; and changing the destination's arrow unnecessarily.

## 11. Interviewer-style follow-up questions

**Q1. Why not ordinary BFS?** BFS minimizes number of moves, not modification cost. A longer free route can beat a shorter paid route.

**Q2. Is Dijkstra incorrect?** No. It is correct but has an unnecessary logarithmic factor.

**Q3. What if every arrow change costs 5?** Divide all costs by 5, solve the equivalent 0/1 problem, and multiply the result by 5.

**Q4. What if weights can be 0, 1, or 2?** Standard 0-1 BFS does not apply directly. Use Dijkstra or a suitable integer-weight bucket method.

**Q5. Why store cost in deque entries?** To skip an obsolete entry after a cheaper route to the same cell has been found.

**Q6. How do you return the route?** Record each neighbor's predecessor when improving its distance, then reconstruct backward from the destination.

**Q7. Why stop at the first non-stale destination entry?** Nondecreasing-distance processing guarantees that its cost is optimal.

**Q8. Where else is 0-1 BFS useful?** Minimum edge reversals, free-versus-paid transitions, and unit-cost obstacle removal.

## 12. What to say aloud in the interview

> “I'll model every cell as a graph node. Moving along its arrow costs zero; taking another direction costs one. Brute-force DFS explores exponentially many routes. Dijkstra gives O(mn log(mn)), but every edge weight is zero or one, so I can use 0-1 BFS. I'll relax distances with a deque, putting zero-cost moves at the front and one-cost moves at the back. I skip stale entries and return when the destination is processed at its best distance. The final complexity is O(mn) time and O(mn) space.”

## 13. Pattern takeaway and practice

~~~text
Minimum-cost movement
          ↓
Model as weighted graph
          ↓
Inspect the edge weights
          ↓
All equal?   → ordinary BFS
Only 0 or 1? → 0-1 BFS
Nonnegative? → Dijkstra
          ↓
0 cost: appendleft
1 cost: append
          ↓
O(V+E) shortest paths
~~~

**Practice next:** LeetCode 2290 (Minimum Obstacle Removal to Reach Corner) for another 0-1 BFS; 743 (Network Delay Time) for Dijkstra; 787 (Cheapest Flights Within K Stops) for constrained shortest paths; and 1293 (Shortest Path in a Grid with Obstacles Elimination) for BFS with an additional state dimension.

**Transferable interview insight:** Before selecting a shortest-path algorithm, inspect the **edge-weight structure**. Restricted weights often permit a faster, simpler algorithm than a general-purpose heap.