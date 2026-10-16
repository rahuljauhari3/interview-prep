# DSA Interview Walkthrough — LeetCode 787: Cheapest Flights Within K Stops

**Difficulty:** Medium  
**Patterns:** Bellman–Ford · Dynamic Programming · Bounded Shortest Path · Edge Relaxation  
**Core lesson:** A shortest path is not determined by cost alone when the number of flights is restricted. Track the best cost achievable with a bounded number of edges. Bellman–Ford's layered relaxation solves this cleanly without accidentally using too many flights.

## 1. Interviewer: Problem statement

**Interviewer:** You are given **n** cities and directed flights of the form **[from, to, price]**. Find the cheapest price from **src** to **dst** using **at most k stops**. If no such route exists, return **-1**.

~~~text
n = 4
flights = [
    [0, 1, 100],
    [1, 2, 100],
    [0, 2, 500],
    [2, 3, 100]
]
src = 0
dst = 3
k = 1

Answer: 600
~~~

**Candidate:** Does a stop mean an intermediate city? Are flight prices positive? Are flights directed? Can I revisit a city?

**Interviewer:** Yes. A route with k intermediate stops has at most **k + 1 flights**. Prices are positive, flights are directed, and revisits are not explicitly forbidden.

**Candidate:** So I need the minimum-cost path subject to an **edge-count budget**. The route 0 → 1 → 2 → 3 costs 300 but uses three flights, which exceeds the two-flight budget. The valid route 0 → 2 → 3 costs 600.

## 2. Recognize the pattern

The words **cheapest route** suggest shortest paths, but **at most k stops** changes the problem.

Ordinary Dijkstra's algorithm keeps one best distance per node. That is not enough here: reaching a city cheaply after many flights may leave too few flights to reach the destination, while a more expensive arrival with fewer flights can still be useful.

This gives the key insight:

**The state must include how many flights have been used, or how many are still allowed.**

There are two practical directions:

- A priority queue with states such as **(cost, city, flights_used)**.
- Dynamic programming by flight count, implemented using **Bellman–Ford relaxation**.

For this problem, layered Bellman–Ford is especially concise and avoids subtle pruning errors.

**Interviewer:** How do you translate the stop limit into the algorithm?

**Candidate:** At most k stops means at most **k + 1 edges**. I'll perform exactly k + 1 rounds of relaxation, where each round permits one additional flight.

## 3. Brute force: enumerate possible routes

Build an adjacency list and use DFS. From each city, try every outgoing flight, but never use more than k + 1 flights. Keep the cheapest route that reaches the destination.

~~~python
def brute(n, flights, src, dst, k):
    graph = [[] for _ in range(n)]
    for u, v, price in flights:
        graph[u].append((v, price))

    best = float("inf")

    def dfs(city, used, cost):
        nonlocal best

        if city == dst:
            best = min(best, cost)
            return
        if used == k + 1 or cost >= best:
            return

        for nxt, price in graph[city]:
            dfs(nxt, used + 1, cost + price)

    dfs(src, 0, 0)
    return -1 if best == float("inf") else best
~~~

Because prices are positive, pruning when **cost >= best** is safe. We do not need a global visited set: a city may be reached with different costs and remaining flight budgets. The edge limit guarantees termination.

Let **b** be the maximum number of outgoing flights from a city.

**Time:** O(b^(k+1)) in the worst case, excluding graph construction.  
**Space:** O(n + E + k) for the graph and recursion, where E is the number of flights.

**Interviewer:** What's the repeated work?

**Candidate:** Many routes reach the same city after the same number of flights. I only need to remember the cheapest such route; more expensive routes with the same remaining budget cannot improve the answer.

## 4. First optimization: define a DP state

Let **dp[t][v]** be the minimum cost of reaching city **v** from **src** using **at most t flights**.

The base case is:

~~~text
dp[0][src] = 0
dp[0][v] = infinity for every other city
~~~

For each additional allowed flight, either keep the previous best route or take one final flight **u → v**:

~~~text
dp[t][v] = min(
    dp[t-1][v],
    min(dp[t-1][u] + price(u, v) for each flight u → v)
)
~~~

The first term means **use fewer than t flights**. The second term means **reach u within t - 1 flights, then take u → v**.

We want:

~~~text
dp[k+1][dst]
~~~

This is a standard **bounded shortest-path DP**. Bellman–Ford implements exactly these transitions by relaxing every directed edge once per round.

## 5. Space optimization: keep only two layers

Notice that **dp[t]** depends only on **dp[t-1]**, not on every earlier layer. Therefore, we can store one array **dist** for the previous round and make a copy **next_dist** for the current round.

For each flight **u → v** costing **w**:

