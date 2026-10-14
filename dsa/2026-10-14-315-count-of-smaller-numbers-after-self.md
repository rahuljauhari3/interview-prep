# DSA Interview Walkthrough — LeetCode 315: Count of Smaller Numbers After Self

**Difficulty:** Hard  
**Pattern:** Modified Merge Sort · Divide and Conquer · Counting Inversions  
**Core lesson:** When each position asks how many later elements satisfy a comparison, count cross-half pairs while merging sorted halves. A modified merge sort turns a quadratic comparison problem into O(n log n) without losing original indices.

## 1. Interviewer: Problem statement

Given an integer array `nums`, return an array `counts` where `counts[i]` is the number of elements to the right of index `i` that are strictly smaller than `nums[i]`.

```text
nums   = [5, 2, 6, 1]
counts = [2, 1, 1, 0]
```

Negative values and duplicates are allowed. Equal values do not count as smaller.

## 2. Recognize the pattern

The phrases **for every index**, **to the right**, and **strictly smaller** describe a per-index inversion count.

An inversion is a pair `(i,j)` with `i < j` and `nums[i] > nums[j]`. Standard inversion counting finds the total; here we need how many inversions start at each index.

That suggests divide and conquer with merge sort.

## 3. Brute force

```python
def brute(nums):
    n = len(nums)
    ans = [0] * n

    for i in range(n):
        for j in range(i + 1, n):
            if nums[j] < nums[i]:
                ans[i] += 1

    return ans
```

**Time:** O(n^2).  
**Auxiliary space:** O(1) beyond the answer.

## 4. First optimization idea: process from right to left

Keep values already seen to the right in sorted order. Binary-search how many are smaller, then insert the current value.

With a normal Python list, insertion may shift O(n) elements, so this remains O(n^2) worst-case. A Fenwick tree with coordinate compression can achieve O(n log n), but merge sort gives an equally efficient solution with a direct counting interpretation.

## 5. Modified merge sort

Store `(value, original_index)` pairs.

Split the original range into left and right halves. Since every original index in the left half comes before every original index in the right half, any pair where a left value exceeds a right value is a valid smaller-to-the-right pair.

During an ascending merge:

- If `left_value <= right_value`, output the left item. The number of right-half elements already output is exactly how many strictly smaller right-half values exist for that left item.
- Otherwise output the right item first.

The tie must favor the left item. If equal right values move first, they would incorrectly be counted as strictly smaller.

## 6. Dry run

```text
Original pairs:
[(5,0), (2,1), (6,2), (1,3)]
```

Merge `[5,2]`: `2` moves before `5`, so index 0 gets +1.

Merge `[6,1]`: `1` moves before `6`, so index 2 gets +1.

Final merge `[2,5]` and `[1,6]`:

- `1` moves first.
- When `2` moves, one right-half value has already moved -> index 1 gets +1.
- When `5` moves, one right-half value has already moved -> index 0 gets +1.

Final result:

```text
[2, 1, 1, 0]
```

## 7. Clean final LeetCode solution — Python

```python
class Solution:
    def countSmaller(self, nums: list[int]) -> list[int]:
        n = len(nums)
        items = [(x, i) for i, x in enumerate(nums)]
        temp = [None] * n
        ans = [0] * n

        def sort(lo, hi):
            if hi - lo <= 1:
                return

            mid = (lo + hi) // 2
            sort(lo, mid)
            sort(mid, hi)

            i, j, k = lo, mid, lo

            while i < mid and j < hi:
                if items[i][0] <= items[j][0]:
                    ans[items[i][1]] += j - mid
                    temp[k] = items[i]
                    i += 1
                else:
                    temp[k] = items[j]
                    j += 1
                k += 1

            while i < mid:
                ans[items[i][1]] += j - mid
                temp[k] = items[i]
                i += 1
                k += 1

            while j < hi:
                temp[k] = items[j]
                j += 1
                k += 1

            items[lo:hi] = temp[lo:hi]

        sort(0, n)
        return ans
```

No lambda functions are used, and the input array is not modified.

## 8. Complexity analysis

Merge sort has recurrence:

```text
T(n) = 2T(n/2) + O(n)
     = O(n log n)
```

**Time:** O(n log n).  
**Auxiliary space:** O(n), plus O(log n) recursion depth.

## 9. Correctness proof

By induction, recursive calls correctly count within-half smaller-right relationships and sort each half by value.

During the merge, when a left item is emitted, every right-half item already emitted is strictly smaller, and every strictly smaller right-half item must already have been emitted because that half is sorted. Therefore `j - mid` is exactly the number of new cross-half smaller elements for that left item.

Equal values are not counted because the left item is chosen first on ties.

Every valid pair is counted exactly at the merge level where its two indices first belong to different halves.

## 10. Edge cases and common mistakes

- Strictly increasing -> all zeros.
- Strictly decreasing -> `[n-1, n-2, ..., 0]`.
- All equal -> all zeros.
- Negative values.
- Single element.

Common mistakes:
- Using `<` rather than `<=` when deciding to take from the left.
- Losing original indices after sorting.
- Updating counts for right-half elements instead of left-half elements.
- Forgetting remaining left items after the right half is exhausted.

## 11. Interviewer-style follow-up questions

**How is this related to inversion counting?**  
Summing the returned counts gives the total inversion count.

**Could a Fenwick tree solve it?**  
Yes. Coordinate-compress values, scan from right to left, query the count of smaller ranks, then update the current rank.

**Why choose the left item on equality?**  
Equal values are not strictly smaller, so they must not contribute to `j - mid`.

**What if we only need the total number of inversions?**  
When a right item moves before a left item, add the number of remaining left items to one global count.

**What if values arrive online?**  
Use an order-statistics tree or indexed frequency structure rather than offline merge sort.

## 12. What to say aloud

> “Brute force compares every element with everything to its right, which is O(n²). This is inversion counting per original index, so I can use merge sort. I recursively sort and count within each half, then during the merge I count cross-half pairs. Whenever a left value is emitted, the number of right-half values already emitted is exactly the number of strictly smaller elements to its right. I keep original indices so the counts go to the correct positions and use left-first tie handling so duplicates are not counted. This gives O(n log n) time and O(n) space.”

## 13. Pattern takeaway

```text
For each index, count later values satisfying an inequality
                        ↓
Brute force checks O(n²) pairs
                        ↓
Recognize inversion-style counting
                        ↓
Divide into original left/right halves
                        ↓
Sort and count cross-half pairs during merge
                        ↓
Keep original indices
                        ↓
O(n log n) time, O(n) space
```

**Practice next:** LeetCode 493 Reverse Pairs, 327 Count of Range Sum, 1649 Create Sorted Array through Instructions, and 912 Sort an Array.