---
layout: default
title: Home
---

<h1>{{ site.title }}</h1>
<p>{{ site.description }}</p>

<h2>🐛 오류 해결</h2>
<ul>
  {% for post in site.posts %}
  {% if post.categories contains "오류해결" %}
  <li>
    <span>{{ post.date | date: "%Y-%m-%d" }}</span> —
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  </li>
  {% endif %}
  {% endfor %}
</ul>

<h2>🛠 개발 작업</h2>
<ul>
  {% for post in site.posts %}
  {% if post.categories contains "개발작업" %}
  <li>
    <span>{{ post.date | date: "%Y-%m-%d" }}</span> —
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
  </li>
  {% endif %}
  {% endfor %}
</ul>
