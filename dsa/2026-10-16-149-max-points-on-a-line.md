# DSA Interview Walkthrough — LeetCode 149: Max Points on a Line

**Difficulty:** Hard  
**Patterns:** Computational Geometry · Hash Maps · GCD · Slope Normalization · Counting  
**Problem:** Max Points on a Line (LeetCode 149)

**Core lesson:** When a problem asks for the largest group of points sharing a geometric property, choose one point as an anchor and group the remaining points by that property. For collinearity, the property is slope—but represent slopes as reduced integer pairs, not floating-point numbers.

## 1. Interviewer: Problem statement

**Interviewer:** You are given an array of distinct points on a 2D plane. Return the maximum number of points lying on the same straight line.

~~~text
points = [[1,1], [2,2], [3,3], [4,4], [1,2], [2,3]]

Answer = 4

The four points (1,1), (2,2), (3,3), and (4,4)
lie on the line y = x.
~~~

**Candidate:** Can the line be vertical or horizontal? Are negative coordinates allowed? Can two points have the same coordinates?

**Interviewer:** Vertical and horizontal lines count, negative coordinates are allowed, and all input points are distinct.

**Candidate:** Then I need the largest collinear subset. I cannot assume the answer is a line with a finite slope, and I should avoid floating-point precision problems.

## 2. Recognize the pattern

The keywords are **points**, **same straight line**, and **maximum count**. This is a geometry problem, but the optimization is a counting problem.

A line is determined by any two distinct points. That immediately gives a brute-force idea: try every pair as a candidate line, then count how many points lie on it.

The important optimization is to stop treating every pair as an unrelated line. If I fix one anchor point A, all other points lying on the same line through A have the **same slope relative to A**.

So the plan becomes:

1. Fix an anchor point.
2. Compute the direction from the anchor to every later point.
3. Count equal directions with a hash map.
4. The largest count plus the anchor gives the best line through that anchor.
5. Repeat for all anchors.

**Interviewer:** Why not use a set of points?

**Candidate:** Membership does not reveal collinearity. I need a canonical key representing a line's direction from the current anchor.

## 3. Brute force: choose every pair of points

For every pair A and B, count all points P satisfying the collinearity test.

Three points A=(x1,y1), B=(x2,y2), and P=(x,y) are collinear exactly when their 2D cross product is zero:

~~~text
(x - x1) * (y2 - y1) == (y - y1) * (x2 - x1)
~~~

This avoids division entirely and works for vertical lines.

~~~python
def brute(points):
    n = len(points)
    if n <= 2:
        return n

    best = 2

    for i in range(n):
        x1, y1 = points[i]
        for j in range(i + 1, n):
            x2, y2 = points[j]
            count = 0

            for x, y in points:
                if (x - x1) * (y2 - y1) == (y - y1) * (x2 - x1):
                    count += 1

            best = max(best, count)

    return best
~~~

There are O(n²) pairs, and each pair scans n points.

**Time:** O(n³).  
**Auxiliary space:** O(1).

**Interviewer:** What is being repeated?

**Candidate:** Many different pairs describe the same line. I repeatedly count points on a line that I have effectively seen before.

## 4. First optimization: fix an anchor and count slopes

Suppose the anchor is A=(1,1). For each other point B=(x2,y2), the slope is:

~~~text
slope = (y2 - 1) / (x2 - 1)
~~~

All points on the same line through A share the same slope. A hash map can count them in O(1) expected time per point.

This suggests O(n²) expected time: n anchors, and up to n-1 slope calculations per anchor.

But using a floating-point slope directly is fragile.

**Problem 1 — vertical lines:** If x2 == x1, the denominator is zero.

**Problem 2 — precision:** Two mathematically equal rational slopes should always produce the same key. Floating-point division is an unnecessary source of rounding and equality problems.

**Problem 3 — direction signs:** The vectors (1,1) and (-1,-1) point in opposite directions but lie on the same line through the anchor.

We need a **canonical integer representation** of slope.

## 5. The special technique: normalize with GCD

For two distinct points, define:

~~~text
dx = x2 - x1
dy = y2 - y1
g  = gcd(dx, dy)

dx = dx // g
dy = dy // g
~~~

Python's gcd is nonnegative. Because the points are distinct, dx and dy are not both zero, so g is positive.

Reducing by gcd makes proportional vectors identical up to sign:

~~~text
(2, 4)   -> (1, 2)
(3, 6)   -> (1, 2)
(-2, -4) -> (-1, -2)
~~~

