---
title: Betweenness Centrality (Brandes's Algorithm)
authors: [Toluwanimi Odemuyiwa, John Owens]
summary: A complete tensor algebraic derivation and EDGE cascade showing how Brandes's algorithm eliminates explicit target enumeration by factorizing the classical $O(|V|^3)$ 3-index betweenness centrality contraction into an $O(|V||E|)$ Two-Phase Forward-Backward Einsum Cascade over shortest-path DAGs.
tags: [betweenness-centrality, brandes, graphs, einsum, tensor-algebra, shortest-paths, adjoint-state]
status_intro: |
  **Betweenness Centrality (BC)** is one of the most widely used metrics for network analysis, identifying critical communication bottlenecks, central hubs, and vulnerable bridges in social, transportation, biological, and telecommunication networks.

  Historically, computing exact Betweenness Centrality required evaluating all shortest paths between every pair of vertices $(s, t)$ and checking if an intermediate vertex $v$ lies on those paths. In tensor terms, this corresponds to contracting a dense **3-index tensor** $\mathcal{T} \in \mathbb{R}^{|V| \times |V| \times |V|}$, requiring $O(|V|^3)$ operations and $O(|V|^2)$ storage.

  In 2001, Ulrik Brandes introduced a breakthrough algorithm that reduced this complexity to $O(|V| |E|)$ for unweighted graphs (and $O(|V| |E| + |V|^2 \log |V|)$ for weighted graphs) while reducing auxiliary memory to $O(|V| + |E|)$.

  In the EDGE framework, Brandes's algorithm is formulated as a general **Two-Phase Forward-Backward Einsum Cascade over Directed Acyclic Graphs (DAGs)**. This page derives the transformation from the classical 3-index contraction to the 2-phase tensor cascade, walks through step-by-step traces, presents 10 problem applications, and examines algorithmic suboptimalities in existing software implementations.
problem_statement: |
  Let $G = (V, E)$ be a directed or undirected graph with vertices $V$ and edges $E$. Let $\sigma_{st}$ denote the total number of shortest paths between source $s$ and target $t$ (with $\sigma_{ss} = 1$). Let $\sigma_{st}(v)$ denote the number of shortest paths from $s$ to $t$ that pass through an intermediate vertex $v \in V \setminus \{s, t\}$.

  The **Betweenness Centrality** $C_B(v)$ of vertex $v$ is defined as:

  $$C_B(v) = \sum_{s \neq v \neq t \in V} \frac{\sigma_{st}(v)}{\sigma_{st}}$$

  ### The 3-Index Contraction Bottleneck

  Define the 3-index pair-dependency tensor $\mathcal{T} \in \mathbb{R}^{|V| \times |V| \times |V|}$:

  $$\mathcal{T}_{s, v, t} = \begin{cases} \frac{\sigma_{st}(v)}{\sigma_{st}} & \text{if } s \neq v \neq t \text{ and } \sigma_{st} > 0 \\ 0 & \text{otherwise} \end{cases}$$

  In standard Einsum notation, $C_B(v)$ contracts over all pairs $(s, t)$:

  $$C_B(v) = \mathcal{T}_{s, v, t} \cdot \mathbf{1}_{s} \cdot \mathbf{1}_{t} :: \bigwedge_{s, t} \leftarrow(\cap) \bigvee_{s, t} +(\cup)$$

  Computing $\mathcal{T}_{s, v, t}$ requires enumerating all triples $(s, v, t)$, resulting in $\Theta(|V|^3)$ operations.

  ### Brandes's Algebraic Insight

  Brandes proved that on any single-source shortest-path DAG rooted at $s$, the explicit target index $t$ can be eliminated by defining the **single-source dependency** $\delta_{s\bullet}(v) = \sum_{t \in V} \mathcal{T}_{s, v, t}$, which satisfies the exact reverse topological pull recurrence:

  $$\delta_{s\bullet}(v) = \sum_{w \in \text{Succ}_s(v)} \frac{\sigma_{sv}}{\sigma_{sw}} \left( 1 + \delta_{s\bullet}(w) \right)$$

  where $\text{Succ}_s(v) = \{w \in V \mid (v, w) \in E \text{ and } d(s, w) = d(s, v) + \text{weight}(v, w)\}$.
working_example_part1: |
  ### Trace 1: 4-Node Diamond Graph (Fractional Paths $\sigma=2, \delta=0.5$)

  Consider an unweighted diamond graph with vertices $V = \{0, 1, 2, 3\}$ and edges:
  $$(0, 1), (0, 2), (1, 3), (2, 3)$$

  We evaluate Brandes's algorithm from source $s = 0$.

  #### 1. Forward Pass (BFS Discovery from $s = 0$)
  - **Distance and Path Count Discovery:**
    - Level 0: $d(0, 0) = 0$, $\sigma_{0, 0} = 1$.
    - Level 1: $d(0, 1) = 1, \sigma_{0, 1} = 1$; $d(0, 2) = 1, \sigma_{0, 2} = 1$.
    - Level 2: Node 3 has two shortest-path predecessors ($1$ and $2$).
      $$d(0, 3) = 2, \quad \sigma_{0, 3} = \sigma_{0, 1} + \sigma_{0, 2} = 1 + 1 = 2$$
  - **Successor Sets:** $\text{Succ}_0(0) = \{1, 2\}$, $\text{Succ}_0(1) = \{3\}$, $\text{Succ}_0(2) = \{3\}$, $\text{Succ}_0(3) = \emptyset$.

  #### 2. Backward Adjoint Pass (Reverse Topological Order: $3 \to 1, 2 \to 0$)
  - Initialize dependencies: $\delta_{0\bullet} = [0, 0, 0, 0]$.
  - **Node 3 (Leaf):** $\text{Succ}_0(3) = \emptyset \implies \delta_{0\bullet}(3) = 0$.
  - **Node 1:** $\text{Succ}_0(1) = \{3\}$.
    $$\delta_{0\bullet}(1) = \frac{\sigma_{0, 1}}{\sigma_{0, 3}} (1 + \delta_{0\bullet}(3)) = \frac{1}{2} (1 + 0) = 0.5$$
  - **Node 2:** $\text{Succ}_0(2) = \{3\}$.
    $$\delta_{0\bullet}(2) = \frac{\sigma_{0, 2}}{\sigma_{0, 3}} (1 + \delta_{0\bullet}(3)) = \frac{1}{2} (1 + 0) = 0.5$$
  - **Node 0 (Source):** $\text{Succ}_0(0) = \{1, 2\}$.
    $$\delta_{0\bullet}(0) = \frac{\sigma_{0, 0}}{\sigma_{0, 1}} (1 + 0.5) + \frac{\sigma_{0, 0}}{\sigma_{0, 2}} (1 + 0.5) = \frac{1}{1}(1.5) + \frac{1}{1}(1.5) = 3.0$$

  By symmetry across all 4 sources, the total betweenness scores are:
  $$C_B(0) = 0, \quad C_B(1) = 1.0, \quad C_B(2) = 1.0, \quad C_B(3) = 0$$
working_example_part2: |
  ### Trace 2: 5-Node Bridge Graph (Bottleneck Articulation)

  Consider a 5-node bridge graph with vertices $V = \{0, 1, 2, 3, 4\}$ and edges:
  $$(0, 1), (1, 2), (2, 3), (3, 4)$$

  Evaluating from source $s = 0$:
  - **Distances:** $d(0, v) = [0, 1, 2, 3, 4]$.
  - **Path Counts:** $\sigma_{0, v} = [1, 1, 1, 1, 1]$.
  - **Successor Structure:** Each node $v < 4$ has unique successor $v+1$.

  #### Backward Adjoint Accumulation ($4 \to 3 \to 2 \to 1 \to 0$):
  - $\delta_{0\bullet}(4) = 0$
  - $\delta_{0\bullet}(3) = \frac{1}{1}(1 + 0) = 1.0$ (serves target $\{4\}$)
  - $\delta_{0\bullet}(2) = \frac{1}{1}(1 + 1.0) = 2.0$ (serves targets $\{3, 4\}$)
  - $\delta_{0\bullet}(1) = \frac{1}{1}(1 + 2.0) = 3.0$ (serves targets $\{2, 3, 4\}$)
  - $\delta_{0\bullet}(0) = \frac{1}{1}(1 + 3.0) = 4.0$

  Summing over all 5 sources gives the exact analytical betweenness centrality vector:
  $$C_B = [0.0, \ 6.0, \ 8.0, \ 6.0, \ 0.0]$$
  Node 2 is correctly identified as the primary bottleneck of the network.
edge_expression_walkthrough: |
  The generalized Brandes algorithm operates across three consecutive algebraic phases:

  1. **Phase 1: Forward Primal Discovery (BFS / Dijkstra over Semirings)**
     - Computes shortest-path distances $d_{s, v}$ and extracts the active DAG transition tensor:
       $$A^{\text{DAG}}_{s, u, v} = A_{u, v} \cdot \mathbf{1}(d_{s, v} \equiv d_{s, u} + w_{u, v}) :: \bigwedge \times(\cap)$$
     - Accumulates path counts $\sigma_{s, v}$ along topological BFS levels:
       $$\sigma_{s, v} = A^{\text{DAG}}_{s, u, v} \cdot \sigma_{s, u} :: \bigwedge \times(\cap) \bigvee +(\cup)$$

  2. **Phase 2: Backward Adjoint Accumulation (Reverse Topological Dependency Sweep)**
     - Constructs the normalized edge transfer tensor:
       $$W_{s, v, w} = A^{\text{DAG}}_{s, v, w} \cdot \left(\frac{\sigma_{s, v}}{\sigma_{s, w}}\right) :: \bigwedge \times(\cap)$$
     - Propagates single-source dependencies $\delta_{s\bullet}(v)$ in reverse topological order:
       $$\delta_{s, v} = W_{s, v, w} \cdot (1 + \delta_{s, w}) :: \bigwedge \times(\cap) \bigvee +(\cup)$$

  3. **Phase 3: Source Reduction**
     - Contracts the 2D dependency field across all sources $s \neq v$:
       $$C_B(v) = \delta_{s, v} \cdot \mathbf{1}(s \not\equiv v) :: \bigwedge_{s} \leftarrow(\cap) \bigvee_{s} +(\cup)$$
edge_expression: |
  $$
  \begin{aligned}
    &\triangleright\ \textbf{Tensors} \\
    &A^{V,\, V} \to \text{Boolean},\ \text{adjacency matrix} \\
    &w^{V,\, V} \to \text{Float},\ \text{edge weights (unweighted: } w_{u, v} = 1\text{)} \\
    &d^{S,\, V} \to \text{Float},\ \text{shortest path distances from source } s \\
    &A^{\text{DAG},\, S,\, V,\, V} \to \text{Boolean},\ \text{single-source shortest path DAG adjacency} \\
    &\sigma^{S,\, V} \to \text{Float},\ \text{number of shortest paths from } s \text{ to } v \\
    &W^{S,\, V,\, V} \to \text{Float},\ \text{normalized backward dependency weights} \\
    &\delta^{S,\, V} \to \text{Float},\ \text{backward pair dependency accumulator} \\
    &C_B^{V} \to \text{Float},\ \text{output node betweenness centrality vector} \\[8pt]
    &\triangleright\ \textbf{Phase 1: Forward Primal Discovery (BFS / Dijkstra)} \\
    &d_{s,\, v} = \text{SSSP}(A,\, w,\, s) \\
    &A^{\text{DAG}}_{s,\, u,\, v} = A_{u,\, v} \cdot \mathbf{1}(d_{s,\, v} \equiv d_{s,\, u} + w_{u,\, v}) :: \bigwedge \times(\cap) \\
    &\sigma_{s,\, v} = A^{\text{DAG}}_{s,\, u,\, v} \cdot \sigma_{s,\, u} :: \bigwedge \times(\cap) \bigvee +(\cup) \quad (\text{with } \sigma_{s,\, s} = 1.0) \\[8pt]
    &\triangleright\ \textbf{Phase 2: Backward Adjoint Accumulation (Reverse Topological Sweep)} \\
    &W_{s,\, v,\, w} = A^{\text{DAG}}_{s,\, v,\, w} \cdot \left(\frac{\sigma_{s,\, v}}{\sigma_{s,\, w}}\right) :: \bigwedge \times(\cap) \\
    &\delta_{s,\, v} = W_{s,\, v,\, w} \cdot (1 + \delta_{s,\, w}) :: \bigwedge \times(\cap) \bigvee +(\cup) \\[8pt]
    &\triangleright\ \textbf{Phase 3: Source Reduction} \\
    &C_B(v) = \delta_{s,\, v} \cdot \mathbf{1}(s \not\equiv v) :: \bigwedge_{s} \leftarrow(\cap) \bigvee_{s} +(\cup)
  \end{aligned}
  $$
variants: |
  ### 10 Graph Algorithm Applications of the Forward-Backward Einsum Template

  #### Previously Derived / Published Algorithms
  1. **[Girvan-Newman Community Detection](https://en.wikipedia.org/wiki/Girvan%E2%80%93Newman_algorithm) ([DOI: 10.1073/pnas.122653799](https://doi.org/10.1073/pnas.122653799), [DOI: 10.1016/j.socnet.2007.11.001](https://doi.org/10.1016/j.socnet.2007.11.001)):** Accumulates edge dependencies $\delta_{s\bullet}(v, w) = \frac{\sigma_{sv}}{\sigma_{sw}}(1 + \delta_{s\bullet}(w))$ to partition networks hierarchically.
  2. **[Stress Centrality](https://en.wikipedia.org/wiki/Centrality#Stress_centrality) ([DOI: 10.1007/BF02476438](https://doi.org/10.1007/BF02476438), [DOI: 10.1016/j.socnet.2007.11.001](https://doi.org/10.1016/j.socnet.2007.11.001)):** Accumulates unnormalized path volume $\delta_{s\bullet}^{\text{Stress}}(v) = \sum_{w} (\sigma_{sv} + \delta_{s\bullet}^{\text{Stress}}(w))$.
  3. **Load Centrality ([DOI: 10.1103/PhysRevLett.87.278701](https://doi.org/10.1103/PhysRevLett.87.278701), [DOI: 10.1016/j.socnet.2007.11.001](https://doi.org/10.1016/j.socnet.2007.11.001)):** Divides commodity load equally among outbound branches: $\delta_{s\bullet}^{\text{Load}}(v) = \sum_w \frac{1}{\text{deg}^+(v)}(1 + \delta_{s\bullet}^{\text{Load}}(w))$.
  4. **Bounded-Distance & Decay Centrality ([DOI: 10.1016/j.socnet.2007.11.001](https://doi.org/10.1016/j.socnet.2007.11.001), [DOI: 10.1103/PhysRevLett.105.038701](https://doi.org/10.1103/PhysRevLett.105.038701)):** Truncates BFS at horizon $k$ and applies geometric decay $\alpha^d$.

  #### Opportunities for Exploration in EDGE
  5. **[Network Vitality & Replacement Path Bounds](https://en.wikipedia.org/wiki/Closeness_centrality#Vitality) ([DOI: 10.1007/978-3-540-31955-9_3](https://doi.org/10.1007/978-3-540-31955-9_3), [DOI: 10.1109/SFCS.2001.959902](https://doi.org/10.1109/SFCS.2001.959902), [DOI: 10.1137/120894567](https://doi.org/10.1137/120894567)):** Forward subtree discovery and backward adjoint failure accumulation to compute exact bridge vitalities and detour bounds without brute-force $O(|V|^2 |E|)$ deletion ([Brandes & Erlebach, 2005](https://doi.org/10.1007/978-3-540-31955-9_3); [Nardelli et al., 2001](https://doi.org/10.1016/S0304-3975%2802%2900438-3); [Demetrescu et al., 2008](https://doi.org/10.1137/S0097539705429847); [Grandoni & Williams, 2020](https://doi.org/10.1137/120894567)).
  6. **Reverse-Mode Sensitivities on Computation DAGs ([DOI: 10.1038/323533a0](https://doi.org/10.1038/323533a0), [arXiv:1802.03676](https://arxiv.org/abs/1802.03676), [DOI: 10.18653/v1/2020.acl-main.443](https://doi.org/10.18653/v1/2020.acl-main.443)):** Unifying graph centrality and GNN backpropagation over the Log-Sum-Exp / Gibbs semiring ([Fan et al., 2020](https://doi.org/10.1145/3357384.3357979); [Berthet et al., 2020](https://arxiv.org/abs/2002.08670); [Rush, 2020](https://doi.org/10.18653/v1/2020.acl-main.443)).
  7. **Dynamic & Streaming Centrality ([DOI: 10.1109/TPDS.2013.111](https://doi.org/10.1109/TPDS.2013.111), [DOI: 10.1016/j.jocs.2013.08.007](https://doi.org/10.1016/j.jocs.2013.08.007), [DOI: 10.1145/2555243.2555263](https://doi.org/10.1145/2555243.2555263)):** Incremental Forward-Backward Einsums using sparse cone masks $M^{\text{fwd}}, M^{\text{bwd}}$ to maintain centrality in $O(|\Delta V| |\Delta E|)$ time ([Lee et al., 2013](https://doi.org/10.1109/TPDS.2013.111); [Jamour et al., 2014](https://doi.org/10.1145/2555243.2555263); [Nasre et al., 2014](https://doi.org/10.1007/978-3-662-44465-8_49); [Bergamini et al., 2015](https://doi.org/10.1007/978-3-662-48350-3_12)).
  8. **Group Betweenness Centrality ([DOI: 10.1016/S0378-8733(99)00013-1](https://doi.org/10.1016/S0378-8733%2899%2900013-1), [DOI: 10.1109/TKDE.2007.190671](https://doi.org/10.1109/TKDE.2007.190671), [DOI: 10.1145/2939672.2939832](https://doi.org/10.1145/2939672.2939832), [DOI: 10.1137/1.9781611975055.18](https://doi.org/10.1137/1.9781611975055.18)):** Residual-Path Forward-Backward Einsum evaluating all candidate marginal gains $\Delta(v \mid S)$ in parallel in $O(k |V| |E|)$ time ([Puzis et al., 2007](https://doi.org/10.1103/PhysRevE.76.056709); [Dolev et al., 2009](https://doi.org/10.1016/j.ipl.2009.07.019); [Angriman et al., 2021](https://doi.org/10.1145/3447548.3467368); [Davis, 2019](https://doi.org/10.1145/3322125)).
  9. **Multi-Criteria / Pareto-Optimal Centrality ([DOI: 10.1007/3-540-27659-9](https://doi.org/10.1007/3-540-27659-9), [DOI: 10.1016/0377-2217(84)90077-8](https://doi.org/10.1016/0377-2217%2884%2990077-8), [DOI: 10.1007/978-0-387-75450-5](https://doi.org/10.1007/978-0-387-75450-5)):** State-layered MODAG adjoint propagation over Pareto Vector Dioids ([Hansen, 1980](https://doi.org/10.1007/978-3-642-48782-8_9); [Martins, 1984](https://doi.org/10.1016/0377-2217%2884%2990077-8); [Skriver & Andersen, 2000](https://doi.org/10.1016/S0305-0548%2899%2900037-4); [Machuca et al., 2012](https://doi.org/10.1016/j.ipl.2012.04.010); [Zhang et al., 2018](https://doi.org/10.1016/j.physa.2018.05.109)).
  10. **Probabilistic Cascading Failure Sensitivity ([DOI: 10.1038/nature08932](https://doi.org/10.1038/nature08932), [DOI: 10.1016/j.physa.2014.08.016](https://doi.org/10.1016/j.physa.2014.08.016), [DOI: 10.1016/S0951-8320(00)00077-6](https://doi.org/10.1016/S0951-8320%2800%2900077-6), [DOI: 10.1109/CDC.2011.6160415](https://doi.org/10.1109/CDC.2011.6160415)):** Stochastic forward survival propagation and backward adjoint sensitivity to compute exact Birnbaum node/edge hardening gradients in $O(|V| |E|)$ time ([Motter & Lai, 2002](https://doi.org/10.1103/PhysRevE.66.065102); [Dobson et al., 2007](https://doi.org/10.1063/1.2737822); [Bienstock, 2011](https://doi.org/10.1109/CDC.2011.6160415); [Kempe et al., 2003](https://doi.org/10.1145/956750.956769); [Borgs et al., 2014](https://doi.org/10.1137/1.9781611973402.70); [Tang et al., 2014](https://doi.org/10.1145/2588555.2593670); [Griewank & Walther, 2008](https://doi.org/10.1137/1.9780898717761)).
implementation_notes: |
  ### Algorithmic Suboptimality in Existing Software Implementations

  | Software Routine | Library | Algorithmic Suboptimality in Existing Code | Brandes Factorization Remedy | Algorithmic Impact |
  |---|---|---|---|:---:|
  | `closeness_vitality` | **NetworkX** | Brute-force outer node deletion ($O(|V|^2 |E|)$); discards shared DAG structures | Forward subtree discovery + backward adjoint failure accumulation | **$O(|V|)$ Asymptotic Speedup** ($O(|V| |E|)$ total) |
  | `group_betweenness_centrality` | **NetworkX** | Re-runs full Brandes traversals per candidate in greedy selection ($O(k |V|^2 |E|)$) | Residual-path backward adjoint recurrence evaluating all candidates simultaneously | **$|V|$-Fold Reduction in Graph Traversals** ($O(k |V| |E|)$ total) |
  | `GroupBetweenness` | **NetworKit** | Iterative single-source candidate traversals per greedy round | Batched residual dependency sweep over candidate set | **$|V|$-Fold Reduction in Graph Sweeps** |
  | `igraph_community_edge_betweenness` | **igraph** | Recomputes full-graph edge betweenness from scratch on every edge cut ($O(|E|^2 |V|)$) | Localized incremental adjoint sweep restricted to affected DAG cone | **Reduces step from $O(|V| |E|)$ to $O(|\Delta V| |\Delta E|)$** |
  | `DynBetweenness` | **NetworKit** | Re-traverses entire BFS trees from affected sources | Exploits linear adjoint difference $\Delta \delta_{s\bullet} = \delta^{\text{new}} - \delta^{\text{old}}$ on masked cones | **Eliminates redundant search tree re-traversals** |
  | `load_centrality` | **NetworkX** | Custom ad-hoc path tracking accumulators | Maps directly to Brandes backward recurrence with uniform branching weights | **Unified single-pass closed-form evaluation** |
complexity_costs: |
  - **Classical Contraction:** Constructing and contracting the 3-index pair-dependency tensor $\mathcal{T}_{s, v, t} \implies O(|V|^3)$ time and $O(|V|^2)$ storage.
  - **Brandes Forward-Backward Einsum (Unweighted):** For each source $s \in V$, forward BFS takes $O(|V| + |E|)$ and backward topological accumulation takes $O(|V| + |E|) \implies \mathbf{O(|V| |E|)}$ total time and $\mathbf{O(|V| + |E|)}$ auxiliary memory.
  - **Brandes Forward-Backward Einsum (Weighted):** For each source $s \in V$, forward Dijkstra takes $O(|E| + |V| \log |V|)$ and backward sweep takes $O(|V| + |E|) \implies \mathbf{O(|V| |E| + |V|^2 \log |V|)}$ total time.
related_notes: |
  - [Breadth-First Search (BFS)](breadth-first-search.html)
  - [Dijkstra's Algorithm](dijkstra.html)
  - [Bellman-Ford Algorithm](bellman-ford.html)
  - [Floyd-Warshall All-Pairs Shortest Paths](floyd-warshall.html)
---
