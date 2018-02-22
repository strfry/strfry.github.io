---
permalink: /blog/index.html
layout: post
---
<!--
{% for item in site.posts %}
<h2>{{ item.title }}</h2> 
  <p>{{ item.description }}</p>
  <span class="post-meta">{{ item.date | date: "%b %-d, %Y" }}</span>

  <p><a href="{{ item.url }}">{{ item.title }}</a></p>
{% endfor %}
-->

<div class="home">

  <ul class="post-list">
    {% for post in (site.posts) %}
      <li>

        <h2>
          <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        </h2>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>

      </li>
    {% endfor %}
  </ul>

</div>



