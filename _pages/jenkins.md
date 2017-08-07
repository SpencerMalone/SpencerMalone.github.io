---
layout: single
title:  "Jenkins Posts!"
---
<ul>
  {% for post in site.posts %}
    {% if post.categories contains 'jenkins' %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
    {% endif %}
  {% endfor %}
</ul>