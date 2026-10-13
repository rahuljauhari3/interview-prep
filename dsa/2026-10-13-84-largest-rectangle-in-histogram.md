# DSA Interview Walkthrough — LeetCode 84: Largest Rectangle in Histogram

**Difficulty:** Hard  
**Pattern:** Monotonic Increasing Stack  
**Core lesson:** For each bar, the largest rectangle using that bar as the limiting height extends until the first smaller bar on each side. A monotonic stack discovers those boundaries in one pass.

## Problem
Given histogram bar heights, return the largest rectangular area.

```text
heights = [2,1,5,6,2,3]
answer = 10
```

## Recognition
If bar `i` is the rectangle's height, we need to know how far left and right we can extend before hitting a shorter bar.

Brute force can expand from every bar, but that repeats work.

## Brute force
For every index, expand left and right while neighboring heights are at least the current height.

Worst-case time: `O(n^2)`.

## Monotonic-stack insight
Maintain indices of bars in nondecreasing height order.

When a new bar is **shorter** than the bar at the stack top, the current index is the first smaller bar on the right for that popped bar.

After popping index `j`:
- `height = heights[j]`
- right boundary is current index `i`
- left boundary is the new stack top

Width is:

```text
if stack:
    width = i - stack[-1] - 1
else:
    width = i
```

Append a sentinel height 0 conceptually at the end so every remaining bar is popped.

## Dry run intuition
For `[2,1,5,6,2,3]`:
- 5 and 6 stay on the stack while heights rise.
- When height 2 arrives, 6 pops first, then 5.
- The rectangle using height 5 spans bars `[5,6]`, width 2, area 10.

## Clean final Python solution

```python
class Solution:
    def largestRectangleArea(self, heights: list[int]) -> int:
        stack = []
        best = 0

        for i in range(len(heights) + 1):
            current = heights[i] if i < len(heights) else 0

            while stack and heights[stack[-1]] > current:
                height = heights[stack.pop()]

                if stack:
                    width = i - stack[-1] - 1
                else:
                    width = i

                best = max(best, height * width)

            stack.append(i)

        return best
```

No lambda functions are used.

## Complexity
Each index is pushed once and popped once.

- Time: `O(n)`.
- Space: `O(n)`.

## Correctness
When index `j` is popped, the current index `i` is the first position to its right with smaller height. After the pop, the new stack top is the nearest index to its left with smaller height. Therefore `j`'s maximum-width rectangle at height `heights[j]` is determined exactly at that moment. Since every bar is eventually popped, every possible limiting height is evaluated.

## Common mistakes
- Forgetting the final sentinel and leaving bars unprocessed.
- Using the wrong width formula after popping.
- Storing heights instead of indices, which loses boundary positions.
- Assuming nested stack loops make runtime O(n²).

## Interview follow-ups
- Maximal Rectangle in a binary matrix? Build histogram heights row by row and reuse this algorithm.
- Why use `>` instead of `>=`? Both conventions can work, but duplicate-height handling and boundary semantics must remain consistent.
- Can left/right smaller arrays solve it? Yes, in O(n) time and O(n) space with two monotonic passes.

## What to say aloud
> “For each bar, the best rectangle using that bar as the limiting height extends until the first shorter bar on both sides. I maintain indices in a monotonic increasing stack. When a shorter bar arrives, it closes the right boundary for taller bars, so I pop them and compute their maximum widths. A final zero-height sentinel flushes the stack. Each index is pushed and popped once, so the solution is O(n).”

## Pattern takeaway
```text
Need nearest smaller boundaries
           ↓
Monotonic increasing stack
           ↓
Smaller incoming value finalizes taller bars
           ↓
O(n)
```

**Practice next:** LeetCode 85, 739, and 907.