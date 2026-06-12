---
layout: default
---
<section class="hero">
  <h1>代码之外，架构之内</h1>
  <p>在构建高并发系统与智能应用的实践中，记录每一行代码背后的思考</p>
  <p class="hero-subtitle">Java · Spring Boot · 分布式系统 · AI 应用 · 云原生</p>
  <div class="hero-cta">
    <a href="{{ '/about/' | relative_url }}" class="btn btn-primary">
      <i class="fas fa-user"></i> 关于我
    </a>
    <a href="{{ '/archive/' | relative_url }}" class="btn btn-outline">
      <i class="fas fa-archive"></i> 浏览文章
    </a>
  </div>
</section>

<section class="recent-posts">
  <h2 style="font-size: 1.3rem; font-weight: 600; margin-bottom: 24px; display: flex; align-items: center; gap: 8px;">
    <i class="fas fa-pen-fancy" style="color: var(--color-accent);"></i> 最新文章
  </h2>
  <ul class="post-list">
    {% assign sorted_posts = site.posts | sort: "date" | reverse %}
    {% for post in sorted_posts %}
    <li class="post-card">
      <a href="{{ post.url | relative_url }}" class="post-card-title">{{ post.title | escape }}</a>
      <div class="post-card-meta">
        <time datetime="{{ post.date | date_to_xmlschema }}">
          <i class="far fa-calendar-alt"></i> {{ post.date | date: "%Y 年 %m 月 %d 日" }}
        </time>
        {% if post.categories.size > 0 %}
        <span><i class="far fa-folder-open"></i> {{ post.categories | join: " · " }}</span>
        {% endif %}
        <span><i class="far fa-clock"></i> {% include reading-time.html %}</span>
      </div>
      <div class="post-card-excerpt">
        {{ post.excerpt | strip_html | truncate: 200 }}
      </div>
      {% if post.tags.size > 0 %}
      <div class="post-card-tags">
        {% for tag in post.tags %}
        <span class="tag">{{ tag }}</span>
        {% endfor %}
      </div>
      {% endif %}
    </li>
    {% endfor %}
  </ul>
</section>
