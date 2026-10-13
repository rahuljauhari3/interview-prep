# DSA Interview Walkthrough — LeetCode 907: Sum of Subarray Minimums

**Difficulty:** Medium  
**Pattern:** Monotonic Stack · Contribution Counting · Previous/Next Smaller Element  
**Core lesson:** When a problem asks for an aggregate over *every contiguous subarray*, consider reversing the counting: instead of enumerating subarrays, calculate how much each array element contributes to the final answer.

## 1. Interviewer: Problem statement

**Interviewer:** Given an array of positive integers, return the sum of the minimum element of every nonempty contiguous subarray. Return the answer modulo 1,000,000,007.

For example:

~~~text
arr = [3, 1, 2, 4]

Subarrays and minimums:
[3]          -> 3
[3,1]        -> 1
[3,1,2]      -> 1
[3,1,2,4]    -> 1
[1]          -> 1
[1,2]        -> 1
[1,2,4]      -> 1
[2]          -> 2
[2,4]        -> 2
[4]          -> 4

Answer = 17
~~~

**Candidate:** Are subarrays required to be contiguous? Can values repeat? Are all values positive?

**Interviewer:** Yes, yes, and yes for the standard constraints.

**Candidate:** So two subarrays with different boundaries count separately, even if they have the same minimum?

**Interviewer:** Correct. Explain how you would avoid checking all subarrays.

## 2. Recognize the pattern and build intuition

The phrases **every subarray**, **minimum**, and **sum of those minimums** are the clues.

A direct approach enumerates subarrays and computes their minimum. But there are n(n+1)/2 subarrays, so explicitly visiting each one already takes quadratic time.

Instead, ask a different question:

> For a particular index i, how many subarrays can use arr[i] as their chosen minimum?

If the answer is count[i], then arr[i] contributes arr[i] × count[i] to the final sum.

This is called **contribution counting**: count the number of objects to which each element contributes, rather than generating the objects individually.

The challenge is finding the boundaries of subarrays in which arr[i] is the minimum. That leads to **previous smaller** and **next smaller-or-equal** elements, found efficiently with a monotonic stack.

## 3. Brute force: enumerate and rescan

Choose every start and end index, then scan the selected range to find its minimum.

~~~python
def brute(arr):
    ans = 0
    n = len(arr)

    for i in range(n):
        for j in range(i, n):
            smallest = min(arr[i:j + 1])
            ans += smallest

    return ans % (10**9 + 7)
~~~

There are O(n²) subarrays, and computing each minimum from scratch takes up to O(n).

**Time:** O(n³) worst case.  
**Auxiliary space:** O(n) because slicing creates a temporary list; a manual inner scan could reduce this to O(1).

**Interviewer:** What work is repeated?

**Candidate:** Adjacent subarrays starting at the same index share almost all their elements. I can maintain the minimum while extending the right boundary.

## 4. First optimization: maintain a running minimum

~~~python
def quadratic(arr):
    ans = 0

    for i in range(len(arr)):
        smallest = float("inf")
        for j in range(i, len(arr)):
            smallest = min(smallest, arr[j])
            ans += smallest

    return ans % (10**9 + 7)
~~~

**Time:** O(n²).  
**Auxiliary space:** O(1).

This is a reasonable intermediate solution. But can we do better than examining all O(n²) subarrays?

**Interviewer:** Suppose I point to one element. Can you count all subarrays where it is the minimum without listing those subarrays?

**Candidate:** Yes. I need to know how far I can extend left and right without encountering a smaller competing value.

## 5. Contribution counting: turn subarrays into distances

Consider index i, with value arr[i].

Define:

- L[i]: index of the **previous strictly smaller** element to the left, or -1 if none exists.
- R[i]: index of the **next smaller-or-equal** element to the right, or n if none exists.

Then a subarray assigned to index i can start at any position from L[i]+1 through i and end at any position from i through R[i]-1.

There are:

~~~text
left_choices  = i - L[i]
right_choices = R[i] - i

subarrays assigned to i = left_choices * right_choices
contribution(i) = arr[i] * left_choices * right_choices
~~~

Why multiply? Each legal start can be paired independently with each legal end, and every such pair creates exactly one contiguous subarray containing i.

Therefore:

~~~text
answer = sum(arr[i] * (i - L[i]) * (R[i] - i))
         for every index i
~~~

The remaining task is finding all these boundaries efficiently.

## 6. Why equal values need asymmetric comparisons

Duplicates are the most subtle part of this problem.

Consider:

~~~text
arr = [2, 2]

Subarrays:
[2] at index 0 -> 2
[2] at index 1 -> 2
[2,2]          -> 2

Answer = 6
~~~

For the subarray [2,2], both indices have the minimum value. If we let both claim it, we double-count. If we disqualify both, we miss it.

We need a consistent tie-breaking rule. Here we assign a subarray with repeated minimum values to its **rightmost occurrence of the minimum**.

