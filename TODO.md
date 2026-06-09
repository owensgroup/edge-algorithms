# TODO — verification checklist

Things to eyeball / decide before these pages are considered final. Split into
rendering (already fixed, just spot-check) and content (needs your judgment).

## Rendering — fixed, please spot-check

kramdown was mangling single-`$` inline math that contained markdown-special
characters, so several expressions rendered as raw text or got chopped into
stray HTML tables. Fixed by swapping in markdown-safe LaTeX that renders
identically, across `bellman-ford.md`, `breadth-first-search.md`, and
`shortest-path.md`:

- `\{` → `\lbrace`, `\}` → `\rbrace`  (literal `\{`/`\}` were being eaten as escapes)
- `\!` removed  (cosmetic negative-thin-space; kramdown turned it into a literal `!`)
- `|V|` / `|E|` → `\lvert V\rvert` / `\lvert E\rvert`  (bare pipes in a list line triggered a spurious GFM table)

These are visually identical rewrites, but worth a glance to confirm no set
braces or edge lists shifted — especially the `working_example_part2` trace
tables and the tensor-declaration bullets.

Root cause for future pages: **avoid bare `|`, `\{`, `\}`, `\!` inside single-`$`
inline math.** Display `$$...$$` blocks are safe (kramdown protects them).

---

## bellman-ford.md — content

Trace table was independently recomputed and is **correct** (all four rounds).
Expression is a faithful match to the canonical paper cascade. Open items:

- [ ] **`variants` is wrong as written.** It says "drop negative weights and
  replace min with ANY recovers BFS." Recovering BFS also needs the visited
  gate (`¬P`) re-introduced — and the page's own walkthrough stresses BF has
  "no visited gate." As written the two contradict. Reword to include re-adding
  the `¬P` filter, or cut the claim.
- [ ] **Is `<` a builtin compute op, or always a UDF?** Page (and paper) write
  `<` as if primitive; paper is silent; interpreter encodes it as UDF
  `less_than`. Pin this down.
- [ ] **`<<` update operator.** Page uses the canonical paper semantics (correct).
  Interpreter approximates it as `take_right(union)`, which differs in general
  on the (left-present, right-empty) case. Confirm the page is authoritative and
  decide the IR encoding separately.
- [ ] **Whole-tensor `≡` in the stop condition** (`D_{i+1} ≡ D_i`) is used but
  never formally defined as "all points equal." Decide if that needs spelling out.
- [ ] *(optional)* RELAX prose says the intersection merge "keeps the ∞ of
  unreached sources out of the arithmetic." Formally the point is just skipped —
  ∞ never enters compute. Reword only if you want the precise framing.

## breadth-first-search.md — content

Trace recomputed and **correct**; IR artifact is consistent with the page.
Open items:

- [ ] **`working_example_part2` claims final depths come from "the union of the
  frontiers over all generations."** The written Einsum computes no such union —
  depths just live scattered across `F_0…F_3`. Either reword to a descriptive
  statement, or add + declare an explicit `Depth` tensor (changes the spec).
- [ ] **`other_notes` termination framing.** "guaranteed ... rather than by any
  property of the notation itself" undersells the `⋄` stopping condition, which
  *is* the notation-level mechanism. Reword to credit `⋄` for signalling and `P`
  for guaranteeing.
- [ ] **Advance "+1" assumes unit edges.** The walkthrough says a neighbour
  receives `F_{i,s}+1` without restating that every present `G` entry is 1.
  Restate the premise in the walkthrough.
- [ ] *(minor)* The page reuses the source set `id` to seed the `D`-ranked `P`,
  relying on `S ≡ D ≡ |V|`. Fine, but consider a one-line reminder for readers
  who don't have the paper's declaration rules in hand.
- [ ] *(minor)* Stop-condition narrative: note that `F_4` is the frontier the
  i=3 filter produces, so `‖F_4‖ ≡ 0` is what fires.

## shortest-path.md — RESOLVED (pulled 2026-06-09)

- [x] The boilerplate demo skeleton was **removed** from `_algorithms/` before the
  exit seminar so no placeholder text shows in the grid. Recover it from git
  (`git show HEAD:_algorithms/shortest-path.md`) if you want a template reference,
  or author a real min-plus / SSSP-overview page later.

---

# Post exit-seminar review (added 2026-06-09)

