# DSA Interview Walkthrough — LeetCode 25: Reverse Nodes in k-Group

**Difficulty:** Hard  
**Pattern:** Linked List · Pointer Reversal · Group Processing  
**Core lesson:** Reverse exactly `k` nodes at a time, reconnect each reversed block to the previous and next blocks, and leave the final short block untouched.

## 1. Interviewer prompt

Given the head of a linked list, reverse nodes in groups of size `k`. If fewer than `k` nodes remain at the end, leave them unchanged.

```text
1 -> 2 -> 3 -> 4 -> 5, k = 2
answer: 2 -> 1 -> 4 -> 3 -> 5
```

Do not swap node values; change pointers.

## 2. Recognize the pattern

This is not “reverse the whole list.” The hard part is safely reconnecting repeated reversed segments.

Use a dummy node before the head so reversing the first group is no different from reversing later groups.

For each group:
1. Find the `k`th node from the node before the group.
2. If it does not exist, stop.
3. Save `group_next = kth.next`.
4. Reverse pointers from `group_start` through `kth`.
5. Reconnect the previous group to the new head and move the boundary forward.

## 3. Straightforward extra-space approach

For each block of `k` nodes, collect node references in an array, then reconnect them in reverse order.

This is conceptually simple but uses `O(k)` temporary storage. We can do it in-place.

## 4. In-place reversal insight

Suppose a group is:

```text
prev_group -> 1 -> 2 -> 3 -> next
```

After reversing:

```text
prev_group -> 3 -> 2 -> 1 -> next
```

Before reversal:
- `group_start = prev_group.next`
- `kth = 3`
- `group_next = kth.next`

Reverse until `curr == group_next`. Initialize `prev = group_next` so the old group head automatically points to the next block at the end.

## 5. Clean final Python solution

```python
class Solution:
    def reverseKGroup(self, head, k: int):
        dummy = ListNode(0, head)
        group_prev = dummy

        while True:
            kth = group_prev
            for _ in range(k):
                kth = kth.next
                if not kth:
                    return dummy.next

            group_next = kth.next
            prev = group_next
            curr = group_prev.next

            while curr != group_next:
                nxt = curr.next
                curr.next = prev
                prev = curr
                curr = nxt

            old_start = group_prev.next
            group_prev.next = kth
            group_prev = old_start
```

No lambda functions are used.

## 6. Complexity

Each node participates in at most one reversal pass.

- Time: `O(n)`.
- Auxiliary space: `O(1)`.

The repeated search for the kth node still totals `O(n)` because groups do not overlap.

## 7. Correctness

Before each iteration, `group_prev.next` is the first node of the next unreversed segment. If fewer than `k` nodes remain, the algorithm stops without modifying them. Otherwise it reverses exactly those `k` nodes, reconnects the previous processed list to the new group head, and makes the old group head the boundary before the next group. Thus every full group is reversed exactly once and the remainder is preserved.

## 8. Edge cases and common mistakes

- `k = 1`: list unchanged.
- List length exactly divisible by `k`.
- Final block shorter than `k`.
- Single-node list.

Common mistakes:
- reversing a short final block,
- losing `group_next`,
- reconnecting to the wrong end of the reversed block,
- not using a dummy node and then special-casing the first group,
- advancing `group_prev` to the new head instead of the old head.

## 9. Interviewer follow-ups

**Reverse between positions left and right?** Same pointer-reversal technique, but only one segment.

**Recursive solution?** Possible, but iterative code avoids recursion depth and makes pointer ownership explicit.

**Why initialize `prev = group_next`?** It makes the final node of the reversed group automatically point to the untouched remainder.

## 10. What to say aloud

> “I process the list one complete group at a time. From the node before the group, I locate the kth node; if it does not exist, I leave the remainder unchanged. I save the node after the group, reverse exactly until that boundary, then reconnect the previous block to the kth node, which is now the group head. The old group head becomes the boundary for the next iteration. This reverses each node once, so time is O(n) and extra space is O(1).”

## 11. Pattern takeaway

```text
Repeated fixed-size linked-list segments
              ↓
Dummy node simplifies first group
              ↓
Locate kth node
              ↓
Save group_next
              ↓
Reverse until group_next
              ↓
Reconnect both ends
              ↓
O(n) time, O(1) space
```

**Practice next:** LeetCode 92, 206, and 24.