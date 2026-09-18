---
layout: default
title: Blog
permalink: /blog/
---

## Posts

{% for post in site.posts %}
- [{{ post.title | split: ":" | first | strip }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}
