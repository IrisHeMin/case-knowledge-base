---
layout: default
title: "Scenario Maps: TCP/IP Packet Flow"
permalink: /scenario-maps/tcpip-packet/
---

<h2>📦 TCP/IP: Packet Flow</h2>
<p class="page-description">Troubleshooting maps for connection resets, packet drops (middle device, WFP), and packet modification.</p>

<a href="{{ '/scenario-maps/' | relative_url }}" class="back-link">&larr; Back to Scenario Maps</a>

<ul class="post-list">
  {% for post in site.posts %}
    {% if post.path contains "scenario-maps/" %}
      {% if post.path contains "tcpip-connection" or post.path contains "tcpip-packet" %}
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
