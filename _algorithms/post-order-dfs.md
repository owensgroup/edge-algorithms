---
title: Post-Order Depth-First Search
authors: [Toluwanimi Odemuyiwa]
summary: Depth-first search that records each vertex when its entire subtree is finished rather than when it is first discovered, expressed as a stamped-stack Einsum that touches every vertex twice and writes a finish time on the way out.
tags: [dfs, traversal, post-order, finish-time, graphs]
status_intro: |
  This page gives post-order depth-first search as an extension of the
  [stamped-stack DFS](depth-first-search.html). Pre-order records a vertex the
  moment it is first discovered; post-order records it the moment it is left for
  good, after every vertex beneath it has finished. The change is *when* a vertex
  is popped and *what* is written: a vertex is popped only once it has no
  undiscovered neighbours left, and at that instant its finish time is stamped
  into a $Post$ tensor. Reading $Post$ in increasing finish-time order yields the
  post-order sequence.

  Post-order DFS is the recurring hard case for an Einsum formulation, and this
  page is deliberately honest about that. The descent, the stamp encoding, and
  the finish-time record are solid and reuse machinery already on the pre-order
  page. The one construct that does heavy lifting — deciding "the top has no
  undiscovered neighbours left" *inside* the notation — is not a coordinate
  predicate and so cannot ride a case statement; it is instead turned into a
  value (count the undiscovered neighbours, then compare the count to zero). That
  reduce-then-compare gate, and the same `select`/`<<` constructs the pre-order
  page already flags, are the parts still under review. See **Other Notes** for
  exactly which lines are solid and which are open.
