---
layout: page
title: Posts
permalink: /posts/
---
<ul>

  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
      - <time datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date | date_to_long_string }}</time>
    </li>
    {% if post.excerpt %}
        {{ post.excerpt | truncate: 256 }}  
    {% endif %}
  {% endfor %}
