---
title: Articles
layout: page
permalink: /blog/
nav_order: 2
---

<div class="articles-hero">
  <p>Notes on projects, learnings and findings that I want to share or archive for later use.</p>
</div>

<div class="articles-list">
  {% for post in site.posts %}
    <div class="article-card">
      <div class="article-content">
        <div class="article-header">
          {% if post.image %}
            <div class="article-thumbnail">
              <img src="{{ post.image }}" alt="{{ post.title }}">
            </div>
          {% endif %}
          <div class="article-main">
            <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
            <p>{{ post.description }}</p>
          </div>
        </div>
        <div class="article-meta">
          <span class="post-date">{{ post.date | date: "%B %d, %Y" }}</span>
          {% if post.tech_stack %}
            <div class="tech-stack">
              {% for tech in post.tech_stack %}
                <span class="tech-tag">{{ tech }}</span>
              {% endfor %}
            </div>
          {% endif %}
        </div>
        <a href="{{ post.url }}" class="article-link">Read More →</a>
      </div>
    </div>
  {% endfor %}
</div>