---
layout: page
title: Software Blog
---
{% include JB/setup %}

## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


## Autor

**Franco Morinigo**<br>
Ingeniero en Sistemas y apasionado del Software.<br>
Surfer y músico incansable<br>

<br>
<br>
<a href="https://twitter.com/frannsulo" class="twitter-follow-button" data-show-count="false">Follow @frannsulo</a>
<script>!function(d,s,id){var js,fjs=d.getElementsByTagName(s)[0],p=/^http:/.test(d.location)?'http':'https';if(!d.getElementById(id)){js=d.createElement(s);js.id=id;js.src=p+'://platform.twitter.com/widgets.js';fjs.parentNode.insertBefore(js,fjs);}}(document, 'script', 'twitter-wjs');</script>
