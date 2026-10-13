# DSA Interview Walkthrough — LeetCode 862: Shortest Subarray with Sum at Least K

**Difficulty:** Hard  
**Pattern:** Prefix Sum · Monotonic Deque  
**Core lesson:** When negative values break sliding window, convert subarray sums into differences of prefix sums. Then maintain only prefix indices that can still produce an optimal future answer.

## 1. Interviewer prompt

Given an integer array `nums` and integer `k`, return the length of the shortest nonempty contiguous subarray whose sum is at least `k`. Return `-1` if none exists.

```text
nums = [2,-1,2]
k = 3
answer = 3
```

Negative values are allowed.

## 2. Why ordinary sliding window fails

With only nonnegative values, expanding right can only increase the sum and moving left can only decrease it. Negative values destroy that monotonic behavior.

For example, adding `-5` can make a previously large window too small, so there is no safe “if sum >= k, shrink” rule.

## 3. Prefix-sum formulation

Let:

```text
P[0] = 0
P[i] = nums[0] + ... + nums[i-1]
```

Then the sum of subarray `nums[i:j]` is:

```text
P[j] - P[i]
```

We want:

```text
P[j] - P[i] >= k
```

For each ending prefix `j`, we want the **largest possible valid i** because that gives the shortest length `j-i`.

## 4. Brute force

Check every pair of prefix indices `(i,j)` and test whether `P[j]-P[i] >= k`.

Time: `O(n^2)` after prefix sums.

We need to avoid considering every previous prefix.

## 5. Dominance insight

Suppose `i1 < i2` and `P[i1] >= P[i2]`.

Then `i1` is never better than `i2` for any future end `j`:
- `P[i2]` is smaller or equal, so `P[j]-P[i2]` is at least as large.
- `i2` is later, so `j-i2` is shorter.

Therefore `i1` is dominated and can be discarded.

Maintain candidate prefix indices in a deque whose prefix sums are strictly increasing.

## 6. Two deque operations

For each prefix index `j`:

### A. Pop valid starts from the front

While:

```text
P[j] - P[dq[0]] >= k
```

we found a valid subarray. Update the answer and pop the front. Why pop? Any future end would only produce a longer subarray from that same start.

### B. Pop dominated starts from the back

While:

```text
P[dq[-1]] >= P[j]
```

pop the back because the newer prefix `j` is both later and no larger in prefix value.

Then append `j`.

## 7. Dry run

For `nums = [2,-1,2]`, prefix sums are `[0,2,1,3]`.

- `j=0`, deque `[0]`.
- `j=1`, prefix 2: append -> `[0,1]`.
- `j=2`, prefix 1: prefix at index 1 is 2 >= 1, so index 1 is dominated and removed -> `[0,2]`.
- `j=3`, prefix 3: `3-0=3`, valid length 3. Pop index 0. Result 3.

## 8. Clean final Python solution

```python
from collections import deque

class Solution:
    def shortestSubarray(self, nums: list[int], k: int) -> int:
        prefix = [0]
        for x in nums:
            prefix.append(prefix[-1] + x)

        dq = deque()
        ans = len(nums) + 1

        for j, total in enumerate(prefix):
            while dq and total - prefix[dq[0]] >= k:
                ans = min(ans, j - dq.popleft())

            while dq and prefix[dq[-1]] >= total:
                dq.pop()

            dq.append(j)

        return -1 if ans > len(nums) else ans
```

No lambda functions are used.

## 9. Complexity

Each prefix index enters the deque once and leaves at most once from either end.

- Time: `O(n)`.
- Space: `O(n)` for prefix sums and deque.

## 10. Correctness

The deque stores undominated start candidates in increasing prefix-sum order. Front candidates are the best opportunities to satisfy the `>= k` condition. Once a front candidate produces a valid subarray, later ends can only make that same start produce a longer length, so removing it is safe. Back candidates with larger prefix sums than a newer index are dominated and can never be optimal.

## 11. Common mistakes

- Using ordinary sliding window despite negative values.
- Forgetting `P[0]=0`.
- Reversing the monotonic deque comparison.
- Failing to pop *all* valid fronts.
- Claiming nested while-loops imply `O(n^2)`.

## 12. Interviewer follow-ups

**What if all nums are nonnegative?** A standard sliding window is enough.

**Why does the deque hold indices rather than values?** We need both prefix values and subarray lengths.

**Why pop a valid front?** It cannot yield a shorter future subarray than the one just found.

**Related pattern?** Sliding Window Maximum also uses a monotonic deque, but there it stores candidates by value; here it stores prefix-sum candidates.

## 13. What to say aloud

> “Negative values break the usual sliding-window monotonicity, so I switch to prefix sums. I need `P[j]-P[i] >= k`. I keep candidate start indices in a deque with increasing prefix sums. From the front, I repeatedly remove starts that already satisfy the target because the current end gives their shortest possible answer. From the back, I remove starts dominated by the newer prefix. Each index enters and leaves once, so the algorithm is O(n).”

## 14. Pattern takeaway

```text
Negative values break sliding window
            ↓
Subarray sum = prefix difference
            ↓
Need best prior prefix candidate
            ↓
Discard dominated prefixes
            ↓
Monotonic deque
            ↓
O(n)
```

**Practice next:** LeetCode 239, 1696, and 1425.