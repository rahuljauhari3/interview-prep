# DSA Interview Walkthrough — LeetCode 312: Burst Balloons

**Difficulty:** Hard  
**Pattern:** Interval Dynamic Programming  
**Core lesson:** When deleting elements changes adjacency and makes forward choices messy, reverse the perspective: choose which element is processed **last** inside an interval.

## 1. Interviewer prompt

You have balloons with values `nums`. Bursting balloon `i` earns:

```text
nums[left] * nums[i] * nums[right]
```

where `left` and `right` are the adjacent remaining balloons at that moment. Return the maximum coins obtainable by bursting all balloons.

```text
nums = [3,1,5,8]
answer = 167
```

Missing neighbors behave as value 1.

## 2. Why the obvious state is hard

If we choose the **first** balloon to burst, its future neighbors depend on which other balloons are removed later. That makes the state depend on a complicated remaining set.

The key inversion is:

> Instead of asking which balloon is burst first, choose which balloon is burst **last** inside an interval.

If balloon `k` is last inside an interval, then every balloon between the interval boundaries except `k` is already gone. Therefore the neighbors of `k` at that moment are known exactly: the interval boundaries.

## 3. Add sentinel boundaries

Create:

```text
balloons = [1] + nums + [1]
```

Let `dp[left][right]` be the maximum coins obtainable by bursting all balloons **strictly between** `left` and `right`, while the boundary balloons remain.

## 4. Recurrence

Choose `k` as the last balloon burst inside `(left,right)`.

Before that final burst:
- all balloons in `(left,k)` are already gone,
- all balloons in `(k,right)` are already gone.

Those subproblems are independent.

So:

```text
dp[left][right] = max over k in (left,right):
    dp[left][k]
    + dp[k][right]
    + balloons[left] * balloons[k] * balloons[right]
```

Base case: if `left + 1 == right`, the interval contains no balloon, so value is 0.

## 5. Memoized solution

```python
from functools import cache

class Solution:
    def maxCoins(self, nums: list[int]) -> int:
        balloons = [1] + nums + [1]

        @cache
        def solve(left, right):
            if left + 1 == right:
                return 0

            best = 0

            for k in range(left + 1, right):
                coins = (
                    solve(left, k)
                    + solve(k, right)
                    + balloons[left] * balloons[k] * balloons[right]
                )
                best = max(best, coins)

            return best

        return solve(0, len(balloons) - 1)
```

No lambda functions are used.

## 6. Bottom-up interval DP

Intervals must be solved from smaller to larger.

```python
class Solution:
    def maxCoins(self, nums: list[int]) -> int:
        balloons = [1] + nums + [1]
        n = len(balloons)
        dp = [[0] * n for _ in range(n)]

        for length in range(2, n):
            for left in range(n - length):
                right = left + length

                for k in range(left + 1, right):
                    dp[left][right] = max(
                        dp[left][right],
                        dp[left][k]
                        + dp[k][right]
                        + balloons[left] * balloons[k] * balloons[right]
                    )

        return dp[0][n - 1]
```

## 7. Complexity

There are `O(n^2)` intervals and up to `O(n)` choices for the last balloon in each.

- Time: `O(n^3)`.
- Space: `O(n^2)`.

## 8. Correctness

For any optimal bursting order inside `(left,right)`, exactly one balloon `k` is burst last. At that moment its neighbors are necessarily the fixed boundaries `left` and `right`, so its final contribution is known. All earlier bursts occur entirely in the independent left and right subintervals, whose optimal values are captured by `dp[left][k]` and `dp[k][right]`. Trying every possible final `k` therefore includes the optimal order.

## 9. Edge cases and common mistakes

- Empty `nums` -> 0.
- Single balloon -> its value with sentinel neighbors 1.
- Zeros are handled naturally.

Common mistakes:
- choosing the first balloon and then struggling with changing neighbors,
- defining interval boundaries inconsistently,
- forgetting sentinel 1s,
- filling bottom-up intervals in the wrong order,
- accidentally including the boundary balloons in the subproblem.

## 10. Interviewer follow-ups

**Why is “choose last” powerful?** It freezes adjacency: when the final action occurs, the surviving boundaries are known.

**What other problems use this idea?** Matrix Chain Multiplication and some interval-removal games.

**Can this be optimized below O(n³)?** Not with the standard recurrence under typical constraints; the cubic DP is the expected interview solution.

## 11. What to say aloud

> “The difficulty is that bursting a balloon changes who its neighbors are. Choosing the first balloon creates a messy state, so I reverse the perspective and choose the balloon burst last in each interval. If k is last between fixed boundaries left and right, its final coins are known, and everything before that splits into two independent interval subproblems. That gives an O(n³) interval DP with O(n²) states.”

## 12. Pattern takeaway

```text
Removing elements changes adjacency
            ↓
Forward state is messy
            ↓
Choose final operation in interval
            ↓
Boundaries become fixed
            ↓
Split into left/right interval DP
```

**Practice next:** Matrix Chain Multiplication-style DP, LeetCode 1039, and 1547.