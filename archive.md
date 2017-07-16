---
layout: page
title: Archive
description: All published posts.
header-img: "img/archive.jpg"
---

<div>
  <ul style="list-style: none;">
    {% for post in site.posts %}
    <li>
      <i class="fa fa-book fa-fw" aria-hidden="true"></i>&nbsp;
      <a href="{{ site.url }}{{ post.url }}" title="{{ post.title }}">
      {{ post.title }}
      </a>
    </li>
    {% endfor %}
  </ul>
</div>