To achieve that, use:

- Previous **strictly smaller** element on the left.
- Next **smaller or equal** element on the right.

With arr = [2,2]:

~~~text
Index 0: L=-1, R=1 -> 1*1=1 subarray -> contribution 2
Index 1: L=-1, R=2 -> 2*1=2 subarrays -> contribution 4

Total = 6
~~~

The first 2 is stopped by the equal 2 to its right. The second 2 can extend left over the equal value, so it owns [2,2].

**Interviewer:** Could we instead choose the leftmost minimum?

**Candidate:** Yes. We would reverse the asymmetry: previous smaller-or-equal on the left, next strictly smaller on the right. What matters is that ties are assigned consistently to exactly one occurrence.

## 7. Special technique: monotonic increasing stack

A monotonic stack maintains indices whose values are ordered. Here we keep values **strictly increasing** from bottom to top.

Process array indices left to right.

When the current value is smaller than or equal to the value at the stack top, the current index is the first smaller-or-equal element to the right of that stacked index. We can pop that index and compute its contribution immediately.

Suppose we pop index j while processing index i:

- **Right boundary:** i, because arr[i] <= arr[j].
- **Left boundary:** the new stack top, or -1 if the stack is empty. It is the previous strictly smaller value.
- **Contribution:** arr[j] × (j - left) × (i - j).

After popping everything greater than or equal to the current value, push the current index.

At the end, we still need to process elements with no smaller-or-equal value to their right. We can handle this by running the loop through index n as a **virtual sentinel** that pops all remaining elements.

**Why is this linear?** Each index is pushed once and popped once. The nested while loop does not mean quadratic work: across the entire algorithm, there are at most n pushes and n pops.

## 8. Detailed dry run

Use arr = [3,1,2,4]. The stack stores indices; values are shown in parentheses.

| Step | Action | Contribution | Stack afterward | Running sum |
|---|---|---:|---|---:|
| i=0, value 3 | Push 0 | — | [0(3)] | 0 |
| i=1, value 1 | Pop 0; left=-1, right=1 | 3×1×1 = 3 | [] | 3 |
| i=1, value 1 | Push 1 | — | [1(1)] | 3 |
| i=2, value 2 | Push 2 | — | [1(1),2(2)] | 3 |
| i=3, value 4 | Push 3 | — | [1(1),2(2),3(4)] | 3 |
| i=4, sentinel | Pop 3; left=2 | 4×1×1 = 4 | [1(1),2(2)] | 7 |
| i=4, sentinel | Pop 2; left=1 | 2×1×2 = 4 | [1(1)] | 11 |
| i=4, sentinel | Pop 1; left=-1 | 1×2×3 = 6 | [] | 17 |

Final answer = **17**.

Notice that index 1 (value 1) contributes to six subarrays: it has two choices of starting index (0 or 1) and three choices of ending index (1, 2, or 3).

## 9. Clean final LeetCode solution — Python

~~~python
class Solution:
    def sumSubarrayMins(self, arr: list[int]) -> int:
        mod = 10**9 + 7
        stack = []
        ans = 0
        n = len(arr)

        for i in range(n + 1):
            while stack and (i == n or arr[stack[-1]] >= arr[i]):
                j = stack.pop()
                left = stack[-1] if stack else -1
                ans += arr[j] * (j - left) * (i - j)

            if i < n:
                stack.append(i)

        return ans % mod
~~~

This is a **one-pass monotonic-stack solution**. It uses no lambda functions, avoids separate left/right arrays, and does not modify the input.

The sentinel index n is never used to access arr[n]; the short-circuit condition i == n ensures the array access is skipped.

## 10. Complexity analysis

Let n = len(arr).

Every index is pushed exactly once and popped exactly once. Even though the while loop is nested inside the for loop, its total number of iterations across the whole run is at most n.

**Time:** O(n).  
**Auxiliary space:** O(n) worst case for the stack, such as a strictly increasing array.

The result is taken modulo 1,000,000,007. Python integers do not overflow, but the modulo is required by the problem.

## 11. Correctness proof

We assign each subarray to the **rightmost index attaining its minimum value**.

For any index j, let L be the closest index to its left with value strictly less than arr[j], or -1. Let R be the closest index to its right with value less than or equal to arr[j], or n.

Every subarray starting in (L, j] and ending in [j, R) has arr[j] as its minimum: no strictly smaller value occurs to the left within the range, and no smaller-or-equal value occurs to the right before R. If an equal minimum appears to the left, j is still the rightmost minimum, so it owns the subarray.

Conversely, any subarray assigned to j cannot start at or before L, because that would include a strictly smaller element. It cannot end at or after R, because that would include a smaller element or an equal minimum farther to the right. Thus the legal starts and ends are exactly those counted by (j-L)(R-j).

The monotonic stack identifies these boundaries when j is popped: the current index is its first smaller-or-equal element to the right, and the new stack top is its previous strictly smaller element. The sentinel provides R=n for remaining indices.

