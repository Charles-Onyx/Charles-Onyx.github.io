---
layout: page
title: "文章归档"
---

<div class="archive-section">
  <p style="margin-bottom: 32px; color: var(--color-text-secondary);">
    按时间顺序整理的博客文章，方便快速查找。
  </p>

  {% assign postsByYear = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}
  {% for year in postsByYear %}
  <h2 class="archive-year">{{ year.name }}</h2>
  <ul class="archive-list">
    {% assign postsByMonth = year.items | group_by_exp: "post", "post.date | date: '%B'" %}
    {% for month in postsByMonth %}
    {% for post in month.items %}
    <li class="archive-item">
      <span class="archive-date">{{ post.date | date: "%m / %d" }}</span>
      <a href="{{ post.url | relative_url }}" class="archive-title">{{ post.title }}</a>
    </li>
    {% endfor %}
    {% endfor %}
  </ul>
  {% endfor %}
</div>
