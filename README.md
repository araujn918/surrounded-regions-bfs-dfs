# Surrounded Regions: Why the Answer Starts at the Border, Plus DFS, BFS and Union-Find Code, Common Bugs, and the Pattern to Spot It Next Time

Most people open LeetCode 130, read "capture all regions surrounded by 'X'", and immediately start hunting for enclosed 'O's in the middle of the board. That instinct is exactly what makes this problem take 40 minutes instead of 10. The regions you want aren't the interesting ones. The interesting ones are the ones you're supposed to ignore.

So before any code, get the definition straight, because everything else follows from it.

## What "surrounded" actually means here

You're given an `m x n` board of `'X'` and `'O'`. A region is any set of `'O'` cells connected by shared edges (up, down, left, right — no diagonals). A region counts as surrounded when **none** of its cells sit on the border of the board. Capture it by flipping every `'O'` in it to `'X'`, in place, with no return value.

Take this board:


X X X X
X O O X
X X O X
X O X X


The three `'O'`s at rows 1–2, columns 1–2 are sealed off, so they become `'X'`. The single `'O'` at row 3, column 1 touches the bottom edge, so it survives. Final board:


X X X X
X X X X
X X X X
X O X X


That last cell is the whole lesson. Any `'O'` with a path to the edge of the board can never be surrounded, which means the border decides the answer for the entire grid. Constraints are up to 200 x 200, so an O(m·n) solution is what you're aiming for.

## Why scanning from the inside falls apart

The naive plan goes like this: walk the interior cells, run a DFS from each unvisited `'O'`, and track whether the walk ever reaches the border. If it doesn't, flip everything you visited to `'X'`. If it does, leave things alone.

It works on paper and it's genuinely annoying to implement. You need a visited set to avoid infinite loops, you need to remember the cells you touched so you can either flip them or roll back, and the two cases interleave badly when one region touches another already-processed one. You end up with bookkeeping instead of an algorithm.

There's a bigger problem too. When you start from an interior `'O'`, you don't know its verdict until the search finishes, so you can't commit to anything early. Every cell becomes a promise you might have to break.

## The inversion: treat the border as the source

Flip the question. Instead of asking "which regions are trapped?", ask "which regions escape?" You don't need to trace paths to the edge one cell at a time — you can just start at the edge and walk inward.

Three steps:

1. Run a DFS (or BFS) from every `'O'` sitting on the four borders. Mark everything you reach with a temporary character, say `'.'`.
2. Sweep the whole board. Any `'O'` still there was never reachable from the border, so it's surrounded: flip it to `'X'`.
3. Sweep again (or fold this into step 2) and turn every `'.'` back into `'O'`.

The temporary marker is what keeps step 2 honest. You're using the board itself as your visited set, which is why this version needs no extra grid.

python
class Solution:
    def solve(self, board: List[List[str]]) -> None:
        if not board or not board[0]:
            return
        rows, cols = len(board), len(board[0])

        def dfs(r, c):
            if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] != 'O':
                return
            board[r][c] = '.'          # safe: reachable from the border
            dfs(r + 1, c)
            dfs(r - 1, c)
            dfs(r, c + 1)
            dfs(r, c - 1)

        for r in range(rows):
            dfs(r, 0)
            dfs(r, cols - 1)
        for c in range(cols):
            dfs(0, c)
            dfs(rows - 1, c)

        for r in range(rows):
            for c in range(cols):
                if board[r][c] == 'O':
                    board[r][c] = 'X'   # trapped
                elif board[r][c] == '.':
                    board[r][c] = 'O'   # restore the escapees


Note the border loops just call `dfs` on every border cell without checking whether it's an `'O'`. The guard at the top of `dfs` already handles that, and skipping the redundant check keeps the code short. Some solutions add early returns for boards with fewer than 3 rows or columns; a 1xN or Nx1 board has no interior at all, so the answer is always "change nothing", and the early return is a clean way to say that.

## BFS, DFS and Union-Find: what actually differs

All three approaches are O(m·n) time. They differ in how they handle the frontier.

| Approach | How it marks safe cells | Worth knowing |
| --- | --- | --- |
| Recursive DFS | Marks and recurses on four neighbours | Shortest code, but recursion depth scales with the region size |
| Iterative BFS | Pushes border `'O'`s into a `deque`, marks neighbours as it pops | No recursion limit, slightly more code |
| Union-Find | Unions every border `'O'` with a dummy node, then checks connectivity | Common in editorials, rarely the fastest thing to write under time pressure |

The BFS version matters more than it looks. A 200x200 board that's entirely `'O'` produces a DFS path far longer than Python's default recursion limit of 1000, and you get a `RecursionError` on a board that's perfectly valid input. Either raise the limit explicitly with `sys.setrecursionlimit(10000)`, or write the traversal with an explicit stack or queue. Interviewers notice when a candidate reaches for an explicit stack without being prompted.

For Union-Find, the trick is a virtual node representing "connected to the outside". Union every border `'O'` with it, union all adjacent `'O'` pairs, then flip any `'O'` whose root isn't the virtual node. It's a legitimate answer, just a lot of machinery for a problem that a 20-line DFS handles.

## Where submissions actually break

