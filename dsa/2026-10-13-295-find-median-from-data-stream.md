# DSA Interview Walkthrough — LeetCode 295: Find Median from Data Stream

**Difficulty:** Hard  
**Pattern:** Two Heaps · Streaming Data  
**Core lesson:** Maintain the lower half of values in a max-heap and the upper half in a min-heap. Keep their sizes balanced so the median is always available in O(1) time.

## 1. Interviewer prompt

Design a structure that supports:
- `addNum(num)` to insert a number from a stream,
- `findMedian()` to return the median of all inserted numbers.

Examples:

```text
add 1
add 2
median = 1.5
add 3
median = 2
```

## 2. Recognize the pattern

A median depends on the middle element(s) after sorting. Re-sorting after every insertion wastes work.

We only need to maintain the boundary between the lower half and upper half of the data.

Use:
- a **max-heap** for the lower half,
- a **min-heap** for the upper half.

Python only provides a min-heap, so store negatives in the lower heap.

## 3. Brute force

Keep every value in a list. On `findMedian`, sort the list and read the middle.

- `addNum`: `O(1)` append.
- `findMedian`: `O(n log n)`.

A sorted-list approach improves median lookup to `O(1)` but insertion becomes `O(n)` because array elements shift.

## 4. Two-heap invariants

Maintain:

1. Every value in `small` is <= every value in `large`.
2. `len(small)` is either equal to `len(large)` or exactly one larger.

Then:
- if total count is odd, median is the maximum of `small`,
- if even, median is the average of max(`small`) and min(`large`).

## 5. Insertion strategy

For a new number:
1. Push it into `small` (as a negative value).
2. Move the maximum of `small` into `large` to restore ordering.
3. If `large` becomes larger, move its minimum back into `small`.

This concise sequence maintains both invariants.

## 6. Dry run

Insert 1:

```text
small = [1]
large = []
median = 1
```

Insert 2:

```text
small = [1]
large = [2]
median = (1 + 2) / 2 = 1.5
```

Insert 3:

After rebalancing:

```text
small = [2,1]
large = [3]
median = 2
```

## 7. Clean final Python solution

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.small = []
        self.large = []

    def addNum(self, num: int) -> None:
        heapq.heappush(self.small, -num)
        heapq.heappush(self.large, -heapq.heappop(self.small))

        if len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def findMedian(self) -> float:
        if len(self.small) > len(self.large):
            return float(-self.small[0])

        return (-self.small[0] + self.large[0]) / 2
```

No lambda functions are used.

## 8. Complexity

For `n` values:
- `addNum`: `O(log n)`.
- `findMedian`: `O(1)`.
- Space: `O(n)`.

## 9. Correctness

The ordering invariant guarantees the lower heap contains only values no greater than those in the upper heap. The size invariant ensures the middle of the sorted sequence is always either the top of `small` alone or the two heap tops together. Therefore `findMedian` returns the true median.

## 10. Edge cases and common mistakes

- One value.
- Duplicate values.
- Negative values.
- Very large positive/negative values.

Common mistakes:
- forgetting to negate values in the max-heap,
- balancing heap sizes without ensuring cross-heap ordering,
- allowing one heap to differ by more than one element,
- using integer division for an even-length median.

## 11. Interviewer follow-ups

**Why not one heap?** One heap exposes only one extreme, not both sides of the median boundary.

**Sliding-window median?** Deletion becomes the hard part; use heaps with lazy deletion or a balanced ordered structure.

**If values are known to lie in [0,100]?** A frequency array can answer median queries efficiently with bounded memory.

**What percentile does this generalize to?** The same partition idea can maintain other quantile boundaries, though balancing targets change.

## 12. What to say aloud

> “The median only depends on the boundary between the lower and upper halves. I keep the lower half in a max-heap and the upper half in a min-heap. I maintain all lower values <= upper values, and keep the lower heap equal in size or one larger. Insertions rebalance in O(log n); the median is then available directly from the heap tops in O(1).”

## 13. Pattern takeaway

```text
Streaming values + repeated median
           ↓
Need middle boundary, not full sorting
           ↓
Max-heap lower half
Min-heap upper half
           ↓
Maintain ordering + size invariants
           ↓
O(log n) insert, O(1) median
```

**Practice next:** LeetCode 480, 703, and 215.