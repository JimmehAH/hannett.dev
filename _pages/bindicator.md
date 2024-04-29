---
title: Bindicator
permalink: /portfolio/bindicator
---

## Bindicator Development Log

{% for post in site.categories["bindicator"] %}

* [{{ post.title }}]({{ post.url }})

  {{ post.excerpt }}

{% endfor %}
