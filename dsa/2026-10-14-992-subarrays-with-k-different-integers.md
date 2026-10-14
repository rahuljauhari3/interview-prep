# DSA Interview Walkthrough — LeetCode 992: Subarrays with K Different Integers

**Difficulty:** Hard  
**Pattern:** Sliding Window · Frequency Map · At-Most Counting · Inclusion–Exclusion  
**Core lesson:** When counting subarrays with **exactly K distinct values**, it is often easier to count subarrays with **at most K** and subtract those with **at most K−1**. The at-most condition is monotonic, making a sliding window possible.

## 1. Interviewer: Problem statement

**Interviewer:** Given an integer array **nums** and an integer **k**, return the number of nonempty contiguous subarrays containing exactly **k distinct integers**.

Example:

```text
nums = [1, 2, 1, 2, 3]
k = 2
answer = 7
```

The seven valid subarrays are:

```text
[1,2]
[1,2,1]
[1,2,1,2]
[2,1]
[2,1,2]
[1,2]
[2,3]
```

The two occurrences of [1,2] start at different positions, so they count separately.

**Candidate:** Are subarrays contiguous? Do duplicates count as additional distinct values? Can values be negative?

**Interviewer:** Subarrays are contiguous; duplicates do not increase the distinct count. The original problem uses positive integers, but the algorithm also works with negative values.

**Candidate:** So I count every pair of boundaries whose enclosed elements contain exactly k different values?

**Interviewer:** Correct. Start with brute force.

## 2. Recognize the pattern

Three clues matter:

1. **Contiguous subarray** suggests a window bounded by left and right indices.
2. **Number of distinct values** suggests a frequency dictionary rather than a simple sum.
3. **Exactly K** is the obstacle: unlike "at most K," the exact condition is not directly monotonic when a window expands or shrinks.

For example, a window with two distinct values can become one with three distinct values when extended. Shrinking a window with three distinct values might still leave three, then eventually two, then one.

A conventional sliding window works best with a condition such as **at most K distinct values**: if the window has too many, move its left boundary right until it becomes valid again.

**Interviewer:** Can you transform the exact condition into a monotonic one?

**Candidate:** Yes. Every subarray with at most K distinct values has either at most K−1 or exactly K. Therefore:

```text
exactly(K) = atMost(K) - atMost(K - 1)
```

This is the key reduction.

## 3. Straightforward brute force

The most literal solution enumerates all start/end pairs and builds a set for each subarray. That can take **O(n³)** time because there are O(n²) subarrays and building a set for one can take O(n).

We can improve the brute force immediately: fix the left boundary, expand the right boundary one step at a time, and maintain frequencies and a distinct count.

```python
def brute(nums, k):
    ans = 0

    for left in range(len(nums)):
        freq = {}
        distinct = 0

        for right in range(left, len(nums)):
            x = nums[right]
            if freq.get(x, 0) == 0:
                distinct += 1
            freq[x] = freq.get(x, 0) + 1

            if distinct == k:
                ans += 1
            if distinct > k:
                break

    return ans
```

We can stop when distinct exceeds k because extending a fixed-left subarray cannot reduce the number of distinct values.

**Time:** O(n²) worst case.  
**Auxiliary space:** O(min(n, number of distinct values)) for the frequency dictionary.

**Interviewer:** Where is the repeated work?

**Candidate:** Consecutive starting positions recount many of the same elements. I want to reuse the window instead of restarting it at each left boundary.

## 4. Why a direct exact-K window is awkward

Suppose we maintain a window with exactly two distinct values. For each right endpoint, how many valid starting positions exist?

Consider:

```text
nums = [1, 2, 1, 2]
k = 2
right = 3
```

Three subarrays ending at index 3 have exactly two distinct values:

```text
[1,2,1,2]   starts at 0
[2,1,2]     starts at 1
[1,2]       starts at 2
```

A single left pointer marking one valid window does not directly tell us that there are **three** valid starts. Duplicates make the number of valid starts variable.

One solution uses two windows simultaneously. But the simpler interview approach is to count **at most K**, where the number of valid starts is immediately available.

## 5. Derive atMost(K) with a sliding window

Maintain a window **[left, right]** and a frequency dictionary. Track how many distinct values are currently inside it.

For each new right endpoint:

