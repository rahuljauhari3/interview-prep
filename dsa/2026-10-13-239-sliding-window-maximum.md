# DSA Interview Walkthrough — LeetCode 239: Sliding Window Maximum

**Difficulty:** Hard  
**Pattern:** Fixed-Size Sliding Window · Monotonic Deque · Amortized Analysis  
**Core lesson:** A decreasing deque stores only values that can still become the maximum. Each index enters and leaves at most once, giving linear time.

## 1. Interviewer prompt

Given an integer array `nums` and a window size `k`, return the maximum value in every contiguous window of size `k`.

```text
nums = [1,3,-1,-3,5,3,6,7]
k = 3
answer = [3,3,5,5,6,7]
```

Clarify that the window size is fixed, values may be negative or duplicated, and only the maximum value is required.

## 2. Recognize the pattern

The phrases **fixed-size contiguous window** and **maximum for every window** point to sliding window, but a running sum trick does not work for maxima: if the outgoing element is the maximum, we need to know the next best candidate.

We need a structure supporting:
- add a new element on the right,
- remove expired elements on the left,
- read the maximum quickly.

A max-heap works, but because the window only moves forward, a monotonic deque is better.

## 3. Brute force

Scan every window and calculate its maximum.

```python
def brute(nums, k):
    ans = []
    for left in range(len(nums) - k + 1):
        ans.append(max(nums[left:left + k]))
    return ans
```

Time is `O(nk)`. The slice uses `O(k)` temporary space; index-based scanning can reduce auxiliary space to `O(1)`.

## 4. Better approach: max-heap

Push `(-value, index)` into Python's min-heap and lazily remove stale entries from the root.

```python
import heapq

def heap_solution(nums, k):
    heap = []
    ans = []

    for i, x in enumerate(nums):
        heapq.heappush(heap, (-x, i))

        if i >= k - 1:
            left = i - k + 1
            while heap[0][1] < left:
                heapq.heappop(heap)
            ans.append(-heap[0][0])

    return ans
```

This is `O(n log n)` worst case and may keep stale heap entries.

## 5. Monotonic deque insight

Suppose an older candidate has value `4` and a newer value `7` arrives. The older `4` can never again become a maximum in any future window containing `7`: it is smaller and expires earlier.

So maintain indices in a deque such that their values are strictly decreasing from front to back.

For each new index:
1. Remove expired indices from the front.
2. Remove smaller-or-equal values from the back.
3. Append the current index.
4. Once a full window exists, the front is the maximum.

## 6. Dry run

For `nums = [1,3,-1,-3,5,3,6,7]`, `k = 3`:

| i | value | deque after processing | output |
|---|---:|---|---:|
| 0 | 1 | `[0:1]` | — |
| 1 | 3 | `[1:3]` | — |
| 2 | -1 | `[1:3, 2:-1]` | 3 |
| 3 | -3 | `[1:3, 2:-1, 3:-3]` | 3 |
| 4 | 5 | `[4:5]` | 5 |
| 5 | 3 | `[4:5, 5:3]` | 5 |
| 6 | 6 | `[6:6]` | 6 |
| 7 | 7 | `[7:7]` | 7 |

## 7. Clean final Python solution

```python
from collections import deque

class Solution:
    def maxSlidingWindow(self, nums: list[int], k: int) -> list[int]:
        dq = deque()
        ans = []

        for i, x in enumerate(nums):
            while dq and dq[0] <= i - k:
                dq.popleft()

            while dq and nums[dq[-1]] <= x:
                dq.pop()

            dq.append(i)

            if i >= k - 1:
                ans.append(nums[dq[0]])

        return ans
```

No lambda functions are used.

## 8. Complexity

Each index is appended exactly once and removed at most once. Across the entire scan there are at most `n` pushes and `n` pops.

- Time: `O(n)` amortized.
- Auxiliary space: `O(k)`.

## 9. Correctness

The deque always contains unexpired candidate indices in increasing index order and decreasing value order. Any removed-from-back element is dominated by a newer element that is at least as large and expires later. Therefore the front is always the maximum candidate for the current window.

## 10. Edge cases and mistakes

- `k = 1`: each element is its own window.
- `k = n`: one output, the array maximum.
- Equal values: using `<=` keeps the newer equal candidate.
- Increasing array: old candidates are repeatedly dominated.
- Decreasing array: candidates expire from the front.

Common mistakes:
- storing values instead of indices,
- removing dominated candidates from the front instead of the back,
- outputting before the first full window,
- claiming the nested while-loops make the algorithm `O(n^2)`.

## 11. Interviewer follow-ups

**Sliding Window Minimum?** Maintain an increasing deque.

**Both minimum and maximum?** Maintain two deques.

**Return indices instead of values?** Append `dq[0]`.

**Why is a heap slower?** Heap operations cost logarithmic time and may retain stale entries; the deque exploits the one-directional window movement.

**Why amortized O(1) maintenance?** Each index can be removed only once after being inserted once.

## 12. What to say aloud

> “Brute force scans every window in O(nk). A heap improves the maximum query but still costs logarithmic time. The key observation is dominance: if a newer value is at least as large as an older candidate, the older one can never win again because it also expires sooner. I keep indices in a decreasing deque, expire old indices from the front, remove dominated candidates from the back, and read the maximum from the front. Each index enters and leaves once, so total time is O(n).”

## 13. Pattern takeaway

```text
Fixed-size windows + repeated maximum
                ↓
Brute force O(nk)
                ↓
Heap O(n log n)
                ↓
Newer >= older => older is dominated
                ↓
Monotonic decreasing deque
                ↓
O(n) time, O(k) space
```

**Practice next:** LeetCode 1438, 862, 84, and 1696.