---
layout: page
icon: fas fa-rocket
order: 0
title: Projects
---

<div class="project-grid">
  {% for p in site.data.projects %}
  <div class="project-card">
    <img class="project-thumb" src="{{ p.image }}" alt="{{ p.title }}" loading="lazy">
    <div class="project-body">
      <h3>{{ p.title }}</h3>
      <p>{{ p.description }}</p>
      <div class="project-tags">
        {% for tag in p.tags %}<span class="project-tag">{{ tag }}</span>{% endfor %}
      </div>
      <div class="project-links">
        <a class="project-btn" href="{{ p.github }}" target="_blank" rel="noopener">
          <i class="fab fa-github"></i> GitHub
        </a>
        {% if p.demo %}
        <a class="project-btn project-btn-primary" href="{{ p.demo }}" target="_blank" rel="noopener">
          <i class="fas fa-arrow-up-right-from-square"></i> Demo
        </a>
        {% endif %}
        {% if p.post %}
        <a class="project-btn" href="{{ p.post | relative_url }}">
          <i class="fas fa-pen"></i> 블로그 글
        </a>
        {% endif %}
      </div>
    </div>
  </div>
  {% endfor %}
</div>

<style>
.project-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.25rem;
  margin-top: 1rem;
}
.project-card {
  border: 1px solid var(--card-border-color, #e9ecef);
  border-radius: 12px;
  overflow: hidden;
  background: var(--card-bg, #fff);
  display: flex;
  flex-direction: column;
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}
.project-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12);
}
.project-thumb {
  width: 100%;
  aspect-ratio: 1200 / 630;
  object-fit: cover;
  display: block;
  background: #f1f3f5;
}
.project-body {
  padding: 1rem 1.1rem 1.2rem;
  display: flex;
  flex-direction: column;
  flex: 1;
}
.project-body h3 {
  margin: 0 0 0.4rem;
  font-size: 1.05rem;
}
.project-body p {
  font-size: 0.88rem;
  color: var(--text-muted-color, #6c757d);
  margin-bottom: 0.8rem;
  flex: 1;
}
.project-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  margin-bottom: 0.9rem;
}
.project-tag {
  font-size: 0.72rem;
  padding: 0.15rem 0.55rem;
  border-radius: 999px;
  background: rgba(79, 70, 229, 0.1);
  color: #4f46e5;
}
.project-links {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
}
.project-btn {
  font-size: 0.82rem;
  padding: 0.35rem 0.75rem;
  border-radius: 8px;
  border: 1px solid var(--card-border-color, #dee2e6);
  color: inherit;
  text-decoration: none;
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
}
.project-btn:hover {
  border-color: #4f46e5;
  color: #4f46e5;
}
.project-btn-primary {
  background: #4f46e5;
  border-color: #4f46e5;
  color: #fff;
}
.project-btn-primary:hover {
  color: #fff;
  opacity: 0.9;
}
</style>
