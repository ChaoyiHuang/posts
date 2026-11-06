---
title: Home
layout: default
nav_order: 1
description: Welcome to Chaoyi Huang's blog — notes on software engineering, AI agents, and technology.
---

# Hello

Welcome! This is my blog about software engineering and AI. Full list of posts is under [Blog]({{ site.baseurl }}/blog/), or pick one below:

{% assign posts = site.pages | where: "parent", "Blog" | sort: "nav_order" %}
<ul>
{% for post in posts %}
  <li>
    <a href="{{ post.url | relative_url }}"><strong>{{ post.title }}</strong></a><br>
    <small>{% if post.date %}{{ post.date | date: "%Y-%m-%d" }} — {% endif %}{{ post.description }}</small>
  </li>
{% endfor %}
</ul>
