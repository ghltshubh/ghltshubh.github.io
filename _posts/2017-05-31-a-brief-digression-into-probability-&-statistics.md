---
layout: post
title:  A brief digression into Probability & Statistics
date:   2017-05-31 16:40:16
description: A tour of Data Science using Python
tags: first-blog
categories: data-science machine-learning
---

We measure the sample using statistics in order to draw inferences about the population and its parameters.
Samples are collected through random selection from a population. This process is called **sampling**.

**Types of statistics:**

1. Descriptive: Number that describes or gives a gist of the population data under consideration.
2. Inferential: Making conclusion about the population.
Notion of variability: Degree to which data points are different from each other or the degree to which they vary.

**Methods of sampling data:**

1. Random sampling: Every member and set of members has an equal chance of being included in the sample.
2. Stratified random sampling: The population is first split into groups. The overall sample consists of some members from every group. The members from each group are chosen randomly. Most widely used sampling technique.
3. Cluster random sample: The population is first split into groups. The overall sample consists of every member from some groups. The groups are selected at random.
4. Systematic random sample: Members of the population are put in some order. A starting point is selected at random and every nth member is selected to be in the sample.

**Types of Statistical Studies:**

1. Sample study(informal term): Taking random samples to generate a statistic to estimate the population parameter.
2. Observational study: Observing a correlation but not sure of the causality.
3. Controlled experiment: Experimenting to confirm the observation by forming a control group and treatment group. It is done by randomly assigning people or things to groups. One group receives a treatment and the other group doesn’t.

Running an experiment is the best way to conduct a statistical study. The purpose of a sample study is to estimate certain population parameter while observational study and experiment is to compare two population parameters.

**Describing data**
Central Tendency: There are different ways of understanding central tendency
Mean: Arithmetic mean value
Median: Middle value
Mode: Highest frequency value
e.g. Samples of observations of a variable 

$$x_i$$ = 2, 4, 7, 11, 16.5, 16.5, 19

$$n$$ = 7

Mean $$\bar_{x}$$ = (2 + 4 + 7 + 11 + 15 + 16.5 + 19)/7 = 10.643

Median = 11

Mode = 16.5

Median is preferred when the data is skewed or subject to outliers.

**WARNING:** A median value significantly larger than the mean value should be investigated!

Measuring spread of data:

Range: Maximum value – Minimum value = 19 – 2 = 17

Variance: $$s_{n-1}^2 = \frac{\sum_({x_i} – \bar_{x})^2}{n-1}$$

Standard Deviation: $$s_{n-1} = \sqrt{Variance}$$

Range is a quick way to get an idea of the spread.

IQR takes longer to compute but it sometimes gives more useful insights like outliers or bad data points etc.

Interquartile Range: IQR is amount of spread in the middle 50% of the data set. In the previous e.g.
Q1(25% of data) = (2 + 4 + 7 + 11)/4 = 6
Q2(50% of data) = 11
Q3(75% of data) = (11 + 16.5 + 16.5 + 19)/4 = 15.75
IQR = Q3 – Q1 = 15.75 – 6 = 9.75
