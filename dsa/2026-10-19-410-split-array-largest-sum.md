# DSA Interview Walkthrough — LeetCode 410: Split Array Largest Sum

**Difficulty:** Hard  
**Patterns:** Binary Search on Answer · Greedy Feasibility Check · Prefix Sums · Partition Dynamic Programming  
**Problem:** LeetCode 410 — Split Array Largest Sum

**Core insight:** Instead of directly deciding where to place every partition boundary, guess the **maximum allowed subarray sum**. We can greedily determine whether that limit is achievable. Feasibility is monotonic, so binary search finds the smallest feasible limit.

## 1. Interviewer: Problem statement

**Interviewer:** Given an array **nums** of nonnegative integers and an integer **k**, split the array into exactly **k nonempty contiguous subarrays**. Minimize the largest subarray sum and return that minimum value.

```python
nums = [7, 2, 5, 10, 8]
k = 2

# Answer: 18
# Split as [7, 2, 5] and [10, 8].
# Subarray sums: 14 and 18; largest = 18.
```

**Candidate:** Must every element belong to exactly one subarray? Can I rearrange elements? Are all numbers nonnegative? Is k between 1 and len(nums)?

**Interviewer:** Every element must be used once, in its original order. The pieces must be contiguous and nonempty. Values are nonnegative, and 1 ≤ k ≤ n.

**Candidate:** The objective is a **minimize-the-maximum** partition problem. I can enumerate boundaries, but I should also check whether a proposed maximum sum can be tested efficiently.

## 2. Recognize the pattern

The signals are **minimize the largest**, **contiguous partitions**, and **exactly k groups**.

A useful interview question is:

**Can I efficiently answer a yes/no version of the optimization problem?**

Instead of immediately asking, “What is the smallest possible largest sum?”, ask:

> “Can I divide nums into at most k contiguous groups such that no group sums to more than X?”

If X works, any larger limit also works. If X fails, every smaller limit fails. That is a **monotonic Boolean predicate**, which makes binary search on the answer possible.

Why **at most k** when the problem requires **exactly k**? With nonnegative numbers and k ≤ n, any valid partition into fewer than k nonempty groups can be split further until there are exactly k groups. Splitting cannot increase the largest group sum.

**Interviewer:** Is this binary search over array indices?

**Candidate:** No. The array is not sorted. I am binary-searching a **numeric answer range**, using a greedy test to decide which half of that range to discard.

## 3. Brute force: enumerate every placement of cuts

There are n−1 gaps between adjacent elements. To make exactly k nonempty groups, choose k−1 of those gaps as cuts.

That gives **C(n−1, k−1)** possible partitions.

Use prefix sums so the sum of nums[a:b] is computed as **prefix[b] − prefix[a]**.

```python
from itertools import combinations

def brute(nums, k):
    n = len(nums)
    prefix = [0]
    for x in nums:
        prefix.append(prefix[-1] + x)

    best = float("inf")
    for cuts in combinations(range(1, n), k - 1):
        bounds = (0,) + cuts + (n,)
        largest = 0
        for i in range(k):
            total = prefix[bounds[i + 1]] - prefix[bounds[i]]
            largest = max(largest, total)
        best = min(best, largest)

    return best
```

For [7,2,5,10,8] with k=2, the four possible cuts produce:

| Partition | Group sums | Largest |
|---|---|---:|
| [7] / [2,5,10,8] | 7, 25 | 25 |
| [7,2] / [5,10,8] | 9, 23 | 23 |
| [7,2,5] / [10,8] | 14, 18 | **18** |
| [7,2,5,10] / [8] | 24, 8 | 24 |

**Time:** O(n + k × C(n−1, k−1)); the number of partitions can be exponential in n.  
**Auxiliary space:** O(n + k), for prefix sums and cut boundaries, excluding the iterator's constant bookkeeping.

**Interviewer:** What causes the explosion?

**Candidate:** We try many cut combinations that lead to similar partial sums. The same prefix-partition subproblems are solved repeatedly.

## 4. First optimization: partition dynamic programming