Now standardize the sign. If dx is negative, negate both components. If dx is zero, normalize the vertical direction to dy=1.

We use the hash-map key **(dy, dx)**:

~~~text
(1, 2)   means slope 1/2
(-1, 2)  means slope -1/2
(1, 0)   means a vertical line
(0, 1)   means a horizontal line
~~~

**Interviewer:** Why does a reduced pair work better than a decimal?

**Candidate:** Two integer vectors have the same direction up to sign exactly when their reduced, sign-normalized pairs are equal. The comparison is exact, including for negative and vertical slopes.

## 6. Derive the optimized algorithm

For each anchor index i:

1. Create an empty dictionary called slopes.
2. For every point j > i, calculate dx and dy.
3. Divide both by gcd(dx,dy).
4. Normalize the sign and vertical case.
5. Increment the count for that direction.
6. Update the global answer with **count + 1**, because the anchor itself is also on the line.

Why only j > i instead of every other point? Consider an optimal line containing m points. Let i be the **smallest original index** among those m points. Every other point on that line has index greater than i, so that anchor still counts all m points. We can safely skip earlier points.

The map must be **reset for each anchor**. Equal slopes from different anchors do not necessarily describe the same line: two parallel lines can have the same slope.

## 7. Detailed dry run

~~~text
points:
0: (1,1)
1: (2,2)
2: (3,3)
3: (4,4)
4: (1,2)
5: (2,3)
~~~

Choose anchor i=0, A=(1,1).

| Other point | dx, dy | Reduced (dy, dx) | Count for slope |
|---|---|---|---:|
| (2,2) | (1,1) | (1,1) | 1 |
| (3,3) | (2,2) | (1,1) | 2 |
| (4,4) | (3,3) | (1,1) | 3 |
| (1,2) | (0,1) | (1,0) | 1 |
| (2,3) | (1,2) | (2,1) | 1 |

The largest slope frequency is 3. Including the anchor gives **3 + 1 = 4**.

**Interviewer:** Why isn't the answer 3?

**Candidate:** The dictionary counts only the other points. Every slope group also contains the anchor.

**Interviewer:** What if the anchor were (2,2) and another point were (1,1)?

**Candidate:** The vector would be (-1,-1). After sign normalization it becomes (1,1), the same line direction as (3,3)-(2,2).

## 8. Clean final LeetCode solution — Python

~~~python
from math import gcd

class Solution:
    def maxPoints(self, points: list[list[int]]) -> int:
        n = len(points)
        best = min(n, 2)

        for i in range(n):
            slopes = {}
            x1, y1 = points[i]

            for j in range(i + 1, n):
                x2, y2 = points[j]
                dx, dy = x2 - x1, y2 - y1
                g = gcd(dx, dy)
                dx, dy = dx // g, dy // g

                if dx < 0:
                    dx, dy = -dx, -dy
                elif dx == 0:
                    dy = 1

                key = (dy, dx)
                slopes[key] = slopes.get(key, 0) + 1
                best = max(best, slopes[key] + 1)

        return best
~~~

The solution uses **no lambda functions**, no floating-point division, and does not mutate the input. It handles vertical lines, horizontal lines, negative coordinates, and one- or two-point inputs.

## 9. Complexity analysis

Let n be the number of points and C the largest absolute difference between two input coordinates.

There are n(n-1)/2 anchor-to-point comparisons. Each comparison performs one gcd calculation and an expected O(1) dictionary update.

**Time:** O(n² log C) with the usual Euclidean-algorithm bound for gcd; under the problem's bounded coordinate constraints, this is conventionally reported as **O(n²)**.

**Auxiliary space:** O(n), because one anchor's slope dictionary can hold at most n-1 entries.

Compared with brute force, we have removed an entire factor of n.

## 10. Correctness proof

**Lemma 1 — normalized keys identify exactly the same line through a fixed anchor.** For two distinct points B and C, the vectors AB and AC are collinear if and only if one is a nonzero scalar multiple of the other. Reducing their integer components by gcd and fixing the sign produces identical pairs exactly in this case, including vertical and horizontal directions.

**Lemma 2 — each recorded count gives a valid collinear group.** For a fixed anchor A, all points assigned to the same normalized key lie on the same line through A by Lemma 1. Therefore a count of c corresponds to c+1 collinear points including A. The algorithm never records an impossible group size.

