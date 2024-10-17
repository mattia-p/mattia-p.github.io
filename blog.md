---
title: Articles
layout: page
permalink: /blog/
---

## Articles

<ul>
{% for post in site.blog %}
  <li>
    <a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: '%B %d, %Y' }}
  </li>
{% endfor %}
</ul>