1. Add nums[right] to the frequency map.
2. If this value was absent, increment distinct.
3. While distinct > K, remove nums[left] and advance left.
4. Once the window is valid, add **right − left + 1** to the answer.

**Why right − left + 1?**

After shrinking, [left, right] contains at most K distinct values. Every suffix ending at right and starting at any position from left through right is also valid: removing elements cannot introduce new distinct values.

There are exactly:

```text
right - left + 1
```

such starting positions.

Any start before left is invalid. The left pointer moved past those starts precisely because their windows contained more than K distinct values. As right moves forward, adding elements cannot make those earlier invalid starts valid again.

This counts all at-most-K subarrays ending at each right index **without enumerating them**.

## 6. Detailed dry run

Use:

```text
nums = [1, 2, 1, 2, 3]
k = 2
```

First compute **atMost(2)**:

| right | New value | left after shrinking | Valid starts | New subarrays |
|---|---:|---:|---|---:|
| 0 | 1 | 0 | 0 | 1 |
| 1 | 2 | 0 | 0, 1 | 2 |
| 2 | 1 | 0 | 0, 1, 2 | 3 |
| 3 | 2 | 0 | 0, 1, 2, 3 | 4 |
| 4 | 3 | 3 | 3, 4 | 2 |

At right = 4, the window initially has values 1, 2, and 3. Removing indices 0 and 1 still leaves three distinct values. Removing index 2 eliminates the last 1, so left becomes 3 and the window [2,3] contains only two distinct values.

```text
atMost(2) = 1 + 2 + 3 + 4 + 2 = 12
```

Now compute **atMost(1)**. Because adjacent elements all differ in this example, each right endpoint has exactly one valid one-distinct suffix:

```text
atMost(1) = 1 + 1 + 1 + 1 + 1 = 5

exactly(2) = atMost(2) - atMost(1)
           = 12 - 5
           = 7
```

**Interviewer:** Why subtract atMost(1), rather than counting windows with two distinct values directly?

**Candidate:** The at-most condition gives a clean range of valid starting positions. Subtracting removes precisely the subarrays with fewer than two distinct values, leaving exactly two.

## 7. Clean final LeetCode solution — Python

```python
from collections import defaultdict

class Solution:
    def subarraysWithKDistinct(self, nums: list[int], k: int) -> int:
        def at_most(limit):
            if limit < 0:
                return 0

            freq = defaultdict(int)
            left = distinct = total = 0

            for right, x in enumerate(nums):
                if freq[x] == 0:
                    distinct += 1
                freq[x] += 1

                while distinct > limit:
                    y = nums[left]
                    freq[y] -= 1
                    if freq[y] == 0:
                        distinct -= 1
                    left += 1

                total += right - left + 1

            return total

        return at_most(k) - at_most(k - 1)
```

This solution uses **no lambda functions**, leaves the input unchanged, and performs two linear sliding-window passes. The negative-limit guard also makes the helper safe for k = 0, although the original problem guarantees k >= 1.

## 8. Complexity analysis

Let n be the array length.

Each call to at_most moves **right** forward n times. Although there is a nested while loop, **left** also moves forward at most n times total.

Every frequency-map update takes O(1) expected time, so one pass takes O(n) expected time. We run two passes:

**Time:** O(n) expected.  
**Auxiliary space:** O(d), where d is the number of distinct values stored in the frequency dictionary, at most O(n).

The dictionary may retain keys whose counts become zero; this does not affect correctness or the O(n) space bound.

## 9. Correctness proof

Define A(K) as the number of nonempty subarrays containing at most K distinct values.

**Lemma 1 — at_most(K) computes A(K).** Fix a right endpoint r. After adding nums[r], the algorithm advances left until the current window has at most K distinct values. Every subarray ending at r and starting at a position s >= left is valid because it is a suffix of the valid window. Every start s < left is invalid: those starts were excluded when their windows had more than K distinct values, and extending the right endpoint cannot reduce the distinct count. Thus there are exactly r − left + 1 valid subarrays ending at r. Summing over all right endpoints counts every valid subarray once.

**Lemma 2 — A(K) − A(K−1) counts exactly K distinct values.** Every subarray with at most K distinct values belongs to exactly one of two disjoint categories: fewer than K, or exactly K. The first category is precisely the set counted by A(K−1). Subtracting removes it and leaves the second category.

