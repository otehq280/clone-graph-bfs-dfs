# Clone Graph: BFS and DFS Solutions, the One HashMap Trick, and Why Your Recursion Loops Forever

Clone Graph is LeetCode 133, and it looks like a five-minute problem right up until your first attempt loops forever. The code is short — a dozen lines in Python — but almost everyone who gets it wrong gets it wrong in the same two places: they forget the hash map, or they register the copy one line too late.

Below is the problem in plain terms, both working solutions, a step-by-step trace, the mistakes that actually show up in error messages, and where graph traversal sits in the bigger pattern picture if you're prepping for interviews rather than just submitting one answer.

## What "clone graph" actually means

You're given a reference to a single node in a connected undirected graph. Each node has a value and a list of neighbors. Your job is to return a deep copy: a brand-new graph with the same structure and the same values, where not a single node in the output is the same object as a node in the input.

Two things make that harder than it sounds.

First, you only get one node. You have to discover the rest of the graph by walking it. Second, the graph can contain cycles, and an adjacency list is symmetric — if 1 lists 2 as a neighbor, 2 lists 1 right back. Any naive recursive copy walks into that pair forever.

The test format on LeetCode is an adjacency list, 1-indexed, where the value of a node happens to equal its index. So `[[2,4],[1,3],[2,4],[1,3]]` means node 1 connects to 2 and 4, node 2 connects to 1 and 3, and so on. The constraints are small — up to 100 nodes, unique values, connected graph, no self-loops or duplicate edges — which matters later when you're deciding between recursion and iteration.

Three test cases are worth keeping in mind because they break sloppy solutions:

- `[[2,4],[1,3],[2,4],[1,3]]` — the normal four-node cycle
- `[[]]` — one node, no neighbors
- `[]` — an empty graph, so the input reference itself is null

## The entire problem is one idea

Maintain a hash map from original node to cloned node.

That map does double duty. It guarantees each node is cloned exactly once, and it's what lets you wire up edges between clones instead of accidentally pointing a clone at an original node. Whenever you touch a node, you check the map first. If it's there, you already have its copy — return it and stop. If it isn't, you create the copy, then **put it in the map before you touch any neighbors**.

That last part is the whole trick. Register the copy first, recurse second. If you clone the neighbor list first and only store the mapping on the way back up, a cycle sends you around the loop again before the map ever gets populated.

## DFS: clone, register, then recurse

python
class Solution:
    def cloneGraph(self, node: 'Node') -> 'Node':
        if not node:
            return None

        old_to_new = {}

        def dfs(cur):
            if cur in old_to_new:
                return old_to_new[cur]

            copy = Node(cur.val)
            old_to_new[cur] = copy          # register BEFORE recursing

            for nei in cur.neighbors:
                copy.neighbors.append(dfs(nei))

            return copy

        return dfs(node)


The function returns the clone corresponding to whatever node it was handed, so the caller can push the result straight into a neighbor list without a second lookup. That's what keeps the recursion clean.

Tracing it on `[[2,4],[1,3],[2,4],[1,3]]`:

