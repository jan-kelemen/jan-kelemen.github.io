---
title: "Blog Archive"
permalink: /archive
---

<!-- https://emmatheeng.github.io/projects/blog_setup/blog-tags.html -->
{% for tag in site.tags %}
  {% assign tag_name = tag | first %}
  {% assign tag_name_pretty = tag_name | replace: "_", " " | capitalize %}
  <div>
    <div id="#{{ tag_name | slugize }}"></div>
    <h3 class="post-list-heading line-bottom"> With #{{ tag_name }}: </h3>
    <a name="{{ tag_name | slugize }}"></a>
    <ul class="post-list post-list-narrow">
     {% for post in site.tags[tag_name] %}
        <li><a href="{{ post.url }}">{{ post.date | date: "%F" }} - {{ post.title }}</a></li>
     {% endfor %}
    </ul>
  </div>
{% endfor %}
