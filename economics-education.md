---
layout: default
title: Economics Education
permalink: /economics-education/
---

<h2>📊 Economics Education</h2>
<p class="page-description">Economics learning series from scratch — covering microeconomics and macroeconomics fundamentals, from basic concepts to advanced theories, with real-world case studies and practical insights.</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "economics-education/" %}
    <li class="post-list-item">
      <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
      <div class="post-list-meta">
        📅 {{ post.date | date: "%Y-%m-%d" }}
        {% if post.categories.size > 0 %}
          &middot; 📁 {{ post.categories | join: ", " }}
        {% endif %}
      </div>
    </li>
    {% endif %}
  {% endfor %}
</ul>
