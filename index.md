---
layout: default
title: EDGE Algorithms
---

<section class="intro">
  <h1>EDGE Algorithms</h1>
  <p>
    This site presents graph algorithms written in EDGE (Extended General Einsums for Graph Algorithms),
    with structured sections, working examples, and full LaTeX support via MathJax.
  </p>
</section>

<section class="algorithm-grid">
  {% for algo in site.algorithms %}
    <article class="algo-card">
      <h2><a href="{{ algo.url | relative_url }}">{{ algo.title }}</a></h2>
      {% if algo.summary %}
        <p>{{ algo.summary }}</p>
      {% endif %}
      {% if algo.tags %}
        <ul class="tag-list">
          {% for tag in algo.tags %}
            <li>{{ tag }}</li>
          {% endfor %}
        </ul>
      {% endif %}
      <a class="read-more" href="{{ algo.url | relative_url }}">Read algorithm</a>
    </article>
  {% endfor %}
</section>