problem_statement: |
  Given a directed graph $G$ on $\lvert V\rvert$ vertices and a set of root
  vertices $root\_id$, visit every reachable vertex depth-first and report the
  vertices in *finish-time* (post-order) order: a vertex is emitted only after
  every vertex in its subtree has been emitted.

  The state reuses the stamped stack of the pre-order page and adds two tensors —
  a finish-time record and a count of undiscovered neighbours:

  - $G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False,
    the graph. $G$ is static, so it carries no generational rank.
  - $S^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= -1$, the
    stack of *open* vertices: a present entry at vertex $v$ holds its push stamp
    $\sigma$, and the entry with the largest stamp is the stack top.
  - $D^{I,\, V \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the
    "discovered" mask.
  - $F^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= -1$, the
    one-hot peeked top.
  - $N^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the
    neighbours of the peeked vertex.
  - $U^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the
    *undiscovered* neighbours of the top.
  - $C^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the
    single child chosen to descend into this round.
  - $Cs^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= -1$, that
    child carrying its fresh push stamp.
  - $Uc^{I} \to \text{integer}$, `empty` $= 0$, the *count* of undiscovered
    neighbours. Its empty value is $0$ on purpose (see below).
  - $Z^{I} \to \text{Boolean}$, `empty` $=$ False, the "top is finished" flag
    ($Uc \equiv 0$).
  - $Fin^{I,\, V \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False,
    the top, but only when it is finished.
  - $T^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= -1$, the
    stack after a finished top is popped.
  - $Post^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= -1$, the
    finish-time record: vertex $v$ maps to the iteration at which it finished.

  The stack's empty value $-1$ sits below the stamp range (stamps start at
  $\sigma(0,0) = 0$), so an absent stack entry is never confused with a real
  stamp. The count $Uc$ instead uses `empty` $= 0$, so that "no undiscovered
  neighbours" reads as the value $0$ rather than as an absent entry — that is
  what lets the finished test fire (see the walkthrough).
working_example_part1: |
  Take the same four-vertex directed graph used on the pre-order page, with edges
  $0\to1,\ 0\to2,\ 1\to3,\ 2\to3$ and root set $root\_id = \lbrace 0\rbrace$, so
  $\lvert V\rvert = 4$ and the stamp is $\sigma(i,v) = 4i + v$.

  Initialization pushes the root with its stamp $\sigma(0,0) = 0$ and marks it
  discovered:

  $$S_0 = \lbrace 0:0\rbrace, \qquad D_0 = \lbrace 0\rbrace, \qquad Post_0 = \lbrace \rbrace.$$

  The two orders differ visibly on this graph. Walking in *descending* sibling
  order (the higher-id undiscovered neighbour is descended first), the pre-order —
  the order vertices are first discovered — is $0,\ 2,\ 3,\ 1$. The post-order —
  the order vertices finish — is the reverse-shaped sequence $3,\ 2,\ 1,\ 0$: the
  root $0$ is discovered first but finishes *last*, because it is left open on the
  stack until its whole subtree is done. A textbook recursive post-order DFS that
  recurses into neighbours in descending id confirms $3,\ 2,\ 1,\ 0$.
working_example_part2: |
  Each round peeks the top $F$, finds its undiscovered neighbours $U$, and then
  either *descends* (pick one child $C$, stamp it, push it, leave the top on the
  stack) or *finishes* (the top has no undiscovered neighbours: record its finish
  time and pop it). The branch is decided by the count $Uc = \lVert U\rVert$ and
  the test $Uc \equiv 0$. Stack and stamp entries are shown as
  $\text{vertex} : \text{stamp}$:

  | $i$ | $S_i$ (open stack) | top | $U_i$ | child | $Uc_i$ | action | $S_{i+1}$ | $Post$ |
  |----|----|----|----|----|----|----|----|----|
  | 0 | $\lbrace 0:0\rbrace$ | $0$ | $\lbrace 1,2\rbrace$ | $2$ | $2$ | descend $2$ | $\lbrace 0:0,\ 2:6\rbrace$ | $\lbrace\rbrace$ |
  | 1 | $\lbrace 0:0,\ 2:6\rbrace$ | $2$ | $\lbrace 3\rbrace$ | $3$ | $1$ | descend $3$ | $\lbrace 0:0,\ 2:6,\ 3:11\rbrace$ | $\lbrace\rbrace$ |
  | 2 | $\lbrace 0:0,\ 2:6,\ 3:11\rbrace$ | $3$ | $\lbrace\rbrace$ | — | $0$ | finish $3$ | $\lbrace 0:0,\ 2:6\rbrace$ | $\lbrace 3:2\rbrace$ |
  | 3 | $\lbrace 0:0,\ 2:6\rbrace$ | $2$ | $\lbrace\rbrace$ | — | $0$ | finish $2$ | $\lbrace 0:0\rbrace$ | $\lbrace 3:2,\ 2:3\rbrace$ |
  | 4 | $\lbrace 0:0\rbrace$ | $0$ | $\lbrace 1\rbrace$ | $1$ | $1$ | descend $1$ | $\lbrace 0:0,\ 1:21\rbrace$ | $\lbrace 3:2,\ 2:3\rbrace$ |
  | 5 | $\lbrace 0:0,\ 1:21\rbrace$ | $1$ | $\lbrace\rbrace$ | — | $0$ | finish $1$ | $\lbrace 0:0\rbrace$ | $\lbrace 3:2,\ 2:3,\ 1:5\rbrace$ |
  | 6 | $\lbrace 0:0\rbrace$ | $0$ | $\lbrace\rbrace$ | — | $0$ | finish $0$ | $\lbrace\rbrace$ | $\lbrace 3:2,\ 2:3,\ 1:5,\ 0:6\rbrace$ |

  The defining feature of post-order shows up at the top: vertex $0$ is peeked in
  rounds $0$, $4$, and $6$ but is not popped until round $6$, after both of its
  subtrees ($2 \to 3$ first, then $1$) have finished. A vertex stays on the
  stack as an *open* node and floats back to the top — it always carries the
  largest stamp once its children pop off — so the stack itself records "vertices
  still in progress," and no separate already-peeked list is needed.

  Sorting $Post$ by finish time gives finish-time order
  $3\,(@2),\ 2\,(@3),\ 1\,(@5),\ 0\,(@6)$, i.e. the post-order
  $3,\ 2,\ 1,\ 0$. At $i = 6$ the popped top empties the stack,
  $S_7 = \lbrace\rbrace$ with $\lVert S_7\rVert \equiv 0$, so $\diamond$ fires and
  the iteration halts.
edge_expression_walkthrough: |
  **Peek.** $F_{i,v^\ast} = S_{i,v} :: \lll_{v^\ast} \mathbf{1}(\text{max-val-1})$.
  The populate emits a one-hot tensor at the coordinate holding the largest stamp
  — the stack top. Because stamps strictly increase with push iteration, the
  maximum-stamp vertex is the most recently pushed *open* vertex, giving
  last-in-first-out descent without an explicit pointer.

  **Advance.** $N_{i,d} = G_{s,d} \cdot F_{i,s} :: \bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup)$.
  The take-left intersection keeps an edge only where both $G_{s,d}$ and the
  one-hot top $F_{i,s}$ are present, selecting the out-edges of the peeked vertex;
  $\bigvee \text{ANY}$ collapses the source rank to the Boolean set of neighbours.

  **Mask.** $U_{i,d} = N_{i,d} \cdot \neg D_{i,d} :: \bigwedge \leftarrow(\cap)$.
  The complement $\neg D$ is present exactly at the *un*discovered vertices, so
  the intersection keeps the top's undiscovered neighbours.

  **Child.** $C_{i,d^\ast} = U_{i,d} :: \lll_{d^\ast} \mathbf{1}(\text{max-coord-1})$.
  A second populate selects exactly *one* undiscovered neighbour — the largest
  coordinate — to descend into. Picking one neighbour per round (rather than all,
  as breadth-first search would) is what makes the traversal go deep before wide.

  **Discover.** $D_{i+1,v} = D_{i,v} \cdot C_{i,v} :: \bigwedge \text{OR}(\cup)$.
  The chosen child is folded into the discovered mask so it is never descended
  into twice.

  **Stamp.** $Cs_{i,v} = C_{i,v} \cdot \sigma(i+1,v) :: \bigwedge \rightarrow(\cap)$.
  The child is assigned its push stamp $\sigma(i+1,v) = (i+1)\,\lvert V\rvert + v$,
  strictly larger than anything currently on the stack, so it becomes the new top.

  **Count.** $Uc_i = U_{i,d} :: \bigvee +(\cup)$. A plain reduction sums the
  Boolean $U$ over the neighbour rank, giving the *number* of undiscovered
  neighbours of the top. This is the move that sidesteps the core difficulty:
  "does the top still have somewhere to go" is a fact about the *contents* of $U$,
  not about a coordinate, so it cannot be a case predicate; counting turns it into
  a value we can test. Because $Uc$ has `empty` $= 0$, a top with no undiscovered
  neighbours produces an *absent* (hence $0$) count rather than a missing entry.

  **Done.** $Z_i = Uc_i \cdot 0 :: \bigwedge ==(\cup)$. The union merge compares
  the count against the literal $0$; the *union* is essential, because it lets the
  absent count participate as its empty value $0$, so an exhausted top yields
  $0 == 0 = \text{True}$. (An intersection would drop the absent count and the
  test would never fire.)

  **Finish.** $Fin_{i,v} = F_{i,v} \cdot Z_i :: \bigwedge \leftarrow(\cap)$. The
  top $F$ is gated by the finished flag $Z$: $Fin$ is the top exactly when the top
  is done, and empty otherwise.

  **Pop.** $T_{i,v} = S_{i,v} \cdot \neg Fin_{i,v} :: \bigwedge \leftarrow(\cap)$.
  The finished top is removed from the stack by masked intersection against
  $\neg Fin$. When the top is *not* finished, $Fin$ is empty, $\neg Fin$ covers
  everything, and the whole stack survives — the top stays open. No subtraction is
  needed.

  **Push.** $S_{i+1,v} = T_{i,v} \cdot Cs_{i,v} :: \bigwedge \texttt{<<}(\cup)$.
  The update merge layers the freshly stamped child $Cs$ onto the surviving stack
  $T$. On a descend round $Cs$ is the new child and $T$ is the unchanged stack; on
  a finish round $Cs$ is empty and $T$ is the stack minus the popped top. The two
  operands have disjoint supports in either case — a child is never already on the
  stack, and a finished vertex is never re-pushed — so the union retains both.

  **Record.** $Post_{i+1,v} = Post_{i,v} \cdot (Fin_{i,v} \cdot i) :: \bigwedge \texttt{<<}(\cup)$.
  On a finish round $Fin$ carries the finished top, so $Fin_{i,v} \cdot i$ writes
  the current iteration $i$ as that vertex's finish time, layered onto the existing
  record by $\texttt{<<}$. On a descend round $Fin$ is empty and $Post$ is
  unchanged. Each vertex finishes exactly once, so each gets one finish time.

  **Stop.** $\diamond : \lVert S_{i+1} \rVert \equiv 0$. The occupancy
  $\lVert \cdot\rVert$ counts present entries, so the condition reads "the open
  stack is empty." With `empty` $= -1$ for $S$, occupancy counts entries that are
  *present*, not entries whose stamp value is zero (the root legitimately has
  stamp $0$). Reading $Post$ in increasing value order then yields the post-order.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &S^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\
  &D^{I,\, V \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &F^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\
  &N^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &U^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &C^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &Cs^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\
  &Uc^{I} \to \text{integer},\ \text{empty}=0\\
  &Z^{I} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &Fin^{I,\, V \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &T^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\
  &Post^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\[4pt]
  &\triangleright \textbf{Stamp (order-stamped value encoding)}\\
  &\sigma(i, v) = i \cdot \lvert V\rvert + v \quad (\text{a function of the iteration point})\\[4pt]
  &\triangleright \textbf{Initialization (root} = \text{vertex } 0)\\
  &S_{0,\ v\,:\,v \in root\_id} = \sigma(0, v)\\
  &D_{0,\ v\,:\,v \in root\_id} = \text{True}\\[4pt]
  &\triangleright \textbf{Extended Einsum (one peek per iteration } i)\\
  &\triangleright \textbf{(1) PEEK}\\
  &F_{i,v^\ast} = S_{i,v} :: \textstyle\lll_{v^\ast} \mathbf{1}(\text{max-val-1})\\
  &\triangleright \textbf{(2) ADV}\\
  &N_{i,d} = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup)\\
  &\triangleright \textbf{(3) MASK}\\
  &U_{i,d} = N_{i,d} \cdot \neg D_{i,d} :: \textstyle\bigwedge \leftarrow(\cap)\\
  &\triangleright \textbf{(4) CHILD}\\
  &C_{i,d^\ast} = U_{i,d} :: \textstyle\lll_{d^\ast} \mathbf{1}(\text{max-coord-1})\\
  &\triangleright \textbf{(5) DISCOVER}\\
  &D_{i+1,v} = D_{i,v} \cdot C_{i,v} :: \textstyle\bigwedge \text{OR}(\cup)\\
  &\triangleright \textbf{(6) STAMP}\\
  &Cs_{i,v} = C_{i,v} \cdot \sigma(i+1, v) :: \textstyle\bigwedge \rightarrow(\cap)\\
  &\triangleright \textbf{(7) COUNT}\\
  &Uc_{i} = U_{i,d} :: \textstyle\bigvee +(\cup)\\
  &\triangleright \textbf{(8) DONE}\\
  &Z_{i} = Uc_{i} \cdot 0 :: \textstyle\bigwedge ==(\cup)\\
  &\triangleright \textbf{(9) FINISH}\\
  &Fin_{i,v} = F_{i,v} \cdot Z_{i} :: \textstyle\bigwedge \leftarrow(\cap)\\
  &\triangleright \textbf{(10) POP}\\
  &T_{i,v} = S_{i,v} \cdot \neg Fin_{i,v} :: \textstyle\bigwedge \leftarrow(\cap)\\
  &\triangleright \textbf{(11) PUSH}\\
  &S_{i+1,v} = T_{i,v} \cdot Cs_{i,v} :: \textstyle\bigwedge \texttt{<<}(\cup)\\
  &\triangleright \textbf{(12) RECORD}\\
  &Post_{i+1,v} = Post_{i,v} \cdot (Fin_{i,v} \cdot i) :: \textstyle\bigwedge \texttt{<<}(\cup)\\
  &\diamond : \lVert S_{i+1} \rVert \equiv 0
  \end{aligned}
  $$
other_notes: |
  **What is solid.** Lines (1) through (6) are the pre-order descent verbatim,
  with one change: a second populate (4) picks a *single* child and (5) discovers
  only that child, so the stack accumulates open vertices instead of being emptied
  on each pop. The stamp encoding, the peek-by-max-stamp, the masked advance, and
  the next-stamp assignment are all carried over unchanged and inherit the
  pre-order page's reasoning. The hand-trace was reproduced by an executable
  reference and by a textbook recursive post-order DFS; both give $3,2,1,0$.

  **What is the open gap.** The hard part of post-order is the branch "descend if
  the top still has an undiscovered neighbour, otherwise finish and pop." That
  test is a question about the *contents* of $U$ (is it empty), not about a
  coordinate of the iteration space. EDGE case statements admit any Boolean
  predicate, but a predicate ranges over rank-variable expressions (coordinates),
  not over whether a named tensor is empty, and case statements desugar to a
  $\texttt{<<}$ over those coordinate predicates. So this branch cannot be written
  as a case. The cascade instead reifies the test into a value: COUNT (7) reduces
  $U$ to a number, DONE (8) compares it to $0$, FINISH (9) gates the top, and POP
  (10) / RECORD (12) act only on a finished top. This is believed faithful, but it
  leans on three constructs that are *not yet confirmed canonical*:

  - **The `empty` $= 0$ on $Uc$ plus the union compare in DONE (8).** The finished
    branch fires only because an absent count is read as its empty value $0$ and
    the union merge $==(\cup)$ lets that absent value into the comparison. If
    $Uc$ had `empty` $= -1$, an exhausted top would compute $-1 == 0 = \text{False}$
    and never pop — the search would hang. This relies on a precise reading of
    empty-value-into-a-union-compare semantics, which needs the expert's sign-off.
  - **The two populates (1) `max-val-1` and (4) `max-coord-1`.** These are
    user-defined coordinate operators (the same family as the pre-order page's
    $\text{select-max-val}$); their populate-with-UDF semantics is an open item
    across several pages.
  - **The update merge `\texttt{<<}` in PUSH (11) and RECORD (12).** As on the
    pre-order page, $\texttt{<<}$ has the paper semantics "take right where present,
    else left." It is exercised here only where the two operands have disjoint
    support, so it coincides with a plain union — but the page states the canonical
    $\texttt{<<}$ semantics, and the interpreter encoding is decided separately.

  Net: the descent and the finish-time record are clean; the *finish-detection
  gate* (COUNT $\to$ DONE $\to$ FINISH) is the genuinely new and not-yet-vetted
  piece. It is written here as the most defensible formulation, not as a closed
  result.
variants: |
  Recording the iteration $i$ on *discovery* instead of on finish — writing
  $C_{i,v}\cdot i$ into a record at DISCOVER rather than $Fin_{i,v}\cdot i$ at
  RECORD — gives the pre-order (discovery-time) sequence from the same cascade,
  making pre- versus post-order a choice of *which* stamp to keep. Recording both
  a discovery time and a finish time per vertex yields the
  $(\text{discover}, \text{finish})$ interval pairs that classify tree, back,
  forward, and cross edges. Swapping the child selector (4) from $\text{max-coord-1}$
  to $\text{min-coord-1}$ fixes the opposite (ascending) sibling order; seeding
  $S_0$ and $D_0$ from a multi-vertex $root\_id$ runs a forest of depth-first trees
  and produces a post-order across all of them.
implementation_notes: |
  Each round does two argmaxes over present entries (the peek over stamps and the
  child pick over undiscovered neighbours), one sparse advance against the graph
  row, and a handful of elementwise merges over the vertex rank, plus a single
  scalar reduce-and-compare for the finish gate. Keeping $S$, $D$, $F$, $U$, $C$,
  and $T$ sparse means a round touches work proportional to the open-stack size
  and the out-degree of the peeked vertex, not to $\lvert V\rvert$. Unlike the
  pre-order cascade, a vertex is peeked once per still-open child plus once to
  finish, so the iteration count is larger than the pre-order pop count — see the
  cost note.
complexity_costs: |
  A vertex is discovered at most once (the discovered mask blocks re-descent) and
  each edge is examined at most once per peek of its source, giving
  $O(\lvert V\rvert + \lvert E\rvert)$ total work over the run. The number of
  iterations is one peek per descend plus one peek per finish: each of the
  $\lvert V\rvert$ reachable vertices is descended into at most once and finishes
  exactly once, so the run takes up to $2\lvert V\rvert$ iterations — roughly twice
  the pre-order pop count, reflecting that post-order touches every vertex twice
  (once going in, once leaving). The open stack never holds more than
  $\lvert V\rvert$ stamps.
related_notes: |
  [Depth-first search (stamped-stack)](depth-first-search.html) is the pre-order
  companion: the same stamp-and-stack machine, but it pops on discovery and emits
  discovery order, whereas this page keeps a vertex open until its subtree
  finishes and emits finish order. Breadth-first search is the same skeleton with
  the child step replaced by expanding all neighbours at once and a first-in
  first-out (minimum-stamp) discipline.
---
