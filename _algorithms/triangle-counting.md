---
title: Triangle Counting
authors: [Toluwanimi Odemuyiwa]
summary: Counts the triangles in an undirected graph as a single masked Einsum that multiplies two adjacency hops and keeps only the products that land on an existing edge, then reduces the masked counts to a total.
tags: [triangle-counting, masked-spgemm, graphs]
status_intro: |
  This page gives triangle counting as a masked sparse matrix product. For an
  undirected graph with a symmetric, Boolean adjacency $$A$$, the number of
  triangles through each edge is the count of two-hop paths between its
  endpoints that *also* share an edge: $$T = (A \cdot A) \circ A$$. The two-hop
  count $$A \cdot A$$ is a contraction over the middle vertex; the Hadamard mask
  $$\circ A$$ keeps only the endpoint pairs that are themselves adjacent. The whole
  thing is one non-iterated extended Einsum with three adjacency operands, so
  there is no generational rank and no stop condition; a final reduction collapses
  the per-edge counts to the totals, with the over-counting divided out as
  post-processing.
problem_statement: |
  Given an undirected graph $$G$$ on $$\lvert V\rvert$$ vertices, presented as a
  symmetric Boolean adjacency tensor, count the triangles: the number of
  unordered vertex triples $$\lbrace i,j,k\rbrace$$ that are pairwise adjacent.
  We compute three related quantities — the per-edge triangle count, the
  per-vertex triangle count, and the global total.

  The state is carried by three tensors:

  - $$A^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean}$$,
    `empty` $$=$$ False. The adjacency. $$A$$ is symmetric ($$A_{s,d} = A_{d,s}$$),
    has no self-loops ($$A_{v,v}$$ is `empty`), and is static, so it carries no
    generational rank.
  - $$T^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer}$$,
    `empty` $$= 0$$. The per-edge triangle count: $$T_{i,j}$$ is the number of
    triangles that use edge $$(i,j)$$.
  - $$tris \to \text{integer}$$, `empty` $$= 0$$, a scalar (rank-0) holding the
    global triangle count.

  The mask is what makes this triangle counting rather than two-hop counting:
  $$(A \cdot A)_{i,j}$$ counts common neighbours of $$i$$ and $$j$$ whether or not
  $$i$$ and $$j$$ are adjacent, but only the pairs that are adjacent close a
  triangle.
working_example_part1: |
  Take four vertices with undirected edges
  $$\lbrace 0,1\rbrace,\ \lbrace 0,2\rbrace,\ \lbrace 1,2\rbrace,\ \lbrace 2,3\rbrace$$.
  Vertices $$0,1,2$$ form a triangle; vertex $$3$$ hangs off vertex $$2$$ by a single
  pendant edge and closes no triangle. The symmetric adjacency is

  $$
  A =
  \begin{bmatrix}
  0 & 1 & 1 & 0\\
  1 & 0 & 1 & 0\\
  1 & 1 & 0 & 1\\
  0 & 0 & 1 & 0
  \end{bmatrix}.
  $$

  The two-hop counts $$A \cdot A$$ record, for every pair $$(i,j)$$, how many common
  neighbours they have (the off-diagonal entries; the diagonal is each vertex's
  degree and is irrelevant here):

  $$
  A \cdot A =
  \begin{bmatrix}
  2 & 1 & 1 & 1\\
  1 & 2 & 1 & 1\\
  1 & 1 & 3 & 0\\
  1 & 1 & 0 & 1
  \end{bmatrix}.
  $$

  Several off-diagonal entries are non-zero where there is *no* edge: for
  instance $$(A \cdot A)_{0,3} = 1$$ records the two-hop path
  $$0\to 2\to 3$$, but $$0$$ and $$3$$ are not adjacent, so it closes no triangle. The
  mask removes exactly these.
