# DSA Interview Walkthrough — LeetCode 973: K Closest Points to Origin

**Difficulty:** Medium  
**Pattern:** Heap · Top K Selection  
**Core lesson:** When a problem asks for the smallest or largest `k` items, avoid fully sorting unless needed. A heap keeps only the best `k` candidates.

## 1. Interviewer prompt

Given points `[x,y]`, return the `k` points closest to the origin. Distance is Euclidean.

```text
points = [[1,3],[-2,2]]
k = 1
answer = [[-2,2]]
```

We only need to compare distances, so square roots are unnecessary:

```text
distance² = x² + y²
```

## 2. Recognize the pattern

The phrase **k closest** is a top-k selection problem.

Possible approaches:
- sort every point by distance: `O(n log n)`,
- keep a max-heap of only `k` points: `O(n log k)`,
- quickselect: average `O(n)` but more implementation complexity.

## 3. Straightforward sorting solution

```python
def sort_solution(points, k):
    points.sort(key=lambda p: p[0] * p[0] + p[1] * p[1])
    return points[:k]
```

This is simple but uses a lambda and sorts all `n` points even though only `k` are needed. For interview prep here, we prefer a concise no-lambda heap solution.

## 4. Max-heap of size k

Python has a min-heap, so negate the squared distance to simulate a max-heap.

For each point:
1. push `(-distance², x, y)`,
2. if heap size exceeds `k`, pop the farthest retained point.

At the end, the heap contains exactly the `k` closest points.

## 5. Clean final Python solution

```python
import heapq

class Solution:
    def kClosest(self, points: list[list[int]], k: int) -> list[list[int]]:
        heap = []

        for x, y in points:
            dist = x * x + y * y
            heapq.heappush(heap, (-dist, x, y))

            if len(heap) > k:
                heapq.heappop(heap)

        return [[x, y] for _, x, y in heap]
```

No lambda functions are used.

## 6. Complexity

For `n` points:
- Time: `O(n log k)`.
- Auxiliary space: `O(k)` excluding the output.

If `k` is close to `n`, sorting is also perfectly reasonable.

## 7. Correctness

After processing any prefix of points, the heap contains the `k` closest points from that prefix (or all points if fewer than `k` have been seen). When a new point is inserted, if there are `k+1` candidates, removing the farthest leaves exactly the best `k`. By induction, the final heap contains the global `k` closest points.

## 8. Edge cases and common mistakes

- `k = 1`.
- `k = len(points)`.
- Duplicate points or equal distances.
- Negative coordinates.

Common mistakes:
- computing square roots unnecessarily,
- accidentally using a min-heap of size k and popping the closest point,
- sorting everything without recognizing the top-k pattern,
- assuming result order matters when the problem allows any order.

## 9. Interviewer follow-ups

**Can we do better than O(n log k)?** Quickselect gives average `O(n)` time and in-place partitioning.

**What if points arrive as a stream?** The size-k heap is ideal because memory stays `O(k)`.

**What if we need the kth closest distance only?** Keep the same heap and read the farthest retained distance, or use quickselect.

## 10. What to say aloud

> “I do not need square roots because squared distance preserves ordering. Sorting all n points is O(n log n), but this is a top-k problem, so I can keep only k candidates in a max-heap. I push each point with negative squared distance and, whenever the heap exceeds k, remove the farthest retained point. That gives O(n log k) time and O(k) extra space.”

## 11. Pattern takeaway

```text
Need smallest/largest k items
          ↓
Full sort works but may do extra work
          ↓
Keep heap of size k
          ↓
Evict worst retained candidate
          ↓
O(n log k)
```

**Practice next:** LeetCode 215, 347, and 703.