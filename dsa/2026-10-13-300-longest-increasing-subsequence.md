# DSA Interview Walkthrough — LeetCode 300: Longest Increasing Subsequence

**Difficulty:** Medium  
**Pattern:** Dynamic Programming · Patience Sorting · Binary Search  
**Core lesson:** The O(n²) DP is straightforward, but for length only, maintain the smallest possible tail value for an increasing subsequence of each length. Binary search updates that structure in O(n log n).

## 1. Interviewer prompt

Given an integer array `nums`, return the length of the longest strictly increasing subsequence.

```text
nums = [10,9,2,5,3,7,101,18]
answer = 4
```

One LIS is `[2,3,7,101]`.

The subsequence does not need to be contiguous.

## 2. Recognize the pattern

Subsequence + optimization naturally suggests DP.

A clean DP state is:

```text
dp[i] = length of the longest increasing subsequence ending at i
```

Then look back at every `j < i` with `nums[j] < nums[i]`.

## 3. O(n²) DP

```python
def quadratic(nums):
    n = len(nums)
    dp = [1] * n

    for i in range(n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)
```

- Time: `O(n^2)`.
- Space: `O(n)`.

This is a strong first interview solution.

## 4. Optimization insight: smallest possible tails

Suppose we have two increasing subsequences of the same length:

```text
[2,5]
[2,3]
```

The second one is always at least as good for future extension because tail `3` is smaller than tail `5`.

So for each subsequence length, keep only the **smallest possible ending value**.

Let `tails[i]` be the smallest possible tail of an increasing subsequence of length `i+1` seen so far.

The `tails` array itself is increasing.

## 5. Update with binary search

For each number `x`:
- find the first index `i` where `tails[i] >= x`,
- replace `tails[i]` with `x`,
- if no such index exists, append `x`.

Use `bisect_left` because the subsequence must be **strictly** increasing. Equal values should replace an existing tail, not extend the length.

## 6. Dry run

For `[10,9,2,5,3,7,101,18]`:

```text
10  -> [10]
9   -> [9]
2   -> [2]
5   -> [2,5]
3   -> [2,3]
7   -> [2,3,7]
101 -> [2,3,7,101]
18  -> [2,3,7,18]
```

The final `tails` array is not necessarily an actual subsequence of the input, but its **length** equals the LIS length: 4.

## 7. Clean final Python solution

```python
from bisect import bisect_left

class Solution:
    def lengthOfLIS(self, nums: list[int]) -> int:
        tails = []

        for x in nums:
            i = bisect_left(tails, x)

            if i == len(tails):
                tails.append(x)
            else:
                tails[i] = x

        return len(tails)
```

No lambda functions are used.

## 8. Complexity

Each of `n` values performs one binary search over at most `n` tails.

- Time: `O(n log n)`.
- Auxiliary space: `O(n)`.

## 9. Correctness

Invariant: after processing any prefix, `tails[i]` is the smallest possible ending value among all increasing subsequences of length `i+1` seen so far.

Replacing a larger tail with a smaller one never reduces the best achievable length; it only makes that subsequence length easier to extend later. Appending occurs exactly when `x` is greater than every current tail, proving an increasing subsequence one element longer exists. Therefore the number of tail slots equals the LIS length.

## 10. Edge cases and common mistakes

- Strictly increasing array -> answer n.
- Strictly decreasing -> 1.
- Duplicate values do not extend a strictly increasing subsequence.
- Single value -> 1.

Common mistakes:
- using `bisect_right`, which changes behavior for duplicates,
- claiming `tails` itself must be the actual LIS,
- confusing subsequence with subarray,
- starting with a greedy rule such as “always take the next larger value.”

## 11. Interviewer follow-ups

**Return the actual LIS?** Track predecessor indices and the index representing each tail length while performing the same binary-search method.

**Longest non-decreasing subsequence?** Use `bisect_right` instead of `bisect_left`.

**Why is replacing a tail safe?** A smaller tail leaves at least as many future extension options as a larger tail of the same length.

## 12. What to say aloud

> “The straightforward DP is O(n²): dp[i] is the best increasing subsequence ending at i. To optimize, I only need one representative per subsequence length—the smallest possible tail. Smaller tails dominate larger tails because they are easier to extend. I maintain these tails in sorted order and binary-search the first tail >= each number, replacing it or appending if the number extends the longest length. That gives O(n log n).”

## 13. Pattern takeaway

```text
Longest subsequence
        ↓
O(n²) DP by ending index
        ↓
For each length, smaller tail dominates
        ↓
Maintain minimum tails
        ↓
Binary-search replacement position
        ↓
O(n log n)
```

**Practice next:** LeetCode 354, 673, and 646.