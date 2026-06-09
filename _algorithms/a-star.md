---
title: A* Search
authors: [Toluwanimi Odemuyiwa]
summary: Goal-directed shortest path that orders Dijkstra's pick by distance-so-far plus a heuristic estimate of the remaining distance to a goal, shown with a Manhattan heuristic computed once from per-vertex grid coordinates.
tags: [shortest-paths, a-star, heuristic-search, graphs]
status_intro: |
  This page gives A* search as a one-step extension of Dijkstra. The only change is
  the priority that chooses which vertex to settle next: distance-so-far $g$ plus a
  heuristic estimate $h$ of the remaining distance to a goal, rather than $g$ alone.
  The distance tensor still holds true distances throughout — the heuristic only
  steers the order of expansion — and the search stops as soon as the goal is
  settled. With an admissible heuristic (here, Manhattan distance on a grid) the
  computed distances are exact.
problem_statement: |
  Given a directed, weighted graph $G$ on $\lvert V\rvert$ vertices with non-negative
  weights, a source set $root\_id$, a single goal vertex, and an $(x,y)$ location
  for every vertex, compute the shortest distance from the source to the goal.

  The heuristic is built once from the locations; the search then runs the Dijkstra
  body with a goal-aware priority:

  - $G$, the graph, and $D, Q, DQ, N, C, NewlyRelaxed, T$, exactly as in Dijkstra.
  - $L^{S \equiv \lvert V\rvert,\, XY \equiv 2} \to \text{integer}$, each vertex's
    $(x,y)$ coordinates.
  - $goal^{S \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the one-hot
    target vertex.
  - $H^{S \equiv \lvert V\rvert} \to \text{integer}$, the Manhattan distance from each
    vertex to the goal, computed once (no $I$ rank).
  - $PQ^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, the priority $f = g + h$ of
    the queued vertices, and $M^{I,\, S \equiv \lvert V\rvert}$, the one-hot vertex of
    minimum priority.
working_example_part1: |
  Take four vertices on a grid with edges
  $0\to1\,(1),\ 0\to2\,(1),\ 1\to3\,(1),\ 2\to3\,(5)$, source $0$, and goal $3$.
  The vertex locations are

  $$0:(0,0),\quad 1:(1,0),\quad 2:(0,1),\quad 3:(2,0),$$

  so the Manhattan heuristic to the goal at $(2,0)$ is

  $$H = \lbrace 0:2,\ 1:1,\ 2:3,\ 3:0\rbrace,\qquad H_v = \lvert x_v - 2\rvert + \lvert y_v - 0\rvert.$$

  Initialization sets the source distance to $0$ and queues it:
  $$D_0 = \lbrace 0:0\rbrace,\qquad Q_0 = \lbrace 0\rbrace.$$
working_example_part2: |
  Each round settles the queued vertex of smallest priority $f = g + h$, then runs
  the Dijkstra relaxation body:

  | $i$ | settled (min $f$) | $f$ | relaxations | $D_{i+1}$ | $Q_{i+1}$ |
  |----|----|----|----|----|----|
  | 0 | $0$ | $0\!+\!2$ | $1\!\leftarrow\!1,\ 2\!\leftarrow\!1$ | $\lbrace 0:0,\ 1:1,\ 2:1\rbrace$ | $\lbrace 1,2\rbrace$ |
  | 1 | $1$ | $1\!+\!1$ | $3\!\leftarrow\!2$ | $\lbrace 0:0,\ 1:1,\ 2:1,\ 3:2\rbrace$ | $\lbrace 2,3\rbrace$ |
  | 2 | $3$ | $2\!+\!0$ | goal settled — stop | unchanged | — |

  In round $1$ both vertex $1$ ($f = 1+1 = 2$) and vertex $2$ ($f = 1+3 = 4$) are
  queued at the same distance $g = 1$, but the heuristic breaks toward the goal:
  vertex $1$ is settled, never vertex $2$. The goal is then settled in round $2$ and
  the search stops with $d(0,3) = 2$. Plain Dijkstra would also have settled vertex
  $2$ before halting; the heuristic lets A* skip it.
edge_expression_walkthrough: |
  **Heuristic (once).** $Lg_{xy} = L_{s,xy} \cdot goal_s :: \bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup)$
  takes the goal vertex's own $(x,y)$ row from the location tensor. Then
  $AD_{v,xy} = \text{abs}(L_{v,xy} \cdot Lg_{xy}) :: \bigwedge -(\cap)$ is the
  per-axis absolute difference $\lvert L_v - Lg\rvert$, and
  $H_v = AD_{v,xy} :: \bigvee +(\cup)$ sums over the location axes — the
  Manhattan distance. The heuristic depends only on fixed positions, so it carries
  no iteration rank and is computed once.

  **Pick.** $DQ_{i,s} = Q_{i,s} \cdot D_{i,s}$ gives the distance-so-far $g$ of the
  queued vertices; $PQ_{i,s} = DQ_{i,s} \cdot H_s :: \bigwedge +(\cap)$ adds the
  heuristic to form the priority $f = g + h$. The populate
  $M_{i,s^\ast} = PQ_{i,s} :: \lll_{s^\ast} \mathbf{1}(\text{select-min-s})$ selects the
  minimum-$f$ vertex. Finally $F_{i,s} = M_{i,s} \cdot D_{i,s} :: \bigwedge \rightarrow(\cap)$
  reads the *true* distance $g$ back out of $D$ at the picked coordinate — the
  frontier must carry $g$, not $f$, so that relaxation propagates real distances.

  **Relax / Improve / Record / Update / Dequeue / Enqueue.** Identical to Dijkstra:
  relax the picked vertex's out-edges, keep strict improvements, fold them into $D$,
  remove the settled vertex from the queue, and enqueue whatever improved.

  **Stop.** $\diamond : \lVert F_i \cdot goal \rVert \equiv 1$. Both $F$ and $goal$
  are one-hot, so the occupancy of their intersection is $1$ exactly when the vertex
  just settled is the goal. The search halts the moment the goal is settled, when
  its distance is already final.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Inputs}\\
  &G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &L^{S \equiv \lvert V\rvert,\, XY \equiv 2} \to \text{integer},\ \text{empty}=\infty\\
  &goal^{S \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\[4pt]
  &\triangleright \textbf{Heuristic (computed once, no } I \text{ rank)}\\
  &Lg_{xy} = L_{s,xy} \cdot goal_s :: \textstyle\bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup)\\
  &AD_{v,xy} = \text{abs}(L_{v,xy} \cdot Lg_{xy}) :: \textstyle\bigwedge -(\cap)\\
  &H_v = AD_{v,xy} :: \textstyle\bigvee +(\cup)\\[4pt]
  &\triangleright \textbf{Search state}\\
  &D^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &Q^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &DQ^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &PQ^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
  &M^{I,\, S \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=\infty\\
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
  &PQ_{i,s} = DQ_{i,s} \cdot H_s :: \textstyle\bigwedge +(\cap)\\
  &M_{i,s^\ast} = PQ_{i,s} :: \textstyle\lll_{s^\ast} \mathbf{1}(\text{select-min-s})\\
  &F_{i,s} = M_{i,s} \cdot D_{i,s} :: \textstyle\bigwedge \rightarrow(\cap)\\
  &N_{i,d} = G_{s,d} \cdot F_{i,s} :: \textstyle\bigwedge +(\cap)\ \bigvee \min(\cup)\\
  &C_{i,d} = N_{i,d} \cdot D_{i,d} :: \textstyle\bigwedge <(\cup)\\
  &NewlyRelaxed_{i,d} = C_{i,d} \cdot N_{i,d} :: \textstyle\bigwedge \rightarrow(\cap)\\
  &D_{i+1,d} = D_{i,d} \cdot NewlyRelaxed_{i,d} :: \textstyle\bigwedge \texttt{<<}(\cup)\\
  &T_{i,d} = Q_{i,d} \cdot F_{i,d} :: \textstyle\bigwedge \leftarrow(\text{take\_left\_only})\\
  &Q_{i+1,d} = T_{i,d} \cdot C_{i,d} :: \textstyle\bigwedge \text{OR}(\cup)\\
  &\diamond : \lVert F_i \cdot goal \rVert \equiv 1
  \end{aligned}
  $$
other_notes: |
  The distance tensor $D$ never holds $f$. The heuristic enters only the priority
  $PQ$ used to choose the next vertex; the frontier $F = M \cdot D$ reads the true
  distance $g$ back out of $D$ before relaxing, so the stored distances stay exact.
  Manhattan distance on a grid is a *consistent* (and admissible) heuristic, which
  is what lets A* keep Dijkstra's settle-once structure: a settled vertex is never
  re-opened. A merely admissible but inconsistent heuristic could require
  re-opening settled vertices, which this cascade does not do.
variants: |
  Setting $H \equiv 0$ everywhere makes the priority $f = g$, so the pick collapses
  to Dijkstra's. The goal also acts as a dial on the heuristic: aiming at a single
  vertex gives focused A* search; treating every vertex as a goal makes
  $H \equiv 0$, recovering Dijkstra's undirected single-source exploration. Other
  heuristics reuse the same skeleton — replacing the $\bigvee +$ reduction with
  $\bigvee \max$ gives the Chebyshev distance, for example.
implementation_notes: |
  The heuristic is a one-time setup: a difference and absolute value per coordinate
  axis, reduced to a scalar per vertex. The search loop is Dijkstra's, with one
  extra elementwise add ($PQ = DQ + H$) before the argmin pick and one extra
  take-right ($F = M \cdot D$) after it. Because the search halts on settling the
  goal rather than emptying the queue, an informative heuristic expands far fewer
  vertices than Dijkstra.
complexity_costs: |
  The heuristic costs $O(\lvert V\rvert)$ to build. Worst case (an uninformative
  heuristic) matches Dijkstra at $O((\lvert V\rvert + \lvert E\rvert)\,\log \lvert V\rvert)$; in
  practice an admissible, well-correlated heuristic settles only the vertices on or
  near a shortest path to the goal, far fewer than $\lvert V\rvert$.
related_notes: |
  Dijkstra is A* with a zero heuristic and a queue-empty stop. Bellman-Ford is the
  no-queue ancestor that relaxes every vertex each round and tolerates negative
  weights.
---
