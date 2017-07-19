---
title: Morgan Peat's blog
layout: default
---
# Blog posts

  {% for post in site.posts %}
### ({{ post.title }})[{{ post.url }}]
{{ post.excerpt }}
  {% endfor %}
</ul>
