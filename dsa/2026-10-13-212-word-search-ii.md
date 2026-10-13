# DSA Interview Walkthrough — LeetCode 212: Word Search II

**Difficulty:** Hard  
**Pattern:** Trie (Prefix Tree) · Backtracking · Grid DFS · Prefix Pruning  
**Core lesson:** When many words must be searched in the same grid, searching for each word independently repeats work. Build a trie of all words, then explore the board while pruning any path that is not a prefix of a target word.

## 1. Interviewer prompt

Given an `m x n` board of lowercase letters and a list of words, return every word that can be formed by following horizontally or vertically adjacent cells. A cell cannot be reused within the same word.

Example:

```text
board = [
  ["o","a","a","n"],
  ["e","t","a","e"],
  ["i","h","k","r"],
  ["i","f","l","v"]
]
words = ["oath", "pea", "eat", "rain"]

answer = ["oath", "eat"]
```

Clarifications to ask:
- No diagonal moves.
- A cell may be used by different words, but not twice on the same DFS path.
- Return each word once; order does not matter.

## 2. Recognize the pattern

A single word in a character grid suggests DFS + backtracking. The fact that we need to search **many** words suggests a trie so that common prefixes share work.

The pruning question is:

> Does the string represented by the current board path remain a prefix of at least one target word?

If not, stop immediately.

## 3. Brute force: search every word independently

For each word, try every board cell as a starting point and run DFS.

```python
def brute(board, words):
    rows, cols = len(board), len(board[0])

    def exists(word):
        def dfs(r, c, i):
            if i == len(word):
                return True
            if not (0 <= r < rows and 0 <= c < cols):
                return False
            if board[r][c] != word[i]:
                return False

            ch = board[r][c]
            board[r][c] = "#"

            found = (
                dfs(r + 1, c, i + 1) or
                dfs(r - 1, c, i + 1) or
                dfs(r, c + 1, i + 1) or
                dfs(r, c - 1, i + 1)
            )

            board[r][c] = ch
            return found

        for r in range(rows):
            for c in range(cols):
                if dfs(r, c, 0):
                    return True
        return False

    return [word for word in words if exists(word)]
```

If `B = rows * cols`, `W` is the number of words, and `L` is maximum word length, a loose upper bound is `O(W * B * 4^L)`. The recursion depth is `O(L)`.

The main inefficiency is repeated exploration of shared prefixes.

## 4. Build a trie

A trie stores all prefixes once.

For `oath`, `oats`, and `eat`:

```text
(root)
 ├── o ── a ── t ── h  [oath]
 │              └── s  [oats]
 └── e ── a ── t      [eat]
```

Build it using nested dictionaries and store the complete word at a terminal marker:

```python
trie = {}

for word in words:
    node = trie
    for ch in word:
        node = node.setdefault(ch, {})
    node["$"] = word
```

Storing the complete word means DFS does not need to rebuild the current path string.

## 5. Trie + backtracking

At each cell:

1. Check bounds.
2. If the cell's character is not a child of the current trie node, prune.
3. Advance into the trie child.
4. If the node has `"$"`, record the complete word.
5. Mark the board cell temporarily as visited.
6. Explore the four neighbors.
7. Restore the cell while backtracking.

Remove only the terminal marker after finding a word so duplicate board paths do not duplicate the answer. Keep the trie node itself because a longer word may share that prefix.

## 6. Dry run

For `oath`, one valid path is:

```text
(0,0) o -> (0,1) a -> (1,1) t -> (2,1) h
```

The trie traversal follows `root -> o -> a -> t -> h`. The `h` node is terminal, so `oath` is appended.

For `eat`:

```text
(1,3) e -> (1,2) a -> (1,1) t
```

`pea` never starts because no board cell contains `p`. `rain` starts with `r` but no valid continuation reaches its terminal node.

## 7. Clean final Python solution

