# DSA Interview Walkthrough — LeetCode 207: Course Schedule

**Difficulty:** Medium  
**Pattern:** Directed Graph · Cycle Detection · Topological Sort / DFS Coloring

## Problem
Given `numCourses` and prerequisite pairs `[a,b]`, return whether all courses can be completed. The edge is `b -> a`.

## Recognition
Prerequisites form a directed graph. All courses are finishable iff the graph has no directed cycle.

## Brute force
Repeatedly search dependency chains from each course without memoization. This can revisit the same graph regions many times.

## Approach 1: Kahn's algorithm
Track indegrees, enqueue all zero-indegree courses, and repeatedly remove them.

```python
from collections import deque

class Solution:
    def canFinish(self, numCourses: int, prerequisites: list[list[int]]) -> bool:
        graph = [[] for _ in range(numCourses)]
        indegree = [0] * numCourses

        for course, prereq in prerequisites:
            graph[prereq].append(course)
            indegree[course] += 1

        q = deque()
        for course in range(numCourses):
            if indegree[course] == 0:
                q.append(course)

        done = 0

        while q:
            course = q.popleft()
            done += 1

            for nxt in graph[course]:
                indegree[nxt] -= 1
                if indegree[nxt] == 0:
                    q.append(nxt)

        return done == numCourses
```

## Approach 2: DFS 3-state coloring
Use `0=unvisited`, `1=visiting`, `2=done`. Reaching a visiting node means a back edge and therefore a cycle.

```python
class Solution:
    def canFinish(self, numCourses: int, prerequisites: list[list[int]]) -> bool:
        graph = [[] for _ in range(numCourses)]
        for course, prereq in prerequisites:
            graph[prereq].append(course)

        state = [0] * numCourses

        def has_cycle(course):
            if state[course] == 1:
                return True
            if state[course] == 2:
                return False

            state[course] = 1

            for nxt in graph[course]:
                if has_cycle(nxt):
                    return True

            state[course] = 2
            return False

        for course in range(numCourses):
            if state[course] == 0 and has_cycle(course):
                return False

        return True
```

No lambda functions are used.

## Complexity
Both approaches run in `O(V+E)` time and `O(V+E)` space.

## Common mistakes
- Reversing `[a,b]` into the wrong edge direction.
- Using undirected cycle detection.
- Forgetting disconnected components.
- In DFS, marking a node done before all descendants finish.

## Interview follow-ups
- Need an actual course order? Use Course Schedule II / topological sort.
- Need lexicographically smallest order? Use a min-heap with Kahn's algorithm.
- Need an explicit cycle? DFS with parent/stack reconstruction is convenient.

## What to say aloud
> “This is a directed-cycle problem. I can either topologically eliminate zero-indegree nodes or use DFS coloring. If Kahn processes all vertices, the graph is acyclic; if some remain, they are trapped behind a directed cycle. Both methods are O(V+E).”

## Pattern takeaway
```text
Prerequisites -> directed graph -> detect cycle -> topological sort or DFS coloring
```