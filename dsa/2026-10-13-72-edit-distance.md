# DSA Interview Walkthrough — LeetCode 72: Edit Distance

**Difficulty:** Medium  
**Pattern:** Two-String Dynamic Programming · Memoization · Space Optimization  
**Core lesson:** When two sequences must be transformed into each other using a small set of edits, define DP by how much of each sequence has been processed. The last operation gives the recurrence.

## 1. Interviewer prompt

Given `word1` and `word2`, return the minimum number of insertions, deletions, and replacements needed to convert `word1` into `word2`.

```text
word1 = "horse"
word2 = "ros"
answer = 3
```

Clarify that all three operations cost one, empty strings are allowed, and matching equal characters is free.

## 2. Recognize the pattern

Two strings + minimum transformations + local choices at a mismatch strongly suggests two-dimensional DP.

If the final characters match, no edit is needed for them. Otherwise the last edit must be one of:
- delete from the source,
- insert into the source,
- replace the source character.

Each choice reduces the problem to shorter prefixes.

## 3. Brute force

```python
def brute(word1, word2):
    n, m = len(word1), len(word2)

    def solve(i, j):
        if i == n:
            return m - j
        if j == m:
            return n - i

        if word1[i] == word2[j]:
            return solve(i + 1, j + 1)

        delete = solve(i + 1, j)
        insert = solve(i, j + 1)
        replace = solve(i + 1, j + 1)
        return 1 + min(delete, insert, replace)

    return solve(0, 0)
```

A loose upper bound is exponential, about `O(3^(n+m))`, with `O(n+m)` recursion depth.

The repeated state is `(i, j)`, so memoization reduces the number of distinct subproblems to `O(nm)`.

## 4. Memoized recursion

```python
from functools import cache

def memoized(word1, word2):
    n, m = len(word1), len(word2)

    @cache
    def solve(i, j):
        if i == n:
            return m - j
        if j == m:
            return n - i

        if word1[i] == word2[j]:
            return solve(i + 1, j + 1)

        return 1 + min(
            solve(i + 1, j),
            solve(i, j + 1),
            solve(i + 1, j + 1)
        )

    return solve(0, 0)
```

Time is `O(nm)`, space is `O(nm)` for memoization plus recursion depth.

## 5. Bottom-up recurrence

Define:

`dp[i][j] = minimum edits to transform the first i characters of word1 into the first j characters of word2`.

Base cases:
- `dp[0][j] = j`
- `dp[i][0] = i`

If `word1[i-1] == word2[j-1]`:

```text
dp[i][j] = dp[i-1][j-1]
```

Otherwise:

```text
dp[i][j] = 1 + min(
    dp[i-1][j],     # delete
    dp[i][j-1],     # insert
    dp[i-1][j-1]    # replace
)
```

## 6. Dry run

For `horse -> ros`:

| source \ target | ∅ | r | o | s |
|---|---:|---:|---:|---:|
| ∅ | 0 | 1 | 2 | 3 |
| h | 1 | 1 | 2 | 3 |
| ho | 2 | 2 | 1 | 2 |
| hor | 3 | 2 | 2 | 2 |
| hors | 4 | 3 | 3 | 2 |
| horse | 5 | 4 | 4 | **3** |

The bottom-right cell is the answer.

## 7. Space optimization

Each row depends only on the previous row and the current row's left neighbor. Keep two rows instead of the whole matrix. Put the shorter string on the columns to get `O(min(n,m))` memory.

## 8. Clean final Python solution

```python
class Solution:
    def minDistance(self, word1: str, word2: str) -> int:
        if len(word2) > len(word1):
            word1, word2 = word2, word1

        prev = list(range(len(word2) + 1))

        for i, a in enumerate(word1, 1):
            curr = [i] + [0] * len(word2)

            for j, b in enumerate(word2, 1):
                if a == b:
                    curr[j] = prev[j - 1]
                else:
                    curr[j] = 1 + min(
                        prev[j],
                        curr[j - 1],
                        prev[j - 1]
                    )

            prev = curr

        return prev[-1]
```

No lambda functions are used.

## 9. Complexity

- Time: `O(nm)`.
- Auxiliary space: `O(min(n,m))`.

## 10. Correctness

Every optimal transformation between two prefixes ends in exactly one of four possibilities: match, delete, insert, or replace. The recurrence considers all valid final operations and picks the minimum cost. Base cases correctly handle conversion to or from an empty prefix.

## 11. Edge cases and mistakes

- Both strings empty -> 0.
- One string empty -> length of the other.
- Identical strings -> 0.
- Repeated characters need no special handling.

Common mistakes:
- mixing character indices with prefix lengths,
- forgetting the first row/column,
- charging an edit when characters already match,
- overwriting the previous row too early,
- using greedy mismatch repair.

## 12. Interviewer follow-ups

**Return the edit operations?** Keep the full DP table and backtrack.

**Weighted operations?** Replace the `+1` with operation-specific costs.

**Replacement forbidden?** Remove the diagonal replacement option; this becomes closely related to LCS.

**Distance at most k?** Use banded DP around the main diagonal when appropriate.

**Damerau-Levenshtein?** Add adjacent transposition as another operation/state transition.

## 13. What to say aloud

> “This is two-string DP. Brute force tries insert, delete, and replace at every mismatch, which is exponential. I define dp[i][j] as the minimum edits between the first i and first j characters. Matching characters take the diagonal value for free; otherwise I take one plus the minimum of delete, insert, and replace. That gives O(nm) time. Because each row only depends on the previous row and its own left neighbor, I can compress memory to O(min(n,m)).”

## 14. Pattern takeaway

```text
Two strings + minimum transformation
            ↓
Take/skip/edit recursion
            ↓
Repeated (i,j) states
            ↓
2D DP over prefix lengths
            ↓
Match = diagonal
Mismatch = 1 + min(top, left, diagonal)
            ↓
O(nm) time, O(min(n,m)) space
```

**Practice next:** LeetCode 1143, 583, 712, and 97.