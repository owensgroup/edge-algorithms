# EDGE Algorithms (Jekyll)

This site hosts EDGE (Extended General Einsums for Graph Algorithms) writeups. Each algorithm lives in a single Markdown file and is rendered with a fixed section structure and MathJax.

## Adding a new algorithm

You only need to add one file: a `.md` file inside `_algorithms/`. You do **not** create an `index.html` for each algorithm. Jekyll generates the pages for you.

### Steps

1. Create a new Markdown file in `_algorithms/`, for example:
   - `_algorithms/triangle-counting.md`
2. Use this front matter + section template:

```markdown
---
title: Triangle Counting
summary: Counts triangles using EDGE.
tags: [triangles, graphs]
status_intro: |
  Short status or overview.
problem_statement: |
  Define the problem.
working_example_part1: |
  Example setup.
working_example_part2: |
  Example continuation.
edge_expression_walkthrough: |
  Explain each line of the EDGE expression.
edge_expression: |
  $$T = \sum_{i,j,k} A_{ij} A_{jk} A_{ki}$$
other_notes: |
  Optional notes.
variants: |
  Optional variants.
implementation_notes: |
  Implementation guidance.
complexity_costs: |
  Time/space complexity.
related_notes: |
  Related algorithms.
---
```

3. Save the file. It will automatically appear on the landing page (`/`).

## Local preview

```bash
bundle install
bundle exec jekyll serve
```

Then open the local URL Jekyll prints (usually `http://127.0.0.1:4000`).

## Notes

- MathJax renders LaTeX. Use `$...$` for inline math and `$$...$$` for display math.
- The algorithm page layout reads the named sections from front matter and renders them in order.
