---
title: Dijkstra (Single-Source Shortest Paths)
authors: [Toluwanimi Odemuyiwa]
summary: Single-source shortest paths that settles one vertex per round — always the closest unsettled vertex — and relaxes only its out-edges, expressed as Bellman-Ford's relaxation body wrapped in a priority queue. Requires non-negative edge weights.
tags: [shortest-paths, dijkstra, priority-queue, graphs]
status_intro: |
  This page gives the priority-queue form of Dijkstra's algorithm. It keeps the
  relaxation body of Bellman-Ford unchanged but, instead of relaxing every vertex
  each round, it settles one vertex per iteration — the unsettled vertex with the
  smallest tentative distance — and relaxes only from it. Each vertex is settled
  exactly once. It assumes non-negative edge weights.
problem_statement: |
  Given a directed, weighted graph $G$ on $\lvert V\rvert$ vertices with non-negative
  edge weights and a source set $root\_id$, compute for each vertex the weight of
  the shortest path from a source.

  One distance tensor carries the answer; a Boolean queue tracks which vertices
  remain to be settled, alongside the relaxation scratch tensors:

  - $G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the graph.
  - $D^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the distances
    ($\infty$ marks "not yet reached").
  - $Q^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the queue
    membership — which vertices are still open.
  - $DQ^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the distances
    of the queued vertices.
  - $F^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the single
    vertex settled this round, carrying its (now final) distance.
  - $N, C, NewlyRelaxed$, the relaxation scratch tensors (as in Bellman-Ford).
  - $T^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the queue with
    the settled vertex removed.
working_example_part1: |
  Take the four-vertex directed graph with edges
  $0\to1\,(10),\ 0\to2\,(1),\ 2\to1\,(1),\ 1\to3\,(1)$ and source vertex $0$.
  Initialization sets the source distance to $0$ and places it in the queue; every
  other vertex starts at $\infty$ and out of the queue:

  $$D_0 = \lbrace 0:0\rbrace, \qquad Q_0 = \lbrace 0\rbrace.$$
working_example_part2: |
  Each round settles the closest queued vertex, relaxes only its out-edges, and
  folds any improvements back into the distances and the queue:

  | $i$ | settled (min $g$) | relaxations | $D_{i+1}$ | $Q_{i+1}$ |
  |----|----|----|----|----|
  | 0 | $0$ | $1\!\leftarrow\!10,\ 2\!\leftarrow\!1$ | $\lbrace 0:0,\ 1:10,\ 2:1\rbrace$ | $\lbrace 1,2\rbrace$ |
  | 1 | $2$ | $1\!\leftarrow\!2\ (<10)$ | $\lbrace 0:0,\ 1:2,\ 2:1\rbrace$ | $\lbrace 1\rbrace$ |
  | 2 | $1$ | $3\!\leftarrow\!3$ | $\lbrace 0:0,\ 1:2,\ 2:1,\ 3:3\rbrace$ | $\lbrace 3\rbrace$ |
  | 3 | $3$ | none | unchanged | $\lbrace \rbrace$ — stop |

  The settle order is $0, 2, 1, 3$. Vertex $1$ is reached at $10$ in round $0$ but
  is not settled then — vertex $2$ has the smaller tentative distance, so it is
  settled first and relaxes $1$ down to $2$. Because every weight is non-negative,
  the smallest tentative distance can never be lowered later, so each vertex's
  distance is final the moment it is settled. The queue empties at $i = 3$ and the
  cascade halts at $D = \lbrace 0:0,\ 1:2,\ 2:1,\ 3:3\rbrace$.
