---
layout: default
title: "Scenario Maps: TCP/IP Configuration"
permalink: /scenario-maps/tcpip-config/
---

<h2>⚙️ TCP/IP: Configuration</h2>
<p class="page-description">Troubleshooting maps for adapter missing, IP not assigned, and default gateway issues.</p>

<a href="{{ '/scenario-maps/' | relative_url }}" class="back-link">&larr; Back to Scenario Maps</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "scenario-maps/" %}
      {% if post.path contains "tcpip-adapter" or post.path contains "tcpip-ip-missing" or post.path contains "tcpip-default-gw" %}
      <li class="post-list-item">
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        <div class="post-list-meta">
          📅 {{ post.date | date: "%Y-%m-%d" }}
          {% if post.categories.size > 0 %}&middot; 📁 {{ post.categories | join: ", " }}{% endif %}
        </div>
        {% if post.excerpt %}<p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>{% endif %}
      </li>
      {% endif %}
    {% endif %}
  {% endfor %}
</ul>
