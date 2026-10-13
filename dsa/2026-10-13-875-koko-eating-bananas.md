# DSA Interview Walkthrough — LeetCode 875: Koko Eating Bananas

**Difficulty:** Medium  
**Pattern:** Binary Search on the Answer  
**Core lesson:** When you need the minimum integer value satisfying a monotonic condition, binary-search the answer space rather than the input array.

## 1. Interviewer prompt

Koko has banana piles and `h` hours. She chooses an integer eating speed `k` bananas per hour. In one hour she eats from only one pile; if a pile has fewer than `k`, she finishes it and waits for the next hour. Return the minimum speed needed to finish all piles within `h` hours.

```text
piles = [3,6,7,11]
h = 8
answer = 4
```

## 2. Recognize the pattern

For a proposed speed `k`, the hours required are:

```text
sum(ceil(pile / k))
```

As `k` increases, required hours never increase. Therefore feasibility is monotonic:

```text
False False ... False True True ... True
```

We need the first feasible speed.

## 3. Brute force

Try every speed from `1` through `max(piles)` and return the first one that works.

If `M = max(piles)` and there are `n` piles:

- Time: `O(nM)`.
- Space: `O(1)`.

## 4. Feasibility test

Use integer ceiling division without floating point:

```text
ceil(pile / k) = (pile + k - 1) // k
```

```python
def can_finish(piles, h, k):
    hours = 0

    for pile in piles:
        hours += (pile + k - 1) // k
        if hours > h:
            return False

    return True
```

## 5. Binary-search bounds

- Minimum possible speed: 1.
- Maximum necessary speed: `max(piles)`; at that speed each pile takes at most one hour.

Use lower-bound binary search for the first speed where `can_finish` is true.

## 6. Dry run

For `[3,6,7,11]`, `h=8`:

```text
left=1, right=11
mid=6  -> feasible
left=1, right=6
mid=3  -> not feasible
left=4, right=6
mid=5  -> feasible
left=4, right=5
mid=4  -> feasible
answer=4
```

## 7. Clean final Python solution

```python
class Solution:
    def minEatingSpeed(self, piles: list[int], h: int) -> int:
        def can_finish(k):
            hours = 0

            for pile in piles:
                hours += (pile + k - 1) // k
                if hours > h:
                    return False

            return True

        left, right = 1, max(piles)

        while left < right:
            mid = (left + right) // 2

            if can_finish(mid):
                right = mid
            else:
                left = mid + 1

        return left
```

No lambda functions are used.

## 8. Complexity

- Each feasibility check: `O(n)`.
- Number of checks: `O(log M)`.
- Total time: `O(n log M)`.
- Space: `O(1)`.

## 9. Correctness

If a speed `k` finishes within `h` hours, every larger speed also finishes within `h` hours because no pile can require more hours at a faster speed. Therefore feasibility is monotonic. Binary search maintains the invariant that the minimum feasible speed remains inside `[left,right]` and converges to the first feasible value.

## 10. Edge cases and mistakes

- `h == len(piles)` -> answer is `max(piles)`.
- One pile.
- Very large pile values.

Common mistakes:
- using floor division instead of ceiling division,
- binary-searching pile indices rather than speeds,
- returning an arbitrary feasible speed instead of the minimum,
- using `right = mid - 1` with the wrong lower-bound invariant.

## 11. Interviewer follow-ups

**Why does binary search work on a value domain?** It only needs an ordered domain and monotonic predicate.

**Could right be larger than max(piles)?** Yes, but unnecessary.

**Related problems?** Ship Packages Within D Days, Split Array Largest Sum, and Minimum Days to Make Bouquets.

## 12. What to say aloud

> “For a candidate speed k, I can compute required hours in O(n) using ceiling division. Faster speeds can never require more hours, so feasibility is monotonic. I binary-search speeds from 1 to max pile and look for the first feasible speed. That reduces brute-force O(nM) to O(n log M).”

## 13. Pattern takeaway

```text
Minimum value satisfying condition
           ↓
Can test candidate efficiently
           ↓
Feasibility is monotonic
           ↓
Binary search first True
```

**Practice next:** LeetCode 1011, 410, and 1482.