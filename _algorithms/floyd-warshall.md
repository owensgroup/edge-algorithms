---
title: Floyd-Warshall (All-Pairs Shortest Paths)
authors: [Toluwanimi Odemuyiwa]
summary: All-pairs shortest paths as a per-pivot min-plus closure, where iteration $i$ admits paths that route through pivot vertex $i$ and the distance tensor is tightened against the through-pivot candidate, iterated once over every vertex.
tags: [shortest-paths, all-pairs, floyd-warshall, min-plus, graphs]
status_intro: |
  This page gives the classic per-pivot form of Floyd-Warshall. The distance
  tensor holds all-pairs distances at once, and each iteration $i$ opens up one
  new pivot vertex: paths from $u$ to $w$ are now allowed to route through pivot
  $i$. The update keeps the smaller of the current distance and the
  through-pivot candidate $D[u,i] + D[i,w]$. After one pass over all
  $\lvert V\rvert$ pivots, every entry holds a true shortest distance. It assumes
  the graph has no negative-weight cycle.
problem_statement: |
  Given a directed, weighted graph $G$ on $\lvert V\rvert$ vertices, compute for
  every ordered pair $(u,w)$ the weight of the shortest path from $u$ to $w$.
  Edge weights may be negative as long as there is no negative-weight cycle.

  A single distance tensor carries the whole answer, with one scratch tensor for
  the per-pivot candidate:

  - $G^{U \equiv \lvert V\rvert,\, W \equiv \lvert V\rvert} \to \text{integer}$,
    `empty` $= \infty$, the graph (with $0$ on the diagonal).
  - $D^{I,\, U \equiv \lvert V\rvert,\, W \equiv \lvert V\rvert} \to \text{integer}$,
    `empty` $= \infty$, the all-pairs distances. $\infty$ marks "no path found
    yet"; the iteration rank $I$ ranges over the $\lvert V\rvert$ pivots.
  - $T^{I,\, U \equiv \lvert V\rvert,\, W \equiv \lvert V\rvert} \to \text{integer}$,
    `empty` $= \infty$, the through-pivot candidate distance for pivot $i$.

  Both $U$ and $W$ range over the $\lvert V\rvert$ vertices; $D$ is read as a
  source-by-destination matrix, so the same tensor is projected once with the
  pivot in the second slot ($D_{i,u,i}$, the column reaching the pivot) and once
  with the pivot in the first slot ($D_{i,i,w}$, the row leaving the pivot).
working_example_part1: |
  Take the four-vertex directed graph with edges
  $0\to1\,(3),\ 0\to3\,(7),\ 1\to2\,(1),\ 2\to0\,(2),\ 2\to3\,(3)$.
  Initialization copies the graph into $D_0$, with $0$ on the diagonal and
  $\infty$ for every pair with no direct edge:

  $$
  D_0 =
  \begin{bmatrix}
  0 & 3 & \infty & 7\\
  \infty & 0 & 1 & \infty\\
  2 & \infty & 0 & 3\\
  \infty & \infty & \infty & 0
  \end{bmatrix}
  $$

  Row $u$, column $w$ is the best known distance from $u$ to $w$. This graph is
  chosen so that several entries only tighten after a multi-hop route through a
  pivot becomes available: $0\to2$ needs pivot $1$, and $1\to0$ and $1\to3$ need
  pivot $2$.
working_example_part2: |
  Each iteration $i$ opens pivot vertex $i$ and replaces $D[u,w]$ with
  $\min\bigl(D[u,w],\, D[u,i] + D[i,w]\bigr)$. The entries that strictly improve
  are shown per pivot:

  | $i$ (pivot) | entries that tighten | resulting change |
  |----|----|----|
  | $0$ | $2\to1$ via $2\to0\to1$ | $\infty \to 2+3 = 5$ |
  | $1$ | $0\to2$ via $0\to1\to2$ | $\infty \to 3+1 = 4$ |
  | $2$ | $1\to0$ via $1\to2\to0$; $1\to3$ via $1\to2\to3$ | $\infty \to 1+2 = 3$; $\infty \to 1+3 = 4$ |
  | $3$ | none ($3$ has no out-edges) | unchanged |

  After pivot $0$, $D[2,1] = 5$. After pivot $1$, $D[0,2] = 4$ (and the candidate
  $0\to1\to3 = 3+\infty$ is skipped, since $D[1,3]$ is still $\infty$). After
  pivot $2$, $D[1,0] = 3$ and $D[1,3] = 4$; the candidate $0\to2\to3 = 4+3 = 7$
  ties the existing $D[0,3] = 7$, so $\min$ leaves it unchanged. Pivot $3$ relaxes
  nothing. The final distances are

  $$
  D_{\lvert V\rvert} =
  \begin{bmatrix}
  0 & 3 & 4 & 7\\
  3 & 0 & 1 & 4\\
  2 & 5 & 0 & 3\\
  \infty & \infty & \infty & 0
  \end{bmatrix}
  $$

  which matches the true all-pairs shortest distances (vertex $3$ reaches only
  itself, so its row stays $\infty$ off the diagonal).