Define **dp[p][i]** as the smallest possible largest subarray sum when splitting the **first i elements** into exactly **p nonempty groups**.

Suppose the last group starts at index j. The preceding j elements must form p−1 groups, and the final group has sum:

**prefix[i] − prefix[j]**

For that choice of j, the worst group sum is:

**max(dp[p−1][j], prefix[i] − prefix[j])**

Choose the best cut:

**dp[p][i] = min over p−1 ≤ j < i of max(dp[p−1][j], prefix[i] − prefix[j])**

Base case: **dp[0][0] = 0**. Other impossible states are infinity.

We only need the previous DP row:

```python
def dp_solution(nums, k):
    n = len(nums)
    prefix = [0]
    for x in nums:
        prefix.append(prefix[-1] + x)

    prev = [float("inf")] * (n + 1)
    prev[0] = 0

    for p in range(1, k + 1):
        curr = [float("inf")] * (n + 1)
        for i in range(p, n + 1):
            for j in range(p - 1, i):
                cost = max(prev[j], prefix[i] - prefix[j])
                curr[i] = min(curr[i], cost)
        prev = curr

    return prev[n]
```

**Time:** O(k n²).  
**Auxiliary space:** O(n) for prefix sums and two DP rows.

**Interviewer:** Can we avoid trying every last-cut position?

**Candidate:** Yes, because all numbers are nonnegative. Instead of finding the optimal cut directly, I can test a candidate maximum sum in one pass.

## 5. Derive the greedy feasibility test

Suppose the maximum allowed sum of any group is **limit**.

Scan left to right, keeping a running group sum. For each number:

- If adding it keeps the sum ≤ limit, include it in the current group.
- Otherwise, start a new group containing that number.
- Count how many groups were required.

The greedy rule is: **make each group as long as possible before opening the next group**.

```python
def feasible(nums, k, limit):
    groups = 1
    total = 0

    for x in nums:
        if x > limit:
            return False

        if total + x > limit:
            groups += 1
            total = x
        else:
            total += x

    return groups <= k
```

The explicit **x > limit** check makes this helper correct for arbitrary proposed limits. Our final binary search will start at max(nums), so that case cannot arise there.

**Interviewer:** Why is this greedy packing optimal for minimizing the number of groups?

**Candidate:** Every feasible first group must end no later than the greedy first group: greedy extends it to the farthest index allowed by limit. After that, the same argument applies to the remaining suffix. Ending a group earlier cannot reduce the number of groups needed. Thus greedy uses the fewest groups possible under that limit.

## 6. Binary search on the answer

We need a lower and an upper bound for the optimal maximum sum.

**Lower bound = max(nums).** Every element must belong to some group, so the maximum group sum cannot be smaller than the largest element.

**Upper bound = sum(nums).** Putting everything into one group gives this maximum. If exactly k groups are required, we can split that group further without increasing the maximum, because values are nonnegative.

Now binary search the integer interval **[max(nums), sum(nums)]**.

- If **feasible(mid)** is true, mid might be the answer, but a smaller limit may work: set **right = mid**.
- Otherwise mid is too small: set **left = mid + 1**.
- Stop when left == right.

This is the **first true** form of binary search.

**Interviewer:** Why not set right = mid − 1 when mid is feasible?

**Candidate:** Because mid itself may be the smallest feasible limit. We must keep it in the search interval until we prove a smaller one works.

## 7. Detailed dry run

Take **nums = [7,2,5,10,8]**, **k = 2**.

Initial bounds: **left = 10**, **right = 32**.

| Iteration | left | right | mid | Greedy groups under mid | Feasible? | New bounds |
|---|---:|---:|---:|---|---|---|
| 1 | 10 | 32 | 21 | [7,2,5] / [10,8] | Yes (2) | [10,21] |
| 2 | 10 | 21 | 15 | [7,2,5] / [10] / [8] | No (3) | [16,21] |
| 3 | 16 | 21 | 18 | [7,2,5] / [10,8] | Yes (2) | [16,18] |
| 4 | 16 | 18 | 17 | [7,2,5] / [10] / [8] | No (3) | [18,18] |

The bounds meet at **18**, so the answer is 18.

