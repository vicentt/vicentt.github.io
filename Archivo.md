---
layout: page
title: Archivo
permalink: /archive/
---

<div class="posts" style="text-align: center;">
  {% for post in site.posts %}
    
      <li><span>{{ post.date | date_to_string }}</span> &nbsp; <a href="{{ post.url }}">{{ post.title }}</a></li> 
    
  {% endfor %}
</div>