Seven new algorithm pages were added so the site reads as populated for the seminar
(**13 total** now). Four are **ports** of EDGE already written under
`edge-ir-interpreter/test-algorithms/`; five are **newly authored** (routed through
the edge-expert / edge-auditor lens, but NOT yet through the full edge-* fleet gate —
review before treating as canonical). Every new page mirrors the `a-star.md` format,
obeys the kramdown rules above, and contains no Claude/AI comments. Hand-traces were
recomputed (several machine-checked) but deserve your eye.

Priority: 🔴 correctness gate · 🟡 confirm a judgment call · 🟢 polish / already-verified.

## Deploy / how to show it
- The site builds to `_site/` and deploys via GitHub Actions on push to `main`
  (`.github/workflows/pages.yml`). I did **not** commit or push — do that to update
  the live site, or run `bundle exec jekyll serve` locally.
- The `shortest-path.md` deletion and the 7 new pages are uncommitted working-tree changes.

## NEW — ports from test-algorithms

### depth-first-search.md  (port; upstream-flagged UNVERIFIED)
- [ ] 🔴 Stamped-stack relies on three IR-unsupported constructs (per `dfs/metadata.md`):
  rank-as-value stamp `σ(i+1,v)` as an operand and `S_0 = σ(0,v)`; the `select-max-val`
  populate (UDF, pending registry); the `<<` push merge (no IR symbol — coincides with
  union only because `T` and `S'` have disjoint support). Confirm each is the sanctioned
  paper-side framing.
- [ ] 🟡 Title "Depth-First Search (Stamped-Stack)" plus the `variants` and complexity
  text were written fresh (not from source) — sanity-check the wording.
- [x] 🟢 Trace: pop order [0,2,3,1], final `P` all-visited — matches metadata ground truth.

### maximum-flow.md  (port; upstream-flagged UNVERIFIED — see `max-flow/toluwa-to-verify.md`)
- [ ] 🔴 Empty value of the height tensor `D`: the height rule and neighbour-height gather
  read `D(v)` through intersection, so an empty(=0) height-0 neighbour is dropped before
  the `min`, and a vertex can be relabelled too high. Wants a non-zero sentinel or union
  reads; needs an evaluator run. (Flagged on the page too.)
- [ ] 🔴 This port has NOT been through edge-expert + edge-auditor (only the 3 newly-authored
  pages were audited this round). Run it through the fleet.
- [ ] 🟡 Push tie-break: trace uses lowest-neighbour-index for "pick one admissible edge";
  per-round push targets vary with the selector (final max-flow = 5 is invariant).
- [ ] Upstream items not on the page (impl detail): source self-loop case order, `Lit_0`
  empty convention, metadata "exact" decl claims, missing program JSON.

### widest-path.md  (port)
- [ ] 🟡 Source program seeds the source bottleneck with literal `999` (empty=0); the page
  renders it as `∞` and explains the `999` stand-in in `other_notes`. Confirm you're happy
  showing `∞`.
- [ ] 🟡 Ranks renamed R/C → S/D to match the `G_{s,d}` house convention (operators preserved).
- [x] 🟢 Trace: D = {0:∞, 1:3, 2:5, 3:3}; widest path to 3 is 0→1→3 (bottleneck 3). Verified.

### connected-components.md  (port; label propagation)
- [ ] 🟡 Worked example uses path 0-1-2 + edge 3-4 (matches label-prop metadata ground truth
  `L=[3,3,3,5,5]`, two components) rather than a triangle, so the label visibly crawls across
  the component. Swap to a triangle if you prefer a one-round convergence.
- [ ] 🟡 Ranks R/C → S/D; type kept `float, empty = 0.0` (int also admissible).

## NEW — authored fresh (audited this round; 2 real bugs fixed during the audit)

### floyd-warshall.md  (audit verdict: CORRECT AS-IS)
- [ ] 🟡 Stop is written bare `⋄ : i ≡ |V|`; the paper's fixed-trip-count precedent uses
  subscripted `⋄_i : i ≡ |V|`. Defensible either way — pick one for consistency.
- [x] 🟢 Audit confirmed: pivot-as-iteration-coordinate `D_{i,u,i}` / `D_{i,i,w}` is legal
  (paper BF precedent, not value-as-coordinate); `+(∩)` / `min(∪)` correct under empty=∞;
  in-place sweep is a sound refinement assuming no negative diagonal. Trace verified vs APSP.

### triangle-counting.md  (audit verdict: FIXED)
- [x] 🟢 Auditor parenthesized the reduce op-1: `(A_{i,k}·¹A_{k,j})_{i,j} ·² A_{i,j}`
  (house style; meaning unchanged — revert to the terser form if you prefer).
