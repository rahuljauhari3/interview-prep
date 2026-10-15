# DSA Interview Walkthrough — LeetCode 1235: Maximum Profit in Job Scheduling

**Difficulty:** Hard  
**Patterns:** Weighted Interval Scheduling · Dynamic Programming · Binary Search · Sorting  
**Core lesson:** When you must select non-overlapping intervals to maximize a reward, a greedy choice is not generally sufficient. Sort by finishing time, use DP to compare taking versus skipping each job, and binary-search for the last compatible job.

## 1. Interviewer: Problem statement

**Interviewer:** You are given three arrays: startTime, endTime, and profit. Job i runs from startTime[i] to endTime[i] and earns profit[i]. You may take any number of jobs, but selected jobs must not overlap. Return the maximum total profit.

A job ending at time t is compatible with another job starting at t.

~~~text
startTime = [1, 2, 3, 3]
endTime   = [3, 4, 5, 6]
profit    = [50, 10, 40, 70]

Answer = 120
~~~

**Candidate:** Can I take two jobs that meet exactly at an endpoint? Do I need to return the schedule or only the maximum profit?

**Interviewer:** Meeting at an endpoint is allowed. Return only the maximum profit.

**Candidate:** So I am choosing a subset of weighted intervals, rather than simply fitting in as many jobs as possible.

## 2. Recognize the pattern and build intuition

Three clues matter: **intervals**, **non-overlap constraints**, and **maximum total reward**.

Ordinary interval scheduling asks for the maximum *number* of compatible intervals. There, choosing the job that finishes earliest is optimal. Here jobs have different profits, so an early-finishing job might be less valuable than a longer, high-profit job.

For example:

~~~text
Job A: [1, 2), profit 5
Job B: [2, 3), profit 5
Job C: [1, 3), profit 100
~~~

Earliest-finish greedy chooses A and B for 10. The optimum chooses C for 100.

This is **weighted interval scheduling**, a classic dynamic-programming problem. The key is to ask, for each job: **Is it better to skip this job, or take it and combine it with the best schedule that finishes before it starts?**

Sorting jobs by end time makes that earlier compatible schedule a prefix, which is exactly what a one-dimensional DP can represent.

## 3. Brute force: try taking or skipping each job

First sort jobs by end time. For the first i jobs, either skip the i-th job or take it. If we take it, find the number j of earlier jobs ending no later than its start, then recurse on those first j jobs.

Without memoization, this explores the same prefixes repeatedly.

~~~python
def brute(startTime, endTime, profit):
    jobs = sorted(zip(endTime, startTime, profit))

    def solve(i):
        if i == 0:
            return 0

        end, start, gain = jobs[i - 1]
        j = i - 1
        while j > 0 and jobs[j - 1][0] > start:
            j -= 1

        skip = solve(i - 1)
        take = gain + solve(j)
        return max(skip, take)

    return solve(len(jobs))
~~~

**Time:** O(n · 2^n) as a loose worst-case upper bound, including the linear compatibility scan in each recursive call.  
**Auxiliary space:** O(n) for the recursion stack, excluding the sorted jobs.

**Interviewer:** Where is the repeated work?

**Candidate:** Different take/skip sequences ask for the best profit among the same first i jobs. That answer depends only on i, so I can cache it.

## 4. First optimization: memoization

Define:

**dp(i) = maximum profit obtainable from the first i jobs when jobs are sorted by end time.**

For job i, let its start time be s and its profit be p. Let j be the number of jobs among the first i-1 whose end times are at most s.

There are two choices:

**Skip job i:** earn dp(i-1).

**Take job i:** earn p + dp(j). The first j jobs form the full compatible prefix; every job after that prefix ends too late to precede job i.

Therefore:

~~~text
dp(0) = 0

dp(i) = max(
    dp(i - 1),          # skip job i
    profit[i] + dp(j)   # take job i
)
~~~

Here profit[i] denotes the i-th job in **1-based, end-time-sorted order**, not the original input index.

Memoization ensures each dp(i) is evaluated once. However, scanning backward to find j may still cost O(n) per state.

**Time:** O(n²), including sorting.  
**Space:** O(n) for cached DP states and recursion depth, excluding the sorted job list.

**Interviewer:** Can we find j faster?