~~~text
next_dist[v] = min(next_dist[v], dist[u] + w)
~~~

After all flights are processed, assign **dist = next_dist**.

This reduces DP storage from **O((k+1)n)** to **O(n)**.

### Why the copy is essential

Suppose the flights are listed in this order:

~~~text
0 → 1 (100)
1 → 2 (100)
2 → 3 (100)
0 → 3 (1000)
~~~

With **k = 0**, only **one flight** is allowed, so the correct answer is **1000**.

If we update **dist** in place during a single round, the first edge makes city 1 reachable for 100, the second immediately uses that new value to reach city 2 for 200, and the third reaches city 3 for 300. That incorrectly uses **three flights in one round**.

Using **next_dist = dist.copy()** prevents this. Every relaxation in round t reads only costs from round t - 1, so it adds at most one flight.

**Interviewer:** Is this the same as ordinary Bellman–Ford?

**Candidate:** It uses the same edge-relaxation principle. The difference is that we stop after **k + 1 rounds** rather than the usual n - 1 rounds for unrestricted shortest paths, and we explicitly separate layers to enforce the flight budget.

## 6. Detailed dry run

Use the original example:

~~~text
n = 4
flights = [
    [0, 1, 100],
    [1, 2, 100],
    [0, 2, 500],
    [2, 3, 100]
]
src = 0, dst = 3, k = 1
~~~

We can use at most **2 flights**, so we run two relaxation rounds.

**Initial state — zero flights:**

~~~text
city:       0    1    2    3
dist:       0    ∞    ∞    ∞
~~~

**Round 1 — at most one flight:**

From city 0, we can reach city 1 for 100 and city 2 for 500. City 3 is not reachable in one flight.

~~~text
dist = [0, 100, 500, ∞]
~~~

**Round 2 — at most two flights:**

The flight 1 → 2 improves city 2 from 500 to **200**. The flight 2 → 3 reaches city 3 for **600**, using the **previous round's** cost to city 2, which was 500.

~~~text
dist = [0, 100, 200, 600]
~~~

| Allowed flights | Cost to 0 | Cost to 1 | Cost to 2 | Cost to 3 |
|---|---:|---:|---:|---:|
| 0 | 0 | ∞ | ∞ | ∞ |
| 1 | 0 | 100 | 500 | ∞ |
| 2 | 0 | 100 | 200 | **600** |

The cheaper route **0 → 1 → 2 → 3** costs 300 but needs **three flights**, so it is not yet allowed.

**Interviewer:** What changes if k becomes 2?

**Candidate:** We perform a third round. The previously computed cost 200 to city 2 can now be extended to city 3 for 300, so the answer becomes **300**.

## 7. Clean final LeetCode solution — Python

~~~python
class Solution:
    def findCheapestPrice(
        self, n: int, flights: list[list[int]],
        src: int, dst: int, k: int
    ) -> int:
        dist = [float("inf")] * n
        dist[src] = 0

        for _ in range(k + 1):
            next_dist = dist[:]

            for u, v, price in flights:
                if dist[u] != float("inf"):
                    next_dist[v] = min(next_dist[v], dist[u] + price)

            dist = next_dist

        return -1 if dist[dst] == float("inf") else dist[dst]
~~~

The code is iterative, uses **no lambda functions**, does not mutate the input, and keeps the flight limit correct regardless of the order of flights.

## 8. Complexity analysis

Let **V = n**, **E = len(flights)**, and **L = k + 1**, the maximum number of flights allowed.

Each round copies an array of V distances and relaxes E edges, taking **O(V + E)** time. There are L rounds.

**Time:** **O(L(V + E)) = O((k+1)(n+E))**.  
**Auxiliary space:** **O(V)** for the previous and current distance arrays.

The input flights are read directly; no adjacency list is needed in the optimized solution.

## 9. Correctness proof

**Invariant:** After round t, **dist[v]** is the minimum cost of any route from src to v using at most t flights, or infinity if none exists.

**Base case:** Before any rounds, src has cost 0 and every other city is unreachable. This is exactly the best cost using zero flights.

**Inductive step:** Assume the invariant holds after t - 1 rounds. Any route to v using at most t flights either uses at most t - 1 flights, already represented by **dist[v]**, or ends with some flight **u → v**. In the latter case, the preceding route to u uses at most t - 1 flights, and by the induction hypothesis its minimum cost is **dist[u]**. Relaxing all edges into a separate copied array considers every such final flight without permitting more than one new edge. Taking the minimum therefore yields the optimal cost with at most t flights.

After k + 1 rounds, the invariant gives the minimum cost using at most k + 1 flights, which is exactly the required limit of k intermediate stops. Returning -1 for infinity correctly handles unreachable destinations.

