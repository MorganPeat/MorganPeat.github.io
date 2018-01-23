---
layout: default
---

<div class="posts">
  {% assign published_posts = (site.posts | where:'draft','false') %}
  {% for post in published_posts|limit:5 %}
    <article class="post">

      <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

      <div class="entry">
        {{ post.excerpt }}
      </div>

      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
    </article>
  {% endfor %}
</div>

<hr>

{% include tags.html %}
