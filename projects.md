---
layout: post
title: "Projects"
author: "Tadej"
permalink: /projects/
---
{% for project in site.data.projects %}
<div>
  <a href="{{ project.url }}"><b>{{ project.title }}</b></a>
  <span>{{ project.description }}</span>
</div>
<hr>
{% endfor %}
