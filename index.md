---
layout: default
---
This is the personal blog of Zane Gittins, focusing on cybersecurity, blue team operations, and digital forensics. I'm using this blog to document things I've learned for my own reference and with the hope that it will help others.

[About Me](/about/) | [All Posts](#posts)

## Posts {#posts}

<ul>
  {% for post in site.posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: "%b %-d, %Y" }}</li>
  {% endfor %}
</ul>