# DSA Interview Walkthrough — LeetCode 10: Regular Expression Matching

**Difficulty:** Hard  
**Pattern:** Dynamic Programming · String Matching · State Transitions · Space Optimization  
**Core lesson:** When a pattern contains choices such as “skip this token” or “consume another character,” model the problem as matching two suffixes. The same suffix pairs recur, so memoization or bottom-up DP replaces exponential backtracking with O(mn) time.

## 1. Interviewer: Problem statement

Given a string `s` and a pattern `p`, implement regular-expression matching with only two special symbols:

- `.` matches any single character.
- `*` matches zero or more occurrences of the immediately preceding character.

The pattern must match the entire string, not just a substring.

```text
s = "aab"
p = "c*a*b"
Answer = True

c* matches zero c characters
a* matches two a characters
b  matches the final b
```

Clarifications: `*` applies only to the preceding token, zero occurrences are allowed, and `.*` can match any sequence including empty.

## 2. Recognize the pattern

At a starred token, there are two choices:

1. Skip the token and its star, matching zero occurrences.
2. Consume one matching character from the string while keeping the same pattern position, because the star may match more.

These branches can reach the same pair of string and pattern positions, suggesting dynamic programming on two indices.

Define `F(i, j)` as whether `s[i:]` matches `p[j:]`.

## 3. Brute force: recursive backtracking

```python
def brute(s, p):
    def dfs(i, j):
        if j == len(p):
            return i == len(s)

        first = i < len(s) and (p[j] == "." or s[i] == p[j])

        if j + 1 < len(p) and p[j + 1] == "*":
            return dfs(i, j + 2) or (first and dfs(i + 1, j))

        return first and dfs(i + 1, j + 1)

    return dfs(0, 0)
```

**Time:** Exponential in the worst case.  
**Auxiliary space:** O(m+n) recursion depth.

The repeated work is identical `(i,j)` suffix states reached through different choices.

## 4. First optimization: memoization

```python
from functools import cache

def memoized(s, p):
    @cache
    def dfs(i, j):
        if j == len(p):
            return i == len(s)

        first = i < len(s) and (p[j] == "." or s[i] == p[j])

        if j + 1 < len(p) and p[j + 1] == "*":
            return dfs(i, j + 2) or (first and dfs(i + 1, j))

        return first and dfs(i + 1, j + 1)

    return dfs(0, 0)
```

There are O(mn) states.

**Time:** O(mn).  
**Space:** O(mn) for the cache plus recursion stack.

## 5. Derive the recurrence

Let `dp[i][j]` mean whether `s[i:]` matches `p[j:]`.

If the pattern is exhausted:

```text
dp[i][n] = (i == m)
```

If `p[j]` is not followed by `*`:

```text
dp[i][j] = first and dp[i+1][j+1]
```

If `p[j+1] == "*"`:

```text
dp[i][j] = dp[i][j+2] or (first and dp[i+1][j])
```

The first branch skips the starred token; the second consumes one character and keeps the pattern position so `*` may consume more.

## 6. Dry run: `s = "aab"`, `p = "c*a*b"`

```text
F(0,0): c* cannot consume "a"
        skip c* -> F(0,2)

F(0,2): a* matches first "a"
        consume -> F(1,2)

F(1,2): a* matches second "a"
        consume -> F(2,2)

F(2,2): current string char is "b", not "a"
        skip a* -> F(2,4)

F(2,4): "b" matches "b"
        -> F(3,5)

F(3,5): both suffixes empty -> True
```

## 7. Space optimization

Each state depends on:

- `dp[i][j+2]` on the same row.
- `dp[i+1][j]` and `dp[i+1][j+1]` on the next row.

Compute `i` from `m` down to `0` and `j` from right to left. Keep only the current row and the row below it.

## 8. Clean final LeetCode solution — Python

```python
class Solution:
    def isMatch(self, s: str, p: str) -> bool:
        m, n = len(s), len(p)
        below = [False] * (n + 1)

        for i in range(m, -1, -1):
            row = [False] * (n + 1)
            row[n] = i == m

            for j in range(n - 1, -1, -1):
                first = i < m and (p[j] == "." or s[i] == p[j])

                if j + 1 < n and p[j + 1] == "*":
                    row[j] = row[j + 2] or (first and below[j])
                else:
                    row[j] = first and below[j + 1]

            below = row

        return below[0]
```

No lambda functions are used, and the inputs are not mutated.

## 9. Complexity analysis

Every pair `(i,j)` is evaluated once.

**Time:** O(mn).  
**Auxiliary space:** O(n).

## 10. Correctness proof

At `j=n`, the pattern is empty, so the match is true exactly when the string is also exhausted.

Without a star, a valid match must consume one character from each side, so the recurrence is `first and dp[i+1][j+1]`.

With a star, every valid match either uses zero copies of the token or at least one. Those cases are exactly `dp[i][j+2]` and `first and dp[i+1][j]`. Their OR therefore covers all possibilities.

The bottom-up order guarantees all required states are already available when a state is computed.

## 11. Edge cases and common mistakes

- Literal-only pattern.
- Dot matching exactly one character.
- `*` matching zero occurrences.
- `*` matching multiple occurrences.
- `.*` matching any sequence.
- Empty string with a pattern such as `a*b*`.

Common mistakes:
- Treating `*` as a standalone wildcard.
- Requiring `*` to match at least one character.
- Moving past the starred token immediately after consuming one character.
- Allowing `.` to match empty.
- Accepting substring matches instead of full-string matches.
- Filling the DP in an order that uses states not computed yet.

## 12. Interviewer-style follow-up questions

**Why does memoization remove the exponential repetition?**  
The future depends only on `(i,j)`, and there are only O(mn) such states.

**Why is the star case OR?**  
Skipping and consuming are alternative valid strategies.

**What if `?` meant zero or one occurrence?**  
Use a skip branch or one consume branch, but the consume branch advances past the token pair because repetition is not allowed.

**What if `+` meant one or more occurrences?**  
The first character must match, then either continue at the same token or advance beyond it.

**Is this a complete regex engine?**  
No. It solves only the dot-and-star language in the problem.

## 13. What to say aloud in the interview

> “The hard part is the star operator because it creates two choices: skip the token pair or consume one matching character and stay on the same pattern token. I define a DP state by the string index and pattern index. Plain recursion repeats the same states exponentially, while memoization reduces the problem to O(mn). In a bottom-up solution, each state depends only on the same row to the right and the next row, so I can keep two rows and reduce space to O(n).”

## 14. Pattern takeaway

```text
Matching with optional/repeated tokens
                ↓
Branching choices at "*"
                ↓
State = (string index, pattern index)
                ↓
Repeated states -> memoization
                ↓
Skip token OR consume character
                ↓
Bottom-up suffix DP
                ↓
O(mn) time, O(n) space
```

**Practice next:** LeetCode 44 Wildcard Matching, 72 Edit Distance, 115 Distinct Subsequences, and 97 Interleaving String.