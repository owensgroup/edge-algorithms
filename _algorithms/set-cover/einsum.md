# Exact Cover (Knuth's Algorithm X): corrected cascade

This is the **corrected** sequential cascade for `_algorithms/set_cover.md`. It
is the cascade the interactive walkthrough (`assets/viz/set-cover/`) runs and
checks. The tutorial's `edge_expression` has not been updated yet; this file is
what it should become.

The cascade below is display math, so it renders in GitHub's and VS Code's
Markdown preview. The labels E01–E20 are the step numbers the walkthrough uses.

## Changes from the tutorial

1. **E04 is split into E04 + E05.** The tutorial's
   `TPath = Path · d* :: ⋀ <(∩)` turns path depths into `True`. Run as written,
   the cascade reports {R0, R2, R3} as a solution to Working Example 1, which
   covers C0 twice. `Keep` selects the entries shallower than `d*`, and `TPath`
   keeps their depths. Every later step is renumbered.
2. **`S_0` is an input, `InitS`.** The tutorial gives it in prose. `InitS`
   holds the rows covering the column with the fewest rows, stamped `σ(1, r)`;
   for Working Example 1 that is `{R3: 7}`. `Path_0 = empty` is dropped, since
   every output starts empty.
3. **Every tensor is declared** (the EDGE paper requires it): `Keep`, plus `F`,
   `d*`, `TPath` and `NewPathEntry`, which the tutorial uses without declaring,
   and the inputs `Col`, `Row` (true at every column / row) and `InitS`.
4. **E17's `select-min-val` breaks ties by first arrival.** Any column with the
   fewest active rows is a correct branch. In order, ties go to the lowest
   column, which is what the worked example's table shows.

All other Einsums are exactly as the tutorial writes them.

## Cascade

$$
\begin{aligned}
  &\triangleright\ \text{Tensors} \\
  A^{R,\, C} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  S^{I,\, R} &\to \text{integer},\ \text{empty}=-1 \\
  \mathrm{Path}^{I,\, R} &\to \text{integer},\ \text{empty}=-1 \\
  \mathrm{SelRow}^{I,\, R} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{CovCol}^{I,\, C} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{ActiveCol}^{I,\, C} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{ConflictRow}^{I,\, R} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{ActiveRow}^{I,\, R} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{Success}^{I} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{Sol}^{I,\, R} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{RowCounts}^{I,\, C} &\to \text{integer},\ \text{empty}=0 \\
  \mathrm{ActiveCounts}^{I,\, C} &\to \text{integer},\ \text{empty}=0 \\
  \mathrm{PickedCol}^{I,\, C} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{Candidates}^{I,\, R} &\to \text{Boolean},\ \text{empty}=\text{False} \\
  \mathrm{NewChoices}^{I,\, R} &\to \text{integer},\ \text{empty}=-1 \\
  T^{I,\, R} &\to \text{integer},\ \text{empty}=-1 \\
  \mathrm{Keep}^{I,\, R} &\to \text{Boolean},\ \text{empty}=\text{False} && \text{new} \\
  F^{I,\, R} &\to \text{integer},\ \text{empty}=-1 && \text{was undeclared} \\
  {d^\ast}^{I} &\to \text{integer},\ \text{empty}=0 && \text{was undeclared} \\
  \mathrm{TPath}^{I,\, R} &\to \text{integer},\ \text{empty}=-1 && \text{was undeclared} \\
  \mathrm{NewPathEntry}^{I,\, R} &\to \text{integer},\ \text{empty}=-1 && \text{was undeclared} \\
  \mathrm{Col}^{C},\ \mathrm{Row}^{R} &\to \text{Boolean},\ \text{empty}=\text{False} && \text{input: true at every column / row} \\
  \mathrm{InitS}^{R} &\to \text{integer},\ \text{empty}=-1 && \text{input: the stack seed} \\[6pt]
  &\triangleright\ \text{Stamp function} \\
  \sigma(\mathrm{depth}, r) &= \mathrm{depth} \cdot \lvert R\rvert + r \\[6pt]
  &\triangleright\ \text{Initialization} \\
  S_{0,\, r} &= \mathrm{InitS}_r && \text{was prose} \\[6pt]
  &\triangleright\ \text{Extended Einsum (one pop/backtrack step per iteration } i\text{)} \\
  &\triangleright\ \textbf{peek and pop} \\
  F_{i,r^\ast} &= S_{i,r} :: \lll_{r^\ast} \mathbf{1}(\text{select-max-val}) && \text{E01} \\
  d^\ast_i &= \lfloor F_{i,r} / \lvert R\rvert \rfloor :: \bigvee \max(\cup) && \text{E02} \\
  T_{i,r} &= S_{i,r} \cdot \neg F_{i,r} :: \bigwedge \leftarrow(\cap) && \text{E03} \\
  &\triangleright\ \textbf{update path} \\
  \mathrm{Keep}_{i,r} &= \mathrm{Path}_{i,r} \cdot d^\ast_i :: \bigwedge <(\cap) && \text{E04 (new)} \\
  \mathrm{TPath}_{i,r} &= \mathrm{Path}_{i,r} \cdot \mathrm{Keep}_{i,r} :: \bigwedge \leftarrow(\cap) && \text{E05 (corrected)} \\
  \mathrm{NewPathEntry}_{i,r} &= F_{i,r} \cdot d^\ast_i :: \bigwedge \rightarrow(\cap) && \text{E06} \\
  \mathrm{Path}_{i+1,r} &= \mathrm{TPath}_{i,r} \cdot \mathrm{NewPathEntry}_{i,r} :: \bigwedge \texttt{<<}(\cup) && \text{E07} \\
  &\triangleright\ \textbf{active subset computation} \\
  \mathrm{SelRow}_{i+1,r} &= \mathrm{Path}_{i+1,r} \cdot 0 :: \bigwedge \ge(\cup) && \text{E08} \\
  \mathrm{CovCol}_{i+1,c} &= A_{r,c} \cdot \mathrm{SelRow}_{i+1,r} :: \bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup) && \text{E09} \\
  \mathrm{ActiveCol}_{i+1,c} &= \mathrm{Col}_c \cdot \neg \mathrm{CovCol}_{i+1,c} :: \bigwedge \leftarrow(\cap) && \text{E10} \\
  \mathrm{ConflictRow}_{i+1,r} &= A_{r,c} \cdot \mathrm{CovCol}_{i+1,c} :: \bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup) && \text{E11} \\
  \mathrm{ActiveRow}_{i+1,r} &= \mathrm{Row}_r \cdot \neg \mathrm{ConflictRow}_{i+1,r} \cdot \neg \mathrm{SelRow}_{i+1,r} :: \bigwedge \leftarrow(\cap) && \text{E12} \\
  &\triangleright\ \textbf{success evaluation} \\
  \mathrm{Success}_{i+1} &= \lVert \mathrm{ActiveCol}_{i+1} \rVert \equiv 0 && \text{E13} \\
  \mathrm{Sol}_{i+1,r} &= \mathrm{SelRow}_{i+1,r} \cdot \mathrm{Success}_{i+1} :: \bigwedge \leftarrow(\cap) && \text{E14} \\
  &\triangleright\ \textbf{branch and push (if Success is False)} \\
  \mathrm{RowCounts}_{i+1,c} &= A_{r,c} \cdot \mathrm{ActiveRow}_{i+1,r} :: \bigwedge \leftarrow(\cap)\ \bigvee +(\cup) && \text{E15} \\
  \mathrm{ActiveCounts}_{i+1,c} &= \mathrm{RowCounts}_{i+1,c} \cdot \mathrm{ActiveCol}_{i+1,c} :: \bigwedge \leftarrow(\cap) && \text{E16} \\
  \mathrm{PickedCol}_{i+1,c^\ast} &= \mathrm{ActiveCounts}_{i+1,c} :: \lll_{c^\ast} \mathbf{1}(\text{select-min-val}) && \text{E17 (ties: first arrival)} \\
  \mathrm{Candidates}_{i+1,r} &= (A_{r,c} \cdot \mathrm{PickedCol}_{i+1,c} :: \bigwedge \leftarrow(\cap)\ \bigvee \text{ANY}(\cup)) \cdot \mathrm{ActiveRow}_{i+1,r} :: \bigwedge \leftarrow(\cap) && \text{E18} \\
  \mathrm{NewChoices}_{i+1,r} &= \mathrm{Candidates}_{i+1,r} \cdot \sigma(d^\ast_i + 1, r) :: \bigwedge \rightarrow(\cap) && \text{E19} \\
  S_{i+1,r} &= T_{i,r} \cdot \mathrm{NewChoices}_{i+1,r} :: \bigwedge \texttt{<<}(\cup) && \text{E20} \\[6pt]
  &\diamond : \lVert S_{i+1} \rVert \equiv 0
