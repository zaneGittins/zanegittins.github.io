---
layout: default
---
This is the personal blog of Zane Gittins, focusing on cybersecurity, blue team operations, and digital forensics.

<ul>
  {% for post in site.posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: "%b %-d, %Y" }}</li>
  {% endfor %}
</ul>