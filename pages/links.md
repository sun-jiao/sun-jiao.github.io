---
layout: page
title: Links
description: Links
keywords: Links
comments: true
menu: Links
permalink: /links/
---

My accounts:

<ul>
{% for link in site.data.links %}
  <li><a href="{{ link.url }}" target="_blank">{{ link.name}}</a></li>
{% endfor %}
</ul>
