---
layout: default
title:  "Minotes - Tags"
---

<div id="alltags">
<ul>
{% for tag in site.tags %}
<li style="font-size: {{ tag[1].size }}px">
<a href="#{{ tag[0] }}">{{ tag[0] }}</a>
</li>
{% endfor %}
</ul>
</div>

<div id="taggedposts">
{% capture tags %}
{% for tag in site.tags %}
{{ tag[0] }}
{% endfor %}
{% endcapture %}
{% assign sortedtags = tags | split:' ' | sort %}

{% for tag in sortedtags %}
<h3 id="{{ tag }}">{{ tag }}</h3>
<ul>
{% for post in site.tags[tag] %}
<li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>
{% endfor %}
</div>