By Lemma 1, both calls compute the correct at-most counts. By Lemma 2, their difference is exactly the requested number of subarrays.

## 10. Edge cases and common mistakes

**All identical:** nums = [2,2,2], k = 1 → 6, because every nonempty subarray has one distinct value.

**K exceeds the number of distinct values:** nums = [1,2], k = 3 → 0.

**K = 1:** nums = [1,1,2] → 4: [1] at index 0, [1] at index 1, [1,1], and [2].

**All distinct:** nums = [1,2,3], k = 2 → 2: [1,2] and [2,3].

**Negative integers:** nums = [-1,2,-1], k = 2 → 3. The method depends on equality, not positivity.

**Common mistakes:**

- Counting only one valid subarray per right endpoint instead of **right − left + 1**.
- Decrementing distinct every time an element is removed; decrement it **only when its frequency becomes zero**.
- Trying to maintain exactly K with one left pointer and missing duplicate-related starting positions.
- Using atMost(K) − atMost(K+1), which has the wrong sign and meaning.
- Forgetting that different index ranges count separately even when their values are identical.
- Assuming the nested while loop makes the solution O(n²); left advances at most n times over the entire pass.

## 11. Interviewer-style follow-up questions

**Q1. Why does a sliding window work for at most K but not as directly for exactly K?**

At most K is monotonic under removing elements: once a window is valid, all its suffixes are valid. Exactly K is not preserved under shrinking; removing the last copy of a value changes the distinct count.

**Q2. Why is the count for a fixed right endpoint right − left + 1?**

There are that many possible starts in the inclusive range [left, right], and all produce valid suffixes of the current window.

**Q3. Could you solve this in one pass?**

Yes. Maintain two windows simultaneously: one with at most K distinct values and one with at most K−1. For each right endpoint, the difference between their left boundaries is the number of exactly-K subarrays ending there. It is still O(n) time and O(n) space, but the two-pass version is usually easier to explain.

**Q4. What changes if the array is a stream?**

You can update the two at-most windows as each value arrives and report the cumulative count. The frequency maps and left boundaries must be maintained with access to the active window values.

**Q5. Does the algorithm require positive numbers?**

No. It only tracks whether values are equal and how many times each appears, so any hashable integers work.

**Q6. What if we need exactly K distinct characters in a string?**

The same identity and frequency-map window apply to characters. The algorithm does not depend on the input being an integer array.

**Q7. How would you count subarrays with at least K distinct values?**

Subtract atMost(K−1) from the total number of nonempty subarrays, n(n+1)/2.

**Q8. What if K is zero?**

No nonempty subarray can contain zero distinct values. The answer is zero. With the helper's negative-limit guard, atMost(0) − atMost(−1) also gives zero.

## 12. What to say aloud in the interview

> “The brute-force approach fixes each left boundary and expands right while tracking distinct values, which is O(n²). Since the problem asks for exactly K, a direct window is awkward because several starts may work for the same right endpoint. I’ll instead use exactly(K) = atMost(K) − atMost(K−1). To compute atMost, I maintain a frequency map and shrink the left boundary whenever the distinct count exceeds the limit. Once the window is valid, every suffix ending at the current right index is valid, so I add right − left + 1. Both pointers move forward at most n times, giving O(n) expected time and O(n) worst-case auxiliary space.”

## 13. Pattern takeaway and practice

```text
Count subarrays with EXACTLY K distinct values
                       |
                       v
Brute force: fix left, expand right -> O(n²)
                       |
                       v
Exactly K is awkward for one sliding window
                       |
                       v
exactly(K) = atMost(K) - atMost(K-1)
                       |
                       v
Sliding window with frequency map
                       |
                       v
Shrink while distinct > limit
                       |
                       v
Count all valid suffixes: right - left + 1
                       |
                       v
O(n) expected time, O(n) space
```

**Practice next:** LeetCode 1248 (Count Number of Nice Subarrays) for exactly K odd numbers; 930 (Binary Subarrays With Sum) for at-most subtraction on binary arrays; 904 (Fruit Into Baskets) for at most two distinct values; and 76 (Minimum Window Substring) to contrast a minimum-covering window with a counting window.

**Transferable interview insight:** Whenever you see **exactly K** in a subarray-counting problem, ask whether **at most K** is easier to maintain. If it is, the difference between two cumulative counts may convert a difficult exact condition into two straightforward linear scans.
