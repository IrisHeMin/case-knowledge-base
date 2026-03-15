---
layout: default
title: Education
permalink: /education/
---

<h2>📐 Education</h2>
<p class="page-description">Learning resources organized by subject, grade, and semester. Currently featuring Shanghai elementary math curriculum — more subjects coming soon.</p>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "math-education/" %}
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