\end{aligned}
$$

## How the walkthrough runs E02, E12, E13, E18 and E19

These five are written above exactly as the tutorial writes them. Each runs as
one Einsum; where the written form is shorthand, the walkthrough shows it in
the EDGE paper's parenthesized, labelled form and steps through the labels:

$$
\begin{aligned}
  d^\ast_i &= ((F_{i,r} \cdot^1 \lvert R\rvert)_{i,r} \cdot^2 1) :: \bigwedge^1 /(\cap)\ \bigwedge^2 \lfloor\,\rfloor(\leftarrow)\ \bigvee^2_r \max(\cup) && \text{E02: a unary as a Map against 1 (paper §5.5)} \\
  \mathrm{ActiveRow}_{i+1,r} &= (\mathrm{Row}_r \cdot^1 \neg \mathrm{ConflictRow}_{i+1,r})_{i+1,r} \cdot^2 \neg \mathrm{SelRow}_{i+1,r} :: \bigwedge^1 \leftarrow(\cap)\ \bigwedge^2 \leftarrow(\cap) && \text{E12} \\
  \mathrm{Success}_{i+1} &= \lVert \mathrm{ActiveCol}_{i+1} \rVert \cdot^1 0 :: \bigwedge^1 \equiv(\cap) && \text{E13} \\
  \mathrm{Candidates}_{i+1,r} &= (A_{r,c} \cdot^1 \mathrm{PickedCol}_{i+1,c})_{i+1,r} \cdot^2 \mathrm{ActiveRow}_{i+1,r} :: \bigwedge^1 \leftarrow(\cap)\ \bigvee^1_c \text{ANY}(\cup)\ \bigwedge^2 \leftarrow(\cap) && \text{E18} \\
  \mathrm{NewChoices}_{i+1,r} &= \mathrm{Candidates}_{i+1,r} \cdot^4 (((d^\ast_i \cdot^1 1)_i \cdot^2 \lvert R\rvert)_i \cdot^3 r)_{i,r} :: \bigwedge^1 +(\cap)\ \bigwedge^2 *(\cap)\ \bigwedge^3 +(\cap)\ \bigwedge^4 \rightarrow(\cap) && \text{E19: } \sigma(d^\ast_i+1, r)
\end{aligned}
$$

E13 needs occupancy (`‖T‖`) inside an Einsum, which the IR so far allowed only
in stopping conditions; it was added locally in edge-core-ir and is pending
discussion.
