# DSA Interview Walkthrough — LeetCode 739: Daily Temperatures

**Difficulty:** Medium  
**Pattern:** Monotonic Stack · Next Greater Element  
**Core lesson:** When each position asks for the next greater value to its right, process the array once while keeping unresolved indices in a monotonic stack.

## 1. Interviewer prompt

Given daily temperatures, return how many days must pass until a warmer temperature. If none exists, return 0.

```text
temperatures = [73,74,75,71,69,72,76,73]
answer = [1,1,4,2,1,1,0,0]
```

## 2. Recognize the pattern

Each day asks:

> Where is the first future value that is strictly greater than mine?

That is the **next greater element** pattern.

A monotonic stack lets a new value resolve multiple earlier positions at once.

## 3. Brute force

For every day, scan right until finding a warmer temperature.

```python
def brute(temperatures):
    ans = [0] * len(temperatures)

    for i in range(len(temperatures)):
        for j in range(i + 1, len(temperatures)):
            if temperatures[j] > temperatures[i]:
                ans[i] = j - i
                break

    return ans
```

Time: `O(n^2)`.

## 4. Monotonic-stack insight

Maintain a stack of indices whose warmer future day has not been found yet.

The temperatures at those indices are kept in **decreasing order**.

When a new temperature `t` arrives at index `i`, while `t` is greater than the temperature at the stack top:
- pop that earlier index `j`,
- its first warmer day is now `i`,
- set `answer[j] = i - j`.

Then push `i`.

## 5. Clean final Python solution

```python
class Solution:
    def dailyTemperatures(self, temperatures: list[int]) -> list[int]:
        ans = [0] * len(temperatures)
        stack = []

        for i, temp in enumerate(temperatures):
            while stack and temp > temperatures[stack[-1]]:
                j = stack.pop()
                ans[j] = i - j

            stack.append(i)

        return ans
```

No lambda functions are used.

## 6. Complexity

Each index is pushed once and popped at most once.

- Time: `O(n)` amortized.
- Auxiliary space: `O(n)`.

## 7. Correctness

The stack contains exactly unresolved indices. Because their temperatures are non-increasing, when a hotter day arrives, popping continues until all smaller top values are resolved. The first time an index is popped is the first warmer day to its right, since every earlier scanned day failed to exceed it.

## 8. Edge cases and common mistakes

- Strictly decreasing input -> all zeros.
- Strictly increasing -> mostly ones.
- Equal temperature is **not** warmer.

Common mistakes:
- storing only temperatures instead of indices,
- using `if` instead of `while`,
- using `>=` instead of `>`,
- calling the algorithm O(n²) because of the nested while-loop.

## 9. Interviewer follow-ups

**Next smaller element?** Reverse the comparison.

**Circular array?** Traverse indices twice while controlling which answers may be written.

**Why monotonic?** The stack keeps only candidates whose answer is still unresolved and imposes enough order for a new element to resolve them efficiently.

## 10. What to say aloud

> “Brute force makes every day scan forward. Instead, I reverse the viewpoint: when a new temperature arrives, I ask which earlier unresolved days it can answer. I keep unresolved indices in a decreasing-temperature stack. A warmer current value repeatedly pops colder days and records the distance. Each index is pushed and popped at most once, so the solution is O(n).”

## 11. Pattern takeaway

```text
Next greater value to the right
          ↓
Unresolved indices
          ↓
Monotonic decreasing stack
          ↓
New greater value pops answers
          ↓
O(n)
```

**Practice next:** LeetCode 496, 503, 84, and 901.