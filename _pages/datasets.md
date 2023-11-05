---
layout: page
title: datasets
permalink: /datasets/
description: published dataset(s)
nav: true
nav_order: 3
display_categories: [2022]
horizontal: false
only_highlights: false
---
<!-- datasets.md is linked to _writing directory that contains all competition and datasets details-->

<!-- pages/datasets.md -->
<div class="writing">
{%- if site.enable_project_categories and page.display_categories -%}
  <!-- Display categorized datasets -->
  {%- for category in page.display_categories -%}
  <h2 class="category">{{ category }}</h2>
  {%- if page.only_highlights -%}
    {%- assign categorized_projects = site.worksop | where: "highlighted", true | where: "category", category -%}
  {%- else -%}
    {%- assign categorized_projects = site.datasets | where: "category", category -%}
  {%- endif -%}
  {%- assign sorted_projects = categorized_projects | sort: "importance" | sort: "date" -%}
  <!-- Generate cards for each writing type -->
  <div class="list-style mx-auto">
    {%- for project in categorized_projects -%}
      {% include datasets.html %}
    {%- endfor %}
  </div>
  {% endfor %}

{%- else -%}
<!-- Display writing without categories -->
  {%- if page.only_highlights -%}
  {%- assign sorted_projects = site.datasets | where: "highlighted", true | sort: "importance" | sort: "date" -%}
  {%- else -%}
  {%- assign sorted_projects = site.datasets | sort: "importance" | sort: "date" -%}
  {%- endif -%}
  <!-- Generate cards for each project -->
  <div class="list-style mx-auto">
    {%- for project in sorted_projects -%}
      {% include datasets.html %}
    {%- endfor %}
  </div>
{%- endif -%}

</div>
