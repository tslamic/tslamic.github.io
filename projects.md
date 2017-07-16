---
layout: page
title: "Projects"
description: Some stuff I've done.
header-img: "img/projects.jpg"
---

{% for project in site.data.projects %}
<div class="post-preview">
  <a href="{{ project.url }}">
    <h2 class="post-title">{{ project.title }}</h2>
    <h3 class="post-subtitle">{{ project.description }}</h3>
  </a>
  <p class="post-meta">{{ project.category }}</p>
</div>
<hr>
{% endfor %}