**Candidate:** Yes. The jobs are sorted by end time, so the last compatible job can be found with binary search.

## 5. Second optimization: binary search the compatible prefix

Create an array **ends** containing the sorted end times.

For a job starting at s, we need the number of *earlier* jobs with end time **<= s**. Python's **bisect_right** returns the insertion position immediately after all values equal to s.

~~~text
ends = [3, 4, 5, 6]
start = 3

bisect_right(ends, 3, 0, 2) = 1
~~~

The answer 1 means exactly one earlier job finishes by time 3.

The strict-versus-nonstrict distinction is important: jobs ending at the next job's start are compatible, so use **bisect_right**, not bisect_left.

We also restrict the search to jobs **before the current job**. For the i-th job in 1-based indexing, that means searching ends[0:i-1].

Now each DP transition takes O(log n) rather than O(n). Since we already process jobs in increasing end-time order, a bottom-up DP is straightforward and avoids recursion.

## 6. Detailed dry run

Use the original example. Sort by end time:

| Sorted job | Start | End | Profit |
|---|---:|---:|---:|
| 1 | 1 | 3 | 50 |
| 2 | 2 | 4 | 10 |
| 3 | 3 | 5 | 40 |
| 4 | 3 | 6 | 70 |

Let dp[i] be the best profit among the first i sorted jobs. Initially dp[0] = 0.

**Job 1:** Starts at 1. No earlier job finishes by 1, so j=0.

~~~text
skip = dp[0] = 0
take = 50 + dp[0] = 50
dp[1] = 50
~~~

**Job 2:** Starts at 2. No earlier job finishes by 2, so j=0.

~~~text
skip = dp[1] = 50
take = 10 + dp[0] = 10
dp[2] = 50
~~~

**Job 3:** Starts at 3. Job 1 ends at 3, so j=1.

~~~text
skip = dp[2] = 50
take = 40 + dp[1] = 90
dp[3] = 90
~~~

**Job 4:** Starts at 3. Again j=1.

~~~text
skip = dp[3] = 90
take = 70 + dp[1] = 120
dp[4] = 120
~~~

Final table:

| i | Compatible prefix j | Skip | Take | dp[i] |
|---|---:|---:|---:|---:|
| 0 | — | — | — | 0 |
| 1 | 0 | 0 | 50 | 50 |
| 2 | 0 | 50 | 10 | 50 |
| 3 | 1 | 50 | 90 | 90 |
| 4 | 1 | 90 | 120 | **120** |

The optimal schedule is job 1 [1,3) with profit 50, followed by job 4 [3,6) with profit 70.

**Interviewer:** Why does job 4 combine with dp[1], not dp[3]?

**Candidate:** dp[3] might include jobs that overlap job 4. Only jobs ending by time 3 can precede job 4, and those form the first one-job prefix.

## 7. Clean final LeetCode solution — Python

~~~python
from bisect import bisect_right

class Solution:
    def jobScheduling(
        self, startTime: list[int],
        endTime: list[int],
        profit: list[int]
    ) -> int:
        jobs = sorted(zip(endTime, startTime, profit))
        ends = [end for end, start, gain in jobs]
        dp = [0] * (len(jobs) + 1)

        for i, (end, start, gain) in enumerate(jobs, 1):
            j = bisect_right(ends, start, 0, i - 1)
            dp[i] = max(dp[i - 1], gain + dp[j])

        return dp[-1]
~~~

The code is iterative, uses no lambda functions, and does not modify the input arrays.

## 8. Complexity analysis

Let n be the number of jobs.

Sorting the jobs takes **O(n log n)** time. Constructing ends and dp takes **O(n)** time. Each of n iterations performs one binary search in **O(log n)** time and a constant-time DP update.

**Time:** O(n log n).  
**Auxiliary space:** O(n) for sorted jobs, end times, and the DP table.

This is the standard practical solution for weighted interval scheduling.

## 9. Correctness proof

We prove by induction on i that dp[i] is the maximum total profit obtainable from the first i jobs sorted by end time.

**Base case:** dp[0] = 0 is optimal when no jobs are available.

**Inductive step:** Consider an optimal schedule using the first i jobs.

If it **does not include job i**, all selected jobs lie among the first i-1 jobs. By the induction hypothesis, their maximum profit is dp[i-1].

