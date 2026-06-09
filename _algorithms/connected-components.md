---
title: Connected Components (Label Propagation)
authors: [Toluwanimi Odemuyiwa]
summary: Connected components by max-label propagation, where every vertex repeatedly adopts the largest label among itself and its neighbours until labels stop changing, expressed as a single iterated extended Einsum over a symmetric, reflexive graph tensor.
tags: [connected-components, label-propagation, graphs]
status_intro: |
  This page gives connected components computed by max-label propagation. Each
  vertex starts with a unique label and, every generation, adopts the maximum
  label among itself and its neighbours. Labels only ever increase and are
  bounded, so the labels converge; vertices that end up sharing a label are in
  the same connected component. The whole computation is a single extended
  Einsum threaded through a generational rank $I$, iterated until the label
  vector stops changing.
problem_statement: |
  Given an undirected graph $G$ on $\lvert V\rvert$ vertices, partition the
  vertices into connected components: assign every pair of vertices the same
  component label exactly when a path connects them.

  The state is carried by two tensors:

  - $G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{float}$,
    the adjacency matrix, with `empty` $= 0.0$. $G$ is static, so it carries no
    generational rank.
  - $L^{I,\, S \equiv \lvert V\rvert} \to \text{float}$, the per-vertex label at
    generation $i$, with `empty` $= 0.0$.

  Correctness and termination require $G$ to be **symmetric**, **reflexive** (a
  self-loop on every vertex), and **$0/1$-weighted**, and the initial labels
  $L_0$ to be **unique per vertex** with every vertex seeded (there is no source
  set). The standard seeding is $\text{label}(v) = v + 1$, which keeps every
  label strictly above the `empty` value $0.0$.
working_example_part1: |
  Take five vertices in two components: a path $0 - 1 - 2$ and a single edge
  $3 - 4$. Adding the required self-loops, the symmetric, reflexive, $0/1$
  adjacency matrix is

  $$
  G =
  \begin{bmatrix}
  1 & 1 & 0 & 0 & 0\\
  1 & 1 & 1 & 0 & 0\\
  0 & 1 & 1 & 0 & 0\\
  0 & 0 & 0 & 1 & 1\\
  0 & 0 & 0 & 1 & 1
  \end{bmatrix}.
  $$

  Every vertex is seeded with its own unique label $\text{label}(v) = v + 1$:

  $$L_0 = \lbrace 0:1,\ 1:2,\ 2:3,\ 3:4,\ 4:5\rbrace.$$
working_example_part2: |
  Each generation replaces every vertex's label with the maximum label among
  itself and its neighbours,
  $L_{i+1,d} = \max_{s}\,\lbrace G_{s,d} \times L_{i,s}\rbrace$. The reflexive
  self-loop puts each vertex's own current label into that maximum, so labels
  never decrease.

  | $i$ | $L_i$ | per-vertex update |
  |----|----|----|
  | 0 | $\lbrace 1,\ 2,\ 3,\ 4,\ 5\rbrace$ | initial labels |
  | 1 | $\lbrace 2,\ 3,\ 3,\ 5,\ 5\rbrace$ | $0\!:\!\max(1,2),\ 1\!:\!\max(1,2,3),\ 2\!:\!\max(2,3),\ 3,4\!:\!\max(4,5)$ |
  | 2 | $\lbrace 3,\ 3,\ 3,\ 5,\ 5\rbrace$ | $0\!:\!\max(2,3),\ 1\!:\!\max(2,3,3),\ 2\!:\!\max(3,3)$ |
  | 3 | $\lbrace 3,\ 3,\ 3,\ 5,\ 5\rbrace$ | unchanged $\equiv L_2$ — stop |

  At $i = 3$ the labels equal those at $i = 2$, so $L_3 \equiv L_2$ fires and
  the iteration halts. The final labels are
  $L = \lbrace 0:3,\ 1:3,\ 2:3,\ 3:5,\ 4:5\rbrace$: component
  $\lbrace 0,1,2\rbrace$ settles on the maximum of its labels, $3$, and component
  $\lbrace 3,4\rbrace$ settles on $5$. Vertices sharing a final label are exactly
  the vertices in the same connected component.
