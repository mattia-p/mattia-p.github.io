---
title: Articles
layout: page
permalink: /blog/
nav_order: 2
---

## Articles

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