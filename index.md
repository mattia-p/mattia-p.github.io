---
title: Home
layout: home
nav_order: 1
---

## About 

Welcome!

I am an engineer with a focus on robotics, autonomous systems, safety and controls.
This blog is a collection of notes on projects, learnings or findings that I want to share or archive for later use.

## Recent Articles

{% for post in site.posts %}
  <div class="post-preview">
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
    <p>{{ post.excerpt }}</p>
    <p class="post-tags">
      {% for tag in post.tags %}
        <span class="tag">#{{ tag }}</span>
      {% endfor %}
    </p>
  </div>
  <hr>
{% endfor %}