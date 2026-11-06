---
title: Blog
layout: default
nav_order: 2
has_children: true
description: All posts.
---

# Blog

{% assign posts = site.pages | where: "parent", "Blog" | sort: "nav_order" %}
<ul>
{% for post in posts %}
  <li>
    <a href="{{ post.url | relative_url }}"><strong>{{ post.title }}</strong></a><br>
    <small>{% if post.date %}{{ post.date | date: "%Y-%m-%d" }} — {% endif %}{{ post.description }}</small>
  </li>
{% endfor %}
</ul>
