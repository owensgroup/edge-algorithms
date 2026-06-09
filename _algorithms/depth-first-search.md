---
title: Depth-First Search (Stamped-Stack)
authors: [Toluwanimi Odemuyiwa]
summary: Sequential depth-first search that carries an explicit stack as a tensor of order stamps, popping the most recently pushed vertex each round, expressed as an iterated extended Einsum over a graph tensor.
tags: [dfs, traversal, stack, graphs]
status_intro: |
  This page gives sequential depth-first search expressed as a single extended
  Einsum applied once per pop. The classical recursion stack is made explicit as a
  tensor $$S$$ whose payload is an *order stamp* $$\sigma(i,v) = i\,\lvert V\rvert + v$$:
  a value that increases with the iteration a vertex was pushed, so that the most
  recently pushed vertex always carries the largest stamp. Each round peeks the top
  of the stack (the maximum stamp), expands its undiscovered neighbours, pops the
  peeked vertex, and pushes the newly stamped neighbours, threading state through a
  generational rank $$I$$ until the stack is empty. The stamp encoding lets a
  last-in-first-out stack ride entirely inside ordinary tensor merges and a single
  select-max populate.
problem_statement: |
  Given a directed graph $$G$$ on $$\lvert V\rvert$$ vertices and a set of root vertices
  $$root\_id$$, visit every reachable vertex in depth-first order: always descend from
  the most recently discovered, still-unfinished vertex before backtracking.

  The state is carried by a stack tensor and a visited mask, alongside the per-round
  scratch tensors:

  - $$G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean}$$, `empty` $$=$$ False,
    the graph. $$G$$ is static, so it carries no generational rank.
  - $$S^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$$, `empty` $$= -1$$, the stack: a
    present entry at vertex $$v$$ holds its push stamp $$\sigma$$, and the entry with the
    largest stamp is the stack top.
  - $$P^{I,\, V \equiv \lvert V\rvert} \to \text{Boolean}$$, `empty` $$=$$ False, the
    "visited" mask.
  - $$F^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$$, `empty` $$= -1$$, the one-hot
    peeked top.
  - $$N^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean}$$, `empty` $$=$$ False, the
    neighbours of the peeked vertex.
  - $$U^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean}$$, `empty` $$=$$ False, the
    *undiscovered* neighbours.
  - $$T^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$$, `empty` $$= -1$$, the stack
    after the peeked vertex is popped.
  - $$S'^{I,\, V \equiv \lvert V\rvert} \to \text{integer}$$, `empty` $$= -1$$, the newly
    stamped undiscovered neighbours about to be pushed.

  The empty value $$-1$$ for the stack sits below the stamp range (stamps start at
  $$\sigma(0,0) = 0$$), so an absent entry is never confused with a real stamp.
working_example_part1: |
  Take the four-vertex directed graph with edges
  $$0\to1,\ 0\to2,\ 1\to3,\ 2\to3$$ and root set $$root\_id = \lbrace 0\rbrace$$, so
  $$\lvert V\rvert = 4$$ and the stamp is $$\sigma(i,v) = 4i + v$$.

  Initialization pushes the root with its stamp $$\sigma(0,0) = 0$$ and marks it
  visited:

  $$S_0 = \lbrace 0:0\rbrace, \qquad P_0 = \lbrace 0:\text{True}\rbrace.$$

  Because $$\sigma$$ adds the vertex id $$v$$ to the iteration term, when several
  neighbours are pushed in the same round the higher-id vertex gets the larger
  stamp, so it is peeked first. Sibling expansion therefore runs in descending
  vertex id.