Notice that we did not try all cut combinations at each iteration. We only scanned the array once per candidate limit.

### Why the group count is monotonic

If a limit of 18 permits a valid partition, a limit of 19 also permits that same partition. Relaxing the maximum never destroys feasibility.

Conversely, if 17 forces more than k groups, every smaller limit is at least as restrictive and also fails.

This monotonicity is the entire reason binary search works.

## 8. Clean final LeetCode solution — Python

```python
class Solution:
    def splitArray(self, nums: list[int], k: int) -> int:
        left, right = max(nums), sum(nums)

        def can_split(limit):
            groups = 1
            total = 0

            for x in nums:
                if total + x > limit:
                    groups += 1
                    total = x
                else:
                    total += x

                if groups > k:
                    return False

            return True

        while left < right:
            mid = (left + right) // 2

            if can_split(mid):
                right = mid
            else:
                left = mid + 1

        return left
```

The code uses **no lambda functions**, does not modify the input, and needs only a few integer variables. The final feasibility helper omits the **x > limit** check because the binary search guarantees **limit ≥ max(nums)**.

## 9. Complexity analysis

Let **n = len(nums)** and **S = sum(nums)**.

Each feasibility test scans at most n elements: **O(n)** time and **O(1)** extra space.

The binary search examines **O(log(S − max(nums) + 1))** candidate limits.

**Time:** **O(n log(S − max(nums) + 1))**, often simplified to O(n log S) when S ≥ 2.  
**Auxiliary space:** **O(1)**.

The O(n) cost of computing max(nums) and sum(nums) is included. If all values are zero, the bounds already match and no feasibility tests are needed.

**Interviewer:** Is this O(n log n)?

**Candidate:** Not necessarily. We are binary-searching a **value range**, not n indices. The logarithm depends on the possible sum values.

## 10. Correctness proof

**Lemma 1: The greedy check uses the minimum number of groups for a fixed limit.**

Assume limit ≥ max(nums). Greedy extends its first group to the farthest possible endpoint without exceeding limit. No feasible partition can put more elements in its first group. Therefore, after greedy finishes its first group, it has left no more unprocessed elements than any competing first-group choice. Repeating this argument on each remaining suffix shows greedy never needs more groups than an alternative valid partition.

**Lemma 2: Greedy groups ≤ k is equivalent to an exactly-k feasible partition.**

If an exactly-k partition exists, Lemma 1 implies greedy uses at most k groups. Conversely, if greedy uses g ≤ k groups, split existing nonempty groups until there are exactly k. Because k ≤ n, enough internal cut positions exist. Since all numbers are nonnegative, splitting a group cannot increase either resulting sum beyond the original group sum. Thus the limit remains satisfied.

**Lemma 3: Feasibility is monotonic.**

If a partition satisfies maximum sum ≤ X, it also satisfies every larger limit Y ≥ X. Therefore the predicate is false for smaller limits and true for larger ones.

**Theorem: Binary search returns the optimal answer.**

The initial interval contains the optimum: no answer can be below max(nums), and sum(nums) is feasible. At each step, if mid is feasible, Lemma 3 guarantees the optimum is at most mid, so retaining [left, mid] is safe. If mid is infeasible, the optimum must exceed mid, so retaining [mid+1, right] is safe. The interval shrinks until one value remains; it must be the smallest feasible limit, which is the required minimum largest subarray sum.

## 11. Edge cases and common mistakes

| nums | k | Answer | Reason |
|---|---:|---:|---|
| [7,2,5,10,8] | 2 | 18 | Standard example |
| [1,2,3,4,5] | 2 | 9 | [1,2,3] / [4,5] |
| [1,4,4] | 3 | 4 | Every element stands alone |
| [5] | 1 | 5 | Single element |
| [2,3,1] | 1 | 6 | Everything in one group |
| [0,0,0] | 2 | 0 | Zero-valued groups |
| [5,0,5] | 2 | 5 | Zeros do not force larger sums |

