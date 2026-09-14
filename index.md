---
layout: default
---

## Posts

{% for post in site.posts %}
<article class="post-card">
  <p class="post-date">{{ post.date | date: "%B %-d, %Y" }}</p>
  <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
  <p>{{ post.description }}</p>
  <a class="read-more" href="{{ post.url | relative_url }}">Read the post →</a>
</article>
{% endfor %}

