---
layout: default
title: Home
---

<div class="home-intro">
  <h1>{{ site.title }}</h1>
  <p>{{ site.description }}</p>
</div>

{% assign error_posts = site.posts | where_exp: "post", "post.categories contains '오류해결'" %}
{% assign dev_posts = site.posts | where_exp: "post", "post.categories contains '개발작업'" %}

<div class="post-section">
  <h2>🐛 오류 해결</h2>
  {% if error_posts.size > 0 %}
  <ul class="post-card-list">
    {% for post in error_posts %}
    <li class="post-card">
      <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    </li>
    {% endfor %}
  </ul>
  {% else %}
  <p class="empty-state">아직 작성된 글이 없습니다.</p>
  {% endif %}
</div>

<div class="post-section">
  <h2>🛠 개발 작업</h2>
  {% if dev_posts.size > 0 %}
  <ul class="post-card-list">
    {% for post in dev_posts %}
    <li class="post-card">
      <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    </li>
    {% endfor %}
  </ul>
  {% else %}
  <p class="empty-state">아직 작성된 글이 없습니다.</p>
  {% endif %}
</div>
