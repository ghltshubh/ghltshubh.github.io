---
layout: page
title: workshops
permalink: /workshops/
description: List of worshops and competitions that I have organized.
nav: true
nav_order: 4
display_categories: [workshops, competitions]
horizontal: false
only_highlights: false
---

<!-- pages/workshops.md -->
<div class="writing">
{%- if site.enable_project_categories and page.display_categories -%}
  <!-- Display categorized workshops -->
  {%- for category in page.display_categories -%}
  <h2 class="category">{{ category }}</h2>
  {%- if page.only_highlights -%}
    {%- assign categorized_projects = site.workshops | where: "highlighted", true | where: "category", category -%}
  {%- else -%}
    {%- assign categorized_projects = site.workshops | where: "category", category -%}
  {%- endif -%}
  {%- assign sorted_projects = categorized_projects | sort: "importance" | sort: "date" -%}
  <!-- Generate cards for each workshops type -->
  <div class="list-style mx-auto">
    {%- for project in categorized_projects -%}
      {% include workshops.html %}
    {%- endfor %}
  </div>
  {% endfor %}

{%- else -%}
<!-- Display workshops without categories -->
  {%- if page.only_highlights -%}
  {%- assign sorted_projects = site.workshops | where: "highlighted", true | sort: "importance" | sort: "date" -%}
  {%- else -%}
  {%- assign sorted_projects = site.workshops | sort: "importance" | sort: "date" -%}
  {%- endif -%}
  <!-- Generate cards for each project -->
  <div class="list-style mx-auto">
    {%- for project in sorted_projects -%}
      {% include workshops.html %}
    {%- endfor %}
  </div>
{%- endif -%}

</div>