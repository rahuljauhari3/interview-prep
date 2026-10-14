# DSA Interview Walkthrough — LeetCode 895: Maximum Frequency Stack

**Difficulty:** Hard  
**Pattern:** Hash Map · Stack · Frequency Buckets · Data Structure Design  
**Core lesson:** When a data structure prioritizes both frequency and recency, use one stack per frequency. A dictionary tracks counts and another dictionary stores those stacks, giving average O(1) push and pop.

## 1. Interviewer: Problem statement

**Interviewer:** Design a data structure supporting two operations:

- **push(val):** Add an integer.
- **pop():** Remove and return the integer with the highest current frequency. If multiple values tie, remove the one most recently pushed.

Each pop removes one occurrence. Assume pop is never called on an empty structure.

~~~text
push(5), push(7), push(5), push(7), push(4), push(5)

pop() -> 5
pop() -> 7
pop() -> 5
pop() -> 4
~~~

**Candidate:** Does frequency mean the number of copies still present, rather than the total number ever pushed?

**Interviewer:** Yes. Frequencies decrease after pops.

**Candidate:** For a frequency tie, do we choose whichever value's latest remaining push happened most recently?

**Interviewer:** Exactly.

## 2. Recognize the pattern and build intuition

There are two priorities, in order: **maximum frequency**, then **most recent push**.

A normal stack handles recency but ignores frequency. A counter handles frequency but ignores recency. A heap could combine both priorities, but that costs O(log n) per operation.

Notice something special: a push increases exactly one value's frequency by **one**, and a pop decreases exactly one value's frequency by **one**. Frequencies form discrete layers (1, 2, 3, ...), so we can store a stack for each layer and access the highest layer directly.

That is the central pattern: **bucket by the primary priority, then use a suitable data structure inside each bucket for the tie-break**.

## 3. Brute force: scan active pushes

Keep a list of all currently active pushes, plus a frequency dictionary. On pop, find the largest frequency, scan backward to locate the most recent occurrence of a value with that frequency, and remove it.

~~~python
class BruteFreqStack:
    def __init__(self):
        self.values = []
        self.freq = {}

    def push(self, val):
        self.values.append(val)
        self.freq[val] = self.freq.get(val, 0) + 1

    def pop(self):
        highest = max(self.freq.values())

        for i in range(len(self.values) - 1, -1, -1):
            val = self.values[i]
            if self.freq[val] == highest:
                self.values.pop(i)
                self.freq[val] -= 1
                if self.freq[val] == 0:
                    del self.freq[val]
                return val
~~~

**Push:** O(1) amortized.  
**Pop:** O(n) worst case: finding the maximum, scanning backward, and removing from a Python list can each require linear work.  
**Space:** O(n) for n active occurrences.

**Interviewer:** Why scan backward?

**Candidate:** Among the values tied at maximum frequency, the rightmost active push is the newest, which is exactly the tie-break.

## 4. First optimization: max-heap

On each push, store an event prioritized by:

~~~text
(frequency reached by this push, timestamp)
~~~

A max-heap returns the highest frequency, breaking ties by the latest timestamp. Python's min-heap can represent this with negative frequency and negative timestamp.

Why does a push event retain its original frequency even after later pops? A value's pushes form layers. If a value was pushed three times, there are records for layers 1, 2, and 3. Popping layer 3 leaves layer 2 available.

**Time:** O(log n) per push and pop.  
**Space:** O(n).

This is a valid improvement, but the heap performs general ordering that we do not need. The maximum frequency changes in steps of one, so a frequency-indexed bucket is enough.

## 5. Optimal insight: stacks grouped by frequency

Maintain:

- **freq[val]:** current count of val.
- **group[f]:** stack of values whose pushes raised their frequency to f.
- **max_freq:** highest current frequency.

### Push

Suppose val currently has frequency f-1.

1. Increase its frequency to f.
2. Append val to group[f].
3. Set max_freq to the larger of its old value and f.

### Pop

1. Pop the top of group[max_freq].
2. Decrease the popped value's frequency.
3. If group[max_freq] becomes empty, delete it and decrease max_freq by one.
4. Return the popped value.

**Why does a stack resolve ties?** Every value currently at maximum frequency f has an active record in group[f]. The most recently pushed such record is at the end of that list.

**Interviewer:** When a value drops from frequency 3 to 2, shouldn't we push it into group[2]?

**Candidate:** No. Its earlier second push already created a record in group[2]. Adding another would duplicate an occurrence.

## 6. Detailed dry run

After pushing 5, 7, 5, 7, 4, 5:

| Operation | freq(5) | freq(7) | freq(4) | max_freq |
|---|---:|---:|---:|---:|
| push(5) | 1 | 0 | 0 | 1 |
| push(7) | 1 | 1 | 0 | 1 |
| push(5) | 2 | 1 | 0 | 2 |
| push(7) | 2 | 2 | 0 | 2 |
| push(4) | 2 | 2 | 1 | 2 |
| push(5) | 3 | 2 | 1 | 3 |

The frequency stacks are:

~~~text
group[3] = [5]         <- highest frequency
group[2] = [5, 7]
group[1] = [5, 7, 4]
~~~

**Pop 1:** Remove 5 from group[3]. Its frequency becomes 2. group[3] is empty, so max_freq becomes 2. Return **5**.

**Pop 2:** group[2] contains [5,7]. Both values have frequency 2, but 7 reached frequency 2 later. Return **7**.

