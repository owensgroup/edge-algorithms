---
title: Direction-Optimizing BFS (DO-BFS)
authors: [Sanjana Mali, Toluwanimi Odemuyiwa]
summary: Breadth-first search that dynamically switches between top-down (push from frontier) and bottom-up (pull from unvisited) traversal to cut wasted edge examinations on low-diameter graphs, expressed as EDGE cascades with macros.
tags: [bfs, traversal, direction-optimizing, frontier, graphs]
status_intro: |
  This page gives the Direction-Optimizing BFS of Beamer et al. (2012) in EDGE notation.
  It walks through three formulations in order: **top-down** (push from the frontier),
  **bottom-up** (pull from unvisited vertices), and **hybrid** (switch dynamically between
  the two). For each, the EDGE cascade is preceded by a step-by-step walkthrough that
  connects every tensor operation to its graph-algorithm meaning.

  The page also shows how several published switching heuristics (Beamer, Gunrock, Ligra,
  XBFS) are all different EDGE cascades over the same underlying tensors, making it easy
  to compare them side by side.

problem_statement: |
  Given an unweighted directed graph $G = (V, E)$ and a source vertex $s \in V$, find
  all vertices reachable from $s$ and construct a BFS tree that records, for each
  reachable vertex, a parent that discovered it.

  The state is carried by three tensors, each indexed by the generational rank $I$ that
  counts BFS levels:

  - $G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False.
    $G_{s,d} = $ True iff edge $s \to d$ exists. Static; carries no $I$ rank.
  - $F^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False.
    $F_{i,s} = $ True iff vertex $s$ is in the frontier at level $i$.
  - $\mathit{Tree}^{I,\, S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean}$,
    `empty` $=$ False. $\mathit{Tree}_{i,s,d} = $ True means vertex $s$ is recorded as the
    parent of vertex $d$ by level $i$.

  The $\mathit{Tree}$ tensor is the tensor representation of the parent array: a $(s,d)$ entry
  means "$s$ is the parent of $d$." At the format/implementation level it would typically be
  stored as a compact parent array rather than a dense matrix.

working_example_part1: |
  Consider the five-vertex graph with directed edges
  $1 \to 2,\ 1 \to 3,\ 2 \to 4,\ 3 \to 4,\ 4 \to 5$ and source $s = 1$.

  **Initialization** places the source in the frontier and records it as its own parent:

  $$F_0 = \{1:\text{T}\}, \qquad \mathit{Tree}_0 = \{(1,1):\text{T}\}.$$

  From $\mathit{Tree}$, not-yet-visited vertices are those with no recorded parent:
  at level 0, vertices $2, 3, 4, 5$ are not yet visited.

  **Top-down** expands outward from the frontier. **Bottom-up** instead examines each
  unvisited vertex and asks whether any of its parents lie in the frontier. Both yield
  the same BFS tree; the difference is traversal order, which changes cost when the
  frontier is large.

working_example_part2: |
  **Top-down trace:**

  | Level $i$ | Frontier $F_i$ | New discoveries | Parent edges added |
  |---|---|---|---|
  | 0 | $\{1\}$ | $2, 3$ | $(1,2),\ (1,3)$ |
  | 1 | $\{2, 3\}$ | $4$ | $(2,4)$ |
  | 2 | $\{4\}$ | $5$ | $(4,5)$ |
  | 3 | $\{5\}$ | — (stop) | — |

  $\lVert F_4 \rVert = 0$ fires and the cascade halts. Vertex 4 could have been
  discovered via $3 \to 4$ as well, but only one parent is kept (here, vertex 2).

  **Bottom-up trace** for comparison: at level 1, instead of expanding from vertex 1, we
  scan vertices 2–5 and ask "does any parent lie in $\{1\}$?". Vertex 2 checks: yes
  (edge $1 \to 2$). Vertex 3 checks: yes ($1 \to 3$). Vertices 4 and 5: no (their parents
  are 2, 3 and 4 respectively, none in the frontier yet). The result is identical:
  $F_1 = \{2, 3\}$. The same holds for subsequent levels — same tree, reversed traversal
  order.

