---
layout: page
title: Archivo
permalink: /archive/
---

<div class="posts" style="text-align: center;">
  {% for post in site.posts %}
    

      <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a>
      {{  post.date | date: '%d-%m-%Y' }}
     
    
  {% endfor %}
</div>