edge_expression_walkthrough: |
  **Propagate.**
  $L_{i+1,d} = G_{s,d} \times L_{i,s} :: \bigwedge_s \times(\cap)\ \bigvee_s \max(\cup)$.
  The map $\times(\cap)$ touches a coordinate only where both the edge $G_{s,d}$
  and the source label $L_{i,s}$ are present, multiplying them; with $0/1$ edge
  weights this passes the neighbour's label through unchanged. Empty is not
  zero: the intersection simply skips a point where either operand is absent, it
  does not contribute a $0$. The reduction $\bigvee_s \max(\cup)$ then keeps the
  largest incoming label at each destination $d$.

  Because the gather is over $G_{s,d}$ (edges *into* $d$), a label flows
  $s \to d$ only when $(s,d) \in G$. Symmetry makes labels propagate both ways
  across each undirected edge. The reflexive self-loop $(d,d)$ is what puts
  vertex $d$'s own current label $L_{i,d}$ into the set of values gathered at
  $d$ — the reduction state is seeded at the $\max$ *identity*, not at $L_{i,d}$,
  so it is the self-loop, not the $\max$, that guarantees
  $L_{i+1,d} \ge L_{i,d}$.

  **Stop.** $\diamond : L_{i+1} \equiv L_i$. The stop is value convergence, not
  occupancy: $L$ never empties, it converges in value. The whole-tensor
  $\equiv$ holds when every coordinate of $L_{i+1}$ equals the corresponding
  coordinate of $L_i$. With reflexivity, labels are monotonically
  non-decreasing and bounded above by the largest seed, so this condition is
  always reached.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{float},\ \text{empty}=0.0\\
  &L^{I,\, S \equiv \lvert V\rvert} \to \text{float},\ \text{empty}=0.0\\[4pt]
  &\triangleright \textbf{Initialization (host-supplied data, no source set)}\\
  &G \to \langle \text{symmetric, reflexive, } 0/1 \text{ adjacency}\rangle\\
  &L_0 \to \langle \text{unique per-vertex labels, } \text{label}(v) = v + 1\rangle\\[4pt]
  &\triangleright \textbf{Extended Einsum}\\
  &L_{i+1,d} = G_{s,d} \times L_{i,s} :: \textstyle\bigwedge_s \times(\cap)\ \bigvee_s \max(\cup)\\
  &\diamond : L_{i+1} \equiv L_i
  \end{aligned}
  $$
other_notes: |
  All three graph preconditions are load-bearing. **Reflexivity** is required
  for *termination*, not just correctness: without a self-loop, a vertex never
  gathers its own current label, and the iteration can oscillate forever — on
  the path $0 - 1 - 2$ with $L_0 = \lbrace 1,2,3\rbrace$ it cycles
  $\lbrace 2,3,2\rbrace \to \lbrace 3,2,3\rbrace \to \cdots$ and never settles.
  **Symmetry** is required because the gather is over in-edges; on a directed
  graph this maxes over in-edges only and does not give weakly connected
  components. **$0/1$ weights** are required because the map is $\times$: a
  present weight must be exactly $1$ so it passes the label through unchanged; a
  weight $\ne 1$ scales the propagated label into a meaningless component id, and
  a weight $> 1$ makes labels grow unboundedly and breaks termination. Labels
  must also stay strictly above the `empty` value $0.0$, which the
  $\text{label}(v) = v + 1$ convention guarantees; a label of $0$ would be
  indistinguishable from absent and silently swallowed. An isolated vertex keeps
  its own label through its self-loop and is correctly its own singleton
  component.
variants: |
  Replacing the $\max$ reduction with $\min$ propagates the smallest label
  instead, giving the same components labelled by their minimum seed. Structurally
  this Einsum is the breadth-first-search advance,
  $T_{i,d} = G_{s,d} \cdot F_{i,s} :: \bigwedge_s +(\cap)\ \bigvee_s \text{ANY}(\cup)$,
  with the map $+$ replaced by $\times$ and the reduce $\text{ANY}$ replaced by
  $\max$. Swapping the map $\times(\cap)$ for take-right $\rightarrow(\cap)$ would
  pass the source label through regardless of edge weight, making "edge weights
  do not matter" literally true and removing the scaling and termination hazard
  tied to $\times$.
implementation_notes: |
  Each generation is a sparse matrix-by-label product followed by a max
  reduction over the source rank — the same kernel shape as a BFS advance, with
  multiply-and-max in place of add-and-any. Keeping $L$ sparse (present-only)
  means each generation touches work proportional to the active labels and the
  edges incident to them. Termination is detected by comparing $L_{i+1}$ to
  $L_i$ for whole-tensor equality each round.
complexity_costs: |
  Each generation costs $O(\lvert V\rvert + \lvert E\rvert)$ for the propagate
  step. The number of generations needed for a label to traverse a component is
  bounded by that component's diameter, so the total work is
  $O((\lvert V\rvert + \lvert E\rvert)\cdot \text{diam})$ in the worst case.
related_notes: |
  Breadth-first search is the additive sibling of this Einsum: same advance
  shape, with add-and-any propagating hop distances instead of multiply-and-max
  propagating labels. Like Bellman-Ford, the stop is value convergence
  ($L_{i+1} \equiv L_i$) rather than the occupancy stop used by frontier-emptying
  traversals.
---
