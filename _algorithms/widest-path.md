---
title: Widest Path (Maximum Bottleneck Path)
authors: [Toluwanimi Odemuyiwa]
summary: Single-source maximum-bottleneck paths that widen a wavefront of bottleneck capacities each round and keep the best capacity seen so far, expressed as the max-min mirror of min-plus Bellman-Ford iterated to a fixed point.
tags: [shortest-paths, widest-path, bottleneck, max-min, graphs]
status_intro: |
  This page gives the widest-path (maximum-bottleneck) algorithm as the max-min
  mirror of Bellman-Ford. Where Bellman-Ford extends a path by *adding* an edge
  weight and picks the *minimum* route, widest path extends a path by taking the
  *narrowest* edge along it (a $\min$) and picks the *maximum* such bottleneck (a
  $\max$). Every round widens a wavefront $F$ of reachable bottleneck capacities;
  a running tensor $D$ keeps the widest bottleneck seen so far at each vertex. The
  cascade iterates until $D$ stops changing.
problem_statement: |
  Given a directed graph $G$ on $\lvert V\rvert$ vertices whose edges carry
  *capacities*, and a source set $root\_id$, compute for each vertex the widest
  bottleneck reachable from a source: the path that maximizes the minimum edge
  capacity along it.

  A wavefront tensor and a running-best tensor carry the state:

  - $G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{float}$, `empty` $= 0$, the
    capacity graph. A missing edge is `empty` $= 0$, the identity that drops out of
    the $\max$.
  - $F^{I,\, S \equiv \lvert V\rvert} \to \text{float}$, `empty` $= 0$, the wavefront of
    bottleneck capacities reaching each vertex this round.
  - $D^{I,\, S \equiv \lvert V\rvert} \to \text{float}$, `empty` $= 0$, the widest bottleneck
    known so far, which only ever grows.

  The source carries an unbounded incoming bottleneck. In the max-min semiring the
  identity for "unreached" is $0$ (anything beats nothing), and the source must be
  the opposite extreme. It is seeded with a capacity larger than any real edge so
  that $\min(\text{source}, G_{s,d}) = G_{s,d}$ never clamps a true bottleneck; this
  is the max-min analogue of seeding the source distance to $0$ in min-plus.
working_example_part1: |
  Take the four-vertex directed capacity graph with edges
  $0\to1\,(3),\ 0\to2\,(5),\ 1\to3\,(3),\ 2\to3\,(1)$ and source vertex $0$. As a
  matrix (row $s$, column $d$, $0$ marks no edge):

  $$
  G =
  \begin{array}{c|cccc}
   & d{=}0 & d{=}1 & d{=}2 & d{=}3 \\ \hline
  s{=}0 & 0 & 3 & 5 & 0 \\
  s{=}1 & 0 & 0 & 0 & 3 \\
  s{=}2 & 0 & 0 & 0 & 1 \\
  s{=}3 & 0 & 0 & 0 & 0
  \end{array}
  $$

  Initialization seeds the source with an unbounded bottleneck (a finite stand-in
  for $+\infty$, larger than any edge) and leaves every other vertex at $0$:

  $$F_0 = D_0 = \lbrace 0:\infty\rbrace.$$

  This graph is chosen because the two routes to vertex $3$ disagree: $0\to1\to3$
  has bottleneck $\min(3,3) = 3$, while $0\to2\to3$ starts on the wider edge $5$ but
  is throttled to $\min(5,1) = 1$. The widest path therefore takes the *narrower*
  first edge, which a single greedy step would miss.
working_example_part2: |
  Each round widens the wavefront $F_{i+1,d} = \max_s \min(G_{s,d},\, F_{i,s})$, then
  folds it into the running best $D_{i+1,d} = \max(D_{i,d},\, F_{i+1,d})$:

  | $i$ | $F_{i+1}$ (wavefront) | $D_{i+1}$ |
  |----|----|----|
  | 0 | $\lbrace 1:3,\ 2:5\rbrace$ | $\lbrace 0:\infty,\ 1:3,\ 2:5\rbrace$ |
  | 1 | $\lbrace 3:3\rbrace$ | $\lbrace 0:\infty,\ 1:3,\ 2:5,\ 3:3\rbrace$ |
  | 2 | $\lbrace \rbrace$ | unchanged — stop |

  In round $0$ the source's unbounded bottleneck crosses its two out-edges, giving
  $F_1 = \lbrace 1:\min(\infty,3){=}3,\ 2:\min(\infty,5){=}5\rbrace$. In round $1$ the
  wavefront reaches vertex $3$ along both routes: $\min(G_{1,3},F_1[1]) = \min(3,3) = 3$
  and $\min(G_{2,3},F_1[2]) = \min(1,5) = 1$, and the $\max$ keeps $3$. Round $2$
  pushes from vertex $3$, which has no out-edges, so $F_3$ is empty and $D$ does not
  change; the cascade halts at
  $D = \lbrace 0:\infty,\ 1:3,\ 2:5,\ 3:3\rbrace$. Vertex $3$'s widest bottleneck is
  $3$, confirming $0\to1\to3$ beats $0\to2\to3$.
