---
title: Maximum Flow (Push-Relabel)
authors: [Sanjana Mali, Toluwanimi Odemuyiwa]
summary: Maximum s-t flow by the bulk-synchronous push-relabel method, which seeds a saturating preflow from the source and then repeatedly pushes excess down admissible residual edges and relabels stuck vertices, expressed as one long extended Einsum iterated until no internal vertex holds excess.
tags: [max-flow, push-relabel, residual-graph, graphs]
status_intro: |
  This page gives the bulk-synchronous, full-edge form of the push-relabel
  maximum-flow algorithm. It first saturates every edge out of the source to build
  a preflow, then runs a single extended Einsum once per generation $i$: each active
  internal vertex (one holding excess) pushes flow down one admissible residual edge,
  and any active vertex with no admissible edge is relabeled to a higher label.
  Heights only ever rise, excess only ever moves "downhill," and the cascade halts
  when no internal vertex holds excess — at which point the flow is maximum. This
  artifact has not yet been through the full EDGE agent panel, so the empty-value
  choices (especially for the height tensor $D$) are flagged as unverified below.
problem_statement: |
  Given a directed graph $G$ on $\lvert V\rvert$ vertices with a non-negative capacity
  $C$ on each edge, a source $s$, and a sink $t$, compute a flow $F$ of maximum value
  from $s$ to $t$.

  The state is carried by a flow tensor, a residual tensor, a per-vertex excess, and a
  per-vertex height (label), alongside the source and sink masks and per-round scratch
  tensors:

  - $G^{U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer}$, the graph structure.
  - $C^{U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer}$, the edge capacities.
  - $F^{I,\, U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer}$, the flow on each
    edge at generation $i$ (the answer; reverse entries are negative).
  - $R^{I,\, U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer}$, the residual
    capacity at generation $i$.
  - $E^{I,\, U \equiv \lvert V\rvert} \to \text{integer}$, the excess at each vertex.
  - $D^{I,\, U \equiv \lvert V\rvert} \to \text{integer}$, the height/label of each vertex.
  - $Act^{I,\, U \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the active
    internal vertices (those holding excess).
  - $S^{U \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the one-hot source mask.
  - $T^{U \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the one-hot sink mask.
working_example_part1: |
  Take a four-vertex network with source $s = 0$, sink $t = 3$, two internal
  vertices $1$ and $2$, and capacities

  $$0\to1\,(3),\ 0\to2\,(2),\ 1\to2\,(1),\ 1\to3\,(2),\ 2\to3\,(3).$$

  The two source edges carry total capacity $3 + 2 = 5$ and the two sink edges
  carry total capacity $2 + 3 = 5$, so the maximum flow is $5$.

  Initialization sets every height to $0$ except the source, which is lifted to
  $\lvert V\rvert = 4$ so it can never receive excess back, and zeros the flow and excess:

  $$D_0 = \lbrace 0:4,\ 1:0,\ 2:0,\ 3:0\rbrace,\qquad E_0 = \lbrace \rbrace,\qquad F_0 = \lbrace \rbrace.$$

  The preflow step (generation $i = 1$) then saturates both source edges:
  $F_{1,0,1} = 3,\ F_{1,0,2} = 2$, which floods the two internal vertices with
  excess and adds the reverse residual edges back toward the source:

  $$E_1 = \lbrace 1:3,\ 2:2\rbrace,\qquad Act_1 = \lbrace 1,\ 2\rbrace.$$
working_example_part2: |
  Each generation lets every active vertex push along one admissible residual edge
  ($R > 0$ and height $D(u) = D(v) + 1$), then relabels any active vertex left with no
  admissible edge. The trace below shows heights $D$, excess $E$, and the actions
  taken; the residual graph $R$ updates in lockstep:

  | $i$ | active | push $(u\to v: \text{amt})$ | relabel $(u: D')$ | $D_{i+1}$ | $E_{i+1}$ |
  |----|----|----|----|----|----|
  | 1 | $\lbrace 1,2\rbrace$ | none (no admissible edge) | $1\!:\!1,\ 2\!:\!1$ | $\lbrace 0:4,\ 1:1,\ 2:1\rbrace$ | $\lbrace 1:3,\ 2:2\rbrace$ |
  | 2 | $\lbrace 1,2\rbrace$ | $1\!\to\!3\!:\!2,\ \ 2\!\to\!3\!:\!2$ | $1\!:\!2$ | $\lbrace 0:4,\ 1:2,\ 2:1\rbrace$ | $\lbrace 1:1\rbrace$ |
  | 3 | $\lbrace 1\rbrace$ | $1\!\to\!2\!:\!1$ | none | $\lbrace 0:4,\ 1:2,\ 2:1\rbrace$ | $\lbrace 2:1\rbrace$ |
  | 4 | $\lbrace 2\rbrace$ | $2\!\to\!3\!:\!1$ | none | unchanged | $\lbrace \rbrace$ — stop |

  In generation $1$ both internal vertices sit at height $0$ alongside all of their
  neighbours, so no edge is admissible and both are relabeled to height $1$. In
  generation $2$ each can now push its excess one step down to the sink: vertex $1$
  sends $2$ units on $1\to3$ and vertex $2$ sends $2$ units on $2\to3$, leaving vertex
  $1$ with $1$ leftover unit and no admissible edge, so it relabels to height $2$. That
  lets vertex $1$ push its last unit across $1\to2$ in generation $3$, which gives
  vertex $2$ one unit of excess; vertex $2$ pushes it to the sink in generation $4$.
  No internal vertex now holds excess, so $\lvert Act_5\rvert \equiv 0$ fires and the
  cascade halts. Summing the flow leaving the source gives the maximum flow value
  $F_{0,1} + F_{0,2} = 3 + 2 = 5$.
edge_expression_walkthrough: |
  **Preflow.** $F_{1,u,v} = S_u \cdot C_{u,v} :: \bigwedge *(\cap)$ pushes the full
  capacity out of every source edge. Summing the flow tensor along each axis gives
  the incoming flow $In_{1,v} = F_{1,u,v} :: \bigvee +(\cup)$ and the outgoing flow
  $Out_{1,u} = F_{1,u,v} :: \bigvee +(\cup)$, and the excess is their difference
  $E_{1,u} = In_{1,u} \cdot Out_{1,u} :: \bigwedge -(\cup)$. The initial residual
  $R_1$ is the capacity graph with the source row zeroed (its edges are now
  saturated) and the corresponding reverse edges back toward the source opened: this
  is the ordered case "$0$ if $u = s$; $C_{v,u}$ if $v = s$; $C_{u,v}$ otherwise."

  **Active vertices.** $NST_u = \neg S_u \cdot \neg T_u :: \bigwedge \text{AND}(\cap)$
  marks the internal vertices (neither source nor sink). A vertex is active when it is
  internal and holds positive excess:
  $Act_{i,u} = NST_u \cdot^1 (E_{i,u} \cdot^2 0)_{i,u} :: \bigwedge^1 \text{AND}(\cap)\ \bigwedge^2 >(\cap)$,
  where the inner sub-merge $\cdot^2$ tests $E_{i,u} > 0$ and the outer $\cdot^1$ ANDs that
  against $NST$.

  **Admissibility.** $ActR_{i,u,v} = Act_{i,u} \cdot R_{i,u,v} :: \bigwedge \leftarrow(\cap)$
  keeps the residual edges leaving an active vertex. The height rule
  $Lbl_{i,u,v} = D_{i,u} \cdot^1 (D_{i,v} \cdot^2 1)_{i,v} :: \bigwedge^1 \equiv(\cup)\ \bigwedge^2 +(\cap)$
  is True exactly where $D(u) = D(v) + 1$ — flow may only move strictly downhill by one
  level. Their AND
  $Adm_{i,u,v} = ActR_{i,u,v} \cdot Lbl_{i,u,v} :: \bigwedge \text{AND}(\cap)$
  is the set of admissible residual edges out of active vertices.

  **Push.** The populate
  $PushCand_{i,u,v} = Adm_{i,u,v} \lll_v \mathbf{1}(\text{pick-admissible-edge})$
  selects one admissible neighbour $v$ for each pushing vertex $u$. The amount pushed is
  the smaller of the vertex's excess and the edge's residual, restricted to the chosen
  edge:
  $Delta_{i,u,v} = (E_{i,u} \cdot^1 R_{i,u,v})_{i,u,v} \cdot^2 PushCand_{i,u,v} :: \bigwedge^1 \min(\cap)\ \bigwedge^2 \leftarrow(\cap)$.
  Flow is then adjusted on the chosen edges and their reverses
  $F_{i+1,u,v} = (F_{i,u,v} \cdot^1 Delta_{i,u,v})_{i,u,v} \cdot^2 Delta_{i,v,u} :: \bigwedge^1 +(\cup)\ \bigwedge^2 -(\cup)$,
  excess is updated by what each vertex received versus sent
  ($InPush_{i,u} = Delta_{i,v,u} :: \bigvee +(\cup)$,
  $OutPush_{i,u} = Delta_{i,u,v} :: \bigvee +(\cup)$,
  then
  $E_{i+1,u} = (E_{i,u} \cdot^1 InPush_{i,u})_{i,u} \cdot^2 OutPush_{i,u} :: \bigwedge^1 +(\cup)\ \bigwedge^2 -(\cup)$),
  and the residual is updated symmetrically: used capacity subtracted forward, reverse
  capacity added back
  $R_{i+1,u,v} = (R_{i,u,v} \cdot^1 Delta_{i,u,v})_{i,u,v} \cdot^2 Delta_{i,v,u} :: \bigwedge^1 -(\cup)\ \bigwedge^2 +(\cup)$.

  **Relabel.** Using the updated residual, recompute the admissible mask
  $Adm_{i+1,u,v} = R_{i+1,u,v} \cdot Lbl_{i,u,v} :: \bigwedge \rightarrow(\cap)$ and whether
  each vertex still has any admissible edge
  $HasAdm_{i+1,u} = Adm_{i+1,u,v} :: \bigvee \text{OR}(\cup)$. Recompute the active set on
  the updated excess
  $Act_{i+1,u} = NST_u \cdot^1 (E_{i+1,u} \cdot^2 0)_{i,u} :: \bigwedge^1 \text{AND}(\cap)\ \bigwedge^2 >(\cap)$.
  A vertex relabels when it is still active but stuck:
  $Rel_{i,u} = Act_{i+1,u} \cdot \neg HasAdm_{i+1,u} :: \bigwedge \text{AND}(\cap)$. For those
  vertices, gather residual-neighbour heights
  $NeiLbl_{i,u,v} = (R_{i+1,u,v} \cdot^1 Rel_{i,u})_{i,u,v} \cdot^2 D_{i,v} :: \bigwedge^1 \leftarrow(\cap)\ \bigwedge^2 \rightarrow(\cap)$,
  take the minimum
  $MinNeiLbl_{i,u} = NeiLbl_{i,u,v} :: \bigvee \min(\cup)$, add one
  $NewD_{i,u} = MinNeiLbl_{i,u} \cdot 1 :: \bigwedge +(\cap)$, and write the new height back
  $D_{i+1,u} = NewD_{i,u} \cdot D_{i,u} :: \bigwedge \leftarrow(\cap)$ (relabeled vertices take
  $NewD$; all others keep $D_i$).

  **Stop.** $\diamond : \lvert Act_{i+1}\rvert \equiv 0$. The cascade halts when no internal
  vertex holds excess — every unit of preflow has either reached the sink or been
  returned to the source, and the flow is maximum.
edge_expression: |
  $$
  \begin{aligned}
  &\triangleright \textbf{Tensors}\\
  &G^{U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &C^{U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &F^{I,\, U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &R^{I,\, U \equiv \lvert V\rvert,\, V \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &E^{I,\, U \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &D^{I,\, U \equiv \lvert V\rvert} \to \text{integer},\ \text{empty}=0\\
  &Act^{I,\, U \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &S^{U \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\
  &T^{U \equiv \lvert V\rvert} \to \text{Boolean},\ \text{empty}=\text{False}\\[4pt]
  &\triangleright \textbf{Initialization (}s = \text{source}, t = \text{sink)}\\
  &F_{0,u,v} = 0\\
  &E_{0,u} = 0\\
  &D_{0,u} = 0\\
  &D_{0,\ u\,:\,u = s} = \lvert V\rvert\\
  &S_{u\,:\,u = s} = \text{True}\\
  &T_{u\,:\,u = t} = \text{True}\\[4pt]
  &\triangleright \textbf{Preflow from source (}i = 1\text{)}\\
  &F_{1,u,v} = S_u \cdot C_{u,v} :: \textstyle\bigwedge *(\cap)\\
  &In_{1,v} = F_{1,u,v} :: \textstyle\bigvee +(\cup)\\
  &Out_{1,u} = F_{1,u,v} :: \textstyle\bigvee +(\cup)\\
  &E_{1,u} = In_{1,u} \cdot Out_{1,u} :: \textstyle\bigwedge -(\cup)\\
  &R_{1,u,v} =
  \begin{cases}
  0 & u = s\\
  C_{v,u} & v = s\\
  C_{u,v} & \text{otherwise}
  \end{cases}\\[4pt]
  &\triangleright \textbf{Extended Einsum (one push/relabel round per iteration } i)\\
  &NST_u = \neg S_u \cdot \neg T_u :: \textstyle\bigwedge \text{AND}(\cap)\\
  &Act_{i,u} = NST_u \cdot^1 (E_{i,u} \cdot^2 0)_{i,u} :: \textstyle\bigwedge^1 \text{AND}(\cap)\ \bigwedge^2 >(\cap)\\
  &ActR_{i,u,v} = Act_{i,u} \cdot R_{i,u,v} :: \textstyle\bigwedge \leftarrow(\cap)\\
  &Lbl_{i,u,v} = D_{i,u} \cdot^1 (D_{i,v} \cdot^2 1)_{i,v} :: \textstyle\bigwedge^1 \equiv(\cup)\ \bigwedge^2 +(\cap)\\
  &Adm_{i,u,v} = ActR_{i,u,v} \cdot Lbl_{i,u,v} :: \textstyle\bigwedge \text{AND}(\cap)\\[4pt]
  &\triangleright \textbf{Push Step}\\
  &PushCand_{i,u,v} = Adm_{i,u,v} \lll_v \mathbf{1}(\text{pick-admissible-edge})\\
  &Delta_{i,u,v} = (E_{i,u} \cdot^1 R_{i,u,v})_{i,u,v} \cdot^2 PushCand_{i,u,v} :: \textstyle\bigwedge^1 \min(\cap)\ \bigwedge^2 \leftarrow(\cap)\\
  &F_{i+1,u,v} = (F_{i,u,v} \cdot^1 Delta_{i,u,v})_{i,u,v} \cdot^2 Delta_{i,v,u} :: \textstyle\bigwedge^1 +(\cup)\ \bigwedge^2 -(\cup)\\
  &InPush_{i,u} = Delta_{i,v,u} :: \textstyle\bigvee +(\cup)\\
  &OutPush_{i,u} = Delta_{i,u,v} :: \textstyle\bigvee +(\cup)\\
  &E_{i+1,u} = (E_{i,u} \cdot^1 InPush_{i,u})_{i,u} \cdot^2 OutPush_{i,u} :: \textstyle\bigwedge^1 +(\cup)\ \bigwedge^2 -(\cup)\\
  &R_{i+1,u,v} = (R_{i,u,v} \cdot^1 Delta_{i,u,v})_{i,u,v} \cdot^2 Delta_{i,v,u} :: \textstyle\bigwedge^1 -(\cup)\ \bigwedge^2 +(\cup)\\[4pt]
  &\triangleright \textbf{Relabel Step}\\
  &Adm_{i+1,u,v} = R_{i+1,u,v} \cdot Lbl_{i,u,v} :: \textstyle\bigwedge \rightarrow(\cap)\\
  &HasAdm_{i+1,u} = Adm_{i+1,u,v} :: \textstyle\bigvee \text{OR}(\cup)\\
  &Act_{i+1,u} = NST_u \cdot^1 (E_{i+1,u} \cdot^2 0)_{i,u} :: \textstyle\bigwedge^1 \text{AND}(\cap)\ \bigwedge^2 >(\cap)\\
  &Rel_{i,u} = Act_{i+1,u} \cdot \neg HasAdm_{i+1,u} :: \textstyle\bigwedge \text{AND}(\cap)\\
  &NeiLbl_{i,u,v} = (R_{i+1,u,v} \cdot^1 Rel_{i,u})_{i,u,v} \cdot^2 D_{i,v} :: \textstyle\bigwedge^1 \leftarrow(\cap)\ \bigwedge^2 \rightarrow(\cap)\\
  &MinNeiLbl_{i,u} = NeiLbl_{i,u,v} :: \textstyle\bigvee \min(\cup)\\
  &NewD_{i,u} = MinNeiLbl_{i,u} \cdot 1 :: \textstyle\bigwedge +(\cap)\\
  &D_{i+1,u} = NewD_{i,u} \cdot D_{i,u} :: \textstyle\bigwedge \leftarrow(\cap)\\
  &\diamond : \lvert Act_{i+1}\rvert \equiv 0
  \end{aligned}
  $$
other_notes: |
  The source height is lifted to $\lvert V\rvert$ at initialization, which is the standard
  push-relabel device: no admissible edge can ever point back into the source (its
  neighbours never reach height $\lvert V\rvert - 1$ in a flow problem), so preflow excess that
  cannot reach the sink is routed back along reverse residual edges instead. Heights
  are monotonically non-decreasing across generations, which is what guarantees
  termination; the stop test $\lvert Act_{i+1}\rvert \equiv 0$ fires once all excess has
  drained to the sink or returned to the source.

  Two empty-value choices need a careful look. The flow, residual, excess, and
  capacity tensors all read naturally with `empty` $= 0$: a missing entry behaves
  exactly like a present $0$ wherever they are consumed. The height tensor $D$ is the
  exception. The height rule (the $D_{i,v} +(\cap) 1$ sub-merge) and the
  neighbour-height gather ($\cdots \rightarrow(\cap)\ D_{i,v}$) both read $D(v)$ through
  an intersection, so a legitimately height-$0$ neighbour declared `empty` $= 0$ is
  treated as absent and dropped before reaching the $\min$ reduction — which would let
  the relabel skip height-$0$ neighbours and lift a vertex too high. This points to
  $D$ wanting a non-zero empty sentinel, or those two reads wanting a union /
  pass-through merge instead of an intersection; the resolution needs an evaluator
  run. The explicit $D_{0,u} = 0$ initialization is likewise a no-op when $0$ is the
  empty value.
variants: |
  Replacing the populate $\lll_v$ that picks one admissible edge per vertex with a
  push along *every* admissible edge gives the fully parallel (all-edges) push-relabel
  variant, at the cost of needing to split excess across edges. Choosing the highest
  active vertex first, rather than pushing from all active vertices each generation,
  recovers the sequential highest-label push-relabel order. Replacing the height-rule
  pick and relabel machinery with an augmenting-path search on the residual graph
  recovers the Ford-Fulkerson / Edmonds-Karp family, which grows the flow path by path
  rather than by local push and relabel.
implementation_notes: |
  The preflow is one saturating product over the source row plus the residual case
  setup. Each generation is dominated by the two edge-ranked tensors $F$ and $R$:
  forming the admissible mask, the per-vertex push selection, and the symmetric
  flow / residual / excess updates are all sparse products and merges over the
  $(u, v)$ edge rank, while the active and relabel steps reduce to per-vertex work.
  Keeping $F$, $R$, $E$, and $Act$ sparse means each generation touches work
  proportional to the active edges, not to $\lvert V\rvert^2$. The pick-admissible-edge
  populate is the one nondeterministic step — any admissible neighbour is a valid
  choice — and is naturally a per-vertex selection over its admissible residual edges.
complexity_costs: |
  Generic push-relabel runs in $O(\lvert V\rvert^2\,\lvert E\rvert)$ time; the highest-label
  selection rule improves this to $O(\lvert V\rvert^2 \sqrt{\lvert E\rvert})$. Each relabel
  raises a vertex's height by at least one and heights are bounded by $2\lvert V\rvert - 1$,
  bounding the number of relabels at $O(\lvert V\rvert^2)$; saturating and non-saturating
  pushes are bounded separately, with the non-saturating pushes dominating. The
  bulk-synchronous form here trades a larger generation count for fully parallel
  per-generation work.
related_notes: |
  The augmenting-path methods (Ford-Fulkerson, Edmonds-Karp) solve the same maximum-flow
  problem by repeatedly pushing flow along whole source-to-sink paths in the residual
  graph, rather than by local push and relabel. Breadth-first search is the residual-graph
  traversal those augmenting-path methods build on.
---