edge_expression_walkthrough: |
  This walkthrough covers the **top-down** direction. The tensors are those in the Problem
  Statement.

  **Step 1 — Identify not-yet-visited vertices (NV).**
  A vertex $c$ is not yet visited if it has no parent recorded in $\mathit{Tree}$. Reduce
  $\mathit{Tree}_{i,p,c}$ over the parent rank $p$ with OR, then negate:

  $$NV_{i,c} = \neg \mathit{Tree}_{i,p,c} :: \textstyle\bigvee_p \text{OR}(\cup).$$

  **Step 2 — Gather neighbors of the frontier (Nei).**
  $d$ is a neighbor of the frontier if there exists at least one frontier vertex $s$ with
  edge $s \to d$. AND over $(s,d)$ with intersect, OR-reduce over $s$:

  $$\mathit{Nei}_{i,d} = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge_s \text{AND}(\cap)\ \bigvee_s \text{OR}(\cup).$$

  **Step 3 — Keep only newly discovered vertices (NeiNV).**
  Intersect the neighbor set with the not-visited mask:

  $$\mathit{NeiNV}_{i,d} = \mathit{Nei}_{i,d} \cdot NV_{i,d} :: \textstyle\bigwedge_d \text{AND}(\cap).$$

  **Step 4 — Record candidate parent links (CandTree).**
  A $(s,d)$ edge enters the BFS tree candidate set when $s$ is in the frontier, edge $s\to d$
  exists, and $d$ is newly discovered:

  $$\mathit{CandTree}_{i,s,d} = (G_{s,d} \cdot F_{i,s})_{i,s,d} \cdot \mathit{NeiNV}_{i,d}
    :: \textstyle\bigwedge \text{AND}(\cap)\ \bigwedge \text{AND}(\cap).$$

  **Step 5 — Select exactly one parent per child (Temp).**
  Multiple frontier vertices may discover the same child. The populate action with
  `pick-parent` enforces one parent per child (e.g., smallest-index parent):

  $$\mathit{Temp}_{i,s^*,d} = \mathit{CandTree}_{i,s,d} \lll_{s^*} \mathbf{1}(\text{pick-parent}).$$

  **Step 6 — Update the BFS tree (Tree).**
  OR-union the new parent assignments into the accumulated tree:

  $$\mathit{Tree}_{i+1,s,d} = \mathit{Tree}_{i,s,d} \cdot \mathit{Temp}_{i,s,d}
    :: \textstyle\bigwedge_d \text{OR}(\cup).$$

  **Step 7 — Advance the frontier.**
  The next frontier is exactly the newly discovered vertices:

  $$F_{i+1,d} = \mathit{NeiNV}_{i,d}.$$

  **Step 8 — Stop when the frontier is empty:**

  $$\diamond : \lVert F_{i+1} \rVert \equiv 0.$$

  Here $\lVert \cdot \rVert$ counts present (non-empty) entries, not zero-valued ones, so
  the source at depth 0 does not accidentally trigger early termination.

edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &F^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &\mathit{Tree}^{I,\, S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\[4pt]
  &\triangleright \textbf{Initialization}\\
  &F_{0,\, s\,:\,s = \text{source}} = \text{True}\\
  &\mathit{Tree}_{0,\, s\,:\,s = \text{source},\, d\,:\,d = \text{source}} = \text{True}\\[4pt]
  &\triangleright \textbf{Extended Einsum — Top-Down}\\
  &NV_{i,c} = \neg \mathit{Tree}_{i,p,c} :: \textstyle\bigvee_p \text{OR}(\cup)\\
  &\mathit{Nei}_{i,d} = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge_s \text{AND}(\cap)\ \bigvee_s \text{OR}(\cup)\\
  &\mathit{NeiNV}_{i,d} = \mathit{Nei}_{i,d} \cdot NV_{i,d} :: \textstyle\bigwedge_d \text{AND}(\cap)\\
  &\mathit{CandTree}_{i,s,d} = (G_{s,d} \cdot F_{i,s})_{i,s,d} \cdot \mathit{NeiNV}_{i,d}
    :: \textstyle\bigwedge \text{AND}(\cap)\ \bigwedge \text{AND}(\cap)\\
  &\mathit{Temp}_{i,s^*,d} = \mathit{CandTree}_{i,s,d} \lll_{s^*} \mathbf{1}(\text{pick-parent})\\
  &\mathit{Tree}_{i+1,s,d} = \mathit{Tree}_{i,s,d} \cdot \mathit{Temp}_{i,s,d}
    :: \textstyle\bigwedge_d \text{OR}(\cup)\\
  &F_{i+1,d} = \mathit{NeiNV}_{i,d}\\
  &\diamond : \lVert F_{i+1} \rVert \equiv 0
  \end{aligned}
  $$

other_notes: |
  **Why a Tree tensor instead of a visited bit?**
  The $\mathit{Tree}^{I,S,D}$ tensor encodes parent-child relationships using two vertex
  ranks ($S$ for parent, $D$ for child). This follows the EDGE convention that each set of
  vertices playing a distinct role gets its own named rank. As a result, "is vertex $d$
  visited?" is derived by reducing $\mathit{Tree}_{i,p,d}$ over the parent rank $p$ — it is
  not stored separately. At the format/implementation level, $\mathit{Tree}$ would typically
  be materialized as a compact parent array, not a dense $\lvert V\rvert \times \lvert V\rvert$
  Boolean matrix.

  **Top-Down vs Bottom-Up: same computation, different evaluation order.**
  Both formulations identify edges $(s,d)$ where $s$ is in the frontier and $d$ is not yet
  visited. Top-down organizes the computation around the source rank $s$ (apply frontier mask
  first, then filter unvisited destinations). Bottom-up organizes around the destination rank
  $d$ (apply not-visited mask first, then check whether any parent $s$ is in the frontier).
  Algebraically, the underlying condition is identical; the difference is which mask is applied
  first, which changes the set of edges examined and enables early-exit in the bottom-up case.

  **Memory optimization.** The cascade as written materializes all generations of $F$ and
  $\mathit{Tree}$. In practice only two generations are needed (ping-pong), since each level
  depends only on the previous one:

  $$F_{(i+1)\bmod 2,\,d} = G_{s,d} \cdot F_{i\bmod 2,\,s} \cdot NV_{i\bmod 2,\,d}
    :: \textstyle\bigwedge \text{AND}(\cap)\ \bigvee \text{OR}(\cup).$$

  This is a format/implementation choice, invisible to the EDGE specification.

variants: |
  **Bottom-Up.**
  In the bottom-up direction we scan unvisited vertices and check whether any incoming
  neighbor lies in the frontier. The tensors are the same; only the computation order changes.

  $$
  \begin{aligned}
  &\triangleright \textbf{Extended Einsum — Bottom-Up}\\
  &NV_{i,c} = \neg \mathit{Tree}_{i,p,c} :: \textstyle\bigvee_p \text{OR}(\cup)\\
  &\mathit{CandParents}_{i,s,d} = G_{s,d} \cdot NV_{i,d} :: \textstyle\bigwedge_d \text{AND}(\cap)\\
  &\mathit{InF}_{i,s,d} = \mathit{CandParents}_{i,s,d} \cdot F_{i,s} :: \textstyle\bigwedge_s \text{AND}(\cap)\\
  &\mathit{Temp}_{i,s^*,d} = \mathit{InF}_{i,s,d} \lll_{s^*} \mathbf{1}(\text{pick-parent})\\
  &\mathit{Tree}_{i+1,s,d} = \mathit{Temp}_{i,s,d} \cdot \mathit{Tree}_{i,s,d}
    :: \textstyle\bigwedge_d \text{OR}(\cup)\\
  &F_{i+1,d} = \mathit{InF}_{i,s,d} :: \textstyle\bigvee_s \text{OR}(\cup)\\
  &\diamond : \lVert F_{i+1} \rVert \equiv 0
  \end{aligned}
  $$

  The key difference from top-down is in `CandParents` vs `Nei`: bottom-up restricts to
  unvisited destinations first ($NV_{i,d}$), then checks whether a parent is in the frontier
  ($F_{i,s}$). The `OR` in the frontier update ($F_{i+1}$) also enables early-exit: once a
  not-visited vertex finds one parent in the frontier, it can stop examining further.

  ---

  **Hybrid (Direction-Optimizing).**
  The hybrid cascade uses macros to call either TOP-DOWN or BOTTOM-UP per level based on a
  switching condition. A Boolean tensor $\mathit{TopDown}^I$ tracks the current direction.

  $$
  \begin{aligned}
  &\triangleright \textbf{Additional Tensor}\\
  &\mathit{TopDown}^I \to \text{Boolean},\ \text{empty}=\text{False},\quad \mathit{TopDown}_0 = \text{True}\\[4pt]
  &\triangleright \textbf{Extended Einsum — Hybrid (Beamer switching heuristic)}\\
  &NV_{i,c} = \neg \mathit{Tree}_{i,p,c} :: \textstyle\bigvee_p \text{OR}(\cup)\\
  &\begin{pmatrix}\mathit{Tree}_{i+1},\, F_{i+1},\, \mathit{TopDown}_{i+1}\end{pmatrix}
    = \begin{cases}
        \text{TOP-DOWN}(G, F_i, \mathit{Tree}_i, NV_i, m_f^i, m_u^i) & \text{if } \mathit{TopDown}_i\\
        \text{BOTTOM-UP}(G, F_i, \mathit{Tree}_i, NV_i, n_f^i) & \text{otherwise}
      \end{cases}\\
  &\diamond : \lVert F_{i+1} \rVert \equiv 0
  \end{aligned}
  $$

  Inside the TOP-DOWN macro, the switching condition computes $m_f$ (outgoing edge count
  of the frontier) and $m_u$ (outgoing edge count of unvisited vertices) and switches to
  bottom-up when $m_f > m_u / \alpha$:

  $$
  \begin{aligned}
  &\mathit{MF}_i = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge \text{AND}(\cap)\ \bigvee +(\cup)\\
  &\mathit{MU}_i = G_{s,d} \cdot NV_{i,d} :: \textstyle\bigwedge \text{AND}(\cap)\ \bigvee +(\cup)\\
  &B_i = \mathit{MF}_i \cdot (\mathit{MU}_i / \alpha) :: \textstyle\bigwedge >(\cap)\\
  &\mathit{TopDown}_{i+1} = \neg B_i
  \end{aligned}
  $$

  Inside BOTTOM-UP, switch back to top-down when $n_f < \lvert V\rvert / \beta$
  ($n_f$ = size of the new frontier):

  $$
  \begin{aligned}
  &\mathit{NF}_i = F_{i,s} :: \textstyle\bigvee +(\cup)\\
  &\mathit{BT}_i = \mathit{NF}_i \cdot (\lvert V\rvert / \beta) :: \textstyle\bigwedge <(\cap)\\
  &\mathit{TopDown}_{i+1} = \mathit{BT}_i
  \end{aligned}
  $$

  ---

  **Other Switching Heuristics.**

  All switching heuristics address the same question: is top-down or bottom-up cheaper for
  this level? They differ in how they estimate work.

  *Gunrock* (Wang et al., 2017) approximates using global statistics rather than exact edge
  counts to reduce overhead on GPUs. Let $n_f$ = frontier size, $n_u$ = unvisited count,
  $m = \lvert E\rvert$, $n = \lvert V\rvert$:

  $$
  \begin{aligned}
  &\mathit{NF}_i = F_{i,s} :: \textstyle\bigvee +(\cup)\\
  &\mathit{NU}_i = NV_{i,s} :: \textstyle\bigvee +(\cup)\\
  &\mathit{MF}_i = \mathit{NF}_i \cdot (m / n) :: \textstyle\bigwedge \times(\cap)\\
  &\mathit{MU}_i = \mathit{NU}_i \cdot (n / (n - \mathit{NU}_i)) :: \textstyle\bigwedge \times(\cap)\\
  &\mathit{TB}_i = \mathit{MF}_i \cdot (\mathit{MU}_i \cdot \mathit{do\_a}) :: \textstyle\bigwedge >(\cap)
    \quad\text{(switch to bottom-up)}\\
  &\mathit{BT}_i = \mathit{MF}_i \cdot (\mathit{MU}_i \cdot \mathit{do\_b}) :: \textstyle\bigwedge <(\cap)
    \quad\text{(switch back to top-down)}
  \end{aligned}
  $$

  *Ligra* (Shun & Blelloch, 2013) combines frontier size and frontier out-degree against a
  single threshold:

  $$
  \begin{aligned}
  &\mathit{NF}_i = F_{i,s} :: \textstyle\bigvee +(\cup)\\
  &\mathit{MF}_i = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge \text{AND}(\cap)\ \bigvee +(\cup)\\
  &\mathit{TB}_i = (\mathit{NF}_i + \mathit{MF}_i) \cdot \text{threshold} :: \textstyle\bigwedge >(\cap)
  \end{aligned}
  $$

  *XBFS* (Yang et al., 2024) compares frontier edge count to the total graph edge count:

  $$
  \begin{aligned}
  &\mathit{MF}_i = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge \text{AND}(\cap)\ \bigvee +(\cup)\\
  &\mathit{BU}_i = \mathit{MF}_i \cdot (\lvert E\rvert \cdot \alpha) :: \textstyle\bigwedge >(\cap)
  \end{aligned}
  $$

  Despite the different formulas, the same two EDGE primitives appear in every heuristic:
  $G \cdot F$ with AND$(\cap)$ / $+(\cup)$ counts frontier out-edges, and $F$ with
  $+(\cup)$ counts frontier vertices. The heuristics differ only in how those counts are
  combined and compared.

implementation_notes: |
  The advance step (gathering neighbors, `Nei`) is a sparse Boolean matrix-vector product
  over the graph tensor. The filter (`NeiNV`) and tree-update (`Tree`) steps are elementwise
  merges over the destination rank.

  In top-down mode, work per level is proportional to the number of edges leaving the
  frontier. In bottom-up mode, work per level is proportional to edges entering unvisited
  vertices — more efficient when the frontier is large relative to the unvisited set, because
  each unvisited vertex can exit early once it finds one frontier parent.

  Keeping $F$ and $\mathit{Tree}$ as sparse tensors (present-only) means each level touches
  work proportional to active vertices and their edges, not to $\lvert V\rvert$.

complexity_costs: |
  In top-down mode each vertex enters the frontier exactly once and each edge is examined
  at most once, giving $O(\lvert V\rvert + \lvert E\rvert)$ total work. The same bound holds for
  bottom-up, but with potentially far fewer edge examinations in practice (early-exit in the
  bottom-up phase). The number of levels is the BFS diameter of the source.

related_notes: |
  The basic (top-down-only) form of this cascade is the depth-tracking BFS on this site.
  Replacing the Boolean edge weights with integer weights and the AND/OR operators with +/min
  recovers Bellman-Ford (all vertices re-relax each level rather than being filtered by a
  visited mask).
---