## 10. Edge cases and common mistakes

| Situation | Expected behavior |
|---|---|
| Direct flight exists and k = 0 | Only direct flights may be used |
| Cheaper route requires too many flights | Reject it |
| No route to destination within budget | Return -1 |
| Multiple flights between the same cities | Use the cheapest feasible option |
| A route with fewer than k + 1 flights is optimal | Accept it |
| Flights appear in an unfavorable order | Still return the same answer |
| Source equals destination | Return 0 |

**Common mistakes:** Treating k stops as k flights instead of **k + 1 flights**; relaxing distances in place and accidentally chaining several flights within one round; using a single global visited set; treating flights as undirected; returning the cheapest unrestricted path; and assuming the cheapest arrival at a city is always the most useful arrival regardless of remaining flight budget.

## 11. Why ordinary Dijkstra needs extra state

Consider these routes to city 2:

~~~text
0 → 1 → 2 : cost 200, uses 2 flights
0 → 2     : cost 500, uses 1 flight
2 → 3     : cost 100
~~~

If the entire journey may use at most **2 flights**, the cheapest arrival at city 2 (cost 200) **cannot continue** to city 3. The more expensive arrival at city 2 (cost 500) can continue, producing a valid total cost of 600.

A Dijkstra implementation that stores only **best_cost[city]** may discard the useful state **(city 2, cost 500, one flight used)**.

To use a priority queue correctly, track states such as **(city, flights_used)**, or maintain separate best costs by flight count. Bellman–Ford's layered DP handles this constraint naturally.

## 12. Interviewer-style follow-up questions

**Q1. Why exactly k + 1 rounds?** A route with k intermediate stops contains k + 1 flights. Each relaxation round permits at most one additional flight.

**Q2. Why copy the distance array?** Without copying, an edge relaxed early in a round can immediately feed another edge in the same round, violating the one-new-flight-per-round rule.

**Q3. Why does the recurrence include the old distance?** The problem permits **at most** t flights, not exactly t. A route using fewer flights must remain available.

**Q4. Can Dijkstra solve this problem?** Yes, but the state must include the number of flights used or remaining. A single best distance per city is insufficient.

**Q5. Can we stop early if no distance changes in a round?** Yes. If an entire round makes no improvement, later rounds cannot improve anything either.

**Q6. What if k is very large?** With strictly positive prices, an optimal route never needs a cycle, so at most n - 1 flights suffice when src and dst differ. We can cap the rounds at n - 1.

**Q7. What if we must return the actual route?** Store predecessor information for each DP layer and reconstruct the chosen sequence of flights from the best destination state.

**Q8. What if prices could be negative?** The layered DP still correctly finds the minimum cost under a fixed edge budget. The positive-price assumption mainly supports certain pruning and simple-path arguments.

**Q9. When would you prefer the priority-queue approach?** When the graph is sparse, the destination can be reached quickly, or the number of explored state combinations is much smaller than all (k + 1) relaxation rounds.

## 13. What to say aloud in the interview

> “This is a shortest-path problem with an edge-count constraint. Brute-force DFS explores possible routes up to k + 1 flights, which can be exponential. Ordinary Dijkstra with one distance per city is unsafe because a cheaper arrival may use too many flights. I'll use Bellman–Ford-style dynamic programming. Let dist[v] be the cheapest cost to reach v using at most the number of flights allowed by the previous round. For each of k + 1 rounds, I'll copy dist and relax every directed flight using only the previous round's values. The copy ensures I never add more than one flight per round. This takes O((k + 1)(V + E)) time and O(V) extra space.”

## 14. Pattern takeaway and practice

~~~text
Cheapest path with at most K intermediate stops
                       |
                       v
Translate stops into K + 1 flights
                       |
                       v
Brute force: explore bounded routes
                       |
                       v
State must track the flight budget
                       |
                       v
DP: best cost using at most t flights
                       |
                       v
Relax all edges from the PREVIOUS layer
                       |
                       v
Repeat K + 1 rounds
                       |
                       v
O((K + 1)(V + E)) time, O(V) space
~~~

**Practice next:** LeetCode 743 (Network Delay Time) for unrestricted shortest paths with Dijkstra; LeetCode 1631 (Path With Minimum Effort) for a different path-cost objective; LeetCode 1928 (Minimum Cost to Reach Destination in Time) for a second constrained resource; and LeetCode 1334 (Find the City With the Smallest Number of Neighbors at a Threshold Distance) for all-pairs shortest paths.

**Transferable interview insight:** When a shortest-path problem adds a constraint such as **maximum flights, time, stops, or fuel**, a single distance per node may no longer represent enough information. Identify the missing state dimension before choosing the algorithm.