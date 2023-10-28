---
layout: page
permalink: /datasets/
title: datasets
description: Published datasets
years: [1967, 1956, 1950, 1935, 1905]
nav: true
nav_order: 3
---
<!-- _pages/publications.md -->
<div class="publications">

{%- for y in page.years %}
  <h2 class="year">{{y}}</h2>
  {% bibliography -f {{ site.scholar.bibliography }} -q @*[year={{y}}]* %}
{% endfor %}

</div>
