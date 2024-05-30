---
layout: archive
title: "Algorithm"
permalink: /algorithm/
author_profile: true
---

{% include base_path %}

{% for post in site.algorithm reversed %}
  {% include archive-single.html %}
{% endfor %}