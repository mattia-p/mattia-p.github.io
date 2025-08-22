---
title: Home
layout: home
nav_order: 1
---

<div class="articles-hero">
  <h2>Welcome</h2>
</div>

<div class="about-section">
  <div class="about-content">
    <div class="about-text">
      <h2>About Me</h2>
      <p>Hello there. I am an engineer. Started out in mechanical and controls engineering. Over time, moved also into robotics and software. Currently at Zoox, working on safety-critical computers running on self-driving cars.</p>
      <p>On the side, I tinker a bit. Robotics, programming, more self driving... This blog is where I keep notes and share what I find along the way.</p>
    </div>
    <!-- <div class="about-image">
      <img src="/assets/images/profile.jpg" alt="Profile photo">
    </div> -->
  </div>
</div>

## Recent Articles

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