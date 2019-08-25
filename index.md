---
layout: default
title: Kevin Cang
---
<ul>
  {% for post in site.posts %}
		<small>{{ post.date | date: "%-d %B %Y" }}</small>
    <h1><a href="{{ post.url }}">{{ post.title }}</a></h1>
    {{ post.excerpt }}
  {% endfor %}
</ul>