Every nonempty subarray has exactly one rightmost minimum, so every subarray's minimum is counted **once and only once**. Summing all contributions gives the required answer.

## 12. Edge cases and common mistakes

**Single element:** [5] -> 5.

**All equal values:** [2,2,2] has six subarrays, each with minimum 2, so answer = 12. The asymmetric tie rule prevents double-counting.

**Strictly increasing:** [1,2,3] -> 1+1+1+2+2+3 = 10.

**Strictly decreasing:** [3,2,1] -> 3+2+1+2+1+1 = 10.

**Large array or large values:** Apply the required modulo.

**Mistake — using strict comparisons on both sides:** Duplicate minima can be counted more than once.

**Mistake — using non-strict comparisons on both sides:** Some subarrays with equal minima can be counted zero times.

**Mistake — counting only subarrays centered on the element:** A chosen minimum can appear anywhere inside its subarray; count all independent start/end choices.

**Mistake — forgetting to flush the stack:** Elements with no smaller-or-equal value to the right still contribute. The virtual sentinel handles them.

**Mistake — assuming nested loops imply O(n²):** Every index is pushed and popped at most once, so the total work is linear.

**Mistake — using a stack of values only:** We need indices to calculate distances to the left and right boundaries.

## 13. Interviewer-style follow-up questions

**Q1. Why not use a sliding window?**

We are aggregating over every possible window length and position, not maintaining one fixed or monotonically constrained window. Contribution counting with nearest-smaller boundaries avoids enumerating those windows.

**Q2. Can you solve this with two passes instead of one?**

Yes. Compute previous strictly smaller indices by popping values greater than or equal to the current value in a left-to-right pass. Compute next smaller-or-equal indices by popping values greater than the current value in a right-to-left pass. Then sum the contributions. It is also O(n) time and O(n) space, but uses two boundary arrays.

**Q3. Why does the stack remain strictly increasing?**

Before pushing arr[i], we pop every top value greater than or equal to arr[i]. Any remaining top is strictly smaller than arr[i], so pushing preserves strict increase.

**Q4. What changes if I need the sum of subarray maximums?**

Reverse the inequalities. Count each element's contribution as a maximum using previous strictly greater and next greater-or-equal boundaries.

**Q5. How would you solve LeetCode 2104, Sum of Subarray Ranges?**

The range of a subarray is maximum minus minimum. Compute the sum of subarray maximums and subtract the sum of subarray minimums using two monotonic-stack passes or two contribution calculations.

**Q6. How is this related to Largest Rectangle in Histogram?**

Both use nearest-smaller boundaries. In Largest Rectangle, a bar's height times the width between smaller bars gives an area. Here, the left and right distances are multiplied to count how many subarrays choose that element as their minimum.

**Q7. Does the technique work with negative values?**

Yes, the nearest-smaller and contribution logic still works for arbitrary integers. The LeetCode problem specifies positive values, but positivity is not essential to this algorithm.

**Q8. What if I need the minimum of one specific range repeatedly?**

That is a range minimum query problem. A sparse table or segment tree is more suitable. This problem instead asks for a global sum over all subarrays, making contribution counting the better fit.

## 14. What to say aloud in the interview

> “There are O(n²) subarrays, so I first think of brute-force enumeration. I can maintain a running minimum to get O(n²), but to reach linear time I reverse the counting. For each element, I count how many subarrays select it as their minimum. If I know its previous strictly smaller index and next smaller-or-equal index, the number of choices is the product of the left and right distances. The strict/non-strict asymmetry is important because duplicate minimum values must be assigned to exactly one index. A monotonic increasing stack finds these boundaries: when a smaller-or-equal value arrives, I pop the larger or equal stack values and compute their contributions. Each index is pushed and popped once, so the solution takes O(n) time and O(n) space.”

## 15. Pattern takeaway and practice

~~~text
Sum a statistic across EVERY subarray
                  ↓
Brute force enumerates O(n²) subarrays
                  ↓
Reverse the counting:
How many subarrays does each element contribute to?
                  ↓
Find nearest smaller boundaries
                  ↓
Break duplicate ties asymmetrically
                  ↓
Count left choices × right choices
                  ↓
Monotonic stack computes boundaries in O(n)
                  ↓
Sum contributions modulo 1,000,000,007
~~~

**Practice next:** LeetCode 2104 (Sum of Subarray Ranges) for combining minimum and maximum contributions; 84 (Largest Rectangle in Histogram) for nearest-smaller boundaries; 739 (Daily Temperatures) for next-greater stack intuition; and 1856 (Maximum Subarray Min-Product) for combining a monotonic stack with prefix sums.

**Transferable interview insight:** When asked to sum a property over all subarrays, look for a way to assign each subarray to one responsible index. Then count how many subarrays belong to each index. A monotonic stack is especially useful when that ownership depends on the nearest smaller or greater elements.
