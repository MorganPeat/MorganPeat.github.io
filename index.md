---
layout: default
---

<div class="posts">
  {% assign counter = '0' %}
  {% for post in site.posts %}
    {% if post.draft == true %}
    {% else %}
      {% if counter < '10' %}
        {% capture counter %}{{ counter | plus:'1' }}{% endcapture %}
        <article class="post">

          <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

          <div class="entry">
            {{ post.excerpt }}
          </div>

          <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
        </article>
      {% endif %}
    {% endif %}
  {% endfor %}
</div>

<hr>

{% include tags.html %}
