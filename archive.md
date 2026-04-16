---
layout: page
title: Blog Archive
permalink: /archive.html
---

All posts in reverse chronological order.

{% assign current_year = "" %}
{% for post in site.posts %}
  {% assign post_year = post.date | date: "%Y" %}
  {% if post_year != current_year %}
    {% assign current_year = post_year %}

## {{ current_year }}
  {% endif %}
- {{ post.date | date: "%b %-d, %Y" }}: [{{ post.title }}]({{ post.url }})
{% endfor %}

