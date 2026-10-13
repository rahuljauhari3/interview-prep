# DSA Interview Walkthrough — LeetCode 416: Partition Equal Subset Sum

**Difficulty:** Medium  
**Pattern:** 0/1 Knapsack · Boolean Dynamic Programming · Space Optimization  
**Core lesson:** When every number may be selected at most once and you need to know whether some subset reaches an exact target, think 0/1 knapsack. A 1D DP works, but sums must be processed backward to avoid reusing the same item.

## 1. Interviewer prompt

Given an array of positive integers, determine whether it can be partitioned into two subsets with equal sum.

```text
nums = [1,5,11,5]
answer = True

[11] and [1,5,5] both sum to 11
```

Clarify that subsets need not be contiguous, each index is usable once, duplicates are separate elements, and only a Boolean answer is required.

## 2. Recognize the pattern

Let `total = sum(nums)`. Equal subsets must each sum to `total // 2`.

- If `total` is odd, return `False`.
- Otherwise ask: **does some subset sum to `target = total // 2`?**

That is classic 0/1 subset sum.

## 3. Brute force: take or skip

```python
def brute(nums):
    total = sum(nums)
    if total % 2:
        return False

    target = total // 2

    def dfs(i, remaining):
        if remaining == 0:
            return True
        if i == len(nums) or remaining < 0:
            return False

        return (
            dfs(i + 1, remaining) or
            dfs(i + 1, remaining - nums[i])
        )

    return dfs(0, target)
```

Worst-case time is `O(2^n)` with `O(n)` recursion depth.

Different paths repeatedly reach the same `(index, remaining)` state, so memoization reduces the state count to `O(n * target)`.

## 4. 2D DP recurrence

Define:

`dp[i][s] = True if some subset of the first i numbers sums to s`.

Base case:

```text
dp[0][0] = True
```

For current number `x = nums[i-1]`:

```text
dp[i][s] = dp[i-1][s] OR
           (s >= x AND dp[i-1][s-x])
```

The first term skips `x`; the second takes it.

## 5. Compress to one dimension

Use `dp[s]` to mean whether sum `s` is reachable with numbers processed so far.

```text
dp[s] = dp[s] OR dp[s-x]
```

The critical detail is iterating `s` **backward**.

Why? With `nums = [2]` and `target = 4`, forward iteration would set `dp[2] = True` and then use that newly updated state to set `dp[4] = True`, incorrectly using the same `2` twice.

Backward iteration ensures `dp[s-x]` still refers to the previous iteration.

## 6. Dry run

```text
nums = [1,5,11,5]
total = 22
target = 11
```

Reachable sums after each value:

| processed | reachable sums up to 11 |
|---|---|
| none | `{0}` |
| 1 | `{0,1}` |
| 5 | `{0,1,5,6}` |
| 11 | `{0,1,5,6,11}` |

Once 11 is reachable, a valid equal partition exists.

## 7. Clean final Python solution

```python
class Solution:
    def canPartition(self, nums: list[int]) -> bool:
        total = sum(nums)
        if total % 2:
            return False

        target = total // 2
        dp = [False] * (target + 1)
        dp[0] = True

        for x in nums:
            for s in range(target, x - 1, -1):
                dp[s] = dp[s] or dp[s - x]

            if dp[target]:
                return True

        return False
```

No lambda functions are used.

## 8. Complexity

Let `T = sum(nums) // 2`.

- Time: `O(nT)`.
- Auxiliary space: `O(T)`.

This is pseudo-polynomial because the runtime depends on the numeric target, not just the number of input values.

## 9. Correctness

After processing each number, `dp[s]` is true exactly when a subset of processed elements sums to `s`. Backward iteration guarantees the current number contributes at most once. Therefore `dp[target]` is true exactly when one subset sums to half the total; the remaining elements then form the other half.

## 10. Edge cases and mistakes

- Odd total -> immediately false.
- `[4,4]` -> true.
- Single positive number -> false.
- Duplicate values are separate usable indices.

Common mistakes:
- treating it as a contiguous subarray problem,
- iterating sums forward,
- forgetting `dp[0] = True`,
- trying a greedy largest-first strategy,
- claiming complexity is `O(n^2)` rather than `O(nT)`.

## 11. Interviewer follow-ups

**Return the actual subsets?** Keep a 2D table or predecessor choices and backtrack.

**Count subsets that hit the target?** Store integer counts instead of booleans and still iterate backward.

**Unlimited reuse of numbers?** That becomes unbounded knapsack; forward iteration is usually appropriate.

**Huge values but n around 30?** Meet-in-the-middle may beat sum-indexed DP.

**Negative values?** Use a set of reachable sums or an offset DP; the simple nonnegative indexing no longer applies directly.

## 12. What to say aloud

> “Equal partition means each side must sum to half the total, so an odd total fails immediately. The remaining problem is 0/1 subset sum. Brute force takes or skips every number. In DP, dp[s] tells me whether sum s is reachable. For each number x, I update dp[s] from dp[s-x], but I iterate sums backward so x cannot be reused in the same pass. That gives O(nT) time and O(T) space.”

## 13. Pattern takeaway

```text
Equal partition
      ↓
Total must be even
      ↓
Subset sum target = total / 2
      ↓
0/1 take-or-skip DP
      ↓
Update sums backward
      ↓
O(nT) time, O(T) space
```

**Practice next:** LeetCode 494, 1049, 322, and 474.