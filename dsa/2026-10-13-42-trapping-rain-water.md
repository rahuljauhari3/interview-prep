# DSA Interview Walkthrough — LeetCode 42: Trapping Rain Water

**Difficulty:** Hard  
**Pattern:** Two Pointers · Prefix/Suffix Maxima · Space Optimization  
**Core lesson:** Water above a bar is limited by the shorter of the tallest walls on its left and right. Once one running boundary is known to be smaller, that side can be finalized immediately.

## 1. Interviewer prompt

Given nonnegative bar heights of width one, return how many units of rain water are trapped.

```text
height = [4,2,0,3,2,5]
answer = 9
```

## 2. Recognize the pattern

For each index `i`:

```text
water[i] = min(left_max[i], right_max[i]) - height[i]
```

The shorter boundary determines the spill level.

## 3. Brute force

For every index, scan left for the tallest wall and right for the tallest wall.

Time: `O(n^2)`.

## 4. Prefix/suffix arrays

Precompute:
- `left[i] = max(height[0..i])`
- `right[i] = max(height[i..n-1])`

Then sum `min(left[i], right[i]) - height[i]`.

Time: `O(n)`. Space: `O(n)`.

## 5. Two-pointer insight

Maintain `left`, `right`, `left_max`, and `right_max`.

If `left_max <= right_max`, the left position's water is already determined by `left_max`: we know a right wall at least that tall exists. So add `left_max - height[left]` and move left.

Otherwise finalize the right side symmetrically.

## 6. Clean final Python solution

```python
class Solution:
    def trap(self, height: list[int]) -> int:
        left, right = 0, len(height) - 1
        left_max = right_max = 0
        water = 0

        while left <= right:
            left_max = max(left_max, height[left])
            right_max = max(right_max, height[right])

            if left_max <= right_max:
                water += left_max - height[left]
                left += 1
            else:
                water += right_max - height[right]
                right -= 1

        return water
```

No lambda functions are used.

## 7. Complexity

- Time: `O(n)`.
- Auxiliary space: `O(1)`.

## 8. Correctness

When `left_max <= right_max`, the true right-side maximum for the current left position is at least `right_max`, so at least `left_max`. Therefore the shorter limiting boundary is exactly `left_max` and the current left position can be finalized. The symmetric argument applies to the right side.

## 9. Edge cases and mistakes

- Empty, one-bar, or two-bar input -> 0.
- Strictly increasing/decreasing -> 0.
- `[3,0,3]` -> 3.

Common mistakes:
- using `max(left_max, right_max)` instead of `min`,
- assuming one tall wall is enough,
- mixing different two-pointer invariants,
- counting only local minima.

## 10. Interviewer follow-ups

**Monotonic-stack solution?** Pop basin bottoms when a taller right wall arrives; calculate bounded height and width.

**Difference from Container With Most Water?** That problem chooses two endpoints and maximizes one rectangle; this problem sums water over all positions.

**2D version?** Use a boundary min-heap and flood-fill style expansion (LeetCode 407).

## 11. What to say aloud

> “At each index, trapped water equals the shorter of the best left and right walls minus the bar height. Prefix/suffix arrays give O(n) time and O(n) space. To remove the arrays, I use two pointers with running maxima. If the left maximum is smaller, I already know the right side has a sufficient wall, so the left position's water is final. Otherwise I process the right. Every position is finalized once, so the solution is O(n) time and O(1) space.”

## 12. Pattern takeaway

```text
Each position depends on left and right summaries
                 ↓
Prefix/suffix maxima
                 ↓
Smaller boundary determines water
                 ↓
Two pointers + running maxima
                 ↓
O(n) time, O(1) space
```

**Practice next:** LeetCode 11, 238, 84, and 407.