edge_expression_walkthrough: |
  **Widen.** $F_{i+1,d} = G_{s,d} \cdot F_{i,s} :: \bigwedge_s \min(\cap)\ \bigvee_s \max(\cup)$.
  For every edge $s\to d$ whose source carries a bottleneck this round, form
  $\min(F_{i,s},\, G_{s,d})$: the narrower of the bottleneck already reached at $s$
  and the capacity of the link $s\to d$. The intersection merge keeps the $0$ of
  unreached sources and of missing edges out of the computation, so only real
  one-step extensions of the wavefront contribute. The $\max$ reduction over $s$
  then picks the widest incoming route for each $d$. So
  $F_{i+1,d} = \max_s \min(G_{s,d},\, F_{i,s})$ — the max-min mirror of
  Bellman-Ford's $\min_s\,(D_{i,s} + G_{s,d})$.

  **Keep.** $D_{i+1,d} = D_{i,d} \cdot F_{i+1,d} :: \bigwedge_d \max(\cup)\ \bigvee_d \max(\cup)$.
  Take the larger of the old best-known bottleneck and the new wavefront value, so
  $D$ grows monotonically and never loses a wider route once found. The union merge
  carries a vertex's value forward even when only one operand is present (a first
  arrival beats the $0$ identity). There is no contraction rank here — both sides
  are free in $d$ — so the $\bigvee_d \max(\cup)$ reduction is vacuous and the work
  is the elementwise $\bigwedge_d \max(\cup)$ merge; it is written in full only to
  mirror the two-clause shape of the Widen step.

  **Stop.** $\diamond : D_{i+1} \equiv D_i$. The cascade halts when a full round of
  widening changes no bottleneck — the maximum-bottleneck fixed point.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{float},\ \text{empty}=0\\
  &F^{I,\, S \equiv \lvert V\rvert} \to \text{float},\ \text{empty}=0\\
  &D^{I,\, S \equiv \lvert V\rvert} \to \text{float},\ \text{empty}=0\\[4pt]
  &\triangleright \textbf{Initialization (source} = \text{vertex } 0\text{, unbounded bottleneck)}\\
  &F_{0,\ s\,:\,s \in root\_id} = \infty\\
  &D_{0,\ s\,:\,s \in root\_id} = \infty\\[4pt]
  &\triangleright \textbf{Extended Einsum (one widen round per iteration } i)\\
  &F_{i+1,d} = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge_s \min(\cap)\ \bigvee_s \max(\cup)\\
  &D_{i+1,d} = D_{i,d} \cdot F_{i+1,d} :: \textstyle\bigwedge_d \max(\cup)\ \bigvee_d \max(\cup)\\
  &\diamond : D_{i+1} \equiv D_i
  \end{aligned}
  $$
other_notes: |
  The source seed is the crux of the max-min initialization. In min-plus
  Bellman-Ford the source distance starts at $0$ and everything unreached at
  $\infty$; here the roles flip, so unreached vertices take the $\min$-friendly
  identity $0$ and the source takes an unbounded capacity. A finite stand-in (any
  value larger than every edge capacity, e.g. $999$ in the source program) works
  because $\min(\infty,\, G_{s,d}) = G_{s,d}$: the source's own seed never becomes a
  bottleneck, so the first real edge out of the source sets the wavefront exactly.

  $G$ uses `empty` $= 0$ rather than $+\infty$. In the $\min$ map a $0$ capacity
  would falsely clamp a route to $0$, but the intersection merge $(\cap)$ skips
  points where either operand is `empty`, so missing edges never enter the
  arithmetic at all. The stop condition $D_{i+1} \equiv D_i$ detects convergence:
  because $D$ is monotonically non-decreasing and bottlenecks are bounded above by
  the widest reachable edge, the cascade always terminates.
variants: |
  Flipping the semiring back recovers shortest paths: replace the Widen step's
  $\bigwedge \min(\cap)$ with $\bigwedge +(\cap)$ and its $\bigvee \max(\cup)$ with
  $\bigvee \min(\cup)$, swap the source seed from $+\infty$ to $0$ and the `empty`
  identity from $0$ to $+\infty$, and the Keep step's $\max$ to a $\min$, and the
  cascade is exactly min-plus Bellman-Ford. Replacing $\min$ with a product and
  $\max$ with a sum turns the wavefront into a *most-reliable path* (maximum
  product of edge survival probabilities). Adding a priority queue that settles the
  vertex of widest current bottleneck each round, rather than relaxing all edges,
  gives the Dijkstra-style widest-path variant.
implementation_notes: |
  Widen is a sparse matrix-by-vector product over the wavefront in the max-min
  semiring: a $\min$ in place of multiply and a $\max$ in place of add. Keep is a
  single elementwise $\max$ merge over the destination rank. Because $D$ only grows,
  no separate improvement mask is needed — the $\max$ in Keep both detects and
  applies an improvement in one step, which is why this cascade is one line shorter
  than Bellman-Ford's relax / improve / record / update.
complexity_costs: |
  Each round widens across every edge once at $O(\lvert E\rvert)$, and the wavefront
  reaches one more hop per round, so convergence takes at most $\lvert V\rvert - 1$
  rounds, for $O(\lvert V\rvert \cdot \lvert E\rvert)$ total work — the same bound as
  Bellman-Ford. A Dijkstra-style variant that settles the widest vertex each round
  reduces this to $O((\lvert V\rvert + \lvert E\rvert)\,\log \lvert V\rvert)$.
related_notes: |
  This is the max-min mirror of min-plus Bellman-Ford: the same
  iterate-until-stable skeleton with the two operators flipped. Dijkstra is the
  priority-queue refinement, and the widest-path Dijkstra variant settles the
  vertex of widest bottleneck instead of shortest distance.
---