working_example_part2: |
  Hadamard-masking the two-hop counts by the adjacency, $$T = (A \cdot A) \circ A$$,
  keeps a count only where an edge exists:

  $$
  T = (A \cdot A) \circ A =
  \begin{bmatrix}
  0 & 1 & 1 & 0\\
  1 & 0 & 1 & 0\\
  1 & 1 & 0 & 0\\
  0 & 0 & 0 & 0
  \end{bmatrix}.
  $$

  Reading off the surviving entries: the three edges of the triangle,
  $$(0,1),(0,2),(1,2)$$ and their transposes, each carry a count of $$1$$ (one common
  neighbour, the third triangle vertex). The pendant edge $$(2,3)$$ is present in
  the mask but $$2$$ and $$3$$ have no common neighbour, so $$(A \cdot A)_{2,3} = 0$$
  and $$T_{2,3} = 0$$ — the edge survives the mask but contributes nothing. The
  masked-away non-edges (such as $$(0,3)$$) drop out entirely.

  The totals follow by reduction:

  | quantity | reduction | value |
  |----|----|----|
  | global sum $$\sum_{i,j} T_{i,j}$$ | over both ranks | $$6$$ |
  | global triangles | $$\tfrac{1}{6}\sum_{i,j} T_{i,j}$$ | $$1$$ |
  | per-vertex $$\sum_{j} T_{i,j}$$ | over $$j$$ | $$\lbrace 0:2,\ 1:2,\ 2:2,\ 3:0\rbrace$$ |
  | per-vertex triangles | $$\tfrac{1}{2}\sum_{j} T_{i,j}$$ | $$\lbrace 0:1,\ 1:1,\ 2:1,\ 3:0\rbrace$$ |

  Checking by hand: the only triangle is $$\lbrace 0,1,2\rbrace$$, so the global
  count is $$1$$, vertices $$0,1,2$$ each sit in exactly one triangle, and vertex $$3$$
  sits in none — matching the table. Each triangle is counted six times in
  $$\sum_{i,j} T_{i,j}$$ (three edges, each in both orientations), hence the
  $$\tfrac{1}{6}$$; each triangle through a vertex contributes to two of that
  vertex's incident edges, hence the $$\tfrac{1}{2}$$.
