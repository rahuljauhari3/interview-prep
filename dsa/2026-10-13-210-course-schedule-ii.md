# DSA Interview Walkthrough — LeetCode 210: Course Schedule II

**Difficulty:** Medium  
**Pattern:** Directed Graph · Topological Sort · Kahn's Algorithm  
**Core lesson:** When tasks have prerequisites, model them as a directed graph. Repeatedly process nodes with zero unmet prerequisites. If every node is processed, the result is a valid topological order; otherwise a cycle exists.

## 1. Interviewer prompt

There are `numCourses` courses labeled `0..numCourses-1`. Each pair `[a,b]` means course `b` must be completed before course `a`. Return any valid order, or `[]` if none exists.

```text
numCourses = 4
prerequisites = [[1,0], [2,0], [3,1], [3,2]]

valid answer: [0,1,2,3]
```

The directed edge is `b -> a`.

## 2. Recognize the pattern

Prerequisites + ordering + possibility of impossibility strongly suggests **topological sorting** on a directed graph.

A DAG has at least one topological order. A directed graph with a cycle does not.

## 3. Brute force

Try every permutation of all courses and verify whether every prerequisite appears before its dependent course.

That takes factorial time: roughly `O(V! * (V+E))`.

The waste is obvious: most permutations violate prerequisites early. Instead, only choose courses whose prerequisites are already satisfied.

## 4. Indegree idea

`indegree[v]` = number of prerequisites still required by course `v`.

Any course with indegree zero can be taken immediately.

Build:
- `graph[u]`: courses unlocked when `u` is completed.
- `indegree[v]`: number of incoming prerequisite edges.

Initialize a queue with every zero-indegree course.

## 5. Kahn's algorithm

Repeatedly:
1. Pop an available course.
2. Append it to the result.
3. For each dependent course, decrement indegree.
4. If a dependent reaches zero, enqueue it.

If the result contains all courses, return it. Otherwise some courses are trapped in a directed cycle.

## 6. Dry run

For `[[1,0], [2,0], [3,1], [3,2]]`:

```text
0 -> 1 -> 3
 \-> 2 ->/
```

Initial indegrees:

```text
course:    0 1 2 3
indegree:  0 1 1 2
```

Queue starts with `[0]`.

- Process 0 -> indegrees of 1 and 2 become zero.
- Process 1 -> indegree of 3 becomes 1.
- Process 2 -> indegree of 3 becomes zero.
- Process 3.

All courses are processed.

## 7. Clean final Python solution

```python
from collections import deque

class Solution:
    def findOrder(self, numCourses: int, prerequisites: list[list[int]]) -> list[int]:
        graph = [[] for _ in range(numCourses)]
        indegree = [0] * numCourses

        for course, prerequisite in prerequisites:
            graph[prerequisite].append(course)
            indegree[course] += 1

        q = deque()
        for course in range(numCourses):
            if indegree[course] == 0:
                q.append(course)

        order = []

        while q:
            course = q.popleft()
            order.append(course)

            for nxt in graph[course]:
                indegree[nxt] -= 1
                if indegree[nxt] == 0:
                    q.append(nxt)

        return order if len(order) == numCourses else []
```

No lambda functions are used.

## 8. Complexity

- Time: `O(V + E)`.
- Auxiliary space: `O(V + E)`.

Every vertex enters the queue at most once, and every edge is processed once.

## 9. Correctness

At every step, a node is appended only when all incoming prerequisite edges from unprocessed courses are gone, so all of its prerequisites already appear earlier in the output. If the algorithm stops before processing all nodes, the remaining directed subgraph has no zero-indegree node, which implies a cycle.

## 10. Edge cases and common mistakes

- No prerequisites -> any course ordering works.
- One course -> `[0]`.
- Disconnected prerequisite chains are allowed.
- A cycle anywhere makes the full schedule impossible.

Common mistakes:
- reversing the edge direction,
- putting only course 0 in the initial queue,
- forgetting to check `len(order) == numCourses`,
- using undirected cycle detection,
- assuming the topological order is unique.

## 11. Interviewer follow-ups

**Can DFS solve it?** Yes. Use three states: unvisited, visiting, visited; reverse DFS postorder if no cycle exists.

**Only need to know whether courses can be completed?** Same algorithm; return whether processed count equals `V` (Course Schedule I).

**Lexicographically smallest valid order?** Use a min-heap instead of a deque.

**Return an actual cycle?** Run DFS with parent/recursion-stack tracking on the remaining graph.

**Parallel course durations?** Use DP over a topological order to compute earliest finish times.

## 12. What to say aloud

> “This is a directed dependency graph, so I need a topological order. I track indegree, meaning how many unmet prerequisites each course has. All indegree-zero courses are immediately available. Each time I process one, I remove its outgoing prerequisite edges conceptually by decrementing the dependents' indegrees. Newly zero-indegree courses enter the queue. If I process all vertices, the order is valid; otherwise a cycle remains. The solution is O(V+E).”

## 13. Pattern takeaway

```text
Dependencies + required ordering
             ↓
Directed graph
             ↓
Track indegree
             ↓
Queue indegree-zero nodes
             ↓
Process node, remove outgoing edges
             ↓
Processed all? valid order : cycle
```

**Practice next:** LeetCode 207, 802, 269, and 2050.