---
title: Wesley Rodrigues - Blog Posts
permalink: /blog/
layout: default
---

<!-- Two -->
<section id="two" class="wrapper style3">
  <div class="inner">
  </div>
</section>

<!-- Three -->
<section id="three" class="wrapper style2">
  <div class="inner">
    <div class="grid-style">

      {% for post in site.posts %}
      <div>
        <div class="box card">
          <div class="image fit">
            <img src="{{ post.image_url }}" alt="" />
          </div>
          <div class="content">
            <header class="align-center">
              <h2>{{ post.title }}</h2>
            </header>
            <hr />
            <p>{{ post.description }}</p>
            <div class="align-center">
              <a href="{{ post.url }}" class="button alt">Ler</a>
            </div>
          </div>
        </div>
      </div>
      {% endfor %}

    </div>
  </div>
</section>
