# DSA Interview Walkthrough — LeetCode 287: Find the Duplicate Number

**Difficulty:** Medium  
**Pattern:** Floyd's Cycle Detection · Functional Graph  
**Core lesson:** Treat each array index as a node and `nums[i]` as a pointer to the next node. The duplicate value creates a cycle, so Floyd's tortoise-and-hare algorithm finds it in O(n) time and O(1) extra space.

## 1. Interviewer prompt

Given an array `nums` containing `n + 1` integers where every value is in `[1, n]`, return the repeated number. Do not modify the array and use only constant extra space.

```text
nums = [1,3,4,2,2]
answer = 2
```

## 2. Recognize the pattern

The obvious solutions conflict with the constraints:
- set -> O(n) extra space,
- sorting -> often modifies input or costs extra copying,
- frequency array -> O(n) space.

The special constraint `nums[i] in [1,n]` lets us interpret the array as pointers:

```text
index i -> nums[i]
```

Every node has exactly one outgoing edge. With `n+1` positions pointing into only `n` possible value nodes, some value has multiple incoming edges. That repeated value is the entrance to a cycle in the reachable functional graph.

## 3. Brute-force approaches

A set finds the duplicate in O(n) time but O(n) space.

Sorting can find equal neighbors in O(n log n), but modifying the array violates the standard constraint unless we copy it.

We need a constant-space structural solution.

## 4. Floyd's cycle detection

Phase 1: find a meeting point inside the cycle.

```text
slow = nums[slow]
fast = nums[nums[fast]]
```

Phase 2: reset one pointer to the start. Move both one step at a time. Their next meeting point is the cycle entrance.

That entrance value is the duplicate.

## 5. Clean final Python solution

```python
class Solution:
    def findDuplicate(self, nums: list[int]) -> int:
        slow = nums[0]
        fast = nums[0]

        while True:
            slow = nums[slow]
            fast = nums[nums[fast]]
            if slow == fast:
                break

        slow = nums[0]

        while slow != fast:
            slow = nums[slow]
            fast = nums[fast]

        return slow
```

No lambda functions are used.

## 6. Why phase 2 works

Let the distance from the start to the cycle entrance be `a`, from the entrance to the first meeting point be `b`, and cycle length be `c`.

At the first meeting, fast has traveled twice as far as slow, so the extra distance fast traveled is a whole number of cycles. This implies `a` and the remaining distance around the cycle from the meeting point to the entrance are congruent modulo `c`.

Therefore, starting one pointer at the beginning and one at the meeting point, then moving both one step at a time, causes them to meet exactly at the cycle entrance.

## 7. Complexity

- Time: `O(n)`.
- Auxiliary space: `O(1)`.
- Input remains unmodified.

## 8. Correctness

The array defines a functional graph reachable from index 0. Because there are more positions than possible values, the reachable path must enter a cycle. The value with multiple incoming references is the duplicate, and in this representation that value is exactly the cycle entrance. Floyd's algorithm returns that entrance.

## 9. Edge cases and mistakes

- Duplicate appears twice or many times.
- Duplicate may be 1 or n.
- Smallest valid input `[1,1]`.

Common mistakes:
- using values as if they were arbitrary array indices without checking the given range constraint,
- resetting to index `0` instead of `nums[0]` while using this particular initialization convention,
- returning the first phase's meeting point, which is not necessarily the cycle entrance,
- modifying the array by sign marking when modification is forbidden.

## 10. Interviewer follow-ups

**Can binary search solve it?** Yes. Count how many values are `<= mid`; pigeonhole reasoning gives O(n log n) time and O(1) space.

**Why isn't this a linked list physically?** The array implicitly defines the same one-outgoing-edge structure.

**What if there can be multiple distinct duplicates?** The problem structure changes; Floyd no longer directly identifies all duplicates.

## 11. What to say aloud

> “The constant-space constraint rules out a hash set. Because every value lies in 1..n, I can treat nums[i] as a pointer to the next index. The duplicate creates a cycle in this functional graph. Floyd's slow/fast pointers find a point inside the cycle, then resetting one pointer to the start and moving both one step finds the cycle entrance, which is the duplicate. The solution is O(n) time and O(1) extra space.”

## 12. Pattern takeaway

```text
Array values form valid next indices
            ↓
Functional graph
            ↓
Duplicate creates cycle entrance
            ↓
Floyd slow/fast pointers
            ↓
O(n) time, O(1) space
```

**Practice next:** LeetCode 142, 202, and 141.