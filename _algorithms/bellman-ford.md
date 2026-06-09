---
title: Bellman-Ford (Single-Source Shortest Paths)
authors: [Sanjana Mali, Toluwanimi Odemuyiwa]
summary: Single-source shortest paths that relaxes every edge each round and re-relaxes vertices whenever a strictly shorter path appears, expressed as a four-step extended Einsum iterated to a fixed point.
tags: [shortest-paths, bellman-ford, relaxation, graphs]
status_intro: |
  This page gives the canonical, re-relaxing form of Bellman-Ford. Every round
  relaxes all edges from all currently-reached vertices; a vertex's distance is
  improved again whenever a shorter path is discovered. The cascade iterates until
  no distance changes. It assumes the graph has no negative-weight cycle.
problem_statement: |
  Given a directed, weighted graph $G$ on $\lvert V\rvert$ vertices and a source set
  $root\_id$, compute for each vertex the weight of the shortest path from a
  source. Edge weights may be negative as long as there is no negative cycle.

  A single distance tensor carries the state, alongside per-round scratch tensors:

  - $G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= 0$, the graph.
  - $D^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the distances.
    $\infty$ marks "not yet reached."
  - $N^{I,\, D \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the relaxed
    candidate distances for this round.
  - $C^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the improvement
    mask.
  - $NewlyRelaxed^{I,\, D \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the
    candidates that actually improved a distance.
working_example_part1: |
  Take the four-vertex directed graph with edges
  $0\to1\,(10),\ 0\to2\,(1),\ 2\to1\,(1),\ 1\to3\,(1)$ and source
  vertex $0$. Initialization sets the source distance to $0$; every other vertex
  starts at $\infty$:

  $$D_0 = \lbrace 0:0\rbrace.$$

  This graph is chosen because it separates true Bellman-Ford from a single-visit
  weighted traversal: vertex $1$ is first reached at distance $10$ (via
  $0\to1$) but must later be re-relaxed to $2$ (via $0\to2\to1$).
working_example_part2: |
  Each round relaxes all edges, keeps only strict improvements, and folds them into
  the distances:

  | $i$ | $N_i$ (candidates) | $NewlyRelaxed_i$ | $D_{i+1}$ |
  |----|----|----|----|
  | 0 | $\lbrace 1:10,\ 2:1\rbrace$ | $\lbrace 1:10,\ 2:1\rbrace$ | $\lbrace 0:0,\ 1:10,\ 2:1\rbrace$ |
  | 1 | $\lbrace 1:2,\ 2:1,\ 3:11\rbrace$ | $\lbrace 1:2,\ 3:11\rbrace$ | $\lbrace 0:0,\ 1:2,\ 2:1,\ 3:11\rbrace$ |
  | 2 | $\lbrace 1:2,\ 2:1,\ 3:3\rbrace$ | $\lbrace 3:3\rbrace$ | $\lbrace 0:0,\ 1:2,\ 2:1,\ 3:3\rbrace$ |
  | 3 | $\lbrace 1:2,\ 2:1,\ 3:3\rbrace$ | $\lbrace \rbrace$ | unchanged — stop |

  Vertex $1$ is re-relaxed from $10$ down to $2$ in round $1$, which then pulls
  vertex $3$ from $11$ down to $3$ in round $2$. Round $3$ improves nothing, so
  $D_4 \equiv D_3$ and the cascade halts at $D = \lbrace 0:0,\ 1:2,\ 2:1,\ 3:3\rbrace$.
edge_expression_walkthrough: |
  **Relax.** $N_{i,d} = G_{s,d} \cdot D_{i,s} :: \bigwedge_s +(\cap)\ \bigvee_s \min(\cup)$.
  For every edge $s\tod$ whose source has a finite distance, form
  $D_{i,s} + G_{s,d}$; the intersection merge keeps the $\infty$ of unreached
  sources out of the arithmetic. The $\min$ reduction over $s$ takes the best
  incoming candidate for each $d$. Crucially there is **no visited gate** — a
  vertex with a finite distance is relaxed again every round, which is what makes
  this true Bellman-Ford rather than a single-visit traversal.

  **Improve.** $C_{i,d} = N_{i,d} \cdot D_{i,d} :: \bigwedge_d <(\cup)$. The
  comparison yields True exactly where the new candidate is strictly smaller than
  the current distance. A False result lands in $C$'s empty space and drops out, so
  $C$ is present only at genuinely improved vertices (a first arrival counts,
  since any finite value beats $\infty$).

  **Record.** $NewlyRelaxed_{i,d} = C_{i,d} \cdot N_{i,d} :: \bigwedge_d \rightarrow(\cap)$.
  The take-right intersection keeps the candidate value $N_{i,d}$ exactly where the
  improvement mask $C$ is present.

  **Update.** $D_{i+1,d} = D_{i,d} \cdot NewlyRelaxed_{i,d} :: \bigwedge_d \texttt{<<}(\cup)$.
  The $\texttt{<<}$ merge takes the right operand where it is present and otherwise
  carries the left operand forward: improved vertices take their new distance, and
  every other vertex keeps its current one.

  **Stop.** $\diamond : D_{i+1} \equiv D_i$. The cascade halts when a full round of
  relaxation changes no distance — the shortest-path fixed point.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &C^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &N^{I,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &NewlyRelaxed^{I,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &D^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\[4pt]
  &\triangleright \textbf{Initialization (source} = \text{vertex } 0)\\
  &D_{0,\ s\,:\,s \in root\_id} = 0\\[4pt]
  &\triangleright \textbf{Extended Einsum (one relaxation round per iteration } i)\\
  &N_{i,d} = G_{s,d} \cdot D_{i,s} :: \textstyle\bigwedge_s +(\cap)\ \bigvee_s \min(\cup)\\
  &C_{i,d} = N_{i,d} \cdot D_{i,d} :: \textstyle\bigwedge_d <(\cup)\\
  &NewlyRelaxed_{i,d} = C_{i,d} \cdot N_{i,d} :: \textstyle\bigwedge_d \rightarrow(\cap)\\
  &D_{i+1,d} = D_{i,d} \cdot NewlyRelaxed_{i,d} :: \textstyle\bigwedge_d \texttt{<<}(\cup)\\
  &\diamond : D_{i+1} \equiv D_i
  \end{aligned}
  $$
other_notes: |
  $D$ is declared with ranks $I, S$ but is projected as both $D_{i,s}$ (as a source
  in Relax) and $D_{i,d}$ (as a destination in Improve and Update). It is a single
  tensor; $S$ and $D$ both range over the $\lvert V\rvert$ vertices, so the loop variable used
  at each projection is just the role that vertex plays in that step.

  The stop condition $D_{i+1} \equiv D_i$ detects convergence, not negative cycles:
  on a graph with a negative-weight cycle the distances decrease forever and the
  cascade never halts. This page assumes no negative cycle.
variants: |
  Dropping negative weights and replacing $\min$ with $\text{ANY}$ recovers the
  unweighted breadth-first search. Bounding the iteration count at $\lvert V\rvert$ and
  checking for an improvement on the final round gives negative-cycle detection.
implementation_notes: |
  Relax is a sparse matrix-by-vector product over the distance tensor; the remaining
  three steps are elementwise merges over the destination rank. Because $C$ gates
  Record, $NewlyRelaxed$ is present only at vertices that strictly improved, so each
  round writes back only the distances that actually changed.
complexity_costs: |
  Each round relaxes every edge once at $O(\lvert E\rvert)$, and convergence on a graph without
  negative cycles takes at most $\lvert V\rvert - 1$ rounds, for $O(\lvert V\rvert \cdot \lvert E\rvert)$ total work.
related_notes: |
  Breadth-first search is the unweighted, single-visit specialization of this
  cascade.
---
