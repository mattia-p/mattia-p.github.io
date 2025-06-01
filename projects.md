---
title: Projects
layout: page
permalink: /projects/
nav_order: 3
---

## Projects
<!-- 
{% for project in site.projects %}
  <div class="post-preview">
    <h2><a href="{{ project.url }}">{{ project.title }}</a></h2>
    <p>{{ project.description }}</p>
    {% if project.image %}
      <img src="{{ project.image }}" alt="{{ project.title }}" style="max-width: 40%; height: auto; margin: 1em 0;">
    {% endif %}
  </div>
  <hr>
{% endfor %} -->

{% assign sorted_projects = site.projects | sort: "order" %}

{% for project in sorted_projects %}
  <div class="post-preview">
    <h2><a href="{{ project.url }}">{{ project.title }}</a></h2>
    <p>{{ project.description }}</p>
    {% if project.image %}
      <img src="{{ project.image }}" alt="{{ project.title }}" style="max-width: 40%; height: auto; margin: 1em 0;">
    {% endif %}
  </div>
  <hr>
{% endfor %}
