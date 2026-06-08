---
layout: default
title: Servo generational run
description: A blog series about Servo work, debugging, and memory behavior.
permalink: /series/servo-generational-run/
---

<section class="hero">
  <p class="eyebrow">Series</p>
  <h1>Servo generational run</h1>
  <p class="lede">My Servo work and fixes, including bug fixes, features I work on, and notes about the details behind them.</p>
</section>

{% assign series_posts = site.posts | where: "series", "Servo generational run" %}

{% if series_posts.size > 0 %}
<ul class="post-list">
  {% for post in series_posts %}
  <li>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <p class="post-meta">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
    </p>
    {% if post.description %}<p>{{ post.description }}</p>{% endif %}
  </li>
  {% endfor %}
</ul>
{% else %}
<p>No posts are published in this series yet.</p>
{% endif %}