1. `dfs(1)` — not in the map. Create `1'`, store `1 → 1'`.
2. Neighbor 2 → `dfs(2)` — create `2'`, store it, recurse into 1 and 3.
3. Back at neighbor 1, it's already in the map, so `1'` is returned directly and appended. No second copy, no loop.
4. Continue into 3, which pulls in 4, which points back at 1 and 3 — both already mapped. The whole graph resolves with exactly four new nodes.

If that "already in the map" branch is doing anything else for you, something's off. The base case is null, not "no neighbors," because a single isolated node is a perfectly valid graph.

## BFS: same map, queue instead of call stack

python
from collections import deque

class Solution:
    def cloneGraph(self, node: 'Node') -> 'Node':
        if not node:
            return None

        old_to_new = {node: Node(node.val)}
        q = deque([node])

        while q:
            cur = q.popleft()
            for nei in cur.neighbors:
                if nei not in old_to_new:
                    old_to_new[nei] = Node(nei.val)
                    q.append(nei)
                old_to_new[cur].neighbors.append(old_to_new[nei])

        return old_to_new[node]


The difference in shape: BFS clones a neighbor as soon as it first encounters it, then immediately connects the two clones and only enqueues the neighbor if it was new. The map lookup still prevents duplicate cloning.

Which one should you write? With 100 nodes maximum, recursion depth is a non-issue, so pick whichever you can produce under interview pressure without a bug. The argument for BFS is that it never depends on a call stack, which is the safe default when a problem's node count isn't bounded this tightly. The argument for DFS is that it's four lines shorter and reads closer to the problem statement.

One thing that doesn't change between them: DFS iterative with an explicit stack is a third option, and it's essentially BFS with the deque replaced by a list and `pop()` instead of `popleft()`. If you can write one, you can write all three.

## Time and space

Both solutions run in O(V + E) time — each node is visited once and each edge is examined once, which is the same bound you'll see listed on LeetCode's editorial and on AlgoMonster's write-up for 133. Space is O(V) for the map that holds one clone per node, plus O(V) for the recursion stack or queue.

That second O(V) is worth saying out loud in an interview. Graph problems are often really memory problems in disguise, and the moment you add a second structure — another visited set, a queue of pairs — the constant factor grows even if the big-O doesn't.

## Where people actually break

**Registering the clone too late.** Clone the node, build its neighbor list, then store the mapping, and a two-node cycle recurses until you hit a stack overflow. This is the single most common failure on this problem.

**Mutating the input.** A JavaScript solution that pops neighbors off the original array with `shift()` while iterating will not just clone the graph — it will destroy it. LeetCode runs your code and then compares the input graph against what you returned, and if you've stripped node 1's neighbor list, the checker reports something like "Node with value 2 doesn't exist in the original graph." That error message points at the input graph, not at your output, which is why it confuses people. Iterate over `cur.neighbors` without modifying it.

**Keying on `node.val` instead of the node itself.** It happens to work on this problem because every value is unique, and it fails the moment values repeat or the same value appears in two disconnected components. The map should key on identity, not on payload. Storing the integer value as a secondary index is a legitimate optimization if you're memory-constrained, but it's not the default you want to reach for.

**Returning the wrong reference.** The entry point is the node you were given, so the answer is `old_to_new[node]` (or the return of `dfs(node)`), not the last clone you built.

**Assuming a tree.** Trees have no cycles and one path between nodes, so `cloneTree` needs no map at all — AlgoMonster's write-up of Clone N-ary Tree makes that contrast directly. Graphs don't give you that assumption, and that's precisely why this problem exists as a separate drill.

If you want a step-by-step version of the same reasoning with the diagrams already drawn, 👉 [AlgoMonster's Clone Graph walkthrough](https://algo.monster?fpr=li69) covers 133 with language-specific implementations and complexity notes.

## Practicing the graph pattern instead of memorizing 133

Getting Clone Graph accepted is easy. Knowing why it's the same problem as counting islands, detecting cycles, or finding whether a path exists is the part that transfers to an interview question you haven't seen.

AlgoMonster's approach is built around that distinction. Its decision flowchart starts with "Is it a graph?" and branches from there: trees go one way, DAGs pull in topological sort, shortest-path questions split by whether the graph is weighted (Dijkstra versus plain BFS), connectivity questions point to Disjoint Set Union, and small-constraint problems fall back to DFS or backtracking. Clone Graph lands on graph traversal without any of those extra branches, which is exactly the kind of "nothing special here, just traverse" recognition that saves you five minutes of panic.

The platform's structure is pattern-first rather than problem-first: a lesson explaining when a pattern applies, a code template to adapt, then a handful of problems that use it with small variations. That's useful for a problem like this one, where the reusable piece is "map original to copy, register before you recurse" rather than the specific twelve lines of Python.

Practical limits worth knowing before you pay:

- The content is text and diagram based. There are no video walkthroughs, which reviewers note repeatedly as the main tradeoff against NeetCode or AlgoExpert.
- The problem count is smaller than LeetCode's. If your weak spot is raw volume and speed rather than pattern recognition, a course doesn't fix that on its own.
- System design and behavioral coverage is introductory. Senior-level loops will need something else for those rounds.
- There is no monthly plan — it's free, annual, or lifetime.

## Current AlgoMonster plans and prices

AlgoMonster's own subscribe page currently lists three options. Prices on this page rotate with promotions, and third-party blogs quote all sorts of figures from older sales, so treat the live page as the source of truth.

| Plan | What you get | Price | Billing | Sign up |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems | $0 | — | [Start free](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant and Speedrun; company question bank updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps | $5.84/mo (billed as $69.99/year) | 12 months, cancel or upgrade anytime | [Get yearly access](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates and all future courses, one payment | $89 | Billed once | [Get lifetime access](https://algo.monster?fpr=li69) |

Everything included in the annual plan also unlocks the additional courses AlgoMonster has added beyond interview patterns, including a CS fundamentals track and a build-an-LLM course. Subscribers get access to new courses as they launch rather than paying per course.

## Is it worth paying for one graph problem?

No. Nobody should buy a subscription to solve LeetCode 133, and the free material plus the platform's public problem pages are enough for that.

The case for the annual or lifetime plan is different: it's whether you want a curated path through roughly the same set of patterns — two pointers, sliding window, binary search, BFS/DFS traversals, dynamic programming — without deciding yourself which of a thousand problems to do next. If you already know these patterns and just need reps, that curated path is overhead. If you're starting from "I've done 20 easy problems and interview season is in six weeks," the structure is the product.

A quick cost comparison, using the prices actually on the site: the yearly plan comes to about $5.84 a month, and the lifetime plan is a one-time $89 that never renews. Both sit well under a single month of most paid prep subscriptions. The real cost to weigh isn't the money — it's the four to eight weeks it takes to work through the material, which is time you could also spend grinding a problem bank with no framework behind it.

If you're going to draw the line somewhere, draw it like this: pay once if you expect to interview more than once in the next few years, because the lifetime tier doesn't expire and previewing content for a single problem isn't the point.

## FAQ

**DFS or BFS for Clone Graph?** Either passes, both are O(V + E). Choose DFS if you can write the four-line map check from memory without error, BFS if you'd rather not think about recursion at all. Interviewers care that you identified the cycle problem and solved it with a map, not which traversal you picked.

**Do I need the hash map if the graph is a tree?** No. Trees have no cycles, so a plain post-order recursive copy works — clone children first, then build the parent from the cloned children. The map exists specifically because graphs have cycles and repeated edges.

**Why does LeetCode complain about my input graph instead of my output?** Almost always because your code modified the input: popping or shifting from a neighbor list, or reusing original nodes inside the returned graph in a way that corrupted shared state. Don't mutate the graph you were given.

**Is this worth knowing outside of interviews?** Deep-copying an object graph is a real problem in plenty of codebases — serialization, undo stacks, state snapshotting. The version in production has the same two requirements: one lookup table for already-copied objects, and a decision about cycles made before the first recursive call.

Clone Graph earns its reputation as a Medium not because the code is long, but because the failure mode is silent until it isn't — infinite recursion on a two-node cycle, or a corrupted input graph that reports an error about a node that plainly exists. Once you've internalized "map first, recurse second," the graph traversal family stops being a list of unrelated problems and becomes one pattern with different questions at the top of it. That's the part worth carrying into the next one.
