# DSA Interview Walkthrough — LeetCode 76: Minimum Window Substring

**Difficulty:** Hard  
**Pattern:** Variable Sliding Window · Frequency Map

## Problem
Given strings `s` and `t`, return the minimum substring of `s` containing every character of `t`, including duplicate requirements. Return `""` if impossible.

```text
s = "ADOBECODEBANC"
t = "ABC"
answer = "BANC"
```

## Recognition
This is the smallest contiguous range satisfying a coverage constraint.

Sliding-window template:
1. expand right until the window is valid,
2. shrink left while it remains valid,
3. record the smallest valid window.

## Brute force
Enumerate every substring and compare its character counts against `t`. Even with prefix counts this is at least quadratic in the number of substrings.

## Frequency-map state
Let `need[ch]` be how many copies of each character are required.

Let `window[ch]` count characters currently inside the window.

Instead of repeatedly comparing whole maps, track:
- `required = number of distinct characters in need`,
- `formed = number of distinct required characters whose needed frequency is currently satisfied`.

When `formed == required`, the window is valid.

## Clean final Python solution

```python
from collections import Counter, defaultdict

class Solution:
    def minWindow(self, s: str, t: str) -> str:
        if not s or not t:
            return ""

        need = Counter(t)
        window = defaultdict(int)
        required = len(need)
        formed = 0
        left = 0

        best_len = float("inf")
        best_left = 0

        for right, ch in enumerate(s):
            window[ch] += 1

            if ch in need and window[ch] == need[ch]:
                formed += 1

            while formed == required:
                length = right - left + 1
                if length < best_len:
                    best_len = length
                    best_left = left

                left_ch = s[left]

                if left_ch in need and window[left_ch] == need[left_ch]:
                    formed -= 1

                window[left_ch] -= 1
                left += 1

        if best_len == float("inf"):
            return ""

        return s[best_left:best_left + best_len]
```

No lambda functions are used.

## Why the order while shrinking matters
Before decrementing `window[left_ch]`, check whether that character is exactly at its required count. If so, removing it will make the window invalid, so decrement `formed`.

## Complexity
Each pointer moves only forward and visits each position at most once.

- Time: `O(len(s) + len(t))` expected.
- Space: `O(alphabet size)` / `O(number of distinct characters)`.

## Correctness
Whenever `formed == required`, all required frequencies are satisfied. The inner loop removes left characters until one more removal would violate a requirement, so it explores the shortest valid window for the current right boundary. Considering every right boundary and recording the smallest valid window yields the global minimum.

## Common mistakes
- Tracking only character presence rather than frequencies, which fails for duplicate characters in `t`.
- Comparing whole dictionaries every iteration.
- Updating `formed` when a count merely exceeds the requirement.
- Shrinking before recording a valid answer.
- Forgetting the impossible case.

## Interview follow-ups
- Find an anagram window of fixed length? Same frequency ideas but fixed-size window.
- Longest substring with at most K distinct? Shrink while distinct count exceeds K.
- Why is this O(n) despite nested loops? Left and right each advance at most n times.

## What to say aloud
> “I expand the right pointer until all required character frequencies are satisfied, then shrink from the left as aggressively as possible while preserving validity. I track how many distinct required characters are fully satisfied so I do not compare frequency maps repeatedly. Since both pointers only move forward, the solution is linear.”

## Pattern takeaway
```text
Minimum valid contiguous range -> expand until valid -> shrink while valid -> track frequencies
```