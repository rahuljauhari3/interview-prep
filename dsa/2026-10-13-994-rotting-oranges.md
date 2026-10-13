# DSA Interview Walkthrough — LeetCode 994: Rotting Oranges

**Difficulty:** Medium  
**Pattern:** Multi-Source BFS · Grid Graph  
**Core lesson:** When a process spreads simultaneously from many starting points one edge per unit time, enqueue all starting sources at once and run BFS by levels.

## 1. Interviewer prompt

Each grid cell is:
- `0`: empty,
- `1`: fresh orange,
- `2`: rotten orange.

Every minute, fresh oranges adjacent up/down/left/right to a rotten orange become rotten. Return the minimum number of minutes until no fresh orange remains, or `-1` if impossible.

## 2. Recognize the pattern

Rotting spreads one grid edge per minute, so shortest-time propagation suggests BFS.

The key twist is that several oranges may already be rotten. All of them start spreading at time 0, so this is **multi-source BFS**.

Instead of running BFS separately from each rotten orange, put all rotten oranges into the queue initially.

## 3. Brute force

A straightforward simulation scans the whole grid every minute, finds every fresh orange adjacent to a rotten one, marks them for the next minute, then scans again.

If the process lasts `O(mn)` minutes and each minute scans `O(mn)` cells, this can approach `O((mn)^2)` work.

## 4. Multi-source BFS

Scan the grid once:
- enqueue every rotten orange,
- count every fresh orange.

Then process the queue layer by layer. Every BFS layer represents one minute of spreading.

Whenever a fresh neighbor becomes rotten:
- mutate it to 2 immediately,
- decrement `fresh`,
- enqueue it.

Mutating immediately prevents the same orange from being enqueued multiple times.

## 5. Clean final Python solution

```python
from collections import deque

class Solution:
    def orangesRotting(self, grid: list[list[int]]) -> int:
        rows, cols = len(grid), len(grid[0])
        q = deque()
        fresh = 0

        for r in range(rows):
            for c in range(cols):
                if grid[r][c] == 2:
                    q.append((r, c))
                elif grid[r][c] == 1:
                    fresh += 1

        minutes = 0
        directions = [(1,0), (-1,0), (0,1), (0,-1)]

        while q and fresh:
            for _ in range(len(q)):
                r, c = q.popleft()

                for dr, dc in directions:
                    nr, nc = r + dr, c + dc

                    if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                        grid[nr][nc] = 2
                        fresh -= 1
                        q.append((nr, nc))

            minutes += 1

        return minutes if fresh == 0 else -1
```

No lambda functions are used.

## 6. Complexity

Every cell is processed at most a constant number of times.

- Time: `O(mn)`.
- Auxiliary space: `O(mn)` worst case for the queue.

## 7. Correctness

All initially rotten oranges enter the queue at time 0. BFS processes all nodes at distance `d` from any source before nodes at distance `d+1`, so the first time a fresh orange is reached is the earliest minute it can rot. Therefore the number of completed BFS layers is exactly the minimum time required. If fresh oranges remain after the queue empties, no path from any rotten orange reaches them.

## 8. Edge cases and common mistakes

- No fresh oranges initially -> 0.
- Fresh oranges but no rotten sources -> `-1` unless fresh count is zero.
- Isolated fresh region -> `-1`.
- Multiple initial rotten oranges.

Common mistakes:
- running one BFS per rotten orange,
- incrementing minutes per node instead of per BFS layer,
- enqueueing a fresh orange without immediately marking it rotten,
- forgetting the all-fresh/all-empty edge cases.

## 9. Interviewer follow-ups

**Why multi-source BFS?** All rotten oranges begin at the same time, so they are all distance-zero sources.

**Could we store the timestamp with each cell?** Yes. Push `(r,c,time)` and track the maximum time instead of explicit layer loops.

**What if diagonal spread is allowed?** Add four diagonal directions.

**Related problems?** 01 Matrix, Walls and Gates, and shortest distance from multiple sources.

## 10. What to say aloud

> “This is shortest-time propagation on an unweighted grid graph. Because all initially rotten oranges spread simultaneously, I enqueue all of them before BFS starts. Each BFS layer is one minute. When I first reach a fresh orange I mark it rotten immediately, decrement the fresh count, and enqueue it. If fresh reaches zero, the number of processed layers is the minimum time; if the queue empties first, some oranges are unreachable.”

## 11. Pattern takeaway

```text
Simultaneous spreading from many sources
              ↓
Unweighted graph distance
              ↓
Initialize BFS with every source
              ↓
Process one level per time unit
              ↓
O(number of cells)
```

**Practice next:** LeetCode 542, 286, and 1162.