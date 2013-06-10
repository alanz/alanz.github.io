---
layout: page
title: Hello World!
tagline: work in progress
---
{% include JB/setup %}

<div class='post'>
  <h1><a href='{{site.posts.first.url}}'>{{site.posts.first.title}}</a></h1>
  <div class='body'>{{ site.posts.first.content }}</div>
  <a href='{{site.posts.first.url}}#disqus_thread'>Comments</a>
</div>

### Older Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



