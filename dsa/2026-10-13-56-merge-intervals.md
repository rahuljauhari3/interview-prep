# DSA Interview Walkthrough — LeetCode 56: Merge Intervals

**Difficulty:** Medium  
**Pattern:** Sorting · Greedy Intervals

## Problem
Given intervals `[start,end]`, merge all overlapping intervals and return the non-overlapping result.

```text
[[1,3],[2,6],[8,10],[15,18]]
-> [[1,6],[8,10],[15,18]]
```

## Recognition
Overlapping intervals become easy once sorted by start time. After sorting, a new interval can overlap only with the last merged interval; earlier merged intervals end no later.

## Brute force
Repeatedly compare interval pairs and merge overlaps until no changes remain. This can degrade toward quadratic behavior and is harder to reason about.

## Greedy idea
1. Sort intervals by start.
2. Iterate left to right.
3. If the current start is greater than the last merged end, append a new interval.
4. Otherwise extend the last merged end.

## Clean final Python solution

```python
class Solution:
    def merge(self, intervals: list[list[int]]) -> list[list[int]]:
        intervals.sort()
        merged = []

        for start, end in intervals:
            if not merged or start > merged[-1][1]:
                merged.append([start, end])
            else:
                merged[-1][1] = max(merged[-1][1], end)

        return merged
```

No lambda functions are used. Python list sorting naturally compares the start first.

## Complexity
- Sorting: `O(n log n)`.
- Scan: `O(n)`.
- Output space: `O(n)` worst case.

## Correctness
After sorting by start, assume `merged` correctly represents all processed intervals and is non-overlapping. For the next interval, if its start exceeds the last merged end, it cannot overlap any earlier merged interval, so appending is correct. Otherwise it overlaps the last merged interval, and replacing the end with the larger end exactly represents their union. The invariant is preserved.

## Edge cases and common mistakes
- One interval.
- Completely disjoint intervals.
- One interval contained inside another.
- Touching intervals such as `[1,4]` and `[4,5]` overlap under this problem's definition.

Common mistakes:
- forgetting to sort,
- comparing with every prior interval after sorting,
- using `start >= last_end` when touching endpoints should merge,
- mutating input when the interviewer explicitly forbids it; copy before sorting if needed.

## Interview follow-ups
- Insert one new interval into an already sorted disjoint list? LeetCode 57.
- Minimum meeting rooms? Sort starts/ends or use a heap.
- Remove minimum intervals to eliminate overlaps? Greedy by earliest ending time.

## What to say aloud
> “Sorting by start converts a global overlap problem into a local one. Once sorted, I only compare the current interval with the last merged interval. If they overlap, I extend the end; otherwise I start a new merged interval. Sorting dominates the runtime at O(n log n).”

## Pattern takeaway
```text
Intervals + overlap -> sort by start -> compare only with last merged interval
```