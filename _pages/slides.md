---
layout: archive
title: Slides
permalink: /slides/
---

{% for page in site.pages %}
{% if page.layout == "slides" %}
[{{ page.title }}]({{ page.url | prepend: site.baseurl }})
{% endif %}
{% endfor %}