---
layout: post
title:  Hypothesis Testing Revisited
date:   2017-06-02 16:40:16
description: "How to test a hypothesis"
tags: statistics
categories: data-science machine-learning
---

There are two equivalent approaches to hypothesis testing:

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-7-1.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>


**Critical Value approach:** Critical values for a test of hypothesis depend upon a test statistic, which is specific to the type of test, and the significance level, 𝛼, which defines the sensitivity of the test. A value of 𝛼 = 0.05 implies that the null hypothesis is rejected 5 % of the time when it is in fact true. The choice of 𝛼 is somewhat arbitrary, although in practice values of 0.1, 0.05, and 0.01 are common. Critical values are essentially cut-off values that define regions where the test statistic is unlikely to lie; for example, a region where the critical value is exceeded with probability 𝛼 if the null hypothesis is true. The null hypothesis is rejected if the test statistic lies within this region which is often referred to as the rejection region(s).

**Steps for critical value approach:**

1. Specify the null and alternative hypothesis.
2. Using the sample data and assuming the null hypothesis is true, calculate the value of the test statistic.
3. Using the distribution of the test statistic, look up the critical value such that the probability of making a Type I error is equal to alpha (which you specify).
4. Compare the test statistic to the critical value. If the test statistic is more extreme in the direction of the alternative than the critical value, reject the null hypothesis. Otherwise, do not reject the null hypothesis.

**P-value approach:** The p-value is the probability of the test statistic being at least as extreme as the one observed given that the null hypothesis is true. A small p-value is an indication that the null hypothesis is false.

**NOTE:** It is good practice to decide in advance of the test how small a p-value is required to reject the test. This is exactly analogous to choosing a significance level, 𝛼, for test. For example, we decide either to reject the null hypothesis if the test statistic exceeds the critical value (for 𝛼 = 0.05) or analogously to reject the null hypothesis if the p-value is smaller than 0.05.

P-value approach is most commonly cited in literature but that is as matter of convention.
