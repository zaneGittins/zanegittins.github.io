---
layout: default
---
This is the personal blog of [Zane Gittins](/about/), focusing on cybersecurity, blue team operations, and digital forensics.

## Posts {#posts}

<ul>
  {% for post in site.posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: "%b %-d, %Y" }}</li>
  {% endfor %}
</ul>