edge_expression_walkthrough: |
  The iteration rank $I$ is the pivot rank: at iteration $i$ the pivot vertex is
  vertex $i$ itself. There is no separate "select the pivot" step, because the
  pivot coordinate *is* the iteration-space variable $i$ — exactly as the
  sequentialized Bellman-Ford cascade reads the running distance at coordinate
  $k$ via $\tilde{D}_{i,k,k}$. The iteration variable is used directly as a tensor
  coordinate, which is an ordinary rank variable expression (a function of an
  iteration-space variable), not a value-as-coordinate.

  **Through-pivot candidate.**
  $T_{i,u,w} = D_{i,u,i} \cdot D_{i,i,w} :: \bigwedge +(\cap)$. The left operand
  $D_{i,u,i}$ is the column reaching the pivot (best distance $u \to i$); the
  right operand $D_{i,i,w}$ is the row leaving the pivot (best distance
  $i \to w$). Both the pivot slots are pinned to $i$, so this is an outer product
  over the free ranks $u$ and $w$ with $+$ compute. The intersection merge is
  what makes it a *path*: a candidate exists only where both legs exist, so an
  unreached leg (an $\infty$) keeps $\infty$ out of the addition and the pair is
  simply skipped — never $\infty + \text{finite}$.

  **Tighten.**
  $D_{i+1,u,w} = D_{i,u,w} \cdot T_{i,u,w} :: \bigwedge \min(\cup)$. The $\min$
  compute over the union of the current distance and the through-pivot candidate
  keeps the smaller of the two at every pair. The union merge is required because
  either operand may be present where the other is empty: a pair with a direct
  edge but no through-pivot route keeps $D$ ($T$ empty), and a pair first reached
  through this pivot takes $T$ ($D$ empty, i.e. $\infty$, which any finite
  candidate beats). Where both are present, $\min$ chooses.

  **Stop.**
  $\diamond : i \equiv \lvert V\rvert$. The cascade runs exactly once through all
  $\lvert V\rvert$ pivots. Unlike Bellman-Ford or Dijkstra this is a fixed
  trip-count, not a convergence test: opening every vertex as a pivot once is
  what the $(\min,+)$ closure requires, and admitting pivots in any order yields
  the same final distances.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{U \equiv \lvert V\rvert,\, W \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &D^{I,\, U \equiv \lvert V\rvert,\, W \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &T^{I,\, U \equiv \lvert V\rvert,\, W \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\[4pt]
  &\triangleright \textbf{Initialization (copy the graph, } 0 \text{ on the diagonal)}\\
  &D_{0,\ u,\ w} = G_{u,w}\\[4pt]
  &\triangleright \textbf{Extended Einsum (open one pivot } i \text{ per iteration)}\\
  &T_{i,u,w} = D_{i,u,i} \cdot D_{i,i,w} :: \textstyle\bigwedge +(\cap)\\
  &D_{i+1,u,w} = D_{i,u,w} \cdot T_{i,u,w} :: \textstyle\bigwedge \min(\cup)\\
  &\diamond : i \equiv \lvert V\rvert
  \end{aligned}
  $$
other_notes: |
  The two pivot reads $D_{i,u,i}$ and $D_{i,i,w}$ are slices of the *same*
  distance tensor at the current iteration: the pivot column and the pivot row.
  Pinning a rank to the iteration variable $i$ is a rank variable expression
  (the coordinate is the constant function returning $i$), so no value is ever
  promoted to a coordinate — the only thing placed in the pivot slot is the
  loop variable, never a distance read out of $D$.

  Reading the pivot row and column from $D_i$ (the start-of-iteration distances)
  rather than from a distance tensor being mutated within the iteration is what
  makes the per-iteration value well defined under the operational definition:
  iteration $i$ projects into $D_i$, computes, and writes $D_{i+1}$. The
  classic in-place sequential implementation updates $D$ as it sweeps $(u,w)$;
  that is a valid eingebraic refinement because, for a fixed pivot $i$, the pivot
  row $D[i,\cdot]$ and pivot column $D[\cdot,i]$ cannot be improved *by that same
  pivot* (it would require a zero- or negative-weight self-loop at $i$), so the
  in-place reads see the same values.

  The fixed trip count assumes no negative-weight cycle; with one, the diagonal
  $D[i,i]$ would go negative and the distances would no longer be true shortest
  paths. A negative diagonal entry after the run is exactly the negative-cycle
  certificate.
variants: |
  Restricting the algebra to Boolean values with $\text{OR}$ in place of $\min$
  and $\text{AND}$ in place of $+$ turns the same cascade into the transitive
  closure (reachability between all pairs). Keeping $(\min,+)$ but relaxing from a
  single source instead of all pairs — collapsing the $U$ rank to one row and
  iterating to a fixed point rather than a fixed pivot count — recovers
  Bellman-Ford. Blocked Floyd-Warshall tiles the $U$ and $W$ ranks and processes
  the diagonal, cross, and remaining blocks per pivot panel; this is a mapping
  and format choice over the same expression, not a change to the einsum.
implementation_notes: |
  Each iteration is two passes over the $\lvert V\rvert \times \lvert V\rvert$
  distance tensor: an outer product of the pivot column and pivot row (with $+$)
  to form the candidate, then an elementwise $\min$ against the current
  distances. Because the candidate merge is an intersection, only pairs whose
  both legs are finite contribute, so a sparse distance tensor touches work
  proportional to (in-edges of the pivot) $\times$ (out-edges of the pivot). The
  pivot row and column can be broadcast along their orthogonal axis, which is the
  basis for the blocked and distributed variants.
complexity_costs: |
  The cascade runs exactly $\lvert V\rvert$ iterations, each doing
  $O(\lvert V\rvert^2)$ work to update every pair, for $O(\lvert V\rvert^3)$ total
  — independent of edge count, which makes Floyd-Warshall attractive for dense
  graphs and for all-pairs queries where running single-source $\lvert V\rvert$
  times would cost more. Storage is the $O(\lvert V\rvert^2)$ distance tensor.
related_notes: |
  Bellman-Ford is the single-source min-plus relaxation this generalizes to all
  pairs; running it from every source is the alternative all-pairs route, cheaper
  on sparse graphs but costlier on dense ones. The transitive closure is the
  Boolean specialization of this same cascade.
---
