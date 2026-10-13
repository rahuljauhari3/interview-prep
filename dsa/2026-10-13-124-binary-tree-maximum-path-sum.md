# DSA Interview Walkthrough — LeetCode 124: Binary Tree Maximum Path Sum

**Difficulty:** Hard  
**Pattern:** Tree DFS · Postorder · Tree DP  
**Core lesson:** At each node, compute the best downward path you can return to the parent, while separately using both children to update a global answer for a path that may turn at the current node.

## 1. Interviewer prompt

Given a binary tree, return the maximum path sum. A path may start and end at any nodes, but adjacent nodes in the path must be connected by an edge and no node can be used twice.

```text
       -10
       /  \
      9   20
         /  \
        15   7

answer = 42
path = 15 -> 20 -> 7
```

## 2. Recognize the pattern

Tree path problems often need two different values:

1. What can this subtree contribute upward to its parent?
2. What complete answer can be formed entirely inside this subtree?

A parent can continue through only **one** child, otherwise the path would branch. But a complete path whose highest point is the current node may use **both** children.

## 3. Brute-force thinking

One brute-force idea is to consider many possible start/end node pairs and compute path sums, which is far too expensive.

Instead, process the tree bottom-up with postorder DFS so each node combines already-solved child information exactly once.

## 4. Define the recursive return value

Let `gain(node)` be the maximum sum of a path that:
- starts at `node`,
- goes downward,
- uses at most one child branch,
- can be extended by the parent.

For a child contribution, negative gain is harmful, so clamp it at zero:

```text
left = max(0, gain(node.left))
right = max(0, gain(node.right))
```

The best path that **turns at this node** is:

```text
node.val + left + right
```

Use that to update the global answer.

The value returned upward is:

```text
node.val + max(left, right)
```

because the parent can extend through only one branch.

## 5. Dry run

At node 15:
- left = 0, right = 0
- best through node = 15
- return 15

At node 7:
- return 7

At node 20:
- left = 15
- right = 7
- complete path through 20 = `20 + 15 + 7 = 42`
- return upward = `20 + max(15,7) = 35`

At node -10:
- best left gain = 9
- best right gain = 35
- through-root path = 34, which is smaller than 42

Final answer: 42.

## 6. Clean final Python solution

```python
class Solution:
    def maxPathSum(self, root) -> int:
        best = float("-inf")

        def dfs(node):
            nonlocal best
            if not node:
                return 0

            left = max(0, dfs(node.left))
            right = max(0, dfs(node.right))

            best = max(best, node.val + left + right)

            return node.val + max(left, right)

        dfs(root)
        return best
```

No lambda functions are used.

## 7. Complexity

- Time: `O(n)` because each node is visited once.
- Auxiliary space: `O(h)` recursion stack, where `h` is tree height.

## 8. Correctness

For each node, DFS correctly computes the best extendable downward path because any extendable path can choose at most one child. Negative child contributions are never useful, so replacing them with zero is safe. Every valid maximum path has some highest node where its left and right portions meet; when DFS processes that node, `node.val + left + right` evaluates exactly that candidate. Therefore the global maximum becomes the optimal path sum.

## 9. Edge cases and common mistakes

- All values negative: initialize the global answer to negative infinity, not zero.
- Single node: answer is that node's value.
- One-sided tree: works naturally.

Common mistakes:
- returning `node.val + left + right` to the parent,
- forgetting to clamp negative child gains,
- initializing `best = 0`, which breaks all-negative trees,
- confusing root-to-leaf paths with arbitrary paths.

## 10. Interviewer follow-ups

**Return the actual path?** Store which child branch produced each gain and record the node where the global maximum is updated; reconstruct both sides afterward.

**What if path must start at root?** Then the global turn logic changes; only downward root-starting paths matter.

**Why postorder?** The current node needs solved child gains before it can compute its own state.

**Related problems?** Diameter of Binary Tree uses the same “return one branch, update answer with two branches” structure.

## 11. What to say aloud

> “The key distinction is between what I can return to the parent and what can form a complete answer at this node. A parent can extend through only one child, so I return node value plus the larger nonnegative child gain. But a full path may use both sides, so I update a global maximum with node value plus both nonnegative gains. Postorder DFS visits every node once, so the solution is O(n).”

## 12. Pattern takeaway

```text
Tree path problem
      ↓
Need child information first
      ↓
Postorder DFS
      ↓
Return one branch upward
      ↓
Use two branches for local complete answer
      ↓
Global maximum
```

**Practice next:** LeetCode 543, 687, and 1372.