```python
class Solution:
    def findWords(self, board: list[list[str]], words: list[str]) -> list[str]:
        trie = {}
        for word in words:
            node = trie
            for ch in word:
                node = node.setdefault(ch, {})
            node["$"] = word

        rows, cols = len(board), len(board[0])
        ans = []

        def dfs(r, c, node):
            if not (0 <= r < rows and 0 <= c < cols):
                return

            ch = board[r][c]
            if ch not in node:
                return

            nxt = node[ch]
            if "$" in nxt:
                ans.append(nxt.pop("$"))

            board[r][c] = "#"
            dfs(r + 1, c, nxt)
            dfs(r - 1, c, nxt)
            dfs(r, c + 1, nxt)
            dfs(r, c - 1, nxt)
            board[r][c] = ch

        for r in range(rows):
            for c in range(cols):
                dfs(r, c, trie)

        return ans
```

No lambda functions are used, and the board is restored after every DFS path.

## 8. Complexity

Let `S = sum(len(word) for word in words)` and `L = max(len(word))`.

- Trie construction: `O(S)` time and space.
- Grid search: worst-case exponential in `L`; a simple loose upper bound is `O(B * 4^min(L,B))`.
- Recursion stack: `O(min(L,B))`.

The worst-case exponential factor remains, but trie pruning and shared prefix exploration greatly reduce repeated work in practice.

## 9. Correctness

Every returned word is valid because DFS follows only adjacent cells, prevents reuse on the same path, and reports a word only after following trie edges to its terminal marker.

Every valid target word is eventually found because the outer loops try its first board cell, DFS explores all four legal directions, and all prefixes of that word exist in the trie, so its valid path is never incorrectly pruned.

Removing the terminal marker after the first discovery guarantees uniqueness without removing longer words sharing that prefix.

## 10. Edge cases and common mistakes

- Single-cell board.
- Word longer than the total number of cells.
- Several words sharing a prefix, e.g. `at` and `atom`.
- Repeated board letters.
- Same word appearing through multiple different paths.

Common mistakes:
- Forgetting to restore the board after DFS.
- Allowing diagonal moves.
- Using one global visited set across different search paths.
- Deleting an entire trie node after finding a shorter word.
- Rebuilding the current string on every recursion unnecessarily.
- Claiming trie pruning makes worst-case complexity linear.

## 11. Interviewer follow-ups

**How is this different from LeetCode 79 Word Search?**  
79 searches for one word, so plain backtracking is enough. 212 benefits from a trie because many words share prefixes.

**Can we prune the trie further?**  
Yes. After DFS finishes using a child, if that child has no children or terminal marker left, remove it from its parent.

**Why store the whole word at the terminal node?**  
It lets us append the answer immediately without carrying or rebuilding the path string.

**Could we use a visited matrix?**  
Yes, at `O(B)` extra memory. In-place marking is simpler when temporary mutation is allowed.

**What if `#` can appear in the board alphabet?**  
Use a separate visited matrix or another collision-free representation.

**Can we prefilter impossible words?**  
Yes. Remove words longer than `B`, and optionally compare their character frequencies against the board.

## 12. What to say aloud

> “This is DFS with backtracking, but because we have many target words I do not want to search each word independently. I build a trie of all words and search the board and trie simultaneously. If a board character is not a valid trie child, I prune immediately. I mark a board cell temporarily so it cannot be reused on the current path, restore it while backtracking, and when I reach a terminal trie node I add the stored word and remove only that terminal marker to avoid duplicates. Trie construction is linear in the total dictionary size; board exploration is still exponential in maximum word length in the worst case, but shared-prefix pruning removes enormous repeated work.”

## 13. Pattern takeaway

```text
Many words + paths through a grid
                ↓
Single-word DFS repeated for every word
                ↓
Repeated exploration of shared prefixes
                ↓
Build a trie
                ↓
DFS board + trie simultaneously
                ↓
Missing trie child? prune
                ↓
Terminal node? record word
                ↓
Mark -> explore -> restore
```

**Practice next:** LeetCode 79 Word Search, 208 Implement Trie, and 211 Design Add and Search Words Data Structure.