edge_expression_walkthrough: |
  **Count and mask (one Einsum).** The whole computation is a single Einsum with
  three adjacency operands and two operation labels:

  $$T_{i,j} = (A_{i,k} \cdot^1 A_{k,j})_{i,j} \cdot^2 A_{i,j} :: \textstyle\bigwedge^1_k \times(\cap)\ \bigvee^1_k +(\cup)\ \bigwedge^2 \times(\cap).$$

  Walking the iteration space $$(i,j,k)$$: at each point the machinery projects
  into $$A$$ three times to read $$A_{i,k}$$, $$A_{k,j}$$, and $$A_{i,j}$$.

  - Operation label $$1$$ ($$\cdot^1$$) is the two-hop count. The map
    $$\bigwedge^1_k \times(\cap)$$ multiplies $$A_{i,k}$$ by $$A_{k,j}$$ under an
    intersection merge, so it fires only at a $$k$$ that is a neighbour of *both*
    $$i$$ and $$j$$ — a genuine two-hop path $$i\to k\to j$$. The reduce
    $$\bigvee^1_k +(\cup)$$ then sums over the middle vertex $$k$$, giving the number
    of common neighbours of $$i$$ and $$j$$. This is the $$(A \cdot A)_{i,j}$$ entry.

  - Operation label $$2$$ ($$\cdot^2$$) is the mask. The map $$\bigwedge^2 \times(\cap)$$
    multiplies the two-hop count by $$A_{i,j}$$ under intersection, so the result
    survives only where $$i$$ and $$j$$ are themselves adjacent. Because $$A_{i,j}$$ is
    Boolean ($$1$$ where present), multiplying preserves the count; the intersection
    is what enforces the mask. There is no reduce on label $$2$$: the $$(i,j)$$ ranks
    pass straight through to the output.

  The result $$T_{i,j}$$ is the number of triangles that use edge $$(i,j)$$ — the
  masked product $$(A \cdot A) \circ A$$.

  **Reduce to totals.** Two follow-on Einsums collapse $$T$$. The per-vertex count
  reduces over $$j$$, $$tpv_i = T_{i,j} :: \bigvee_j +(\cup)$$, and the global sum
  reduces over both ranks, $$tsum = T_{i,j} :: \bigvee_{i,j} +(\cup)$$. Each is a
  default sum reduction. The combinatorial normalisation — $$\tfrac{1}{2}$$
  per-vertex, $$\tfrac{1}{6}$$ global — is a final scalar scaling left as
  post-processing rather than folded into the masked product, since it does not
  change which coordinates are touched (see Other Notes).

  **No stop condition.** Triangle counting is a single, non-iterated Einsum:
  there is no generational rank $$I$$ and therefore no $$\diamond$$ stopping
  condition. The reductions run once.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &A^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &T^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &tpv^{S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &tsum \to \text{integer},\ \text{empty}=0\\[4pt]
  &\triangleright \textbf{Initialization}\\
  &A \to \langle\text{user-specified, symmetric, no self-loops}\rangle\\[4pt]
  &\triangleright \textbf{Extended Einsum (masked two-hop count, single pass)}\\
  &T_{i,j} = (A_{i,k} \cdot^1 A_{k,j})_{i,j} \cdot^2 A_{i,j} :: \textstyle\bigwedge^1_k \times(\cap)\ \bigvee^1_k +(\cup)\ \bigwedge^2 \times(\cap)\\[4pt]
  &\triangleright \textbf{Reductions to totals}\\
  &tpv_i = T_{i,j} :: \textstyle\bigvee_j +(\cup)\\
  &tsum = T_{i,j} :: \textstyle\bigvee_{i,j} +(\cup)\\[4pt]
  &\triangleright \textbf{Post-processing normalization}\\
  &\text{per-vertex triangles} = \tfrac{1}{2}\, tpv_i,\qquad
   \text{global triangles} = \tfrac{1}{6}\, tsum
  \end{aligned}
  $$
other_notes: |
  **One Einsum versus a cascade.** The masked product can equally be written as a
  two-step cascade — first $TT_{i,j} = A_{i,k} \cdot A_{k,j} :: \bigwedge_k
  \times(\cap)\ \bigvee_k +(\cup)$, then
  $$T_{i,j} = TT_{i,j} \cdot A_{i,j} :: \bigwedge \times(\cap)$$ — which materializes
  the full two-hop count $$A \cdot A$$ before masking. The single-Einsum
  triple-product form above is preferred because it never materializes the dense
  $$A \cdot A$$: under the ODE the operation-label-$$2$$ intersection with $$A_{i,j}$$
  restricts which $$(i,j)$$ pairs are effectual, so an implementation may push the
  mask down and compute the $$k$$-reduction only at the $$(i,j)$$ pairs that are
  edges. Both forms denote the same $$T$$.

  **Where the normalization lives.** The $$\tfrac{1}{2}$$ and $$\tfrac{1}{6}$$ are
  stated as post-processing, not folded into the masked Einsum, deliberately. The
  scaling is a single scalar multiply that touches no new coordinates and changes
  no merge behaviour, so folding it in would only obscure the masked-product
  structure. A formulation that counts each triangle exactly once instead of
  six times restricts the iteration space rather than dividing afterward; see
  Variants.

  **Empty is not zero.** With `empty` $$=$$ False for $$A$$, the intersection merges
  on both operation labels touch a point only where the relevant edges are
  actually present. A masked-away non-edge such as $$(0,3)$$ in the worked example
  is *absent* from $$T$$, not stored as a zero; the over-count division operates on
  the present entries only.
variants: |
  **Count each triangle once.** Constraining the iteration space to ordered
  triples eliminates the redundancy that the $$\tfrac{1}{6}$$ corrects. Writing the
  enumeration form $$O_{s,d,n} = A_{s,\,d:d<s} \cdot A_{d,\,n:n<d} \cdot A_{s,n}$$
  touches only triples with $$n < d < s$$, so each triangle is produced exactly
  once and the global count is a plain $$\bigvee +$$ reduction with no division.
  This is the node-iterator / FlexMiner style of triangle *enumeration*; it lists
  triangles rather than scoring edges.

  **Directed / oriented counting.** Orienting every edge from lower to higher
  vertex ID and masking with the oriented adjacency counts each triangle once
  without enumerating all triples, the standard work-reduction for triangle
  counting on large graphs.

  **Clustering coefficient.** Dividing each vertex's triangle count $\tfrac{1}{2}
  tpv_i$$ by $$\binom{\deg(i)}{2}$$ — itself a reduction $$\deg_i = A_{i,j} ::
  \bigvee_j +(\cup)$ followed by a scalar map — gives the local clustering
  coefficient, reusing the same masked product.
implementation_notes: |
  The masked Einsum is a masked sparse matrix product (masked SpGEMM): for each
  present edge $$(i,j)$$ in the mask $$A$$, intersect the neighbour lists of $$i$$ and
  $$j$$ and count the overlap. Keeping $$A$$ sparse and pushing the operation-label-$$2$$
  mask down means the inner $$k$$-reduction runs only over edges, not over the full
  $$\lvert V\rvert \times \lvert V\rvert$$ pair space, so the work is proportional to
  summed over edges of the cost of intersecting their endpoints' adjacency lists.
  Ordering each intersection from the smaller neighbour list, and orienting edges
  low-to-high (see Variants), are the usual constant-factor and asymptotic
  improvements. The two reductions are a single pass over the present entries of
  $$T$$.
complexity_costs: |
  The masked product costs $$\sum_{(i,j)\in E} \min(\deg(i),\deg(j))$$ neighbour-list
  comparisons, which is $$O\left(\sum_{(i,j)\in E}\min(\deg(i),\deg(j))\right)$$ and
  bounded by $$O(\lvert E\rvert^{3/2})$$ for the oriented variant — the standard bound
  for matrix-product triangle counting on sparse graphs. The reductions to
  per-vertex and global totals add $$O(\lvert E\rvert)$$ over the present entries of
  $$T$$. Storing $$T$$ as a sparse tensor keeps memory at $$O(\lvert E\rvert)$$ rather
  than $$O(\lvert V\rvert^2)$$.
related_notes: |
  Triangle counting shares the gather-then-mask shape of reachability BFS, where
  $$G \cdot F$$ gathers neighbours and an intersection with $$\neg P$$ masks; here the
  second adjacency operand masks the gathered two-hop counts instead. The
  enumeration variant connects to subgraph matching and $$k$$-clique counting, which
  generalize the masked product to longer paths and larger cliques.
---
