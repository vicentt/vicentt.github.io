---
layout: page
title: Archivo
permalink: /archive/
---

<div class="posts" style="text-align: center;">
  {% for post in site.posts %}
    

      <h4><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h4>
      {{  post.date | date: '%d-%m-%Y' }}
     
    
  {% endfor %}
</div>
