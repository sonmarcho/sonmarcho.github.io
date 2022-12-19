---
layout: page
title: "Blog Posts"
---

<!-- <ul> -->
<!--   {% for post in site.posts %} -->
<!--     {{ post.date | date_to_string }} <br> -->
<!--     <a href="{{ post.url }}">{{ post.title }}</a> -->
<!--   {% endfor %} -->
<!-- </ul> -->

{% for post in site.posts %}
<article>
  <p style="color: rgb(130,130,130)">
  <time datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date |
  date_to_long_string }}</time><br>
  <font size="+2">
    <a href="{{ post.url }}"> {{ post.title }} </a>
  </font>
  </p>
</article>
{% endfor %}

{% for blog in site._blog %}
<article>
  <p style="color: rgb(130,130,130)">
  <time datetime="{{ blog.date | date: "%Y-%m-%d" }}">{{ blog.date |
  date_to_long_string }}</time><br>
  <font size="+2">
    <a href="{{ blog.url }}"> {{ blog.title }} </a>
  </font>
  </p>
</article>
{% endfor %}
