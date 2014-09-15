---
layout: page
title: "治学方法"
date: 
modified:
excerpt:
image:
  feature: so-simple-sample-image-2.jpg
---

<ul class="post-list">
{% for post in site.categories.methodology %} 
  <li><article><a href="{{ site.url }}{{ post.url }}">{{ post.title }} <span class="entry-date"><time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %d, %Y" }}</time></span></a></article></li>
{% endfor %}
</ul>