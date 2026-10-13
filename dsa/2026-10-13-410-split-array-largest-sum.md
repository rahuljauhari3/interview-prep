# DSA Interview Walkthrough — LeetCode 410: Split Array Largest Sum

**Difficulty:** Hard  
**Pattern:** Binary Search on the Answer · Greedy Feasibility  
**Core lesson:** When asked to minimize the maximum value across contiguous groups, turn the optimization into a yes/no threshold question. If feasibility is monotonic, binary-search the answer.

## 1. Interviewer prompt

Given an array of nonnegative integers and an integer `k`, split the array into exactly `k` nonempty contiguous subarrays. Minimize the largest subarray sum.

```text
nums = [7,2,5,10,8]
k = 2
answer = 18
```

One optimal split is `[7,2,5] | [10,8]` with maximum group sum 18.

## 2. Recognize the pattern

The key phrase is **minimize the maximum**.

Instead of directly choosing cuts, ask:

> If I am allowed a maximum group sum of `limit`, can I split the array into at most `k` groups?

If a limit works, every larger limit also works. That monotonic false-to-true pattern suggests binary search on the answer.

## 3. Brute force

Choose `k-1` cuts among the `n-1` gaps and evaluate every partition.

There are `C(n-1, k-1)` possible cut sets, so brute force is combinatorial/exponential for middle values of `k`.

A DP improvement is possible:

`dp[p][i] = minimum possible largest sum when splitting first i elements into p groups`.

Using prefix sums:

```text
dp[p][i] = min over j:
    max(dp[p-1][j], prefix[i] - prefix[j])
```

This gives `O(k n^2)` time.

## 4. Greedy feasibility test

For a proposed limit, scan left to right. Keep adding numbers to the current group. If the next number would make the group exceed `limit`, start a new group.

```python
def can_split(nums, k, limit):
    groups = 1
    total = 0

    for x in nums:
        if total + x > limit:
            groups += 1
            total = 0
        total += x

    return groups <= k
```

Why is this greedy correct? Because all numbers are nonnegative. Extending a group as far as possible cannot force *more* groups than cutting earlier.

Why `groups <= k` instead of exactly `k`? If we can form fewer than `k` groups, we can split existing nonempty groups further. With nonnegative values, splitting does not increase either new group's sum.

## 5. Binary-search bounds

The answer must be at least `max(nums)`, because some group contains the largest element.

The answer is at most `sum(nums)`, because the whole array fits under that limit.

So search:

```text
left = max(nums)
right = sum(nums)
```

Use lower-bound binary search for the first feasible limit.

## 6. Dry run

For `[7,2,5,10,8]`, `k=2`:

| left | right | mid | feasible? |
|---:|---:|---:|---|
| 10 | 32 | 21 | yes |
| 10 | 21 | 15 | no |
| 16 | 21 | 18 | yes |
| 16 | 18 | 17 | no |

Answer: 18.

## 7. Clean final Python solution

```python
class Solution:
    def splitArray(self, nums: list[int], k: int) -> int:
        def can_split(limit):
            groups = 1
            total = 0

            for x in nums:
                if total + x > limit:
                    groups += 1
                    total = 0
                total += x

            return groups <= k

        left, right = max(nums), sum(nums)

        while left < right:
            mid = (left + right) // 2

            if can_split(mid):
                right = mid
            else:
                left = mid + 1

        return left
```

No lambda functions are used.

## 8. Complexity

Let `S = sum(nums)`.

- Each feasibility check: `O(n)`.
- Binary search: `O(log(S + 1))` checks.
- Total time: `O(n log(S + 1))`.
- Auxiliary space: `O(1)`.

## 9. Correctness

For a fixed limit, the greedy scan uses the minimum number of groups because delaying a cut until necessary leaves no more remaining work than cutting earlier. Feasibility is monotonic in the limit. Binary search therefore converges to the smallest feasible value, which is exactly the minimum possible largest subarray sum.

## 10. Edge cases and common mistakes

- `k = 1` -> `sum(nums)`.
- `k = n` -> `max(nums)`.
- All zeros -> 0.

Common mistakes:
- using `groups == k` as the feasibility test,
- searching cut positions instead of the answer space,
- setting the lower bound below `max(nums)`,
- using `right = mid - 1` for lower-bound search,
- applying this greedy proof when negative values are allowed.

## 11. Interviewer follow-ups

**Return the actual groups?** Find the optimal limit first, then reconstruct cuts while guaranteeing exactly `k` nonempty groups.

**Why is binary search valid on values rather than indices?** Binary search only needs an ordered domain and a monotonic predicate.

**Negative values allowed?** The greedy feasibility proof breaks; DP is safer.

**At most k groups?** The same feasibility check works directly.

**Related problems?** Koko Eating Bananas, Capacity to Ship Packages Within D Days, and Minimum Number of Days to Make m Bouquets.

## 12. What to say aloud

> “Brute force tries cut positions, and DP can solve it in O(kn²). The stronger observation is that for a candidate maximum sum L, I can greedily compute the minimum number of groups needed. If that count is at most k, L is feasible; if not, it is too small. Feasibility is monotonic, so I binary-search L from max(nums) to sum(nums). Each check is O(n), giving O(n log S) time and O(1) extra space.”

## 13. Pattern takeaway

```text
Minimize the maximum
        ↓
Guess threshold L
        ↓
Can we satisfy constraints under L?
        ↓
Greedy feasibility check
        ↓
False ... False, True ... True
        ↓
Binary search first True
```

**Practice next:** LeetCode 1011, 875, 1482, and 1231.