# DSA Interview Walkthrough — LeetCode 721: Accounts Merge

**Difficulty:** Medium  
**Pattern:** Union-Find / DSU · Connected Components

## Problem
Each account is `[name, email1, email2, ...]`. If two accounts share any email, they belong to the same person. Merge all connected accounts and return each person's name followed by sorted unique emails.

## Recognition
Shared emails create an undirected connectivity relation. If account A shares an email with B and B shares one with C, all three belong to one connected component.

That is a natural Union-Find problem.

## Brute force
Build a graph between accounts by comparing every pair for shared emails. Pairwise account comparison is expensive and unnecessary.

Instead, use each email as a direct bridge between account indices.

## DSU strategy
Maintain:
- `email_owner[email] = first account index seen with this email`,
- DSU parent/size arrays over account indices.

When another account contains the same email, union the two account indices.

After all unions, group every email by the root of its owner account.

## Clean final Python solution

```python
from collections import defaultdict

class Solution:
    def accountsMerge(self, accounts: list[list[str]]) -> list[list[str]]:
        n = len(accounts)
        parent = list(range(n))
        size = [1] * n

        def find(x):
            while parent[x] != x:
                parent[x] = parent[parent[x]]
                x = parent[x]
            return x

        def union(a, b):
            a, b = find(a), find(b)
            if a == b:
                return

            if size[a] < size[b]:
                a, b = b, a

            parent[b] = a
            size[a] += size[b]

        email_owner = {}

        for i, account in enumerate(accounts):
            for email in account[1:]:
                if email in email_owner:
                    union(i, email_owner[email])
                else:
                    email_owner[email] = i

        groups = defaultdict(list)

        for email, owner in email_owner.items():
            root = find(owner)
            groups[root].append(email)

        ans = []
        for root, emails in groups.items():
            ans.append([accounts[root][0]] + sorted(emails))

        return ans
```

No lambda functions are used.

## Complexity
Let `E` be the total number of email occurrences.

DSU work is near-linear: `O(E α(n))`. Sorting emails inside the final groups dominates in many inputs, giving roughly `O(E log E)` in the worst case. Space is `O(E+n)`.

## Correctness
Whenever two accounts share an email, they are unioned, so every direct connection is represented. Union-Find's transitivity also merges chains of shared emails, exactly matching connected components. Grouping each email by the final root therefore collects precisely the emails belonging to one person.

## Common mistakes
- Treating only directly shared accounts as merged and missing transitive chains.
- Grouping by the original owner index without calling `find` after all unions.
- Forgetting to deduplicate shared email keys naturally through the `email_owner` map.
- Forgetting to sort final email lists.

## Interview follow-ups
- Could DFS solve it? Yes: build an email/account graph and find components.
- Why union account indices instead of emails? Either representation works; account DSU is compact when accounts are the natural component objects.
- What if names conflict inside a connected component? The original problem guarantees consistent names for the same person; otherwise clarify business rules.

## What to say aloud
> “Emails define connectivity between account records. I map each email to the first account that owns it. When another account uses that email, I union their account indices. After processing all emails, I group each unique email by the root of its owner and sort the group. DSU handles transitive merging efficiently.”

## Pattern takeaway
```text
Shared identifiers -> connected components -> Union-Find
```