edge_expression_walkthrough: |
  **Pick.** $DQ_{i,s} = Q_{i,s} \cdot D_{i,s} :: \bigwedge \rightarrow(\cap)$ keeps
  the distance of every still-queued vertex. The populate
  $F_{i,s^\ast} = DQ_{i,s} :: \lll_{s^\ast} \mathbf{1}(\text{select-min-s})$ then selects
  the queued vertex of minimum distance and emits it at its own coordinate, so $F$
  is a one-hot tensor holding the closest unsettled vertex and its distance.

  **Relax.** $N_{i,d} = G_{s,d} \cdot F_{i,s} :: \bigwedge +(\cap)\ \bigvee \min(\cup)$.
  Exactly as in Bellman-Ford, but the source is the single settled vertex $F$
  rather than all of $D$ — so this round only relaxes the out-edges of the vertex
  just settled.

  **Improve.** $C_{i,d} = N_{i,d} \cdot D_{i,d} :: \bigwedge <(\cup)$. True exactly
  where the new candidate is strictly smaller than the current distance.

  **Record.** $NewlyRelaxed_{i,d} = C_{i,d} \cdot N_{i,d} :: \bigwedge \rightarrow(\cap)$.
  Keeps the candidate value where the improvement mask is present.

  **Update.** $D_{i+1,d} = D_{i,d} \cdot NewlyRelaxed_{i,d} :: \bigwedge \texttt{<<}(\cup)$.
  Improved vertices take their new distance; every other vertex keeps its current one.

  **Dequeue.** $T_{i,d} = Q_{i,d} \cdot F_{i,d} :: \bigwedge \leftarrow(\text{take\_left\_only})$.
  The take-left-only merge keeps a queue member exactly where the settled vertex
  $F$ is *absent*, so the just-settled vertex is removed from the queue. (This is
  the occupancy-level form of $Q \cdot \neg F$, and it does not require negating
  the integer-valued $F$.)

  **Enqueue.** $Q_{i+1,d} = T_{i,d} \cdot C_{i,d} :: \bigwedge \text{OR}(\cup)$.
  The next queue is the surviving members together with every vertex that just
  improved — reusing the improvement mask $C$.

  **Stop.** $\diamond : \lVert Q_{i+1} \rVert \equiv 0$. The cascade halts when the
  queue is empty, i.e. every reachable vertex has been settled.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &D^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &Q^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &DQ^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &F^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &N^{I,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &C^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &NewlyRelaxed^{I,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &T^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\[4pt]
  &\triangleright \textbf{Initialization (source} = \text{vertex } 0)\\
  &D_{0,\ s\,:\,s \in root\_id} = 0\\
  &Q_{0,\ s\,:\,s \in root\_id} = \text{True}\\[4pt]
  &\triangleright \textbf{Extended Einsum (one vertex settled per iteration } i)\\
  &DQ_{i,s} = Q_{i,s} \cdot D_{i,s} :: \textstyle\bigwedge \rightarrow(\cap)\\
  &F_{i,s^\ast} = DQ_{i,s} :: \textstyle\lll_{s^\ast} \mathbf{1}(\text{select-min-s})\\
  &N_{i,d} = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge +(\cap)\ \bigvee \min(\cup)\\
  &C_{i,d} = N_{i,d} \cdot D_{i,d} :: \textstyle\bigwedge <(\cup)\\
  &NewlyRelaxed_{i,d} = C_{i,d} \cdot N_{i,d} :: \textstyle\bigwedge \rightarrow(\cap)\\
  &D_{i+1,d} = D_{i,d} \cdot NewlyRelaxed_{i,d} :: \textstyle\bigwedge \texttt{<<}(\cup)\\
  &T_{i,d} = Q_{i,d} \cdot F_{i,d} :: \textstyle\bigwedge \leftarrow(\text{take\_left\_only})\\
  &Q_{i+1,d} = T_{i,d} \cdot C_{i,d} :: \textstyle\bigwedge \text{OR}(\cup)\\
  &\diamond : \lVert Q_{i+1} \rVert \equiv 0
  \end{aligned}
  $$
other_notes: |
  The queue $Q$ carries only membership (Boolean), while distances live in $D$;
  separating the two roles is what keeps the queue update to a single dequeue plus
  enqueue. A settled vertex never re-enters the queue: under non-negative weights
  its distance is already final, so the improvement mask $C$ is never true for it
  again, and the dequeue has already removed it.

  Like Bellman-Ford, the single distance tensor $D$ is declared with ranks $I, S$
  but projected as both $D_{i,s}$ (a source, in the pick and relax) and $D_{i,d}$
  (a destination, in the improve and update steps); $S$ and $D$ both range over the
  $\lvert V\rvert$ vertices.
variants: |
  Removing the priority queue and relaxing every vertex each round — replacing the
  pick with $N_{i,d} = G_{s,d} \cdot D_{i,s}$ — recovers Bellman-Ford, which also
  admits negative weights at the cost of re-relaxing every vertex. Ordering the
  pick by $g + h$ for an admissible heuristic $h$ instead of by $g$ alone, and
  stopping once a goal is settled, gives A* search.
implementation_notes: |
  The pick is an argmin over the queued distances, naturally a min-priority-queue
  (heap) operation; relax is a sparse product over the single settled vertex's
  out-edges; the remaining steps are elementwise merges over the destination rank.
  Keeping $Q$, $F$, and the distance tensor sparse means each iteration touches
  work proportional to the settled vertex's out-degree.
complexity_costs: |
  Each vertex is settled exactly once and each edge is relaxed once across the whole
  run. With a binary-heap priority queue the pick and the queue updates cost
  $O(\log \lvert V\rvert)$ each, for $O((\lvert V\rvert + \lvert E\rvert)\,\log \lvert V\rvert)$ total work.
related_notes: |
  Bellman-Ford is the no-queue ancestor that relaxes every vertex each round and
  tolerates negative weights. A* adds a goal-directed heuristic to the pick.
---
