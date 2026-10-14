# DSA Interview Walkthrough — LeetCode 4: Median of Two Sorted Arrays

**Difficulty:** Hard  
**Pattern:** Binary Search on Partition · Two Pointers · Order Statistics  
**Core lesson:** When two arrays are already sorted, you do not need to merge them to find the median. Find a partition that puts exactly half the elements on the left and ensures every left-side element is no greater than every right-side element.

## 1. Interviewer: Problem statement

Given two sorted integer arrays `nums1` and `nums2`, return the median of their combined values in logarithmic time.

```text
nums1 = [1, 3]
nums2 = [2]
median = 2.0

nums1 = [1, 2]
nums2 = [3, 4]
median = 2.5
```

One array may be empty, but not both. Duplicates and negative values are allowed.

## 2. Recognize the pattern

The clues are **two sorted arrays**, **median**, and **logarithmic runtime**.

The median is an order statistic. Instead of building the full merged order, binary-search a partition position.

We want:

1. Exactly half the total elements on the left, rounded up.
2. Every left-side element <= every right-side element.

Once that partition exists, the median comes directly from the boundary values.

## 3. Brute force: merge

```python
def merge_median(a, b):
    i = j = 0
    merged = []

    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            merged.append(a[i])
            i += 1
        else:
            merged.append(b[j])
            j += 1

    merged.extend(a[i:])
    merged.extend(b[j:])

    k = len(merged)
    if k % 2:
        return float(merged[k // 2])
    return (merged[k // 2 - 1] + merged[k // 2]) / 2
```

**Time:** O(m+n).  
**Auxiliary space:** O(m+n).

## 4. First optimization: two pointers without storing the merge

```python
def scan_median(a, b):
    m, n = len(a), len(b)
    i = j = 0
    prev = curr = 0

    for _ in range((m + n) // 2 + 1):
        prev = curr
        if i < m and (j == n or a[i] <= b[j]):
            curr = a[i]
            i += 1
        else:
            curr = b[j]
            j += 1

    if (m + n) % 2:
        return float(curr)
    return (prev + curr) / 2
```

**Time:** O(m+n) worst case.  
**Auxiliary space:** O(1).

To meet the logarithmic target, we must skip over ranges rather than consume elements one by one.

## 5. Key insight: binary-search a partition

Always make `A` the shorter array. Let its length be `m` and the other array's length be `n`.

Choose partition positions `i` in A and `j` in B such that:

```text
half = (m + n + 1) // 2
j = half - i
```

Define:

```text
a_left  = A[i-1] if i > 0 else -infinity
a_right = A[i]   if i < m else +infinity
b_left  = B[j-1] if j > 0 else -infinity
b_right = B[j]   if j < n else +infinity
```

The partition is valid when:

```text
a_left <= b_right
b_left <= a_right
```

For odd total length:

```text
median = max(a_left, b_left)
```

For even total length:

```text
median = (max(a_left, b_left) + min(a_right, b_right)) / 2
```

## 6. How binary search moves

If `a_left > b_right`, we took too many A elements on the left, so move `i` left.

If `b_left > a_right`, we took too few A elements on the left, so move `i` right.

The direction is monotonic because shifting `i` changes the boundary values predictably in sorted arrays.

## 7. Dry run

```text
A = [1, 3, 8, 9, 15]
B = [7, 11, 18, 19, 21, 25]
Total = 11
half = 6
```

Try `i=2`, so `j=4`:

```text
A: [1,3 | 8,9,15]
B: [7,11,18,19 | 21,25]

a_left = 3
a_right = 8
b_left = 19
b_right = 21
```

`b_left > a_right`, so move right.

Try `i=4`, `j=2`:

```text
A: [1,3,8,9 | 15]
B: [7,11 | 18,19,21,25]
```

Now:

```text
9 <= 18
11 <= 15
```

The partition is valid. Since the total is odd:

```text
median = max(9, 11) = 11
```

## 8. Clean final LeetCode solution — Python

```python
class Solution:
    def findMedianSortedArrays(
        self, nums1: list[int], nums2: list[int]
    ) -> float:
        if len(nums1) > len(nums2):
            nums1, nums2 = nums2, nums1

        m, n = len(nums1), len(nums2)
        half = (m + n + 1) // 2
        left, right = 0, m

        while left <= right:
            i = (left + right) // 2
            j = half - i

            a = nums1[i - 1] if i else float("-inf")
            b = nums1[i] if i < m else float("inf")
            c = nums2[j - 1] if j else float("-inf")
            d = nums2[j] if j < n else float("inf")

            if a <= d and c <= b:
                if (m + n) % 2:
                    return float(max(a, c))
                return (max(a, c) + min(b, d)) / 2

            if a > d:
                right = i - 1
            else:
                left = i + 1
```

No lambda functions are used, and neither input array is modified.

## 9. Complexity analysis

We binary-search the shorter array.

**Time:** O(log(min(m,n)+1)).  
**Auxiliary space:** O(1).

## 10. Correctness proof

The partition always contains `half` combined elements on the left.

Because each input array is sorted, only the two cross-array inequalities can violate the ordering between left and right. Once both hold, `max(a_left,b_left)` is the largest left value and `min(a_right,b_right)` is the smallest right value.

Those are exactly the middle value for odd length or the two middle values for even length.

If `a_left > b_right`, no partition with an even larger `i` can work, so moving left is safe. If `b_left > a_right`, no partition with a smaller `i` can work, so moving right is safe.

## 11. Edge cases and common mistakes

- One empty array.
- Two single-element arrays.
- Duplicates.
- Negative values.
- Very unequal lengths.

Common mistakes:
- Not ensuring the first array is shorter.
- Using `(m+n)//2` without adjusting odd-length handling.
- Checking only one cross-array inequality.
- Using strict `<` instead of `<=` and mishandling duplicates.
- Accessing out-of-range partition boundaries instead of using sentinels.
- Moving binary-search bounds in the wrong direction.

## 12. Interviewer-style follow-up questions

**Why search the shorter array?**  
It minimizes iterations and keeps the derived partition in the other array valid.

**Why use infinity sentinels?**  
They make empty left/right partition boundaries obey the same comparison logic as ordinary values.

**Can this find the k-th smallest element?**  
Yes. Choose partitions whose left side contains exactly `k` elements and apply the same boundary logic.

**Why isn't two-pointer merging enough?**  
It is O(m+n), while the requested complexity is logarithmic.

**What if arrays are unsorted?**  
The monotonic partition argument fails; sort or use selection methods instead.

**What if values arrive continuously?**  
Use the two-heaps streaming-median technique rather than this static-array partition method.

## 13. What to say aloud

> “The linear solution is to merge until the middle, but the logarithmic requirement suggests binary search. I search for a partition in the shorter array, derive the partition in the other array so that the left side contains half the values, and check the two cross-array boundary inequalities. If A's left boundary is too large I move left; if B's left boundary is too large I move right. Once the partition is valid, the median is determined by the boundary values. This runs in O(log(min(m,n))) time and O(1) space.”

## 14. Pattern takeaway

```text
Two sorted arrays + median
           ↓
Merge gives O(m+n)
           ↓
Need logarithmic time
           ↓
Binary-search partition in shorter array
           ↓
Force half of total values onto left side
           ↓
Check cross-boundary inequalities
           ↓
Valid partition -> median from boundaries
           ↓
O(log(min(m,n))) time
```

**Practice next:** LeetCode 153 Find Minimum in Rotated Sorted Array, 33 Search in Rotated Sorted Array, 378 Kth Smallest Element in a Sorted Matrix, and 295 Find Median from Data Stream.