working_example_part2: |
  Each round peeks the largest stamp $$F$$, expands its undiscovered neighbours,
  removes the peeked vertex to form $$T$$, stamps the survivors as $$S'$$ with
  $$\sigma(i+1,\cdot)$$, and pushes them onto $$T$$ to form the next stack. Stack and
  stamp entries are shown as $$\text{vertex} : \text{stamp}$$:

  | $$i$$ | $$S_i$$ (stack) | peeked $$F_i$$ | $$U_i$$ (new) | $$S'_i$$ (stamped) | $$S_{i+1}$$ | $$P_{i+1}$$ |
  |----|----|----|----|----|----|----|
  | 0 | $$\lbrace 0:0\rbrace$$ | $$0$$ | $$\lbrace 1,2\rbrace$$ | $$\lbrace 1:5,\ 2:6\rbrace$$ | $$\lbrace 1:5,\ 2:6\rbrace$$ | $$\lbrace 0,1,2\rbrace$$ |
  | 1 | $$\lbrace 1:5,\ 2:6\rbrace$$ | $$2$$ | $$\lbrace 3\rbrace$$ | $$\lbrace 3:11\rbrace$$ | $$\lbrace 1:5,\ 3:11\rbrace$$ | $$\lbrace 0,1,2,3\rbrace$$ |
  | 2 | $$\lbrace 1:5,\ 3:11\rbrace$$ | $$3$$ | $$\lbrace \rbrace$$ | $$\lbrace \rbrace$$ | $$\lbrace 1:5\rbrace$$ | $$\lbrace 0,1,2,3\rbrace$$ |
  | 3 | $$\lbrace 1:5\rbrace$$ | $$1$$ | $$\lbrace \rbrace$$ | $$\lbrace \rbrace$$ | $$\lbrace \rbrace$$ | $$\lbrace 0,1,2,3\rbrace$$ |

  The pop (discovery) order is $$0,\ 2,\ 3,\ 1$$. In round $$0$$ the root expands to
  $$1$$ and $$2$$, stamped $$5$$ and $$6$$; the larger stamp $$6$$ wins the peek, so vertex
  $$2$$ is descended before vertex $$1$$ (descending sibling order). Vertex $$2$$ leads to
  $$3$$, which is peeked next; $$3$$'s only neighbour is already visited, so the round
  just pops it. With $$2$$ and $$3$$ exhausted the stack falls back to vertex $$1$$, which
  is popped in round $$3$$; its neighbour $$3$$ is already visited, nothing is pushed,
  and the stack empties. At $$i = 3$$ the new stack $$S_4 = \lbrace \rbrace$$ has
  $$\lVert S_4 \rVert \equiv 0$$, so $$\diamond$$ fires and the iteration halts with every
  vertex visited.
edge_expression_walkthrough: |
  **Peek.** $$F_{i,v^\ast} = S_{i,v} :: \lll_{v^\ast} \mathbf{1}(\text{select-max-val})$$.
  The populate scans the stack and emits a one-hot tensor at the coordinate carrying
  the largest stamp — the stack top — with $$F$$ holding that stamp value. Because
  stamps strictly increase with push iteration, the maximum-stamp vertex is exactly
  the most recently pushed, giving last-in-first-out order without any explicit
  pointer.

  **Advance.** $$N_{i,d} = G_{s,d} \cdot F_{i,s} :: \bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup)$$.
  The take-left intersection $$\leftarrow(\cap)$$ keeps an edge only where both the
  graph entry $$G_{s,d}$$ and the one-hot top $$F_{i,s}$$ are present, so it selects the
  out-edges of the single peeked vertex; $$\bigvee \text{ANY}$$ collapses the source
  rank to a Boolean set of its neighbours $$N$$.

  **Mask.** $$U_{i,d} = N_{i,d} \cdot \neg P_{i,d} :: \bigwedge \leftarrow(\cap)$$.
  The complement $$\neg P$$ is present exactly at the *un*visited vertices; the
  take-left intersection keeps the neighbours that are still undiscovered. These are
  the vertices that will actually be pushed.

  **Pop.** $$T_{i,v} = S_{i,v} \cdot \neg F_{i,v} :: \bigwedge \leftarrow(\cap)$$.
  This removes the peeked vertex from the stack by masked intersection: keep every
  stack entry *except* the one $$F$$ points at. No subtraction is needed — the
  take-left intersection against $$\neg F$$ drops exactly the popped coordinate.

  **Stamp.** $$S'_{i,v} = U_{i,v} \cdot \sigma(i+1,v) :: \bigwedge \rightarrow(\cap)$$.
  Each undiscovered neighbour is assigned its push stamp for the next round,
  $$\sigma(i+1,v) = (i+1)\,\lvert V\rvert + v$$. The take-right intersection writes the
  stamp value at the coordinates present in $$U$$, so $$S'$$ is the set of new entries to
  push, already carrying stamps strictly larger than anything currently on the stack.

  **Push.** $$S_{i+1,v} = T_{i,v} \cdot S'_{i,v} :: \bigwedge \texttt{<<}(\cup)$$.
  The update merge $$\texttt{<<}$$ takes the right value where it is present and falls
  back to the left otherwise, layering the freshly stamped neighbours $$S'$$ on top of
  the popped stack $$T$$. In this algorithm $$T$$ and $$S'$$ have disjoint supports — a
  popped vertex is never re-pushed in the same round — so the union retains both.

  **Visit.** $$P_{i+1,v} = P_{i,v} \cdot S'_{i,v} :: \bigwedge \text{OR}(\cup)$$.
  A vertex is visited at round $$i+1$$ if it was already visited or just stamped for
  pushing, marking the new arrivals discovered.

  **Stop.** $$\diamond : \lVert S_{i+1} \rVert \equiv 0$$. The occupancy
  $$\lVert \cdot \rVert$$ counts present entries, so the condition reads "the stack is
  empty." With `empty` $$= -1$$ for $$S$$, occupancy counts entries that are *present*,
  not entries whose stamp value is zero (the root legitimately has stamp $$0$$).
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &S^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\
  &P^{I,\, V \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &F^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\
  &N^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &U^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &T^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\
  &S'^{I,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=-1\\[4pt]
  &\triangleright \textbf{Stamp (order-stamped value encoding)}\\
  &\sigma(i, v) = i \cdot \lvert V\rvert + v \quad (\text{a function of the iteration point})\\[4pt]
  &\triangleright \textbf{Initialization (root} = \text{vertex } 0)\\
  &S_{0,\ v\,:\,v \in root\_id} = \sigma(0, v)\\
  &P_{0,\ v\,:\,v \in root\_id} = \text{True}\\[4pt]
  &\triangleright \textbf{Extended Einsum (one pop} + \text{expand per iteration } i)\\
  &\triangleright \textbf{(1) PEEK}\\
  &F_{i,v^\ast} = S_{i,v} :: \textstyle\lll_{v^\ast} \mathbf{1}(\text{select-max-val})\\
  &\triangleright \textbf{(2) ADV}\\
  &N_{i,d} = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup)\\
  &\triangleright \textbf{(3) MASK}\\
  &U_{i,d} = N_{i,d} \cdot \neg P_{i,d} :: \textstyle\bigwedge \leftarrow(\cap)\\
  &\triangleright \textbf{(4a) POP}\\
  &T_{i,v} = S_{i,v} \cdot \neg F_{i,v} :: \textstyle\bigwedge \leftarrow(\cap)\\
  &\triangleright \textbf{(4b) STAMP}\\
  &S'_{i,v} = U_{i,v} \cdot \sigma(i+1, v) :: \textstyle\bigwedge \rightarrow(\cap)\\
  &\triangleright \textbf{(4c) PUSH}\\
  &S_{i+1,v} = T_{i,v} \cdot S'_{i,v} :: \textstyle\bigwedge \texttt{<<}(\cup)\\
  &\triangleright \textbf{(5) VISIT}\\
  &P_{i+1,v} = P_{i,v} \cdot S'_{i,v} :: \textstyle\bigwedge \text{OR}(\cup)\\
  &\diamond : \lVert S_{i+1} \rVert \equiv 0
  \end{aligned}
  $$
