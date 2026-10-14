# DSA Interview Walkthrough — LeetCode 332: Reconstruct Itinerary

**Difficulty:** Hard  
**Pattern:** Directed Multigraph · Eulerian Path · Hierholzer’s Algorithm · Min-Heap  
**Core lesson:** When you must use every edge exactly once, think Eulerian trail. Hierholzer’s algorithm constructs the route by adding airports to the answer only after exhausting their outgoing tickets. A min-heap handles the lexicographic tie-break.

## 1. Interviewer: Problem statement

Given airline tickets `[from, to]`, reconstruct an itinerary that starts at `JFK`, uses every ticket exactly once, and is lexicographically smallest among valid complete itineraries.

```text
tickets = [
    ["JFK", "SFO"],
    ["JFK", "ATL"],
    ["SFO", "ATL"],
    ["ATL", "JFK"],
    ["ATL", "SFO"]
]

answer = ["JFK", "ATL", "JFK", "SFO", "ATL", "SFO"]
```

Tickets are directed; duplicate tickets are separate edges; airports may be revisited.

## 2. Recognize the pattern

The crucial phrase is **use every ticket exactly once**. Airports are vertices and tickets are directed edges, so the graph is a directed multigraph.

This is not a shortest-path or Hamiltonian-path problem. It is an **Eulerian trail** problem.

A visited-airport set would be wrong because revisiting airports is allowed; ticket usage is what matters.

## 3. Brute force: try every unused ticket

```python
def brute(tickets):
    used = [False] * len(tickets)
    path = ["JFK"]
    best = None

    def dfs():
        nonlocal best

        if len(path) == len(tickets) + 1:
            if best is None or path < best:
                best = path[:]
            return

        for i, (src, dst) in enumerate(tickets):
            if not used[i] and src == path[-1]:
                used[i] = True
                path.append(dst)
                dfs()
                path.pop()
                used[i] = False

    dfs()
    return best or []
```

**Time:** factorial in the worst case.  
**Auxiliary space:** O(E).

## 4. Why naive lexical greedy fails

```text
tickets = [
    ["JFK", "KUL"],
    ["JFK", "NRT"],
    ["NRT", "JFK"]
]
```

Choosing the smallest next destination gives `JFK -> KUL`, which gets stuck. The only complete route is:

```text
JFK -> NRT -> JFK -> KUL
```

Lexical order still matters, but we must defer final placement of vertices.

## 5. Hierholzer’s algorithm

1. Traverse an unused outgoing edge.
2. Continue until the current airport has no unused outgoing tickets.
3. Add that airport to the route.
4. Backtrack and continue from earlier airports.
5. Reverse the route at the end.

For lexicographic order, choose the smallest unused destination. Use a min-heap for each departure airport.

The key insight is that the construction stack is not the final route. Airports are committed only when their outgoing edges are exhausted.

## 6. Dry run: greedy trap

For the `JFK/KUL/NRT` example:

```text
JFK heap = [KUL, NRT]
NRT heap = [JFK]
stack = [JFK]
route = []
```

- Take `JFK -> KUL`; KUL has no outgoing edge, so append KUL to route.
- Back at JFK, take `JFK -> NRT`.
- Take `NRT -> JFK`.
- This JFK has no remaining tickets; append it.
- Finish NRT, then original JFK.

Reverse finish order:

```text
[JFK, NRT, JFK, KUL]
```

## 7. Clean final LeetCode solution — Python

```python
from collections import defaultdict
import heapq

class Solution:
    def findItinerary(self, tickets: list[list[str]]) -> list[str]:
        graph = defaultdict(list)

        for src, dst in tickets:
            heapq.heappush(graph[src], dst)

        stack = ["JFK"]
        route = []

        while stack:
            src = stack[-1]

            if graph[src]:
                stack.append(heapq.heappop(graph[src]))
            else:
                route.append(stack.pop())

        return route[::-1]
```

No lambda functions are used. Duplicate tickets remain as duplicate heap entries.

## 8. Complexity analysis

Let E be the number of tickets.

Every ticket is inserted into and removed from a heap once.

**Time:** O(E log(E+1)).  
**Auxiliary space:** O(V+E).

If lexicographic order did not matter, ordinary Hierholzer traversal could be O(V+E).

## 9. Correctness

Each ticket is represented by exactly one heap entry and is consumed exactly once.

Hierholzer’s reverse-finish construction splices exhausted trails into earlier unfinished trails, producing a continuous Eulerian trail when reversed.

Using the smallest available outgoing edge gives the lexicographically smallest Eulerian itinerary under the problem’s existence guarantee.

## 10. Edge cases and common mistakes

- One ticket.
- Duplicate tickets.
- Revisiting airports.
- Cycles.
- Dead-end airports.

Common mistakes:
- Tracking visited airports instead of tickets.
- Returning the traversal stack directly.
- Appending an airport when first visited rather than when exhausted.
- Forgetting to reverse the finish order.
- Using a set and thereby losing duplicate tickets.

## 11. Interviewer-style follow-up questions

**Why is this not shortest path?**  
Every ticket must be used; the objective is edge coverage, not path cost.

**Could we sort adjacency lists instead of heaps?**  
Yes. Sort destinations in reverse lexicographic order once and use `pop()`.

**What conditions allow a directed Eulerian trail?**  
Degree balance plus appropriate connectivity: normally one start with outdegree one larger than indegree, one end with indegree one larger than outdegree, and all others balanced.

**What if validity were not guaranteed?**  
Validate degree/connectivity conditions or verify that the final route uses all E edges.

**What if we had to visit each airport exactly once?**  
That is Hamiltonian-path territory; Hierholzer does not apply.

## 12. What to say aloud

> “Because every ticket must be used exactly once, this is an Eulerian-trail problem. Brute-force backtracking is factorial. A naive smallest-destination greedy choice can hit a dead end, so I use Hierholzer’s algorithm: traverse unused tickets, and append an airport to the answer only when all its outgoing tickets are exhausted. I store outgoing destinations in min-heaps for lexical order. Reversing the finish order gives the itinerary in O(E log E) time.”

## 13. Pattern takeaway

```text
Use every EDGE exactly once
            ↓
Directed multigraph
            ↓
Eulerian trail
            ↓
Naive lexical greedy can dead-end
            ↓
Hierholzer: append on exhaustion
            ↓
Min-heap for lexical choice
            ↓
Reverse finish order
```

**Practice next:** LeetCode 2097 Valid Arrangement of Pairs, 753 Cracking the Safe, and 207 Course Schedule.