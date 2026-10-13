# DSA Interview Walkthrough — LeetCode 1235: Maximum Profit in Job Scheduling

**Difficulty:** Hard  
**Pattern:** Weighted Interval Scheduling · Dynamic Programming · Binary Search  
**Core lesson:** Sort jobs, then at each job choose between skipping it or taking it and jumping to the next non-overlapping job found by binary search.

## 1. Interviewer prompt

You are given `startTime`, `endTime`, and `profit` arrays describing jobs. Select non-overlapping jobs to maximize total profit.

```text
start  = [1,2,3,3]
end    = [3,4,5,6]
profit = [50,10,40,70]
answer = 120
```

One optimal choice is the job `[1,3]` with profit 50 and `[3,6]` with profit 70.

## 2. Recognize the pattern

This is interval scheduling, but unlike the classic “maximum number of meetings” problem, jobs have different profits. Greedy earliest-finish is not enough.

The decision at each job is binary:
- skip it,
- take it and continue from the first compatible future job.

That suggests dynamic programming. Sorting allows the compatible next job to be found efficiently with binary search.

## 3. Brute force

At each job, recursively choose take or skip. Without memoization this explores up to `2^n` subsets.

A state can be defined by the index of the next job under consideration, so memoization already reduces repeated work.

## 4. Sort jobs by start time

Create tuples:

```text
(start, end, profit)
```

and sort by start time.

For job `i`, if we take it, we need the first future job whose start time is at least `jobs[i].end`.

Because starts are sorted, binary search finds that index in `O(log n)`.

## 5. DP recurrence

Let `dp(i)` be the maximum profit obtainable starting from sorted job index `i`.

Then:

```text
dp(i) = max(
    dp(i + 1),
    jobs[i].profit + dp(next_non_overlapping_index)
)
```

The first term skips the job; the second takes it.

## 6. Clean final Python solution

```python
from bisect import bisect_left
from functools import cache

class Solution:
    def jobScheduling(
        self,
        startTime: list[int],
        endTime: list[int],
        profit: list[int]
    ) -> int:
        jobs = sorted(zip(startTime, endTime, profit))
        starts = [job[0] for job in jobs]

        @cache
        def dp(i):
            if i == len(jobs):
                return 0

            start, end, gain = jobs[i]
            nxt = bisect_left(starts, end)

            skip = dp(i + 1)
            take = gain + dp(nxt)
            return max(skip, take)

        return dp(0)
```

No lambda functions are used.

## 7. Complexity

- Sorting: `O(n log n)`.
- `n` DP states, each doing one binary search: `O(n log n)`.
- Auxiliary space: `O(n)` for memoization and arrays, plus recursion stack.

## 8. Correctness

At every job, any optimal schedule either excludes it or includes it. If excluded, the best remaining profit is `dp(i+1)`. If included, no job starting before its end may be chosen, so the next valid state is exactly the first start time `>= end`. Taking the maximum of these exhaustive possibilities gives the optimal result.

## 9. Edge cases and common mistakes

- Jobs may start exactly when another ends; that is allowed.
- Several jobs may share the same start/end times.
- Highest-profit single job may still be worse than multiple compatible jobs.

Common mistakes:
- using unweighted interval-scheduling greedy,
- binary-searching the wrong field,
- requiring `start > end` instead of `start >= end`,
- sorting one array without keeping the other job fields aligned.

## 10. Interviewer follow-ups

**Bottom-up version?** Build `dp[i]` from right to left using the same binary-search jump.

**Sort by end time instead?** Also possible; then find the last compatible previous job and use classic weighted-interval DP.

**Return chosen jobs?** Store which branch wins at each state and reconstruct.

**Why not greedy by profit?** A high-profit long job may block several smaller jobs whose combined profit is larger.

## 11. What to say aloud

> “This is weighted interval scheduling. Greedy is not sufficient because intervals have different profits. I sort jobs by start time and define dp(i) as the best profit from job i onward. I either skip job i or take it, add its profit, and binary-search for the first job starting at or after its end. Memoization gives n states, each with a logarithmic jump lookup, so the total is O(n log n).”

## 12. Pattern takeaway

```text
Weighted non-overlapping intervals
            ↓
Sort intervals
            ↓
Take or skip each interval
            ↓
Binary-search compatible next interval
            ↓
DP over interval index
            ↓
O(n log n)
```

**Practice next:** LeetCode 435, 646, and 1751.