**Pop 3:** group[2] now contains [5]. Return **5**. The group empties, so max_freq becomes 1.

**Pop 4:** group[1] contains [5,7,4]. All remaining values have frequency 1, and 4 was pushed most recently. Return **4**.

The result is **[5,7,5,4]**.

## 7. Clean final LeetCode solution — Python

~~~python
class FreqStack:
    def __init__(self):
        self.freq = {}
        self.group = {}
        self.max_freq = 0

    def push(self, val: int) -> None:
        f = self.freq.get(val, 0) + 1
        self.freq[val] = f
        self.group.setdefault(f, []).append(val)
        self.max_freq = max(self.max_freq, f)

    def pop(self) -> int:
        val = self.group[self.max_freq].pop()
        self.freq[val] -= 1

        if self.freq[val] == 0:
            del self.freq[val]

        if not self.group[self.max_freq]:
            del self.group[self.max_freq]
            self.max_freq -= 1

        return val
~~~

No lambda functions, sorting, heap, or explicit timestamps are required.

## 8. Complexity analysis

Let n be the number of active occurrences.

**Push:** O(1) average/amortized time for dictionary operations and list append.

**Pop:** O(1) average time for dictionary operations and list pop from the end.

**Auxiliary space:** O(n). Each active push corresponds to exactly one record in a frequency stack. The frequency dictionary contains at most n distinct active values.

The bounds are average/amortized rather than strict worst-case guarantees because Python dictionaries use hashing and lists sometimes resize.

## 9. Correctness proof

Maintain these invariants:

1. freq[val] is the number of active copies of val.
2. Every active value at frequency f has its f-th push record in group[f].
3. Within each group, records appear in chronological push order.
4. max_freq is the largest active frequency.

Initially the structure is empty, so the invariants hold.

**Push:** Increasing val's count from f-1 to f and appending it to group[f] adds exactly its new f-th occurrence. Append order preserves recency. Updating max_freq preserves the maximum.

**Pop:** Every value at maximum frequency f has an active record in group[f]. The last such record represents the most recent push among the tied values, so popping it is exactly what the problem requires. Its frequency decreases by one; its previous layer record remains in group[f-1]. If group[f] becomes empty, no value still has frequency f. Because a pop decreases a count by only one, the new maximum is f-1 (or zero if empty).

Thus the invariants hold after every operation, and each pop returns the required value.

## 10. Edge cases and common mistakes

**One element:** push(8), pop() returns 8 and leaves max_freq = 0.

**All identical:** Three pushes of 2 create layers 1, 2, and 3. Three pops return 2 each time.

**Frequency tie:** push(5), push(7), pop() returns 7 because both frequencies are 1.

**Reinsert after pop:** push(5), push(5), pop(), push(5), pop() returns 5 both times.

**Zero and negative integers:** Work normally as dictionary keys.

Common mistakes include using a plain stack and ignoring frequency; using only a counter and losing recency; storing a set instead of a stack per frequency; moving a popped value into a lower group again; forgetting to decrement its count; and decreasing max_freq after every pop instead of only when the top group becomes empty.

## 11. Interviewer-style follow-up questions

**Q1. Why is group[max_freq] always the right place to pop?** Every value tied at maximum frequency has a record there, and the group's top is the most recent of those records.

**Q2. Why can max_freq decrease by only one?** One pop decreases only one frequency by one. If the old maximum disappears, the popped value occupies the immediately lower layer unless the structure is empty.

**Q3. Why not use a heap?** A heap works in O(log n), but direct frequency buckets reduce the operations to average O(1).

**Q4. What if a value's frequency reaches zero?** Delete its count entry. A future push starts again at frequency 1.

**Q5. Why don't we need timestamps?** Append order inside each frequency stack already captures recency.

**Q6. Could we delete an arbitrary occurrence efficiently?** Not with this simple design. Arbitrary deletion would require extra indexing and maintenance of frequency-group order.

**Q7. Is this thread-safe?** No. Concurrent pushes and pops would need synchronization around updates to the dictionaries and max_freq.

**Q8. How is this related to LFU Cache?** Both prioritize frequency and use recency to break ties, but LFU Cache also needs keyed lookup and arbitrary eviction.

## 12. What to say aloud in the interview

> “Brute force scans the active pushes for the most frequent value and breaks ties by scanning backward, but pop is O(n). A heap can improve that to O(log n). We can do better because every push changes a frequency by exactly one. I'll maintain a map from values to counts, a stack for each frequency, and the current maximum frequency. Push appends to the stack for the new frequency. Pop removes the top of the maximum-frequency stack, decreases the count, and lowers the maximum only if that group empties. The stacks automatically enforce recency ties. This gives average O(1) push and pop with O(n) space.”

## 13. Pattern takeaway and practice

~~~text
Maximum frequency, with recency tie-break
                    |
                    v
Brute-force scan -> O(n) pop
                    |
                    v
Heap by (frequency, timestamp) -> O(log n)
                    |
                    v
Frequency changes by only one
                    |
                    v
Map frequency -> stack of push records
                    |
                    v
Track max_freq directly
                    |
                    v
Average O(1) push/pop, O(n) space
~~~

**Practice next:** LeetCode 460 (LFU Cache), 432 (All O(1) Data Structure), 155 (Min Stack), and 146 (LRU Cache).

**Transferable interview insight:** When priorities change in small discrete steps, direct-access buckets can outperform a general-purpose heap. Choose the structure inside each bucket to enforce the secondary tie-break automatically.
