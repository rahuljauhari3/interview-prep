# DSA Interview Walkthrough — LeetCode 208: Implement Trie

**Difficulty:** Medium  
**Pattern:** Trie / Prefix Tree

## Problem
Implement a trie supporting:
- `insert(word)`
- `search(word)`
- `startsWith(prefix)`

## Recognition
A hash set handles exact word lookup, but repeated prefix queries need shared-prefix structure. A trie stores one node per character transition, so common prefixes are stored once.

## Core structure
Each node needs:
- `children`: mapping from character to child node,
- `is_end`: whether a complete inserted word ends here.

`is_end` is necessary because a path can exist as a prefix without being a full word.

Example after inserting `app` and `apple`:

```text
root -> a -> p -> p* -> l -> e*
```

`*` means `is_end = True`.

## Clean final Python solution

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False


class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root

        for ch in word:
            if ch not in node.children:
                node.children[ch] = TrieNode()
            node = node.children[ch]

        node.is_end = True

    def search(self, word: str) -> bool:
        node = self._find(word)
        return node is not None and node.is_end

    def startsWith(self, prefix: str) -> bool:
        return self._find(prefix) is not None

    def _find(self, text: str):
        node = self.root

        for ch in text:
            if ch not in node.children:
                return None
            node = node.children[ch]

        return node
```

No lambda functions are used.

## Complexity
For a word/prefix of length `L`:
- insert: `O(L)`
- search: `O(L)`
- startsWith: `O(L)`

Space is proportional to the total number of distinct trie nodes, bounded by the total inserted character count.

## Correctness
Following character edges from the root exactly matches a prefix path. `startsWith` succeeds whenever that path exists. `search` additionally requires the final node to be marked `is_end`, so a proper prefix is not mistaken for a complete inserted word.

## Common mistakes
- Omitting `is_end`, making `search("app")` true after inserting only `apple`.
- Recreating shared prefix nodes instead of reusing them.
- Returning false from `startsWith` unless the prefix itself is a full word.
- Using a set of complete strings and scanning them for every prefix query.

## Interview follow-ups
- Wildcard character search? DFS through multiple child branches (LeetCode 211).
- Word Search II? Combine trie prefix pruning with board backtracking.
- Fixed lowercase alphabet? Replace dictionaries with arrays of size 26 for speed/memory tradeoffs.
- Delete a word? Unmark its terminal node and optionally prune unused nodes bottom-up.

## What to say aloud
> “A trie is useful because exact words and prefixes share the same character paths. Each node stores child transitions and an end-of-word flag. Insert walks or creates the path; search walks the same path and checks the terminal flag; startsWith only requires the path to exist. All operations are linear in the input string length.”

## Pattern takeaway
```text
Many string lookups + prefix queries -> share prefixes explicitly -> trie
```