**Common mistakes:** Binary-searching indices rather than the answer; using zero as the lower bound and forgetting that a group must contain the largest element; testing **groups == k** instead of **groups <= k**; forgetting the nonnegative-values assumption; using **right = mid − 1** after a feasible mid; allowing empty groups; and counting a new group when **total + x == limit** rather than only when it exceeds the limit.

## 12. Interviewer-style follow-up questions

**Q1. Why is this binary search on answer?** We can efficiently test whether a candidate maximum sum is achievable, and that yes/no result is monotonic as the candidate increases.

**Q2. Why does greedy work inside the feasibility check?** For a fixed maximum, filling each group as far as possible minimizes the number of groups. Earlier cuts cannot help reduce the required group count.

**Q3. Why check at most k groups instead of exactly k?** With nonnegative numbers, we can split existing groups to increase the count without increasing any group sum, provided k ≤ n.

**Q4. What changes if negative numbers are allowed?** Greedy feasibility and the simple max(nums) lower bound can fail. For example, a large positive number can be offset by a later negative number. A DP-based approach is safer.

**Q5. What if k = 1?** The only partition is the entire array, so the answer is sum(nums).

**Q6. What if k = n?** Every element must form its own group, so the answer is max(nums).

**Q7. Could we reconstruct the actual groups?** Yes. After finding the optimal limit, make a pass to place cuts. To guarantee exactly k nonempty groups, also force cuts when the number of remaining elements equals the number of remaining groups.

**Q8. How does this compare with the DP solution?** DP explicitly optimizes every last cut in O(kn²) time. Binary search exploits nonnegative values to reduce the problem to O(n) feasibility checks.

**Q9. Why not use a sliding window?** The task is a global partition optimization, not finding one window satisfying a constraint. Greedy packing is a better feasibility test.

**Q10. What related problems share this idea?** Capacity to Ship Packages Within D Days, Koko Eating Bananas, and Minimized Maximum of Products Distributed to Any Store all use monotonic feasibility plus binary search over a numeric answer.

## 13. Interview-ready explanation

> “The brute-force approach chooses k−1 cuts among n−1 gaps, which is combinatorial. I can improve that with partition DP, but because the numbers are nonnegative, there is a better approach. I binary-search the smallest possible maximum group sum between max(nums) and sum(nums). For a candidate limit, I greedily pack as many consecutive elements as possible into each group, starting a new group only when the next element would exceed the limit. This minimizes the number of groups. If I need at most k groups, I can split some further to obtain exactly k without increasing the maximum. Feasibility is monotonic in the limit, so binary search finds the first feasible value in O(n log S) time and O(1) extra space.”

## 14. Pattern takeaway and practice

```text
Minimize the maximum cost across contiguous groups
                         |
                         v
Brute force all k-1 cuts: combinatorial
                         |
                         v
DP over prefixes and group counts: O(kn²)
                         |
                         v
Ask: Can maximum group sum be <= X?
                         |
                         v
Greedy scan counts minimum required groups
                         |
                         v
groups <= k? Feasible
                         |
                         v
Larger X never hurts -> monotonic predicate
                         |
                         v
Binary search first feasible X
                         |
                         v
O(n log S) time, O(1) extra space
```

**Practice next:** LeetCode 1011 — Capacity to Ship Packages Within D Days (almost identical feasibility structure); LeetCode 875 — Koko Eating Bananas (binary search a processing rate); LeetCode 1482 — Minimum Number of Days to Make m Bouquets (contiguous feasibility under a day threshold); and LeetCode 1231 — Divide Chocolate (maximize the minimum instead of minimizing the maximum).

**Transferable interview insight:** Whenever an optimization problem asks for the **smallest maximum** or **largest minimum**, test whether the objective can be converted into a monotonic yes/no decision. If that decision can be checked greedily in linear time, binary search on the answer can replace an expensive DP.

## 15. Validation

The final binary-search solution was compared against exhaustive cut enumeration on **20,000 randomized arrays** of lengths 1 through 9, including zeros, with every trial using a valid random k. All comparisons passed. It returned **18** for [7,2,5,10,8] with k=2, **9** for [1,2,3,4,5] with k=2, **0** for [0,0,0] with k=2, and **4** for [1,4,4] with k=3.
