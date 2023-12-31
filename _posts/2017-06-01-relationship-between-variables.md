---
layout: post
title:  Relatioship between Variables
date:   2017-06-01 16:40:16
description: "Relationship between variables and how to measure it"
tags: statistics
categories: data-science machine-learning
---

## Scatterplot

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-6-1.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

- Used to determine relationship between two continuous variables.
- One variable plotted on x-axis, another on y-axis.
- Positive Correlation: Higher x-values correspond to higher y-values.
- Negative Correlation: Higher x-values correspond to lower y-values.
- Examples:
    - Body weight and BMI
    - Height and Pressure etc.

## Scatterplot Matrix

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-6-2.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

## Linear vs Non-Linear Relationship

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-6-3.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

Variables changing proportionately in response to each other show linear relationship. Linear relationship is an abstract concept it depends what can be called linear and what can’t in a given context. A linear relationship **may** exist locally with a non linear relationship globally.

**Summary Tables**

Method for understanding the relationship between two variables when at least one the variables is discrete.
Example: Summary information about ages of active psychologists by demographics.


<table  style="width:100%">
<tr>
<th rowspan="2" style="text-align:center"><strong>Ages</strong></th>
<th rowspan="2" style="text-align:center"><strong>(1) Total Active Psychologists</strong></th>
<th colspan="2" style="text-align:center"><strong>Active Psychologists by Gender</strong></th>
<th colspan="4" style="text-align:center"><strong>Active Psychologists by Race/Ethnicity</strong></th>
</tr>
<tr>
<td style="text-align:center"><strong>(2) Female</strong></td>
<td style="text-align:center"><strong>(3) Male</strong></td>
<td style="text-align:center"><strong>(4) Asian</strong></td>
<td style="text-align:center"><strong>(5) Black/ African American</strong></td>
<td style="text-align:center"><strong>(6) Hispanic</strong></td>
<td style="text-align:center"><strong>(7) White</strong></td>
</tr>
<tr>
<td style="text-align:center"><strong>Mean</strong></td>
<td style="text-align:center">50.5</td>
<td style="text-align:center">47.9</td>
<td style="text-align:center">55.1</td>
<td style="text-align:center">46.5</td>
<td style="text-align:center">47.9</td>
<td style="text-align:center">46.4</td>
<td style="text-align:center">51.1</td>
</tr>
<tr>
<td style="text-align:center"><strong>Median</strong></td>
<td style="text-align:center">51</td>
<td style="text-align:center">48</td>
<td style="text-align:center">57</td>
<td style="text-align:center">43</td>
<td style="text-align:center">46</td>
<td style="text-align:center">44</td>
<td style="text-align:center">53</td>
</tr>
<tr>
<td style="text-align:center"><strong>Std. Dev.</strong></td>
<td style="text-align:center">12.5</td>
<td style="text-align:center">12.4</td>
<td style="text-align:center">11.4</td>
<td style="text-align:center">13.3</td>
<td style="text-align:center">10.3</td>
<td style="text-align:center">11.2</td>
<td style="text-align:center">12.6</td>
</tr>
</table>


**Discrete Variable(s):** Demography: (1), (2), (3), (4), (5), (6), (7)
**Continuous Variable:** Age

**Cross-Tabulation Tables/ Crosstabs/ Contingency Tables**

- Method for summarizing two categorical variables
- In practice, continuous variables may be at times summarized as categorical variables.
- Example: Age could be divided into categories as young, adult and senior citizen, etc. Income could be divided into categories as poor, middle class, upper middle class, wealthy, etc.

**Correlation Coefficient**

- A quantification of the linear relationship between two variables
- Ranges from -1 to +1
- Used for variables on an interval or ratio scale


$$r_{xy} = \sum_{i=1}^{i=n}\frac{\left({x_i\,-\,\bar{x}}\right)\left({y_i\,-\,\bar{y}}\right)}{\left(n\,-\,1\right)s_{x}s_{y}}
 = \frac{\sum{\left( {x_i\,-\,\bar{x}} \right)}\left( {y_i\,-\,\bar{y}} \right)}{\sqrt{\sum\left({x_i\,-\,\bar{x}}\right)^2\sum\left({y_i\,-\,\bar{y}}\right)^2}}$$

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-6-4.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

**NOTE:** Correlation coefficient does not capture nonlinear relationships. Many nonlinear relationships might exist which are not captured (𝑟 = 0) by correlation coefficient.
