# DSA Interview Walkthrough — LeetCode 3: Longest Substring Without Repeating Characters

**Difficulty:** Medium  
**Pattern:** Sliding Window · Hash Map

## Problem
Given a string `s`, return the length of the longest contiguous substring containing no repeated characters.

```text
s = "abcabcbb"
answer = 3
```

## Recognition
This is a longest contiguous range satisfying a local validity condition. That strongly suggests a variable-size sliding window.

Maintain a window `[left, right]` that contains only unique characters.

## Brute force
Try every substring and test whether all characters are unique. A straightforward version is `O(n^3)`; keeping a set per start can reduce it to `O(n^2)`.

## Sliding-window optimization
Store the most recent index of every character.

When character `ch` appears at `right` and was previously seen inside the current window, move `left` just past that old occurrence:

```text
left = max(left, last_seen[ch] + 1)
```

The `max` is critical: a character last seen *before* the current window must not move the left pointer backward.

## Dry run: "abba"

- `a`: window `a`, best 1.
- `b`: window `ab`, best 2.
- second `b`: move left from 0 to 2.
- final `a`: its old index 0 is before left, so keep left at 2.

Answer remains 2.

## Clean final Python solution

```python
class Solution:
    def lengthOfLongestSubstring(self, s: str) -> int:
        last_seen = {}
        left = 0
        best = 0

        for right, ch in enumerate(s):
            if ch in last_seen:
                left = max(left, last_seen[ch] + 1)

            last_seen[ch] = right
            best = max(best, right - left + 1)

        return best
```

No lambda functions are used.

## Complexity
- Time: `O(n)`.
- Space: `O(min(n, alphabet size))`.

## Correctness
At every step, `left` is the earliest position such that the window ending at `right` contains no duplicate caused by the current character. Because `left` never moves backward, the maintained window is valid. Evaluating its length at every `right` considers the longest valid substring ending at each position, so the maximum is globally optimal.

## Edge cases and common mistakes
- Empty string -> 0.
- All identical characters -> 1.
- All unique -> length of string.

Common mistakes:
- using `left = last_seen[ch] + 1` without `max`,
- confusing substring with subsequence,
- clearing the whole window on a duplicate instead of moving only as far as needed.

## Interview follow-ups
- At most K distinct characters? Track frequencies and shrink while distinct count exceeds K.
- Longest substring with at most two distinct? Same pattern with K=2.
- Need the substring itself? Track the best start and length.

## What to say aloud
> “I keep a valid window of unique characters. For each right pointer, I use the last-seen index of the current character. If that occurrence is inside the current window, I move left just beyond it. I never move left backward. Each character is processed once, giving O(n) time.”

## Pattern takeaway
```text
Longest contiguous valid range -> variable sliding window -> move left only enough to restore validity
```