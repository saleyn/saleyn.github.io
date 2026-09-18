---
layout: default
title: Blog
permalink: /blog/
---

## Posts

{% for post in site.posts %}
  {% if post.title.size > 65 %}
    {% assign display_title = post.title | split: ":" | first | strip %}
  {% else %}
    {% assign display_title = post.title %}
  {% endif %}
- [{{ display_title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}
