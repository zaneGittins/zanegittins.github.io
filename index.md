---
layout: default
title: Zane's Blog
---

This is the personal blog for Zane Gittins. 

<ul>
  {% for post in site.posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: "%b %-d, %Y" }}</li>
  {% endfor %}
</ul>