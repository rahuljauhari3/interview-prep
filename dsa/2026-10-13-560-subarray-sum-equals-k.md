# DSA Interview Walkthrough — LeetCode 560: Subarray Sum Equals K

**Difficulty:** Medium  
**Pattern:** Prefix Sum · Hash Map · Frequency Counting  
**Core lesson:** For exact contiguous-subarray sums, rewrite each range sum as a difference of prefix sums. A frequency map then counts all valid starting positions in O(1) expected time per ending position.

## 1. Interviewer prompt

Given an integer array `nums` and integer `k`, return the number of nonempty contiguous subarrays whose sum equals `k`.

```text
nums = [1,2,3]
k = 3
answer = 2
```

Negative numbers and zeros are allowed, and different index ranges count separately.

## 2. Recognize the pattern

Contiguous + exact sum suggests prefix sums. Ordinary sliding window is unreliable because negative numbers destroy monotonicity.

Let `P[j]` be the sum of the first `j` elements. Then:

```text
sum(nums[i:j]) = P[j] - P[i]
```

We want:

```text
P[j] - P[i] = k
P[i] = P[j] - k
```

So for each current prefix, count how many earlier prefixes equal `current_prefix - k`.

## 3. Brute force

Enumerating every start/end pair and recomputing the range sum is `O(n^3)`.

Keeping a running sum for each start reduces this to `O(n^2)`:

```python
def quadratic(nums, k):
    ans = 0

    for i in range(len(nums)):
        total = 0
        for j in range(i, len(nums)):
            total += nums[j]
            if total == k:
                ans += 1

    return ans
```

## 4. Prefix-frequency optimization

Maintain `freq[s] = number of earlier prefix boundaries with sum s`.

Initialize `freq = {0: 1}` to represent the empty prefix before index 0.

For each number:
1. update the running prefix,
2. add `freq[prefix - k]` to the answer,
3. increment `freq[prefix]`.

Query **before** inserting the current prefix, otherwise when `k = 0` we would count an empty subarray.

## 5. Dry run

For `nums = [1,-1,0]`, `k = 0`:

| value | prefix | need | prior matches | answer |
|---:|---:|---:|---:|---:|
| 1 | 1 | 1 | 0 | 0 |
| -1 | 0 | 0 | 1 | 1 |
| 0 | 0 | 0 | 2 | 3 |

There are three valid subarrays: `[1,-1]`, `[0]`, and `[1,-1,0]`.

## 6. Clean final Python solution

```python
class Solution:
    def subarraySum(self, nums: list[int], k: int) -> int:
        freq = {0: 1}
        prefix = 0
        ans = 0

        for x in nums:
            prefix += x
            ans += freq.get(prefix - k, 0)
            freq[prefix] = freq.get(prefix, 0) + 1

        return ans
```

No lambda functions are used.

## 7. Complexity

- Time: `O(n)` expected.
- Auxiliary space: `O(n)` worst case.

## 8. Correctness

At each ending boundary `j`, every earlier boundary `i` with prefix sum `P[j]-k` defines exactly one subarray of sum `k`. The frequency map counts all such starts, and each subarray is counted exactly once when its ending boundary is processed.

## 9. Edge cases and mistakes

- All zeros with `k=0` can produce many matches.
- Negative values are handled naturally.
- One matching element should count.

Common mistakes:
- using a standard sliding window despite negatives,
- forgetting `{0:1}`,
- using a set instead of frequency counts,
- inserting the current prefix before querying,
- confusing subarrays with subsequences.

## 10. Interviewer follow-ups

**Longest subarray with sum k?** Store the earliest index for each prefix instead of its frequency.

**Count sums divisible by k?** Count prefix remainders modulo `k`.

**Return all intervals?** Store lists of prior indices; output size may itself be quadratic.

**Streaming version?** Keep `prefix`, `freq`, and `ans` across arrivals.

## 11. What to say aloud

> “Negative numbers make sliding window unreliable. A subarray sum is the difference of two prefix sums, so if the current prefix is P, I need an earlier prefix P-k. I store frequencies of prior prefixes, starting with zero once. For each element I update the prefix, add the count of prefix-k, then insert the current prefix. That gives O(n) expected time and O(n) space.”

## 12. Pattern takeaway

```text
Contiguous exact sum
        ↓
Range sum = difference of prefixes
        ↓
Need earlier prefix = current - k
        ↓
Store prefix frequencies
        ↓
O(n) expected time
```

**Practice next:** LeetCode 525, 974, 930, and 437.