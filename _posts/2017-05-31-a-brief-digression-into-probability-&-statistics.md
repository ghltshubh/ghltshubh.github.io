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

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-5-1.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

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

Mean $$\bar{x} = \frac{\sum_{i=1}^n x_i}{n}$$ = (2 + 4 + 7 + 11 + 15 + 16.5 + 19)/7 = 10.643

Median = 11

Mode = 16.5

Median is preferred when the data is skewed or subject to outliers.

**WARNING:** A median value significantly larger than the mean value should be investigated!

Measuring spread of data:

Range: Maximum value – Minimum value = 19 – 2 = 17

Variance: $$s_{n-1}^2 = \frac{\sum_\left({x_i} – \bar{x}\right)^2}{n-1}$$

Standard Deviation: $$s_{n-1} = \sqrt{Variance}$$

Range is a quick way to get an idea of the spread.

IQR takes longer to compute but it sometimes gives more useful insights like outliers or bad data points etc.

Interquartile Range: IQR is amount of spread in the middle 50% of the data set. In the previous e.g.

Q1(25% of data) = (2 + 4 + 7 + 11)/4 = 6

Q2(50% of data) = 11

Q3(75% of data) = (11 + 16.5 + 16.5 + 19)/4 = 15.75

IQR = Q3 – Q1 = 15.75 – 6 = 9.75

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-5-2.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

*Questioning the underlying reason for distributional non-unimodality frequently leads to greater insight and improved deterministic modeling of the phenomenon under study.*

- Very common in real-world data.
- Best practice is to look into this.

**Plotting Data**

- Bar chart: It is made up of columns plotted on a graph and used for categorical variable.
- Frequency histogram(Histogram): It is made up of columns plotted on a graph and used for quantitative variable. Usually obtained by splitting the range of a continuous variable into equal sized bins(classes).

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-5-3.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

*Both display relative frequencies of different variables. With bar charts, the labels on the X axis are categorical; with histograms, the labels are quantitative. Both are useful in detecting outliers(odd data points).*

- Boxplots: The boxplot (a.k.a. box and whisker diagram) is a standardized way of displaying the distribution of data based on the five number summary: minimum, first quartile, median, third quartile, and maximum. In the simplest boxplot the central rectangle spans the first quartile to the third quartile (the interquartile range or IQR). A segment inside the rectangle shows the median and “whiskers” above and below the box show the locations of the minimum and maximum. The extreme values (within 1.5 times the interquartile range from the upper or lower quartile) are the ends of the lines extending from the IQR. Points at a greater distance from the median than 1.5 times the IQR are plotted individually as asterisks. These points represent potential outliers.

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-5-4.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

**Shape**

- Skewness: Measure of degree of asymmetry of a variable.

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-5-5.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

- Skewness = $$\frac{\sum_{i=1}^n (x_i – \bar{x})^3}{(n-1)s^3}$$

    - Value of 0 indicates a perfectly symmetric variable.
    - Positive skewness: The majority of observations are to the left of the mean.
    - Negative skewness: The majority of observations are to the right of the mean.

- Kurtosis: A measure of how “tailed” a variable is.
    - Variables with a pronounced peak near the mean have high kurtosis.
    - Variables with a flat peak have a low kurtosis.

- Kurtosis = $$\frac{\sum_{i=1}^n (x_i – \bar{x})^4}{(n-1)s^4}$$

- Values for skewness and kurtosis near zero indicate the variable approximates a normal distribution.

**Sample Statistic and Population Parameter**

Each sample statistic has a corresponding unknown population value called a parameter. e.g. population mean, variance etc. are called parameter whereas sample mean, variance etc. are called statistic.

|                    | **Sample Statistic**  | **Population Parameter** |
|--------------------|-------------------|---------------------|
| **Mean**               | $$\bar{x}=\frac{\sum_{i=1}{n} {x_i}}{n}$$ | $$\mu=\frac{\sum_{i=1}{N} x_i}{N}$$ |
| **Variance**           | $$s_{n-1}^2=\frac{\sum({x_i-\bar{x}})^2}{n-1}$$ | $$\sigma^2=\frac{\sum({x_i-\mu})^2}{N}$$  |
| **Standard Deviation** | $$s$$ or $$s_{n-1}$$  | $$\sigma$$         |

*There are many more sample statistics and their corresponding population parameters.*

**Probability**

Probability: The likelihood of an event occurring.

Probability of an event = $$\frac{\text{# of favourable outcomes}}{\text{Total # of possible outcomes}}$$

Conditional Probability: The probability of an event occurring given that another event has occurred.

Conditional Probability of an event = $$P\left(A\bar{B}\right) = \frac{P\left(A\cap{B}\right)}{P\left(B\right)} \implies$$ A is dependent on B

Bayes Theorem: $$P\left(A\bar{B}\right) = \frac{P\left(B\bar{A}\right)P\left(A\right)}{P\left(B\right)}$$

**Probability Distribution:** 

A mathematical function that, stated in simple terms, can be thought of as providing the probability of occurrence of different possible outcomes in an experiment.
Let’s say we have a random variable 𝑋 = # of HEADS from flipping a coin 5 times.



<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/blog-5-6.png" title="" class="img-fluid rounded z-depth-1" %}
    </div>
</div>


[latex]P\left[X=0\right](/latex) = [latex]\frac{{5}\choose{0}}{32}[/latex] = [latex]{\frac{1}{32}}[/latex],  [latex]P\left[X=1\right](/latex) = [latex]\frac{{5}\choose{1}}{32}[/latex] = [latex]{\frac{5}{32}}[/latex]

[latex]P\left[X=2\right](/latex) = [latex]\frac{5\choose2}{32}[/latex] = [latex]{\frac{10}{32}}[/latex],  [latex]P\left[X=3\right](/latex) = [latex]\frac{{5}\choose{3}}{32}[/latex] = [latex]{\frac{10}{32}}[/latex]

[latex]P\left[X=4\right](/latex) = [latex]\frac{{5}\choose{4}}{32}[/latex] = [latex]{\frac{5}{32}}[/latex],  [latex]P\left[X=5\right](/latex) = [latex]\frac{{5}\choose{5}}{32}[/latex] = [latex]{\frac{1}{32}}[/latex]