---
layout: default
title: Blog Posts
---

# Blog Posts

{% for post in site.posts %}
  <article style="margin-bottom: 2rem; padding-bottom: 1rem; border-bottom: 1px solid #ccc;">
    <h2><a href="{{ post.url | relative_url }}" style="text-decoration: none;">{{ post.title }}</a></h2>
    <time style="color: #666; font-size: 0.9em;">{{ post.date | date: "%B %d, %Y" }}</time>
    {% if post.excerpt %}
      <p>{{ post.excerpt | strip_html | truncatewords: 50 }}</p>
    {% endif %}
    <a href="{{ post.url | relative_url }}">Read more &rarr;</a>
  </article>
{% endfor %}

{% if site.posts.size == 0 %}
  <p>No posts yet.</p>
{% endif %}
