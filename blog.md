---
title:  Wesley Rodrigues - Blog Posts
permalink: /blog/
layout: blogpage
---

# Blog Posts

<ul class="posts_list">
  {% for post in site.posts %}
    <li class="blog_entry">
      <a href="{{ post.url }}">
        <img src="{{ post.image_url }}" alt="">
        <div class="blog_entry_text">
          <h3>{{ post.title }}</h3>
          <p class="description">{{ post.description }}</p>
          <p><strong>Escrito em: </strong> {{ post.date | date_to_string }}</p>
          <p><strong>Tags: </strong> 
          {% for t in post.tags %}
          {{ t }}, 
          {% endfor %}
          </p>
          <p><strong>Idioma: </strong>{{ post.lang }}</p>
        </div>
      </a>
    </li>
  {% endfor %}
</ul>