**Lemma 3 — an optimal group is considered.** Take any line containing the maximum possible number m of input points. Choose the point on that line with the smallest original index i. Every other point on that line is processed in the inner loop for anchor i. By Lemma 1, all m-1 points share one key, so the algorithm records a group of size m.

By Lemmas 2 and 3, the result is neither larger nor smaller than the true optimum. Therefore the algorithm returns the maximum number of collinear points.

## 11. Edge cases and common mistakes

| Points | Answer | Reason |
|---|---:|---|
| [[1,1]] | 1 | Single point |
| [[1,1],[2,3]] | 2 | Any two distinct points define a line |
| [[0,0],[0,1],[0,2]] | 3 | Vertical line |
| [[0,0],[1,0],[2,0]] | 3 | Horizontal line |
| [[-1,-1],[0,0],[1,1]] | 3 | Opposite directions, same line |
| [[0,0],[1,1],[2,2],[1,2]] | 3 | Three collinear points |
| [[0,0],[1,1],[2,2],[0,1],[1,2],[2,3]] | 3 | Parallel lines must not be merged |

**Common mistakes:** Using float slopes as dictionary keys; dividing by zero on vertical lines; forgetting to normalize negative directions; confusing (dx,dy) with (dy,dx) between calculations; omitting the anchor from the count; reusing the same slope dictionary across anchors; and forgetting that points are guaranteed distinct in this problem.

## 12. Interviewer-style follow-up questions

**Q1. Why is a hash map useful?** For one anchor, the problem becomes counting equal normalized directions. A dictionary groups them in expected constant time per point.

**Q2. Why not sort points first?** Sorting is unnecessary. The algorithm relies only on original indices to avoid duplicate pair processing, not on coordinate order.

**Q3. Why is gcd necessary?** Without reduction, vectors (1,2) and (2,4) would produce different keys even though they describe the same slope.

**Q4. Why normalize the sign?** Vectors (1,1) and (-1,-1) are opposite rays of the same infinite line. They must share one key.

**Q5. Why reset the dictionary for every anchor?** A slope alone does not uniquely identify a line globally. Different parallel lines share slopes, but lines through one fixed anchor are uniquely determined by slope.

**Q6. What if duplicate coordinates were allowed?** A zero vector has no defined slope. Count duplicates separately for each anchor and add that duplicate count to every line through the anchor. The original problem guarantees distinct points.

**Q7. What if the problem asks for the actual points on the best line?** Remember the best anchor and slope key, then scan all points and collect those with the same normalized direction (or test collinearity by cross product).

**Q8. Can the algorithm beat O(n²)?** This pairwise-anchor solution is the standard practical approach for the given constraints. Specialized assumptions may allow other methods, but there is no comparable simple general improvement.

**Q9. Why not use y = mx + b?** Vertical lines have no finite m, and floating-point slope/intercept comparisons are fragile. An anchor plus an exact normalized direction is simpler.

## 13. What to say aloud in the interview

> “Brute force picks every pair of points to define a line and scans all points for collinearity, taking O(n³). To optimize, I'll fix one anchor and group every other point by its slope from that anchor. Equal slopes from the same anchor mean the points lie on the same line. I won't use floating-point division; I'll represent the slope as the reduced integer pair (dy, dx), using gcd and sign normalization. For each anchor, a hash map counts matching slopes, and the best group size is the maximum frequency plus one for the anchor. That gives O(n²) time under bounded coordinates and O(n) space.”

## 14. Pattern takeaway and practice

~~~text
Largest group of points sharing a straight line
                         |
                         v
Brute force: every pair defines a candidate line
                         |
                         v
For each pair, scan all points: O(n³)
                         |
                         v
Fix one anchor; equal slopes mean one line
                         |
                         v
Normalize slope with gcd and sign convention
                         |
                         v
Hash map: normalized slope -> frequency
                         |
                         v
Add 1 for the anchor and maximize
                         |
                         v
O(n²) practical time, O(n) space
~~~

**Practice next:** LeetCode 356 (Line Reflection) for geometric symmetry with hash sets; LeetCode 447 (Number of Boomerangs) for distance-frequency counting per anchor; LeetCode 939 (Minimum Area Rectangle) for coordinate-pair hashing; and LeetCode 587 (Erect the Fence) for cross products and convex hulls.

**Transferable interview insight:** When the problem asks for the largest set of items sharing a relation, try fixing one item and converting the relation into a **canonical hashable key**. For geometry, exact integer representations often make both correctness and implementation much cleaner.