- [ ] 🟡 Confirm: single three-operand einsum vs two-step cascade; the "push the mask down"
  claim (does the ODE license reordering the mask ahead of the k-reduce, or is that
  implementation-level?). Mask is spelled `×(∩)` (canonical) vs `←(∩)`.
- [ ] 🟡 Normalization (½ per-vertex, ⅙ global) left as post-processing; `variants` notes the
  once-counted `n<d<s` enumeration form. Decide which to lead with.
- [x] 🟢 Trace: 4 vertices, 1 triangle {0,1,2}, per-vertex [1,1,1,0]; pendant edge (2,3)
  survives the mask with 0 common neighbours (a nice teaching point). Verified.

### spmv.md  (audit verdict: FIXED)
- [x] 🟢 Auditor fixed a real bug: `variants` listed the BFS advance as `(×, ANY)` but printed
  `+(∩)`; corrected to `(+, ANY)` (depth advance) and kept `(AND, OR)` for Boolean reachability
  — now consistent with `breadth-first-search.md` and the paper.
- [x] 🟢 Core einsum `×(∩)` multiply + `+(∪)` reduce verified; the semiring-generalization
  framing checked as accurate (does not overclaim). Trace y = {0:11, 1:2, 2:24, 3:18} verified.

## NEWEST — added in the second pass

### bubble-sort.md  (recovered from the paper's commented-out `cascade:bubblesort`)
- [ ] 🔴 The paper sketch has an off-by-one: it reads the prior pass's final row as
  `Z_{i-1,J,·}` with `J ≡ N-1`, which indexes out of bounds at step `j=J`. The page
  publishes steps `j = 0…J-1` with final row `Z_{i,J-1}` (verified to sort). Confirm `J`
  is a step-count (final row `J-1`) vs. rows `0…J` with a trailing identity row.
- [ ] 🟡 Confirm a single piecewise `Z_{i,j,n}` einsum branching on `n` (n<j, n=j, n=j+1,
  n>j+1) is sanctioned (vs. separate einsums per region); and the shifted reads
  `Z_{i,j-1,j}` / `A_{j+1}` are RVEs, not value-as-coordinate (Floyd-Warshall precedent).
- [ ] 🟡 `empty = 0` works as a sentinel only because the example is all-positive; general
  data needs an out-of-range sentinel. Inner stop spelled `⋄⋄ : j ≡ J` (sketch had `j ≥ J`).
- [x] 🟢 Trace on [3,1,4,2] hand- and Python-verified; ODE clean (pure map/copy, no value-as-coord).

### post-order-dfs.md  (OPEN PROBLEM — grounded in your `2026.06.05.dfs_postorder.ipynb`)
- [ ] 🔴 Post-order is NOT cleanly closed in EDGE; the page states this plainly. The descent
  (PEEK/ADV/MASK/CHILD/DISCOVER/STAMP) and RECORD/STOP are solid (inherit the pre-order page);
  the OPEN piece is the finish-detection gate COUNT → DONE → FINISH → POP.
- [ ] 🔴 "Top has no undiscovered neighbours" is a test on tensor CONTENTS, not a coordinate,
  so it cannot be a case predicate. The page reifies it via a COUNT-reduce then `==0`; confirm
  this is the sanctioned EDGE pattern (your notebook abandoned a `\begin{cases}` version for
  exactly this reason). Also relies on `empty=0` participating in a union compare `==(∪)`.
- [ ] 🟡 Two populates (`max-val-1` / `max-coord-1`) and the `<<` update — the standing
  populate-with-UDF and `<<` open items shared with the pre-order DFS page.
- [x] 🟢 Trace verified two ways: post-order [3,2,1,0] vs. pre-order [0,2,3,1]. The page's
  verdict is honest about the open gate — do NOT present it as closed without edge-expert sign-off.

## Capture the new authored ones as test-algorithms artifacts (post-seminar)
- [ ] Floyd-Warshall, triangle counting, SpMV, bubble sort, and post-order DFS are NOT yet
  under `edge-ir-interpreter/test-algorithms/`. Per the usual workflow, run them through
  edge-expert → edge-auditor → edge-syntax-translator → edge-coder and add
  `einsum.md` / `build_*.py` / `*_program.json` / `metadata.md`. (Post-order DFS only after
  its finish-gate construct is resolved.)

## Minor / pre-existing
- [ ] `a-star.md` still contains a bare `\!` (the kramdown hazard noted at the top) — spot-check it renders.
- [ ] The `bellman-ford.md` and `breadth-first-search.md` content items above are still open.
