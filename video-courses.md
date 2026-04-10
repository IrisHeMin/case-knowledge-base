---
layout: default
title: Video Courses
permalink: /video-courses/
---

<h2>🎬 Video Courses</h2>
<p class="page-description">Windows 网络技术系列视频课程。以故事驱动的方式，从宏观场景出发，逐步深入每个技术细节。每集 10-15 分钟，通俗易懂，适合入门到进阶。</p>

<div class="series-banner" style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 24px; border-radius: 12px; margin-bottom: 24px;">
  <h3 style="color: white; margin-top: 0;">📺 系列一：从零搭建企业网络 — 小明的网络工程师之旅</h3>
  <p style="margin-bottom: 0;">跟随网络工程师小明，从一根网线开始，逐步搭建起完整的企业网络基础设施。每一集解决一个真实场景问题，串联起 Windows 网络技术的全貌。</p>
</div>

{% assign vc_posts = site.posts | where: "type", "video-course" | sort: "episode" %}

{% if vc_posts.size > 0 %}
<div class="episode-list" style="display: flex; flex-direction: column; gap: 12px;">
  {% for post in vc_posts %}
  <a href="{{ post.url | relative_url }}" class="episode-card" style="display: flex; align-items: center; padding: 16px 20px; background: #f8f9fa; border-radius: 8px; text-decoration: none; color: inherit; border-left: 4px solid {% if post.episode == 0 %}#667eea{% else %}#764ba2{% endif %}; transition: transform 0.2s, box-shadow 0.2s;">
    <span style="font-size: 1.6em; margin-right: 16px; min-width: 48px; text-align: center;">{{ post.episode_icon }}</span>
    <div>
      <div style="font-weight: 600; font-size: 1.05em;">EP{{ post.episode | prepend: '00' | slice: -2, 2 }} · {{ post.title }}</div>
      <div style="color: #666; font-size: 0.9em; margin-top: 4px;">{{ post.description }}</div>
    </div>
  </a>
  {% endfor %}
</div>
{% else %}
<p>Coming soon...</p>
{% endif %}

<style>
.episode-card:hover {
  transform: translateX(4px);
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}
</style>
