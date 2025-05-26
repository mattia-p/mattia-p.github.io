---
title: Home
layout: home
---

Welcome!

This website will host some notes on projects, learnings or findings that I want to archive for later use, or share.

## Recent Articles

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}