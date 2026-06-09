---
title: Shortest Path Faster Algorithm (SPFA)
authors: [Toluwanimi Odemuyiwa]
summary: Single-source shortest paths that relaxes only vertices whose distance recently improved, by wrapping Bellman-Ford's relaxation body in a work queue — sitting exactly between Bellman-Ford (relax everything) and Dijkstra (relax the closest), and tolerating negative edge weights.
tags: [shortest-paths, spfa, bellman-ford, work-queue, graphs]
status_intro: |
  This page gives SPFA, the queue-based form of Bellman-Ford. It keeps Bellman-Ford's
  relaxation body unchanged but, instead of re-relaxing every vertex each round, it
  maintains a work queue and processes only vertices whose tentative distance has just
  improved. It is the middle member of the Bellman-Ford $\to$ SPFA $\to$ Dijkstra family:
  Bellman-Ford relaxes *all* sources each round, SPFA relaxes *some* queued source, and
  Dijkstra relaxes the *closest* queued source. SPFA and Dijkstra share one cascade and
  differ by a single token — the queue selector — as the EDGE expressions below make
  explicit. Like Bellman-Ford, SPFA tolerates negative edge weights (but not negative
  cycles).
problem_statement: |
  Given a directed, weighted graph $G$ on $\lvert V\rvert$ vertices and a source set
  $root\_id$, compute for each vertex the weight of the shortest path from a source.

  The state is the same set of tensors as Dijkstra: a distance tensor carries the answer,
  a Boolean queue tracks which vertices still need processing, alongside the relaxation
  scratch tensors.

  - $G^{S \equiv \lvert V\rvert,\, D \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the graph.
  - $D^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the distances
    ($\infty$ marks "not yet reached").
  - $Q^{I,\, S \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the queue
    membership — which vertices still have work to do.
  - $DQ^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the distances
    of the queued vertices.
  - $F^{I,\, S \equiv \lvert V\rvert} \to \text{integer}$, `empty` $= \infty$, the single
    vertex processed this round, carrying its current distance.
  - $N, C, NewlyRelaxed$, the relaxation scratch tensors (as in Bellman-Ford).
  - $T^{I,\, D \equiv \lvert V\rvert} \to \text{Boolean}$, `empty` $=$ False, the queue with
    the processed vertex removed.
working_example_part1: |
  Take the four-vertex directed graph with edges
  $0\to1\,(10),\ 0\to2\,(1),\ 2\to1\,(1),\ 1\to3\,(1)$ and source vertex $0$ — the same
  graph used on the Dijkstra page, so the two traces can be compared directly.
  Initialization sets the source distance to $0$ and places it in the queue; every other
  vertex starts at $\infty$ and out of the queue:

  $$D_0 = \lbrace 0:0\rbrace, \qquad Q_0 = \lbrace 0\rbrace.$$
working_example_part2: |
  Each round pulls *some* queued vertex (here `select-any` is shown picking the
  lowest-numbered queued vertex, a deliberately unhelpful order), relaxes its out-edges,
  and folds any improvements back into the distances and the queue:

  | $i$ | $Q_i$ (with $g$) | processed $F$ | candidates $N_i$ | improved $C_i$ | $D_{i+1}$ | $Q_{i+1}$ |
  |----|----|----|----|----|----|----|
  | 0 | $\lbrace 0:0\rbrace$ | $0$ | $\lbrace 1:10,\ 2:1\rbrace$ | $\lbrace 1,2\rbrace$ | $\lbrace 0:0,\ 1:10,\ 2:1\rbrace$ | $\lbrace 1,2\rbrace$ |
  | 1 | $\lbrace 1:10,\ 2:1\rbrace$ | $1$ | $\lbrace 3:11\rbrace$ | $\lbrace 3\rbrace$ | $\lbrace 0:0,\ 1:10,\ 2:1,\ 3:11\rbrace$ | $\lbrace 2,3\rbrace$ |
  | 2 | $\lbrace 2:1,\ 3:11\rbrace$ | $2$ | $\lbrace 1:2\rbrace$ | $\lbrace 1\rbrace$ | $\lbrace 0:0,\ 1:2,\ 2:1,\ 3:11\rbrace$ | $\lbrace 1,3\rbrace$ |
  | 3 | $\lbrace 1:2,\ 3:11\rbrace$ | $1$ | $\lbrace 3:3\rbrace$ | $\lbrace 3\rbrace$ | $\lbrace 0:0,\ 1:2,\ 2:1,\ 3:3\rbrace$ | $\lbrace 3\rbrace$ |
  | 4 | $\lbrace 3:3\rbrace$ | $3$ | $\lbrace \rbrace$ | $\lbrace \rbrace$ | unchanged | $\lbrace \rbrace$ — stop |

  Final $D = \lbrace 0:0,\ 1:2,\ 2:1,\ 3:3\rbrace$, the same answer Dijkstra gives. But
  the path there differs: because SPFA processed vertex $1$ (at the stale distance $10$)
  *before* vertex $2$ relaxed it down to $2$, vertex $1$ is processed twice and vertex $3$
  is relaxed twice ($11$, then $3$). Dijkstra, picking the *minimum*-distance queued vertex
  each round, would settle in the order $0, 2, 1, 3$ and process each vertex exactly once.
  That re-relaxation is the price SPFA pays for not maintaining a priority order, and the
  reason a different pick order yields a different (but equally correct) trace.
edge_expression_walkthrough: |
  **Pick.** $DQ_{i,s} = Q_{i,s} \cdot D_{i,s} :: \bigwedge \rightarrow(\cap)$ keeps the
  distance of every still-queued vertex. The populate
  $F_{i,s^\ast} = DQ_{i,s} :: \lll_{s^\ast} \mathbf{1}(\text{select-any-s})$ then selects *any*
  one queued vertex and emits it at its own coordinate, so $F$ is a one-hot tensor holding
  the chosen vertex and its current distance. This single line is the only difference from
  Dijkstra, which selects the *minimum*-distance vertex (`select-min-s`) instead of any.

  **Relax.** $N_{i,d} = G_{s,d} \cdot F_{i,s} :: \bigwedge +(\cap)\ \bigvee \min(\cup)$.
  Exactly as in Bellman-Ford, but the source is the single chosen vertex $F$ rather than
  all of $D$ — so this round only relaxes the out-edges of the vertex just pulled from the
  queue.

  **Improve.** $C_{i,d} = N_{i,d} \cdot D_{i,d} :: \bigwedge <(\cup)$. True exactly where
  the new candidate is strictly smaller than the current distance.

  **Record.** $NewlyRelaxed_{i,d} = C_{i,d} \cdot N_{i,d} :: \bigwedge \rightarrow(\cap)$.
  Keeps the candidate value where the improvement mask is present.

  **Update.** $D_{i+1,d} = D_{i,d} \cdot NewlyRelaxed_{i,d} :: \bigwedge \texttt{<<}(\cup)$.
  Improved vertices take their new distance; every other vertex keeps its current one.

  **Dequeue.** $T_{i,d} = Q_{i,d} \cdot F_{i,d} :: \bigwedge \leftarrow(\text{take\_left\_only})$.
  The take-left-only merge keeps a queue member exactly where the processed vertex $F$ is
  *absent*, removing the just-processed vertex from the queue. (This is the occupancy-level
  form of $Q \cdot \neg F$, and it does not require negating the integer-valued $F$.)

  **Enqueue.** $Q_{i+1,d} = T_{i,d} \cdot C_{i,d} :: \bigwedge \text{OR}(\cup)$. The next
  queue is the surviving members together with every vertex that just improved — so a
  vertex re-enters the queue whenever a shorter path to it is found, which is what allows
  re-relaxation.

  **Stop.** $\diamond : \lVert Q_{i+1} \rVert \equiv 0$. The cascade halts when the queue is
  empty: no vertex has pending improvements to propagate.
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
  &\triangleright \textbf{Extended Einsum (one queued vertex processed per iteration } i)\\
  &DQ_{i,s} = Q_{i,s} \cdot D_{i,s} :: \textstyle\bigwedge \rightarrow(\cap)\\
  &F_{i,s^\ast} = DQ_{i,s} :: \textstyle\lll_{s^\ast} \mathbf{1}(\text{select-any-s})\\
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
  **Relationship to Dijkstra.** The two cascades are identical except for the pick: SPFA's
  $\mathbf{1}(\text{select-any-s})$ becomes Dijkstra's $\mathbf{1}(\text{select-min-s})$.
  Under non-negative weights, always pulling the minimum-distance vertex guarantees its
  distance is already final, so Dijkstra processes each vertex once. SPFA makes no such
  guarantee and may re-process a vertex when a shorter path to it appears later — which is
  exactly why it still works when some weights are negative.

  **Nondeterminism.** `select-any-s` does not fix *which* queued vertex is pulled, so the
  per-iteration trace ($Q$, $D$ at intermediate $i$) is not unique. The *converged* $D$ is
  unique on graphs without negative cycles, so the final answer is deterministic even
  though the schedule is not. An implementation typically realizes the queue as a FIFO.

  **Negative cycles.** SPFA inherits Bellman-Ford's tolerance of negative edge weights but,
  like it, does not converge in their presence: on a negative cycle the improvement mask
  keeps firing, vertices keep re-entering the queue, and $\lVert Q\rVert$ never reaches $0$.
  The queue-empty stop signals "no more improvements," not "negative cycle detected."
variants: |
  Replacing the pick's `select-any-s` with `select-min-s` — pull the closest queued vertex
  instead of any — gives Dijkstra, which then processes each vertex exactly once but
  requires non-negative weights. Going the other way, removing the queue entirely and
  relaxing every vertex each round (replacing the pick with $N_{i,d} = G_{s,d} \cdot D_{i,s}$)
  recovers Bellman-Ford. The three are one skeleton under different source-selection rules.
implementation_notes: |
  The queue $Q$ is an unordered work-list (a FIFO suffices), not the priority queue Dijkstra
  needs, so enqueue and dequeue are $O(1)$ rather than $O(\log \lvert V\rvert)$ — SPFA trades
  Dijkstra's heap for possible re-processing. The pick is a select over queued distances; the
  relax is a sparse product over the chosen vertex's out-edges; the remaining steps are
  elementwise merges over the destination rank. Keeping $Q$, $F$, and the distances sparse
  means each iteration touches work proportional to the processed vertex's out-degree.
complexity_costs: |
  Each relaxation can re-enqueue a vertex, so the worst case matches Bellman-Ford at
  $O(\lvert V\rvert \cdot \lvert E\rvert)$. In practice SPFA is often far faster than that
  bound because only vertices with pending improvements are processed, though adversarial
  graphs can force the full $O(\lvert V\rvert \cdot \lvert E\rvert)$ work.
related_notes: |
  Bellman-Ford is the no-queue ancestor that relaxes every vertex each round. Dijkstra is
  the same cascade with the queue made a priority queue (`select-min-s`), processing each
  vertex once under non-negative weights. A* adds a goal-directed heuristic to Dijkstra's
  pick.
---
