---
title: Projects
layout: page
permalink: /projects/
nav_order: 3
---

<div class="projects-hero">
  <h1>Projects</h1>
  <p>A collection of projects exploring robotics, autonomous systems, and programming in general.</p>
</div>

<div class="projects-grid">
  {% for project in site.projects %}
    <div class="project-card">
      {% if project.image %}
        <div class="project-image">
          <img src="{{ project.image }}" alt="{{ project.title }}">
        </div>
      {% endif %}
      <div class="project-content">
        <h3><a href="{{ project.url }}">{{ project.title }}</a></h3>
        <p>{{ project.description }}</p>
        {% if project.tech_stack %}
          <div class="tech-stack">
            {% for tech in project.tech_stack %}
              <span class="tech-tag">{{ tech }}</span>
            {% endfor %}
          </div>
        {% endif %}
        <a href="{{ project.url }}" class="project-link">View Project →</a>
      </div>
    </div>
  {% endfor %}
</div>