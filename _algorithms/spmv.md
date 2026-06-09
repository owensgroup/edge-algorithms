---
title: Sparse Matrix-Vector Multiply (SpMV)
authors: [Toluwanimi Odemuyiwa]
summary: The foundational sparse-linear-algebra primitive, expressed as a single extended Einsum whose intersection merge skips absent matrix entries and whose reduction sums each output row, with a semiring dial that turns the same expression into BFS-advance and shortest-path relaxation.
tags: [spmv, sparse-linear-algebra, semiring, building-blocks]
status_intro: |
  This page gives sparse matrix-vector multiply, $y = A x$, as a single extended
  Einsum. It is the smallest interesting EDGE expression on the site: one map, one
  reduce, no generational rank, and no stopping condition. What makes it *sparse* is
  not a special operator but the intersection merge $(\cap)$, which contributes a
  product only where both a stored matrix entry and the matching vector entry are
  present. Absent entries of $A$ are skipped, never multiplied as zeros, so the work
  tracks the number of stored nonzeros rather than $\lvert V\rvert^2$. The same
  skeleton, with the $(\times, +)$ pair swapped for another (compute, reduce) pair,
  is exactly the inner product that drives BFS, shortest paths, and many other graph
  algorithms on this site.
problem_statement: |
  Given a sparse matrix $A$ with rows indexed by $i$ and columns indexed by $j$, and
  a dense or sparse vector $x$ indexed by $j$, compute the output vector $y$ indexed
  by $i$, where

  $$y_i = \sum_{j} A_{i,j}\, x_j.$$

  Read as a graph, $A$ is the adjacency tensor and $x$ a per-vertex value: row $i$ of
  $y$ gathers a weighted contribution from every neighbour $j$ of $i$. Two tensors
  carry the input and one carries the output:

  - $A^{I \equiv \lvert V\rvert,\, J \equiv \lvert V\rvert} \to \text{integer}$,
    `empty` $= 0$, the sparse matrix. Only stored (nonzero) entries are present;
    every absent entry is the empty value, distinct from a stored $0$.
  - $x^{J \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= 0$, the input vector.
  - $y^{I \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= 0$, the output
    vector. There is no generational rank on any tensor, because SpMV is a single,
    non-iterated product.
working_example_part1: |
  Take the $4 \times 4$ matrix $A$ with these stored entries (every other entry is
  absent), and the vector $x$:

  $$
  A = \begin{bmatrix} 2 & \cdot & 3 & \cdot \\ \cdot & 1 & \cdot & \cdot \\ 4 & \cdot & \cdot & 5 \\ \cdot & \cdot & 6 & \cdot \end{bmatrix},
  \qquad x = \begin{bmatrix} 1 \\ 2 \\ 3 \\ 4 \end{bmatrix}.
  $$

  As present-only sets, the stored entries are

  $$A = \lbrace (0,0):2,\ (0,2):3,\ (1,1):1,\ (2,0):4,\ (2,3):5,\ (3,2):6\rbrace,$$
  $$x = \lbrace 0:1,\ 1:2,\ 2:3,\ 3:4\rbrace.$$

  Six of the sixteen $(i,j)$ cells are present, so only six products are ever formed.
working_example_part2: |
  SpMV is single-shot, so instead of an iteration trace we compute each output row
  directly. For row $i$, the intersection merge pairs each stored $A_{i,j}$ with
  $x_j$, multiplies, and the reduction sums the products:

  | $i$ | present $(i,j)$ in row | products $A_{i,j}\cdot x_j$ | $y_i$ |
  |----|----|----|----|
  | 0 | $(0,0),\ (0,2)$ | $2\cdot 1,\ 3\cdot 3$ | $2 + 9 = 11$ |
  | 1 | $(1,1)$ | $1\cdot 2$ | $2$ |
  | 2 | $(2,0),\ (2,3)$ | $4\cdot 1,\ 5\cdot 4$ | $4 + 20 = 24$ |
  | 3 | $(3,2)$ | $6\cdot 3$ | $18$ |

  giving $y = \lbrace 0:11,\ 1:2,\ 2:24,\ 3:18\rbrace$.

  The absent cells never enter the arithmetic. Column $j = 1$, for instance, is
  present in $x$ ($x_1 = 2$) but only row $1$ stores an entry in that column, so $x_1$
  contributes to $y_1$ and nothing else. Likewise the empty entries $A_{0,1}$,
  $A_{0,3}$, and so on are simply not in the iteration that contributes work: the
  intersection merge fires only where *both* operands are present, which is precisely
  what makes the product sparse rather than $O(\lvert V\rvert^2)$.
edge_expression_walkthrough: |
  **Multiply (map).** The merge $\bigwedge_j \times(\cap)$ walks the column rank $j$
  and touches a coordinate only where both the matrix entry $A_{i,j}$ and the vector
  entry $x_j$ are present. At each such coordinate it computes the product
  $A_{i,j} \cdot x_j$. Where $A_{i,j}$ is absent there is no intersection, so the
  point is skipped entirely; it is never multiplied as a zero. This is the entire
  sparsity story: `empty` is not the same as the value $0$, and the $(\cap)$ merge
  consumes only the stored structure of $A$.

  **Sum (reduce).** The reduction $\bigvee_j +(\cup)$ collapses the column rank $j$,
  adding together all the products that landed at the same output row $i$. The
  $(\cup)$ marks that the accumulation ranges over the union of contributing columns
  for that row. Each output coordinate $y_i$ receives the sum of its row's products.

  Because the map names $j$ on its merge and the reduce names $j$ on its reduction,
  $j$ is consumed and only $i$ survives to the output. There is no generational rank
  $I$ and no $\diamond$ stop condition: the whole computation is one pass over the
  stored $(i,j)$ pairs.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &A^{I \equiv \lvert V\rvert,\, J \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &x^{J \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &y^{I \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\[4pt]
  &\triangleright \textbf{Extended Einsum (single, non-iterated product)}\\
  &y_{i} = A_{i,j} \cdot x_{j} :: \textstyle\bigwedge_j \times(\cap)\ \bigvee_j +(\cup)
  \end{aligned}
  $$
other_notes: |
  `empty` $\neq$ zero is the load-bearing distinction. If $A$'s absent entries were
  treated as stored zeros, the iteration would walk all $\lvert V\rvert^2$ cells and
  multiply a great many zeros: dense matrix-vector multiply. By declaring `empty`
  $= 0$ and using the intersection merge, the expression visits only the stored
  $(i,j)$ pairs, and the additive identity of the absent entries is honoured by their
  *absence* from the sum rather than by an explicit $+0$.

  The choice of intersection ($\cap$) rather than union ($\cup$) for the multiply is
  what encodes "skip the structural zeros." A union merge on the multiply would force
  the point to be visited wherever *either* operand is present, which is neither what
  SpMV computes nor what a sparse kernel does. The union belongs on the reduction,
  where it correctly accumulates over whichever columns happen to contribute to a row.
variants: |
  SpMV is a single instance of a more general $(\text{compute}, \text{reduce})$
  inner product, and swapping that pair turns it into other primitives without
  touching the iteration structure:

  - $(\times,\ +)$ over the reals is the SpMV shown here.
  - $(+,\ \text{ANY})$ is the depth-tracking BFS *Advance*, and
    $(\text{AND},\ \text{OR})$ on Boolean tensors is the reachability *Advance*:
    both gather the neighbours of a frontier across the graph. See
    Breadth-First Search, whose advance step
    $T_{i,d} = G_{s,d} \cdot F_{i,s} :: \bigwedge_s +(\cap)\ \bigvee_s \text{ANY}(\cup)$
    is exactly this product with a generational rank added.
  - $(+,\ \min)$ is min-plus (tropical) relaxation: add an edge weight to a source
    distance and keep the smallest per destination. Iterating that product is
    Bellman-Ford; see Shortest Paths.

  In every case the matrix stays sparse for the same reason: the multiply-side merge
  is an intersection, so only stored edges contribute.
implementation_notes: |
  The expression is a single sparse gather-and-accumulate. With $A$ stored row-major
  (e.g. CSR), the natural schedule iterates rows $i$ in the outer loop and, for each
  row, its stored columns $j$ in the inner loop, reading $x_j$ and accumulating into
  $y_i$ once per stored nonzero. Storing $A$ column-major (CSC) instead schedules the
  same Einsum as a column-outer scatter into $y$. Both are concordant traversals of
  the one expression; the merge and reduce do not change, only the loop order and the
  tensor formats do.
complexity_costs: |
  Each stored nonzero of $A$ is touched exactly once to form one product and one
  addition, so SpMV costs $O(\text{nnz}(A))$ work, where $\text{nnz}(A)$ is the number
  of stored entries. In graph terms this is $O(\lvert E\rvert)$, linear in the edges,
  as opposed to the $O(\lvert V\rvert^2)$ of dense matrix-vector multiply. The output
  $y$ has at most $\lvert V\rvert$ present entries.
related_notes: |
  Breadth-First Search uses this product as its Advance step, with a Boolean
  semiring and a generational rank. Bellman-Ford and the other shortest-path pages
  iterate the min-plus version of the same product. SpMV is the un-iterated kernel
  underneath all of them.
---