If it **includes job i**, every other selected job must finish by the start time of job i. Because jobs are sorted by end time, all such eligible jobs form the prefix of length j returned by binary search. By the induction hypothesis, their maximum profit is dp[j]. Adding job i yields gain + dp[j].

Every optimal schedule falls into exactly one of these cases, and each case's candidate can be achieved. Thus dp[i] is their maximum and is optimal.

By induction, dp[n] is the maximum profit over all jobs.

## 10. Edge cases and common mistakes

**One job:** Return its profit.

**All jobs overlap:** Choose the highest-profit job, not necessarily the shortest or earliest-ending one.

**No jobs overlap:** Take every job; the result is the sum of all profits.

**Touching endpoints:** [1,3) and [3,5) are compatible. Equality must be allowed.

**Equal end times:** Sorting still works. The binary search considers only earlier jobs, and positive-duration jobs cannot be chained merely because they share an end time.

**A long valuable job versus several short jobs:** DP compares their total profits rather than applying a greedy rule.

**Common mistakes:** Sorting start, end, and profit arrays independently (destroying job associations); choosing earliest finish greedily despite unequal profits; using bisect_left and excluding jobs ending exactly at the next start; confusing a job's index with the number of jobs in a DP prefix; binary-searching the entire ends array instead of only earlier jobs; and combining a job with dp[i-1] even when that prefix contains overlapping jobs.

## 11. Interviewer-style follow-up questions

**Q1. Why sort by end time rather than start time?** End-time sorting makes all jobs compatible *before* the current job a prefix, so a simple prefix DP works. Start-time sorting also works, but naturally leads to a suffix DP that binary-searches the next compatible job.

**Q2. Why not greedily choose the job with the highest profit?** A single profitable job may block two or more jobs whose combined profit is higher.

**Q3. Why use bisect_right?** The compatibility rule is previous_end <= current_start. bisect_right counts end times equal to the current start as compatible.

**Q4. Can you reconstruct the chosen jobs?** Yes. Store whether taking each job beats skipping it, then trace backward: when taking job i, jump to its compatible prefix j; otherwise move to i-1.

**Q5. What if profits can be negative?** With job selection optional, dp[0]=0 and the skip option naturally avoid negative-profit jobs. The same recurrence works.

**Q6. Can you reduce DP space to O(1)?** Not with this straightforward recurrence: a job may need dp[j] for an arbitrary earlier j. We retain the history for constant-time DP lookups.

**Q7. What if overlapping jobs are allowed but we can run at most two simultaneously?** This becomes a different capacity-constrained scheduling problem; the simple one-dimensional weighted-interval DP no longer captures all relevant states.

**Q8. What if a fixed cooldown is required between jobs?** Change the compatibility search to previous_end <= current_start - cooldown. The DP structure remains the same.

## 12. Interview-ready explanation

> “This is weighted interval scheduling. Brute force tries taking or skipping each job, which is exponential. I sort jobs by end time and define dp[i] as the maximum profit from the first i jobs. For job i, either I skip it and use dp[i-1], or I take it and add its profit to dp[j], where j is the number of earlier jobs ending by its start. Because end times are sorted, I can find j with binary search, including equal endpoints. This gives O(n log n) time and O(n) space.”

## 13. Pattern takeaway and practice

~~~text
Choose non-overlapping intervals to maximize reward
                       |
                       v
Earliest-finish greedy fails with unequal profits
                       |
                       v
Sort jobs by end time
                       |
                       v
For each job: SKIP or TAKE
                       |
                       v
TAKE requires last compatible earlier job
                       |
                       v
Binary search compatible prefix j
                       |
                       v
dp[i] = max(dp[i-1], profit[i] + dp[j])
                       |
                       v
O(n log n) time, O(n) space
~~~

**Practice next:** LeetCode 2008 (Maximum Earnings From Taxi) for weighted scheduling; 1751 (Maximum Number of Events That Can Be Attended II) for an added limit on chosen events; 646 (Maximum Length of Pair Chain) for a contrasting greedy interval problem; and 435 (Non-overlapping Intervals) for earliest-finish greedy.

**Transferable insight:** When a greedy interval algorithm stops working because intervals have different rewards, keep the sorting insight but replace the greedy decision with **take/skip dynamic programming**. Binary search often turns the compatibility lookup into an O(log n) operation.