---
layout: single
title: "Tags"
permalink: /tags/
---

{% assign project_tags = site.projects | map: "tags" | compact | flatten %}
{% assign post_tags = site.posts | map: "tags" | compact | flatten %}
{% assign all_tags = project_tags | concat: post_tags | uniq | sort %}

<h2>Tags Used in Projects and Posts</h2>
<ul>
  {% for tag in all_tags %}
    <li><a href="#{{ tag | slugify }}">{{ tag }}</a></li>
  {% endfor %}
</ul>

<hr>

{% for tag in all_tags %}
  <h3 id="{{ tag | slugify }}">{{ tag }}</h3>
  <ul>
    {% for project in site.projects %}
      {% if project.tags contains tag %}
        <li><a href="{{ project.url }}">{{ project.title }}</a></li>
      {% endif %}
    {% endfor %}
    {% for post in site.posts %}
      {% if post.tags contains tag %}
        <li><a href="{{ post.url }}">{{ post.title }}</a></li>
      {% endif %}
    {% endfor %}
  </ul>
{% endfor %}