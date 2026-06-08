---
layout: default
title: Blog
description: Notes on browser engines, systems programming, and software work.
---

## Series

<ul class="series-list">
  <li>
    <a href="{{ '/series/servo-generational-run/' | relative_url }}">Servo generational run</a>
    <p>My Servo work and fixes, including bug fixes, features I work on, and notes about the details behind them.</p>
  </li>
</ul>

## Posts

{% if site.posts.size > 0 %}
<ul class="post-list">
  {% for post in site.posts %}
  <li>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <p class="post-meta">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
      {% if post.series %}<span>{{ post.series }}</span>{% endif %}
    </p>
    {% if post.description %}<p>{{ post.description }}</p>{% endif %}
  </li>
  {% endfor %}
</ul>
{% else %}
<p>No posts are published yet.</p>
{% endif %}