other_notes: |
  The whole stack lives in one integer tensor and the last-in-first-out discipline
  is carried entirely by the stamp values: pushes in later rounds get strictly larger
  stamps, so the select-max populate always returns the most recently pushed vertex.
  Nothing in the cascade stores an explicit top-of-stack pointer. Because $$\sigma$$
  also adds the vertex id, siblings pushed in a single round are ordered by id, and
  the higher id is peeked first — fixing a deterministic sibling order (descending
  here). The pop step uses masked intersection against $$\neg F$$ rather than a set
  difference, so the stack shrinks without any subtraction operator. Termination is
  guaranteed by the monotonically growing visited mask $$P$$, bounded by
  $$\lvert V\rvert$$: once a vertex is visited it is never stamped again, so it is pushed
  at most once and the stack must eventually empty.
variants: |
  Swapping the peek from $$\text{select-max-val}$$ to $$\text{select-min-val}$$ over the
  same stamp turns the last-in-first-out stack into a first-in-first-out queue,
  recovering breadth-first order from the identical cascade — the stamp encoding makes
  the data structure a one-operator dial. Replacing the Boolean advance with a
  weighted relaxation and a $$\min$$ reduction generalizes the expansion toward
  shortest-path traversals. Seeding $$S_0$$ and $$P_0$$ from a multi-vertex $$root\_id$$
  runs a forest of depth-first trees from several roots at once.
implementation_notes: |
  The peek is a single argmax over the present stack entries; the advance is a sparse
  product of the one-hot top against the graph row; the mask, pop, stamp, visit, and
  push are elementwise merges over the vertex rank. Keeping $$S$$, $$P$$, $$F$$, $$U$$, and
  $$T$$ sparse (present-only) means each round touches work proportional to the stack
  size and the out-degree of the popped vertex, not to $$\lvert V\rvert$$. The stamp is
  computed from the iteration point alone, so it needs no extra state: it is the
  arithmetic value $$i\,\lvert V\rvert + v$$ at each pushed coordinate.
complexity_costs: |
  Each vertex is pushed at most once (the visited mask blocks re-pushing) and each
  edge is examined at most once across the run, giving $$O(\lvert V\rvert + \lvert E\rvert)$$
  total work. The number of iterations is the number of pops, which is the count of
  reachable vertices — one pop per vertex removed from the stack. The stack never
  holds more than $$\lvert V\rvert$$ stamps.
related_notes: |
  Breadth-first search is the same cascade with a first-in-first-out discipline:
  swapping the peek to select the minimum stamp turns this stack into a queue.
  Bellman-Ford generalizes the advance to weighted, re-relaxing traversals.
---