- **Forgetting to restore.** Flipping the safe cells straight to `'O'` in a single pass without distinguishing them from trapped cells is the #1 bug. If you mark safe cells with `'X'`, you've destroyed the information you needed.
- **Scanning the interior only.** Looping from `1` to `rows - 1` and `1` to `cols - 1` looks elegant, but your DFS still needs to be able to walk out to the border, and any off-by-one in those bounds silently drops regions.
- **Recursion depth**, as above. Test an all-`'O'` board mentally before you submit.
- **Expecting a return value.** `solve` mutates in place and returns `None`. If you `return board`, some judges accept it and some don't. Follow the signature.
- **Diagonal adjacency.** Cells touching only at a corner are not connected. It's easy to write a direction array from memory and include `(1, 1)` by accident.

## What the interviewer is actually grading

They're checking two things beyond correctness: whether you recognize that the border is the source of truth, and whether you can state complexity without hand-waving.

Time is O(m·n), since each cell is visited a constant number of times across the marking and sweep passes. Space is O(m·n) in the worst case — not for a visited array (there isn't one), but for the recursion stack or queue. Saying "O(m·n) auxiliary space because of the traversal frontier" is the answer that lands. Saying "O(1) because it's in place" is the answer that gets a follow-up question you don't want.

## The pattern underneath it

Strip away the `'X'`/`'O'` dressing and this is flood fill from the boundary. The same skeleton solves Number of Islands, Walls and Gates, Rotting Oranges, and Pacific Atlantic Water Flow. The transferable move is the direction of the search: when a problem asks you to identify something *enclosed*, *interior*, *unreachable from outside*, or *captured in place*, the cheap move is almost always to start at the boundary and work inward rather than guess and verify from the middle.

That inversion is a pattern, not a trick. It shows up in enough problems that recognizing it — "boundary is the source, interior is the derived set" — is worth more than memorizing this particular solution.

If pattern recognition is the part you keep missing, [👉 this is the kind of thing AlgoMonster structures its whole curriculum around](https://algo.monster?fpr=li69). Instead of presenting 130 as an isolated puzzle, the site places grid traversal, flood fill and boundary-sourced searches side by side so the shared skeleton is visible. Lesson content is written by Google engineers and competitive programmers, and if you want to see the multi-language solutions for this exact problem first, [👉 problem #130 is published on AlgoMonster's problem pages](https://algo.monster/liteproblems/130?fpr=li69).

## What you get for free vs. paid on AlgoMonster

The free tier is real but narrow: a subset of the lessons and problems is readable without an account upgrade, which is enough to judge whether the text-and-diagram teaching style works for you and not much more. That's why the drill-down pages for individual problems are the sensible first stop — you can compare their explanation against the one above and see which one makes the border-first idea click.

Pro unlocks all patterns, lessons and problems, unlimited use of the AI assistant, unlimited Speedrun sessions for timed practice, and the editorial library. Recent snapshots of the subscribe page describe it as 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials. Reviews of the platform consistently flag what it isn't: there are no video walkthroughs, and the problem count sits in the low hundreds rather than LeetCode's thousands. It's a framework product, not a volume product.

## Current plans and prices

Prices on AlgoMonster move with promotions, so treat this table as the shape of the offer rather than a fixed quote — snapshots of the official subscribe page taken at different times have shown the yearly plan at both $99 and $119, and the lifetime plan at both $189 and $459 (the higher-priced lifetime listing bundled a 1:1 coaching session with the founder plus an internal referral for qualified candidates).

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons, problems and editorials; enough to sample the teaching style | $0 | — | [ Open the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro – Monthly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; editorial library; company-sourced questions refreshed monthly | $18.99 | Per month, cancel or upgrade anytime | [ Choose the monthly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | Everything in monthly, plus 12 months of full access and a member discount on live bootcamps | $99–$119 (list price shown as $198/year in current promos) | Billed annually, roughly $8.25–$9.92/mo equivalent | [ Pick up Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in yearly, plus lifetime updates to included Pro content | $189 (one-time; a $459 tier with coaching and referral perks has also been listed) | Billed once, yours forever | [ Buy Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

Two things worth pointing out, because a lot of recent review posts get them wrong. First, several 2026 write-ups state flatly that AlgoMonster has no monthly option — the subscribe page currently lists one at $18.99/month, which makes a 6-to-8-week interview sprint cheaper than committing to a year. Second, the yearly price is volatile enough that buying at the wrong moment can cost you $20 to $90 for identical access. Check the page before you assume a number from a blog post, including this one.

Partner sites circulate extra-discount codes that stack an additional 10% on top of the sale price at checkout. Those codes come and go, and they expire without notice, so apply one in the promotion field and confirm the total dropped before paying.

## Which plan makes sense for what

If you're working through a pattern list for an interview in the next couple of months, the monthly plan covers it and costs less than a yearly subscription you'd stop using. If you interview every two or three years — the usual rhythm for engineers chasing promotions or outside offers — the lifetime plan is the one that doesn't need a renewal decision; Lodely's pricing breakdown puts the break-even at roughly two to four years of yearly renewals depending on which promotional price you paid.

What it won't do is replace LeetCode. The problem bank is deliberately small, the platform is text and diagrams only, and there's no mock interviewing or deep system design content. If you already know what to study and just want volume, you're paying for the wrong thing. If your gap is "I can solve problems I've seen and freeze on ones I haven't", the pattern-first structure is aimed exactly at that, and Surrounded Regions is a decent test case: you can verify for yourself whether seeing it as boundary-sourced flood fill is more useful than seeing it as puzzle 130.
