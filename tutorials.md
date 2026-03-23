---
layout: default
title: Tutorials
permalink: /tutorials/
---

<h2>📖 Tutorials</h2>
<p class="page-description">Step-by-step tutorials for building projects from scratch. Each series walks you through the entire process with detailed explanations — from tools setup to deployment.</p>

<ul class="post-list">
  {% for post in site.posts reversed %}
    {% if post.type == "tutorial" %}
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
