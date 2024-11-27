---
title: "Complex Survey Notes"
author: "Adam Peterson"
date: "2024-11-26"
output: 
  html_document:
    code_folding: hide
    toc: true
    toc_float: true
bibliography: surveybib.bib
editor_options: 
  markdown: 
    wrap: 80
---




# Preface

## Overview

What follows is a coded work through of Thomas Lumley's "Complex Surveys: A
Guide to Analysis in R" [@lumley2011complex] and the methods underlying
[design based](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC2856970/)
statistics more broadly. This write-up reflects my understanding of the
material with additions made to try and clarify ideas further. These include
simulations, derivations or references that I found helpful in working through
Lumley's material. Most of the data sets that I use throughout these notes are
from Lumley's website for the
[book](https://r-survey.r-forge.r-project.org/survey/). 


### How to use these notes?

I'd imagine there are two general ways one might use these notes:

1. As a quick reference on any of the code or topics.

2. If you are working through the book, you may find these notes useful to 
double check Lumley's code or you're understanding of the principles tested in
the exercises. 

  * Note that Lumley's code was written at the time of publishing and much has
  changed, both in the [survey package](https://isi-iass.org/home/wp-content/uploads/Survey_Statistician_2023_July_N88_019.pdf)
  and R more generally. I try to keep this as close to his original code as
  possible while updating when necessary. I also often show how to produce the 
  same code using the [`srvyr` package](http://gdfe.co/srvyr/), but this is less
  relevant for the later chapters. 
     * I would not always use the same method if I was writing this kind of code
     but I try to replicate his to make it easy for any reader with the book to
     follow along.
  
  * My answers to the exercises are not guaranteed to be correct. I would 
  strongly encourage you to avoid copying them if you are, for example, 
  working through this textbook for a class.

### How shouldn't I use these notes?

These notes are no substitute for buying the 
[book](https://www.amazon.com/Complex-Surveys-Guide-Analysis-Using/dp/0470284307) 
itself. I would encourage you to buy the book if you're intent on working 
through these notes.

## Drawing Samples in R

It wasn't very long into reading this book that I found that weighted sampling
in R is, unfortunately, not well set-up for complex designs.  It
would not be possible, for example, to simply use the base R `sample` or 
popular `tidyverse` package `dplyr`'s function `slice_sample()` to draw a 
weighted sample from a population for example with appropriate inclusion 
probabilities.Further details are in 
[this](https://stats.stackexchange.com/q/639211/) stats exchange post.

Instead a function from the
[`sampling`](https://cran.r-project.org/web/packages/sampling/index.html)
package would have to be used. I use this function below in any setting where a
non-uniform sample with inclusion probabilities is needed.

Its worth further pointing out that the topic of how samples themselves are
drawn is a complicated one its own right and that the functions in the
`sampling` package each have pros and cons according to the target estimand or
question of interest. Drawing samples with replicate weights 
-- discussed in Chapter 2 -- is a similarly complex question which I haven't 
yet resolved to my satisfaction.

# Chapter 1: The Basics

## Design vs. Model

This book focuses on "Design-based" Inference. That is the methods in this book
focus on the design from which the data are constructed, rather than the data 
itself. In a traditional survey setting the data are assumed to be fixed and 
the probabilities of sampling different entities are used to derive the desired
estimate. These inclusion probabilities or their inverse, "sampling weights", 
are used to re-balance the data so that they more accurately reflect the target
population distribution. Different sampling techniques --- clustering, 2-phase, 
etc. --- are used to either decrease the variance of the resulting estimate, 
the cost associated with the design or both.

### Horvitz Thompson Estimation

The Horvitz Thompson Estimator (HTE) is the starting point for non-uniform
random estimates. If we observe measure $X_i$ on subject $i$ drawn with 
probability $\pi_i$ from a population of $N$ total subjects the HTE is 
formulated as follows:

$$
HTE(X) = \sum_{i=1}^{N} \frac{1}{\pi_i}X_i, \tag{1.1}
$$

which is an unbiased estimator as shown in the [Chapter 1 Appendix].

The variance of this estimate is:

$$
V[HTE(X)] = \sum_{i,j} \left ( \frac{X_i X_j}{\pi_{ij}} - 
\frac{X_i}{\pi_i} \frac{X_j}{\pi_j} \right ), \tag{1.2}
$$

which follows from the Bernoulli covariance using indicator variables $R_i=1$ if
individual $i$ is in the sample, $R_i=0$ otherwise. A proof is provided in the
[Questions From Chapter 1].

### Design And Misspecification Effects

[@kish1965survey] defined the notion of a *design effect* as the ratio of a
variance of an estimate in a complex sample to the variance of the same estimate
in a simple random sample (SRS). The motivation for this entity being that it
can guide researchers in terms of how much sample size they may need; If the
sample size for a given level of precision is known for a simple random sample,
the sample size for a complex design can be obtained by multiplying by the
design effect.

While larger sample sizes may be necessary to maintain the same level of
variance as a SRS, the more complex may still be more justified because of the
lower cost associated. See [@meng2018statistical] for an example of where design
effects are used in a modern statistical setting by comparing competing
estimators.

### Other preliminary items

From this point Lumley works through an introduction to the datasets used in the
book and the idea that we'll often be taking samples from datasets where we know
the "true" population and computing estimates from there. This isn't always the
case and there may some subtlety worth discussing how to interpret results once
we get into topics like regression, but for the most part his description makes
sense.

One thing I found lacking in this introductory section is the motivation for why
we might take non-uniform samples. It isn't until Chapter 3 that Lumley
discusses probability proportional to size (PPS) sampling, but this is very
often the reason why a non-uniform sample is used.

If we have some measure that is right skewed in our population of interest and
we'd like to estimate the mean, we could take a SRS to estimate the mean but the
variance on that item would be lower than if we sampled proportional to the
right skew measure itself. I'll demonstrate with the following quick example,
suppose we want to measure the income of a population. Incomes are often right
skewed, but we can get a lower variance estimate if we take a weighted sample.

I generate a right skewed population and visualize the distribution.


::: {.cell}

```{.r .cell-code}
population <- tibble(
  id = 1:1E3,
  income = 5E4 * rlnorm(1E3, meanlog = 0, sdlog = 0.5)
)
population %>%
  ggplot(aes(x = income)) +
  geom_histogram() +
  geom_vline(aes(xintercept = mean(income)), linetype = 2, color = "red") +
  ggtitle("Simulated Income Distribution") +
  xlab("Income (USD)")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/pps_sample_setup-1.png){width=672}
:::
:::


Here I'll take a uniform and weighted sample of size 50. Note that the
differences in the samples are subtle. They might not look all that different on
visual inspection.


::: {.cell}

```{.r .cell-code}
uniform_sample <- population %>%
  slice_sample(n = 50) %>%
  transmute(
    income = income,
    method = "uniform",
    pi = 50 / 1E3
  )

weighted_sample <- population %>%
  mutate(
    pi = sampling::inclusionprobabilities(floor(population$income), 50),
    in_weighted_sample = sampling::UPbrewer(pi) == 1
  ) %>%
  filter(in_weighted_sample) %>%
  transmute(
    income = income,
    pi = pi,
    method = "weighted"
  )

rbind(
  uniform_sample,
  weighted_sample
) %>%
  ggplot(aes(x = income, fill = method)) +
  geom_histogram() +
  ggtitle("Sample Comparisons") +
  xlab("Income (USD)") +
  theme(legend.title = element_blank(), legend.position = "top")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/pps_draw-1.png){width=672}
:::
:::


Finally I'll estimate the population mean from both samples and include the
design effect calculation in the weighted sample estimate.


::: {.cell}

```{.r .cell-code}
uniform_sample %>%
  as_survey_design() %>%
  summarize(
    mean_income = survey_mean(income)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  mean_income mean_income_se
        <dbl>          <dbl>
1      64409.          5695.
```
:::
:::

::: {.cell}

```{.r .cell-code}
weighted_sample %>%
  as_survey_design(probs = pi) %>%
  summarize(mean_income = survey_mean(income, deff = TRUE))
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
  mean_income mean_income_se mean_income_deff
        <dbl>          <dbl>            <dbl>
1      57548.          4724.             1.01
```
:::
:::


We see that the weighted estimate standard error is not quite half the uniform
estimate. Accordingly the design effect for the weighted sample is less than 1.

## Exercises

1. Doesn't make sense to reproduce here.

2. Don't make sense to reproduce here.

3. Each visit to the front page of a newspaper's website has (independently) a
1/1000 chance of resulting in a questionnaire on voting intentions in a
forthcoming election. Assuming that everyone who is given the questionnaire
responds, why are the results not a probability sample of:

-   Voters?
-   Readers of the newspaper?
-   Readers of the newspaper's online version?

Lumley lists 4 properties needed for a sample to be considered a probability
sample.

*  (i) Every individual (unit of analysis) in the population must have a non-zero
    probability of ending up in the sample ($\pi_i>0 \forall i$)

*  (ii) $\pi_i$ must be known for every individual who does end up in the sample.

*  (iii) Every pair of individuals in the sample must have a non-zero probability of
    both ending up in the sample ($\pi_{i,j} \forall i, j$)

*  (iv) The probability $\pi_{i,j}$ must be known for every pair that does end up in
    the sample.

(i) is not guaranteed when considering voters --- there are voters who don't 
read the paper who have will have $\pi_i = 0$ --- or the broader heading of 
"readers" of the newspaper - since those who only read the physical paper will 
have a \$\pi\_i = 0 \$. For "readers of the newspaper's online version" the 
sample would only be a probability sample if the time window was further 
specified, as there could be online readers who do not visit during the survey 
window, and would thus be assigned a $\pi_i=0$.

4. You are conducting a survey that will estimate the proportion of women who
used anti-malarial insecticide-treated bed nets every night during their last
pregnancy. With a simple random sample you would need to recruit 50 women in any
sub-population where you wanted a standard error of less than 5 percentage
points in the estimate. You are using a sampling design that has given design
effects of 2-3 for proportions in previous studies in similar areas.

  * Will you need a larger or smaller sample size than 50 for a sub-population 
  to get the desired precision?

Larger, a design effect $>1$ indicates that the variance is larger in the
complex design with the same sample size - consequently the sample size will
need to be increased to maintain the same level of precision.

  * Approximately what sample size will you need to get the desired precision?

100 - 150. Derived from multiplying 50 by 2 and 3.

5. Systematic sampling involves taking a list of the population and choosing,
for example, every 100th entry in the list.

  * Which of the necessary properties of a probability sample does this have?

Items ii-iv from the list enumerated above. The only condition that is not
satisfied is that not every item has a nonzero probability of being chosen.

  * For systematic sampling with a random start, the procedure would be to
    choose a random starting point from 1, 2, ..., 1000 and then take every
    100th entry starting at the random point. Which of the necessary properties
    of a probability sample does this procedure have?

This satisfies all items from the above list.

  * For systematic sampling with multiple random starts we might choose 5 random
    starting points in 1, 2, ....., 5000 and then take every 500th entry
    starting from each of the 5 random points. Which of the necessary properties
    of a probability sample does this procedure have?

Again, this satisfies all items from the above list.

  * If the list were shuffled into a random order before a systematic sample was
    taken, which of the properties would the procedure have.

Again, all of them. The key is adding the *known* randomness and not excluding
any items from selection.

  * Treating a systematic sample as if it were a simple random sample often
    gives good results. Why would this be true?

This would be because the items are not ordered in any particular fashion prior
to taking the "systematic sampling". In this setting a systematic sample is
equivalent to a simple random sample.

6. Why must all the sampling probabilities be non-zero to get a valid 
population estimate?

If any of the sampling probabilities are zero, that would introduce bias in
shifting the estimate away from the portion of the population that would
*always* be unobserved under repeated sampling.

7. Why must all the pairwise probabilities be non-zero to get a valid
uncertainty estimate.

This is basically a second order statement equivalent to the previous. If any
pair is unable to be observed together that is a form of selection bias that
would shift the sample estimate away from the true population value.

8. A probability design assumes that people who are sampled will actually be
included in the sample, rather than refusing. Look up the response rates for the
most recent year of BRFSS and NHANES.

Lumley is highlighting the fact that even though we set up samples thinking
that every sample will be observed that is rarely the case. Looking at 
just the most recent NHANES 
[data](https://www.cdc.gov/nchs/hus/sources-definitions/nhanes.htm#sample-size-response-rate)
I see response rates at ~ 78% for the un-weighted, 6-year household survey.

9. In a telephone study using random digit dialing, telephone numbers are
sampled with equal probability from a list. When a household is recruited, why
is it necessary to ask how many telephones are in the household, and what should
be done with this information in computing the weights.

It is necessary to ask how many telephones are in the household to down weight
the *a priori* sampling probability accordingly because every additional
telephone line increases the odds that a given house is sampled. For example in
a simple population with two houses, where house one has 5 telephones and house
two has 2 telephones, and we're looking to take a $n=1$ sample, but we don't
know the number of telephones *a priori*, house one has a $\frac{5}{7}$
probability of being sampled. If that is the house that is chosen its weight
needs to go from 2 to $\frac{5}{7}$ to better reflect its sampling probability.
In a real sample this would be corrected relative to all the other households
number of telephones or perhaps a population average of the number of
telephones.

10.

Derive the Horvitz Thompson variance estimator for the total as follows.

*  Write $R_i = 1$ if individual $i$ is in the sample, $R_i=0$ otherwise. Show 
that $V[R_i] = \pi_i(1-\pi_i)$ and that $Cov[R_i,R_j]=\pi_{ij} - \pi_i\pi_j$.

This follows in a straightforward fashion from the assumption that $R_i$ is
distributed according to the Bernoulli distribution and $R_i \perp R_j$. This is
an accurate model for sampling with replacement, or sampling from large
populations with small sample sizes without replacement, but less true for small
sample sizes without replacement.

b)  Show that the variance of the Horvitz Thompson estimator is:

$$
V[\hat{T}_{HT}] = \sum_{i=1}^{N}\sum_{j=1}^{N} \check{x}_i\check{x}_j(\pi_{ij} - \pi_i \pi_j)
$$ We have, $$
\hat{T}_{HT} := \sum_{i=1}^{N} \frac{X_i I(X_i \in {S})}{\pi_i} \\
V[\hat{T}_{HT}] = V\left[\sum_{i=1}^{N}\frac{X_i I(X_i \in {S})}{\pi_i} \right] \\ 
=\sum_{i=1}^{N}\sum_{j=1}^{N} Cov\left[\frac{X_i I(i \in {S})}{\pi_i},
\frac{X_j I(j \in {S})}{\pi_j}\right]\\
= \sum_{i=1}^{N}\sum_{j=1}^{N} \frac{X_i}{\pi_i}\frac{X_j}{\pi_j}Cov(I(i \in 
{S}),I(j \in {S})) \\
= \sum_{i=1}^{N}\sum_{j=1}^{N} \frac{X_i}{\pi_i}\frac{X_j}{\pi_j} (\pi_{ij} - 
\pi_i \pi_j)
$$

which is equivalent to the above, where $\check{x_i} = \frac{X_i}{\pi_i}$.

  * Show that an unbiased estimator of the variance is 
$$
\hat{V}[\hat{T}_{HT}] = \sum_{i=1}^{N}\sum_{j=1}^{N} 
\frac{R_i R_j}{\pi_{ij}}\check{x_i}\check{x_j}(\pi_{ij} - \pi_i \pi_j)
$$

To show the expression above is unbiased for $\hat{V}[\hat{T}_{HT}]$ we must
show that $E\left [\hat{V}[\hat{T}_{HT}] \right] = V[\hat{T}_{HT}]$ $$
E \left [\hat{V}[\hat{T}_{HT} ] \right] = E \left[\sum_{i=1}^{N}\sum_{j=1}^{N} \frac{R_iR_j}{\pi_{ij}} \check{x}_i \check{x}_j(\pi_{ij} - \pi_i\pi_j) \right] \\ 
= \sum_{i=1}^{N} \sum_{j=1}^{N} \frac{E[R_iR_j]}{\pi_{ij}} \check{x}_i \check{x}_j (\pi_{ij} - \pi_i \pi_j) \\
= \sum_{i=1}^{N} \sum_{j=1}^{N} \frac{\pi_{ij}}{\pi_{ij}} \check{x}_i\check{x}_j(\pi_{ij} - \pi_i \pi_j)  \\
= \sum_{i=1}^{N} \sum_{j=1}^{N}  \check{x}_i\check{x}_j(\pi_{ij} - \pi_i \pi_j)  \\
\blacksquare
$$

d)  Show that the previous expression simplifies to equation 1.2

$$
\sum_{i=1}^{N} \sum_{j=1}^{N} \frac{R_iR_jx_ix_j}{\pi_{ij}\pi_i\pi_j}(\pi_{ij} - \pi_i \pi_j) \\
=
\sum_{i=1}^{n} \sum_{j=1}^{n} \frac{x_ix_j}{\pi_{ij}\pi_i\pi_j}(\pi_{ij} - \pi_i \pi_j) \\
= 
\sum_{i=1}^{n} \sum_{j=1}^{n} x_i x_j(\frac{1}{\pi_i \pi_j} - \frac{1}{\pi_{ij}}) \\
= 
\sum_{i=1}^{n} \sum_{j=1}^{n} \frac{x_ix_j}{\pi_i \pi_j}  - \frac{x_i x_j}{\pi_{ij}}\
$$

I'm not sure how the signs switch on the last line to reproduce expression 1.2

11. Another popular way to write the Horvitz-Thompson variance estimator is

$$
\hat{V}[\hat{T}_{HT}] = \sum_{i=1}^{n} x_i^{2} \frac{1-\pi_i}{\pi_i^2} + 
\sum_{i\neq j}x_ix_j\frac{\pi_{ij} - \pi_i \pi_j}{\pi_i\pi_j\pi_{ij}}
$$

Show that this is equivalent to equation 1.2

We need to show that the above is equivalent to
$$
\sum_{i,j} \frac{X_iX_j}{\pi_{ij}} - \frac{X_i}{\pi_i}\frac{X_j}{\pi_j}
$$

First we fix $i \neq j$ in the above expression and we find
$$
\sum_{i\neq j} \frac{X_iX_j}{\pi_{ij}} - \frac{X_i}{\pi_i}\frac{X_j}{\pi_j} = 
\sum_{i\neq j} X_iX_j(\frac{1}{\pi_{ij}} - \frac{1}{\pi_i} \frac{1}{\pi_j}) \\
= \sum_{i \neq j} X_iX_j(\frac{\pi_i \pi_j - \pi_{ij}}{\pi_{ij}\pi_i\pi_j})
$$

which is the latter part in the desired expression save for a sign, which again 
I must be missing somehow or is an error in the book. 

Now we take $i=j$ and return to expression 1.2 in which we have,

$$
\sum_{i=j} \frac{X_i X_j}{\pi_{ij}} - \frac{X_i}{\pi_i} \frac{X_j}{\pi_j} \\ 
= \sum_{i=j} \frac{X_i^2}{\pi_{ii}} - \frac{X_i}{\pi_i} \frac{X_i}{\pi_i} \\
= \sum_{i=j} X_i^2 (\frac{1}{\pi_{ii}} - \frac{1}{\pi_i^2} ) \\
= \sum_{i=j} X_i^2 (\frac{\pi_i^2 - \pi_{ii}}{\pi_i^2 \pi_{ii}})
$$
Clearly we have to formulate $\pi_{ii}$ in terms of $\pi_i$ but isn't immediately
clear to me how to do so. We know that for the two terms to be equal we must have

$$
\frac{\pi_i^2 - \pi_{ii}}{\pi_i^2 \pi_{ii}} = \frac{1 - \pi_i^2}{\pi_i^2}  \\
\iff \\
\pi_{ii} = \frac{\pi_i^2}{2- \pi_i^2}
$$
Which I suppose we'll take to be the expression of a co-inclusion probability of
an entity sampled with itself (this must assume sampling with replacement) for
this expression to be true.

## Chapter 1 Appendix

The HTE is an unbiased estimator of the population total - I reproduce the
expression from above, but now make explicit the indicator variables that
express which observations are included in our sample, $S$.

$$
HTE := \sum_{i=1}^{N} \frac{X_i I(X_i \in S)}{\pi_i} \\
E[HTE] = E\left [\sum_n \frac{X_i I(X_i \in S)}{\pi_i} \right ] \\
= \sum_n E \left [\frac{X_iI(X_i \in S)}{\pi_i} \right ] \\
= \sum_n \frac{X_iE[I(X_i \in S)]}{\pi_i} \\ 
= \sum_n \frac{X_i \pi_i}{\pi_i} = \sum_n X_i
$$

# Chapter 2: Simple and Stratified Sampling

## Starting from Simple Random Samples

When dealing with a sample of size $n$ from a population of size $N$ the HTE of
the total value of $X_i$ in the population can be written as

$$
\begin{equation}
HTE(X) = \hat{T_X} =  \sum_{i=1}^{n} \frac{X_i}{\pi_i}.
\end{equation}
$$

For a simple random sample, the variance can be more explicitly written as

$$
\begin{equation}
V[\hat{T_X}] = \frac{N-n}{N} \times N^{2} \times \frac{V[X]}{n},
\end{equation}
$$

where $\frac{N-n}{N}$ is the finite population correction factor. This factor is
[derived](https://stats.stackexchange.com/questions/5158/explanation-of-finite-population-correction-factor)
from the hypergeometric distribution and explains the reduction in uncertainty
that follows from sampling a large portion of the population. Consequently, if
the sample is taken with replacement --- the same individual or unit has the
possibility to be sampled twice --- this term is no longer relevant. It should
be noted that sampling with replacement is not usually used however, but
sometimes this language is used to refer to the fact that the finite correction
factor may not be used.

The second term, $N^2$, rescales the estimate from the mean to the total, while
the final term is simply the scaled variance of $X$.

A point worth deliberating on, that Lumley notes as well, is that while the
above equations suggest that a larger sample size is always better that is not
always the case in reality. Non-response bias or the cost of surveys can
dramatically diminish the *quality* of the dataset, even if the size is large. I
state this is worth deliberating on because it is a matter of increasing
importance in the world of "Big Data" - where it can be easy to delude oneself
with confidence in their estimates because their sample is large, even when the
sample is not well designed. See [@meng2018statistical] for a larger discussion
of this topic.

It follows from the above that the HTE for the population size is defined as
$\hat{N} = \sum_{i=1}^{n} \frac{1}{\pi_i}$. This holds true in the case where,
as here $\pi_i = \frac{n}{N}$, a bit trivial, but also in those where $\pi_i$
may be defined differently.

## Confidence Intervals

The sampling distribution for the estimates --- typically sample means and sums
--- across "repeated surveys" is Normal by the Central Limit Theorem, so the
typical $\bar{x} \pm 1.96 \sqrt{\frac{\sigma^2_X}{n}}$, expression is used to
calculate a 95% confidence interval. Lumley offers the following example from
the California Academic Performance Index (API)
[dataset](https://www.cde.ca.gov/re/pr/api.asp) to illustrate this idea.


::: {.cell hash='ComplexSurveyNotes_cache/html/clt_demo_4d669ceebfa02dde5877f52b81c7585a'}

```{.r .cell-code}
data(api)
mn_enroll <- mean(apipop$enroll, na.rm = TRUE)
p1 <- apipop %>%
  ggplot(aes(x = enroll)) +
  geom_histogram() +
  xlab("Student Enrollment") +
  geom_vline(xintercept = mn_enroll, linetype = 2, color = "red") +
  ggtitle("Distribution of School Enrollment")
p2 <- replicate(n = 1000, {
  apipop %>%
    sample_n(200) %>%
    pull(enroll) %>%
    mean(., na.rm = TRUE)
})
mn_sample_mn <- mean(p2)
p2 <- tibble(sample_ix = 1:1000, sample_mean = p2) %>%
  ggplot(aes(x = sample_mean)) +
  geom_histogram() +
  xlab("Student Enrollment Averages") +
  geom_vline(
    xintercept = mn_sample_mn,
    linetype = 2, color = "red"
  ) +
  ggtitle("Distribution of Sample Means")
p1 + p2
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/clt_demo-1.png){width=1344}
:::
:::


## Complex Sample Data in R

What follows is a work-up of basic survey estimates using the California API
data set composed of student standardized test scores. I'll work through the code
once using the `survey` package and a second time using the `srvyr` package,
which has a [tidyverse](https://www.tidyverse.org/) friendly API. These are all
demonstrated alongside Lumley's commentary in the book and I'd encourage you to
read the section for more details on this part.

Much of the computational work in this book begins with creating a design
object, from which weights and other information can then be drawn on for any
number/type of estimates.

For example, we create a basic design object below, where we look at a classic
simple random sample (SRS) of the schools in the API dataset. Let's take a look
at the dataset first.


::: {.cell}

```{.r .cell-code}
dplyr::as_tibble(apisrs)
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 200 × 39
   cds       stype name  sname  snum dname  dnum cname  cnum  flag pcttest api00
   <chr>     <fct> <chr> <chr> <dbl> <chr> <int> <chr> <int> <int>   <int> <int>
 1 15739081… H     "McF… McFa…  1039 McFa…   432 Kern     14    NA      98   462
 2 19642126… E     "Sto… Stow…  1124 ABC …     1 Los …    18    NA     100   878
 3 30664493… H     "Bre… Brea…  2868 Brea…    79 Oran…    29    NA      98   734
 4 19644516… E     "Ala… Alam…  1273 Down…   187 Los …    18    NA      99   772
 5 40688096… E     "Sun… Sunn…  4926 San …   640 San …    39    NA      99   739
 6 19734456… E     "Los… Los …  2463 Haci…   284 Los …    18    NA      93   835
 7 19647336… M     "Nor… Nort…  2031 Los …   401 Los …    18    NA      98   456
 8 19647336… E     "Gla… Glas…  1736 Los …   401 Los …    18    NA      99   506
 9 19648166… E     "Max… Maxs…  2142 Moun…   470 Los …    18    NA     100   543
10 38684786… E     "Tre… Trea…  4754 San …   632 San …    37    NA      90   649
# ℹ 190 more rows
# ℹ 27 more variables: api99 <int>, target <int>, growth <int>, sch.wide <fct>,
#   comp.imp <fct>, both <fct>, awards <fct>, meals <int>, ell <int>,
#   yr.rnd <fct>, mobility <int>, acs.k3 <int>, acs.46 <int>, acs.core <int>,
#   pct.resp <int>, not.hsg <int>, hsg <int>, some.col <int>, col.grad <int>,
#   grad.sch <int>, avg.ed <dbl>, full <int>, emer <int>, enroll <int>,
#   api.stu <int>, pw <dbl>, fpc <dbl>
```
:::
:::


In the code below `fpc` stands for the aforementioned finite population
correction factor and `id=~1` designates the unit of analysis as each individual
row in the dataset.


::: {.cell}

```{.r .cell-code}
srs_design <- svydesign(id = ~1, fpc = ~fpc, data = apisrs)
srs_design
```

::: {.cell-output .cell-output-stdout}
```
Independent Sampling design
svydesign(id = ~1, fpc = ~fpc, data = apisrs)
```
:::
:::


In order to calculate the mean enrollment based on this sample the,
appropriately named, `svymean` function can be used.


::: {.cell}

```{.r .cell-code}
svymean(~enroll, srs_design)
```

::: {.cell-output .cell-output-stdout}
```
         mean     SE
enroll 584.61 27.368
```
:::
:::


This is the same as the typical computation - which makes sense, this is a SRS!


::: {.cell}

```{.r .cell-code}
c(
  "Mean" = mean(apisrs$enroll),
  "SE" = sqrt(var(apisrs$enroll) / nrow(apisrs))
)
```

::: {.cell-output .cell-output-stdout}
```
     Mean        SE 
584.61000  27.82121 
```
:::
:::


Instead of specifying the finite population correction factor, the sampling
weights could be used - since this is a SRS, all the weights should be the same.


::: {.cell}

```{.r .cell-code}
as_tibble(apisrs) %>% distinct(pw)
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 1
     pw
  <dbl>
1  31.0
```
:::
:::

::: {.cell}

```{.r .cell-code}
nofpc <- svydesign(id = ~1, weights = ~pw, data = apisrs)
nofpc
```

::: {.cell-output .cell-output-stdout}
```
Independent Sampling design (with replacement)
svydesign(id = ~1, weights = ~pw, data = apisrs)
```
:::
:::


Use `svytotal` to calculate the estimate of the total across all schools, note
that the standard error will be different between the two designs because of the
lack of fpc.


::: {.cell}

```{.r .cell-code}
svytotal(~enroll, nofpc)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 3621074 172325
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~enroll, srs_design)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 3621074 169520
```
:::
:::


Totals across groups can be calculated using the `~` notation with a categorical
variable.


::: {.cell}

```{.r .cell-code}
svytotal(~stype, srs_design)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
stypeE 4397.74 196.00
stypeH  774.25 142.85
stypeM 1022.01 160.33
```
:::
:::


`svycontrast` can be used to calculate the difference or addition of two
different estimates - below we estimate the difference in the 2000 and 1999
scores based on the SRS design.


::: {.cell}

```{.r .cell-code}
svycontrast(svymean(~ api00 + api99, srs_design), quote(api00 - api99))
```

::: {.cell-output .cell-output-stdout}
```
         nlcon     SE
contrast  31.9 2.0905
```
:::
:::


### Now again with the `srvyr` package


::: {.cell}

```{.r .cell-code}
dstrata <- apisrs %>%
  as_survey_design(fpc = fpc)
dstrata %>%
  mutate(api_diff = api00 - api99) %>%
  summarise(
    enroll_total = survey_total(enroll, vartype = "ci"),
    api_diff = survey_mean(api_diff, vartype = "ci")
  ) %>%
  gt()
```

::: {.cell-output-display}
```{=html}
<div id="rzlfxjjzhq" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#rzlfxjjzhq table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#rzlfxjjzhq thead, #rzlfxjjzhq tbody, #rzlfxjjzhq tfoot, #rzlfxjjzhq tr, #rzlfxjjzhq td, #rzlfxjjzhq th {
  border-style: none;
}

#rzlfxjjzhq p {
  margin: 0;
  padding: 0;
}

#rzlfxjjzhq .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#rzlfxjjzhq .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#rzlfxjjzhq .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#rzlfxjjzhq .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#rzlfxjjzhq .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#rzlfxjjzhq .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#rzlfxjjzhq .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#rzlfxjjzhq .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#rzlfxjjzhq .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#rzlfxjjzhq .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#rzlfxjjzhq .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#rzlfxjjzhq .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#rzlfxjjzhq .gt_spanner_row {
  border-bottom-style: hidden;
}

#rzlfxjjzhq .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#rzlfxjjzhq .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#rzlfxjjzhq .gt_from_md > :first-child {
  margin-top: 0;
}

#rzlfxjjzhq .gt_from_md > :last-child {
  margin-bottom: 0;
}

#rzlfxjjzhq .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#rzlfxjjzhq .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#rzlfxjjzhq .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#rzlfxjjzhq .gt_row_group_first td {
  border-top-width: 2px;
}

#rzlfxjjzhq .gt_row_group_first th {
  border-top-width: 2px;
}

#rzlfxjjzhq .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#rzlfxjjzhq .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#rzlfxjjzhq .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#rzlfxjjzhq .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#rzlfxjjzhq .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#rzlfxjjzhq .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#rzlfxjjzhq .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#rzlfxjjzhq .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#rzlfxjjzhq .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#rzlfxjjzhq .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#rzlfxjjzhq .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#rzlfxjjzhq .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#rzlfxjjzhq .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#rzlfxjjzhq .gt_left {
  text-align: left;
}

#rzlfxjjzhq .gt_center {
  text-align: center;
}

#rzlfxjjzhq .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#rzlfxjjzhq .gt_font_normal {
  font-weight: normal;
}

#rzlfxjjzhq .gt_font_bold {
  font-weight: bold;
}

#rzlfxjjzhq .gt_font_italic {
  font-style: italic;
}

#rzlfxjjzhq .gt_super {
  font-size: 65%;
}

#rzlfxjjzhq .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#rzlfxjjzhq .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#rzlfxjjzhq .gt_indent_1 {
  text-indent: 5px;
}

#rzlfxjjzhq .gt_indent_2 {
  text-indent: 10px;
}

#rzlfxjjzhq .gt_indent_3 {
  text-indent: 15px;
}

#rzlfxjjzhq .gt_indent_4 {
  text-indent: 20px;
}

#rzlfxjjzhq .gt_indent_5 {
  text-indent: 25px;
}

#rzlfxjjzhq .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#rzlfxjjzhq div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="enroll_total">enroll_total</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="enroll_total_low">enroll_total_low</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="enroll_total_upp">enroll_total_upp</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="api_diff">api_diff</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="api_diff_low">api_diff_low</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="api_diff_upp">api_diff_upp</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="enroll_total" class="gt_row gt_right">3621074</td>
<td headers="enroll_total_low" class="gt_row gt_right">3286789</td>
<td headers="enroll_total_upp" class="gt_row gt_right">3955360</td>
<td headers="api_diff" class="gt_row gt_right">31.9</td>
<td headers="api_diff_low" class="gt_row gt_right">27.77764</td>
<td headers="api_diff_upp" class="gt_row gt_right">36.02236</td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


## Stratified Sampling

Simple random samples are not often used in complex surveys because there is a
justified concern that some strata (e.g. racial ethnic group, age group, etc.)
may be underrepresented in the sample if a simple random sample were used.
Similarly, complex designs can give the same precision at a lower cost.
Consequently, a sample may be constructed so that some units are guaranteed to
be included within a given strata - improving the resulting variance. When this
is a simple random sample, the HTE and variance of the total population is
simply the sum of the strata specific estimates;
$\hat{T}_{HT} = \sum_{s=1}^{S} \hat{T}^{s}_X$, where there are $S$ strata within
the population.

For example, in the `apistrat` data set a stratified random sample of 200
schools is recorded such that schools are sampled randomly within school type (
elementary, middle school or high school).

In the code below we can designate the strata using the categorical variable
`stype`, which denotes each of the school type as strata.


::: {.cell}

```{.r .cell-code}
strat_design <- svydesign(
  id = ~1,
  strata = ~stype,
  fpc = ~fpc,
  data = apistrat
)
strat_design
```

::: {.cell-output .cell-output-stdout}
```
Stratified Independent Sampling design
svydesign(id = ~1, strata = ~stype, fpc = ~fpc, data = apistrat)
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~enroll, strat_design)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 3687178 114642
```
:::
:::

::: {.cell}

```{.r .cell-code}
svymean(~enroll, strat_design)
```

::: {.cell-output .cell-output-stdout}
```
         mean     SE
enroll 595.28 18.509
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~stype, strat_design)
```

::: {.cell-output .cell-output-stdout}
```
       total SE
stypeE  4421  0
stypeH   755  0
stypeM  1018  0
```
:::
:::


Note that are standard errors are 0 for the within strata estimates because if
we have strata information on each member of the population, then we know the
strata counts without any uncertainty.

Several points worth noting about stratified samples before moving on.

-   Stratified samples get their power from "conditioning" on the strata
    information that explain some of the variability in the measure --- 
    analogous to regression estimators improved variance when conditioning on 
    informative covariates.

-   Whereas a SRS might have a chance of leaving out an elementary or middle
    school, and leaving a higher estimate of enrollment, because of a higher
    number of high schools in the sample, keeping a fixed number of samples
    within each strata removes this problem.

-   Stratified analysis may also refer to something entirely different from what
    we're discussing here --- where a subgroup has some model or estimate fit
    only on that subgroup's data exclusively.

### Now again with the `srvyr` package


::: {.cell}

```{.r .cell-code}
srvyr_strat_design <- apistrat %>%
  as_survey_design(
    strata = stype,
    fpc = fpc
  )
srvyr_strat_design
```

::: {.cell-output .cell-output-stdout}
```
Stratified Independent Sampling design
Called via srvyr
Sampling variables:
  - ids: `1` 
  - strata: stype 
  - fpc: fpc 
Data variables: 
  - cds (chr), stype (fct), name (chr), sname (chr), snum (dbl), dname (chr),
    dnum (int), cname (chr), cnum (int), flag (int), pcttest (int), api00
    (int), api99 (int), target (int), growth (int), sch.wide (fct), comp.imp
    (fct), both (fct), awards (fct), meals (int), ell (int), yr.rnd (fct),
    mobility (int), acs.k3 (int), acs.46 (int), acs.core (int), pct.resp (int),
    not.hsg (int), hsg (int), some.col (int), col.grad (int), grad.sch (int),
    avg.ed (dbl), full (int), emer (int), enroll (int), api.stu (int), pw
    (dbl), fpc (dbl)
```
:::
:::

::: {.cell}

```{.r .cell-code}
srvyr_strat_design %>%
  summarise(
    enroll_total = survey_total(enroll),
    enroll_mean = survey_mean(enroll)
  ) %>%
  gt()
```

::: {.cell-output-display}
```{=html}
<div id="ezhynegmqh" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#ezhynegmqh table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#ezhynegmqh thead, #ezhynegmqh tbody, #ezhynegmqh tfoot, #ezhynegmqh tr, #ezhynegmqh td, #ezhynegmqh th {
  border-style: none;
}

#ezhynegmqh p {
  margin: 0;
  padding: 0;
}

#ezhynegmqh .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#ezhynegmqh .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#ezhynegmqh .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#ezhynegmqh .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#ezhynegmqh .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ezhynegmqh .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ezhynegmqh .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ezhynegmqh .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#ezhynegmqh .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#ezhynegmqh .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#ezhynegmqh .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#ezhynegmqh .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#ezhynegmqh .gt_spanner_row {
  border-bottom-style: hidden;
}

#ezhynegmqh .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#ezhynegmqh .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#ezhynegmqh .gt_from_md > :first-child {
  margin-top: 0;
}

#ezhynegmqh .gt_from_md > :last-child {
  margin-bottom: 0;
}

#ezhynegmqh .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#ezhynegmqh .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#ezhynegmqh .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#ezhynegmqh .gt_row_group_first td {
  border-top-width: 2px;
}

#ezhynegmqh .gt_row_group_first th {
  border-top-width: 2px;
}

#ezhynegmqh .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ezhynegmqh .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#ezhynegmqh .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#ezhynegmqh .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ezhynegmqh .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ezhynegmqh .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#ezhynegmqh .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#ezhynegmqh .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#ezhynegmqh .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ezhynegmqh .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ezhynegmqh .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ezhynegmqh .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ezhynegmqh .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ezhynegmqh .gt_left {
  text-align: left;
}

#ezhynegmqh .gt_center {
  text-align: center;
}

#ezhynegmqh .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#ezhynegmqh .gt_font_normal {
  font-weight: normal;
}

#ezhynegmqh .gt_font_bold {
  font-weight: bold;
}

#ezhynegmqh .gt_font_italic {
  font-style: italic;
}

#ezhynegmqh .gt_super {
  font-size: 65%;
}

#ezhynegmqh .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#ezhynegmqh .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#ezhynegmqh .gt_indent_1 {
  text-indent: 5px;
}

#ezhynegmqh .gt_indent_2 {
  text-indent: 10px;
}

#ezhynegmqh .gt_indent_3 {
  text-indent: 15px;
}

#ezhynegmqh .gt_indent_4 {
  text-indent: 20px;
}

#ezhynegmqh .gt_indent_5 {
  text-indent: 25px;
}

#ezhynegmqh .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#ezhynegmqh div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="enroll_total">enroll_total</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="enroll_total_se">enroll_total_se</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="enroll_mean">enroll_mean</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="enroll_mean_se">enroll_mean_se</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="enroll_total" class="gt_row gt_right">3687178</td>
<td headers="enroll_total_se" class="gt_row gt_right">114641.7</td>
<td headers="enroll_mean" class="gt_row gt_right">595.2821</td>
<td headers="enroll_mean_se" class="gt_row gt_right">18.50851</td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::

::: {.cell}

```{.r .cell-code}
srvyr_strat_design %>%
  group_by(stype) %>%
  survey_count()
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 3 × 3
# Groups:   stype [3]
  stype     n  n_se
  <fct> <dbl> <dbl>
1 E     4421.     0
2 H      755.     0
3 M     1018.     0
```
:::
:::


## Replicate Weights

Replicate weights exploit sub-sampling to derive more generalizable statistics
than sampling weights. This is particularly useful when estimating a
"nonparametric" statistic like the median or a quantile which doesn't have an
easily derived variance.

For a basic idea of why this works, Lumley notes that one could estimate the
variance of a total by using two independent half samples estimating the same
total, i.e. if $\hat{T}_A$ and $\hat{T}_B$ are both from two independent half
samples estimating $\hat{T}$ then the variance of the difference of the two half
samples is proportional to the variance of the original total:

$$
E\left[ (\hat{T}_A - \hat{T}_B)^2 \right] = 2 V[\hat{T}_A] = 4 V[\hat{T}].
$$

There are multiple ways one might set up these splits that are more efficient
than the straightforward "half" sample described above - Lumley discusses 3
variants briefly:

1.  [Balanced Repeated
    Replication](https://en.wikipedia.org/wiki/Balanced_repeated_replication)
    (BRR) Based on the work of [@mccarthy1966replication].

-   [@judkins1990fay], extends BRR to handle issues with sparse signals and
    small samples.

2.  [Jackknife](https://en.wikipedia.org/wiki/Jackknife_resampling)

-   Because BRR and Fay's method is difficult with other designs using
    overlapping subsamples, Jackknife and the bootstrap are intended to be more
    flexible.

3.  [Bootstrap](https://en.wikipedia.org/wiki/Bootstrapping_(statistics))

-   This is the method I'm most familiar with, outside of complex designs.
-   Lumley states that using the Bootstrap in this setting involves taking a
    sample (with replacement) of observations or clusters and multiplying the
    sampling weight by the number of times the observation appears in the
    sample.

Each of these ideas relies on the fundamental idea that we can calculate the
variance of our statistic of interest by using --- sometimes carefully chosen
--- subsamples of our original sample to calculate our statistic of interest and
more importantly, the variance of that statistic. Lumley's use of the equation
above gives the basic idea but I believe the more rigorous justification appeals
to theory involving [empiricial distribution
functions](https://en.wikipedia.org/wiki/Empirical_distribution_function), as
much of the theory underlying these ideas relies on getting a good estimate of
the empirical distribution.

It isn't explicitly clear which of these techniques is most popular currently,
but my guess would be that the bootstrap is the most used. This also happens to
be the method that Lumley has provided the most citations for in the text. I've
also [run into](https://github.com/walkerke/tidycensus/issues/552) cases where
the US Census IPUMS data [uses](https://usa.ipums.org/usa/repwt.shtml#q50)
[successive difference
weights](http://www.asasrms.org/Proceedings/y2011/Files/302108_67867.pdf).

All this to say that replicate weights are powerful for producing
"non-parametric" estimates, like quantiles and so on, and different weighting
techniques may be more or less appropriate depending on the design and data
involved.

### Replicate Weights in R

Lumley first demonstrates how to setup a survey design object when the weights
are already provided. I've had trouble accessing the 2005 California Health
Interview Survey
[data](http://healthpolicy.ucla.edu/chis/data/Pages/GetCHISData.aspx) on my own
but he thankfully provides a link to the data on his
[website](https://r-survey.r-forge.r-project.org/svybook/).


::: {.cell}

```{.r .cell-code}
chis_adult <- as.data.frame(read_dta("Data/ADULT.dta")) %>%
  # have to convert labeled numerics to regular numerics for
  # computation in survey package.
  mutate(
    bmi_p = as.numeric(bmi_p),
    srsex = factor(srsex, labels = c("MALE", "FEMALE")),
    racehpr = factor(racehpr, labels = c(
      "LATINO", "PACIFIC ISLANDER",
      "AMERICAN INDIAN/ALASKAN NATIVE",
      "ASIAN", "AFRICAN AMERICAN",
      "WHITE",
      "OTHER SINGLE/MULTIPLE RACE"
    ))
  )
chis <- svrepdesign(
  variables = chis_adult[, 1:418],
  repweights = chis_adult[, 420:499],
  weights = chis_adult[, 419, drop = TRUE],
  ## combined.weights specifies that the replicate weights
  ## include the sampling weights
  combined.weights = TRUE,
  type = "other", scale = 1, rscales = 1
)
chis
```

::: {.cell-output .cell-output-stdout}
```
Call: svrepdesign.default(variables = chis_adult[, 1:418], repweights = chis_adult[, 
    420:499], weights = chis_adult[, 419, drop = TRUE], combined.weights = TRUE, 
    type = "other", scale = 1, rscales = 1)
with 80 replicates.
```
:::
:::


When *creating* replicate weights in R one specifies a replicate type to the
`type` argument.


::: {.cell}

```{.r .cell-code}
boot_design <- as.svrepdesign(strat_design,
  type = "bootstrap",
  replicates = 100
)
boot_design
```

::: {.cell-output .cell-output-stdout}
```
Call: as.svrepdesign.default(strat_design, type = "bootstrap", replicates = 100)
Survey bootstrap with 100 replicates.
```
:::
:::


By default, the `as.svrepdesign()` function assumes the replicate weights are
jackknife replicates.


::: {.cell}

```{.r .cell-code}
## jackknife is the default
jk_design <- as.svrepdesign(strat_design)
jk_design
```

::: {.cell-output .cell-output-stdout}
```
Call: as.svrepdesign.default(strat_design)
Stratified cluster jackknife (JKn) with 200 replicates.
```
:::
:::


Once the design object is created the mean of a variable can computed
equivalently as before using the `svymean()` function. We'll compare the
bootstrap and jackknife estimates, noting that the bootstrap has a higher
standard error than the jackknife.


::: {.cell}

```{.r .cell-code}
svymean(~enroll, boot_design)
```

::: {.cell-output .cell-output-stdout}
```
         mean     SE
enroll 595.28 21.113
```
:::
:::

::: {.cell}

```{.r .cell-code}
svymean(~enroll, jk_design)
```

::: {.cell-output .cell-output-stdout}
```
         mean     SE
enroll 595.28 18.509
```
:::
:::


Of course, part of the motivation in using replicate weights is that you're able
to estimate standard errors for non-trivial estimands, especially those that may
not be implemented in the `survey` package. Lumley demonstrates this using a
sample from the [National Wilms Tumor Study
Cohort](https://pubmed.ncbi.nlm.nih.gov/18027087/), in order to estimate the
five year survival probability via a
[Kaplan-Meier](https://en.wikipedia.org/wiki/Kaplan%E2%80%93Meier_estimator)
Estimator.


::: {.cell}

```{.r .cell-code}
library(addhazard)
nwtsco <- as_tibble(nwtsco)
head(nwtsco)
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 6 × 12
   trel  tsur relaps  dead study stage histol instit   age    yr specwgt tumdiam
  <dbl> <dbl>  <int> <int> <int> <int>  <int>  <int> <dbl> <int>   <int>   <int>
1 21.9  21.9       0     0     3     1      1      1  2.08  1979     750      14
2 11.3  11.3       0     0     3     2      0      0  4.17  1979     590       9
3 22.1  22.1       0     0     3     1      1      1  0.75  1979     356      13
4  8.02  8.02      0     0     3     2      0      0  2.67  1979     325       9
5 20.5  20.5       0     0     3     2      0      0  3.67  1979     970      17
6 14.4  14.4       1     1     3     2      0      1  2.58  1979     730      15
```
:::
:::

::: {.cell}

```{.r .cell-code}
cases <- nwtsco %>% filter(relaps == 1)
cases <- cases %>% mutate(wt = 1)
ctrls <- nwtsco %>% filter(relaps == 0)
ctrls <- ctrls %>%
  mutate(wt = 10) %>%
  sample_n(325)
ntw_sample <- rbind(cases, ctrls)

fivesurv <- function(time, status, w) {
  scurve <- survfit(Surv(time, status) ~ 1, weights = w)
  ## minimum probability that corresponds to a survival time > 5 years
  return(scurve$surv[min(which(scurve$time > 5))])
}

des <- svydesign(id = ~1, strata = ~relaps, weights = ~wt, data = ntw_sample)
jkdes <- as.svrepdesign(des)
withReplicates(jkdes, quote(fivesurv(trel, relaps, .weights)))
```

::: {.cell-output .cell-output-stdout}
```
       theta     SE
[1,] 0.83669 0.0016
```
:::
:::


The estimated five year survival probability of 84% (95% CI: 84%,85%) uses the
`fivesurv` function which computes the kaplan meier estimate of five year
survival probability fora given time status and weight. The `withReplicates`
function then re-estimates this statistic using each set of replicates and
calculates the standard error from the variability of these estimates.

Its worth noting that this is the standard error for estimating the five year
survival in the NWTS cohort, not the hypothetical superpopulation of all
children with Wilms' tumor.

### Now again with the `srvyr` package


::: {.cell}

```{.r .cell-code}
boot_design <- as_survey_rep(strat_design,
  type = "bootstrap",
  replicates = 100
)
boot_design
```

::: {.cell-output .cell-output-stdout}
```
Call: Called via srvyr
Survey bootstrap with 100 replicates.
Data variables: 
  - cds (chr), stype (fct), name (chr), sname (chr), snum (dbl), dname (chr),
    dnum (int), cname (chr), cnum (int), flag (int), pcttest (int), api00
    (int), api99 (int), target (int), growth (int), sch.wide (fct), comp.imp
    (fct), both (fct), awards (fct), meals (int), ell (int), yr.rnd (fct),
    mobility (int), acs.k3 (int), acs.46 (int), acs.core (int), pct.resp (int),
    not.hsg (int), hsg (int), some.col (int), col.grad (int), grad.sch (int),
    avg.ed (dbl), full (int), emer (int), enroll (int), api.stu (int), pw
    (dbl), fpc (dbl)
```
:::
:::

::: {.cell}

```{.r .cell-code}
boot_design %>% summarise(Mean = survey_mean(enroll))
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
   Mean Mean_se
  <dbl>   <dbl>
1  595.    22.1
```
:::
:::


It's not clear or straightforward to me from reading the `srvyr`
[docs](http://gdfe.co/srvyr/articles/extending-srvyr.html) how to estimate the
weighted survival function probability --- I may return to this later.

### Final Notes on Replicate Weights

Lumley finishes this section by noting that the bootstrap typically works better
when all the strata are large. While a strata correction is available it is
likely not correct for small or unequal strata.

Separately, Lumley note that both the jackknife and bootstrap can incorporate
finite population correction factors.

Finally, the BRR designs implemented in the `survey` package will use at most
excess 4 replicate splits for $K < 180$ and at most 5% when $K > 100$. It is not
clear to me from the reading, which is more likely to be used for
$100 < K < 180$.

## Other population summaries

While population means, totals, and differences are typically easy to estimate
via the horvitz thompson estimator there are other population statistics such as
the median or regression estimates that are more complex. These require using
the replicate weights described in the previous [section](#Replicate%20Weights)
or making certain linearization / interpolation assumptions which may or may not
influence the resulting estimate.

### Quantiles

Estimation of quantiles involves estimating arbitary points along the
[cumulative distribution
function](https://en.wikipedia.org/wiki/Cumulative_distribution_function)(cdf).
For example, the 90th percentile has 90% of the estimated population size below
it and 10% above. In this case, for cdf $F_X(x)$, we want to estimate
$x: F_X(x) = 0.9$. However, estimating the cdf presents some technical
difficulties in that the [empirical cumulative distribution
function](https://en.wikipedia.org/wiki/Empirical_distribution_function) (ecdf),
is not typically a "smooth" estimate for any given $x$ --- as the estimate is
highly dependent upon the sample. Consequently, Lumley's function,
`svyquantile()` interpolates linearly between two adjacent observations when the
quantile is not uniquely defined.


::: {.cell}

```{.r .cell-code}
samp <- rnorm(20)
plot(ecdf(samp))
```

::: {.cell-output-display}
![Empirical Cumulative Distribution Function - note the jumps at distinctive points along the x-axis.](ComplexSurveyNotes_files/figure-html/sample_ecdf-1.png){width=672}
:::
:::


Confidence intervals are constructed similarly, using the ecdf, though it should
be noted that estimating extreme quantiles poses difficulties because of the low
density values in the area.

A first calculation to demonstrate this using replicate weights with the CA
health interview study, estimating different quantiles of BMI.


::: {.cell}

```{.r .cell-code}
svyquantile(~bmi_p, design = chis, quantiles = c(0.25, 0.5, 0.75))
```

::: {.cell-output .cell-output-stdout}
```
$bmi_p
     quantile ci.2.5 ci.97.5         se
0.25    22.68  22.66   22.81 0.03767982
0.5     25.75  25.69   25.80 0.02763161
0.75    29.18  29.12   29.29 0.04270393

attr(,"hasci")
[1] TRUE
attr(,"class")
[1] "newsvyquantile"
```
:::
:::


The same thing can be done with the stratified design. Here the uncertainty is
computed via the estimates of the ecdf and finding the pointwise confidence
interval for different points along the curve.


::: {.cell}

```{.r .cell-code}
svyquantile(~api00,
  design = strat_design, quantiles = c(0.25, 0.5, 0.75),
  ci = TRUE
)
```

::: {.cell-output .cell-output-stdout}
```
$api00
     quantile ci.2.5 ci.97.5       se
0.25      565    535     597 15.71945
0.5       668    642     694 13.18406
0.75      756    726     778 13.18406

attr(,"hasci")
[1] TRUE
attr(,"class")
[1] "newsvyquantile"
```
:::
:::


You can see how to construct the same estimate below using the `srvyr` package.


::: {.cell}

```{.r .cell-code}
srvyr_strat_design %>%
  summarize(quantiles = survey_quantile(api00, quantiles = c(0.25, 0.5, 0.75)))
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 6
  quantiles_q25 quantiles_q50 quantiles_q75 quantiles_q25_se quantiles_q50_se
          <dbl>         <dbl>         <dbl>            <dbl>            <dbl>
1           565           668           756             15.7             13.2
# ℹ 1 more variable: quantiles_q75_se <dbl>
```
:::
:::


### Contingency Tables

Lumley's main points in this section focus on the complications in
interpretation of typical contingency table tests of association in a design
based setting. Specifically, he points out that it is not obvious how the null
distribution should be generated without making some kind of modeling
assumptions. Quoting from the book (text in parentheses from me):

> For example, if there are 56,181,887 women and 62,710,561 men in a population
> it is not possible for the proportions of men and women who are unemployed to
> be the same, since these population sizes have no common factors. We would
> know without collecting any employment data that the finite-population null
> hypothesis (of equal proportions) was false. A more interesting question is
> whether the finite population could have arisen from a process that had no
> association between the variables: is the difference at the population level
> small enough to have arisen by chance.... A simpler approach is to treat the
> sample as if it came from an infinite superpopulation and simply ignore the
> finite-population corrections in inference.

The super-population approach offers the more interesting approach
philosophically and thus is implemented in the `survey` package. The `svychisq`
function implements a test for no association as the null using a chi-squared
distribution with a correction for the mean and variance. Lumley discusses
various methods for computing the $\chi^2$ statistic in this setting and their
implementations in `svycontrast()`. I'd suggest looking at the function
documentation if that level of detail is needed.

Lumley demonstrates how to call these functions estimating the proportion of
smokers in each insurance status group from the California Health Interview
Survey.


::: {.cell}

```{.r .cell-code}
tab <- svymean(~ interaction(ins, smoking, drop = TRUE), chis)
tab
```

::: {.cell-output .cell-output-stdout}
```
                                              mean     SE
interaction(ins, smoking, drop = TRUE)1.1 0.112159 0.0021
interaction(ins, smoking, drop = TRUE)2.1 0.039402 0.0015
interaction(ins, smoking, drop = TRUE)1.2 0.218909 0.0026
interaction(ins, smoking, drop = TRUE)2.2 0.026470 0.0012
interaction(ins, smoking, drop = TRUE)1.3 0.507728 0.0036
interaction(ins, smoking, drop = TRUE)2.3 0.095332 0.0022
```
:::
:::

::: {.cell}

```{.r .cell-code}
ftab <- ftable(tab, rownames = list(
  ins = c("Insured", "Uninsured"),
  smoking = c("Current", "Former", "Never")
))
round(ftab * 100, 1)
```

::: {.cell-output .cell-output-stdout}
```
             ins Insured Uninsured
smoking                           
Current mean        11.2       3.9
        SE           0.2       0.1
Former  mean        21.9       2.6
        SE           0.3       0.1
Never   mean        50.8       9.5
        SE           0.4       0.2
```
:::
:::


In the output below we see a very small p-value indicating that the data were
unlikely to be generated from a *process* in which smoking and insurance status
were independent.


::: {.cell}

```{.r .cell-code}
svychisq(~ smoking + ins, chis)
```

::: {.cell-output .cell-output-stdout}
```

	Pearson's X^2: Rao & Scott adjustment

data:  svychisq(~smoking + ins, chis)
F = 130.11, ndf = 1.9923, ddf = 157.3884, p-value < 2.2e-16
```
:::
:::


### Estimates in Subpopulations

Estimation within subpopulations (also called domain estimation) that are
sampled strata is easy since a stratified sample is composed of random samples
within strata by definition; simply compute the desired statistic within the
given strata using the strata-specific random sample.

When the subpopulation of interest is not a strata, things are more difficult.
While the sampling weights would still be correct for representing any given
observation to the total population --- resulting in an unbiased mean point
estimate --- the co-occurrence probabilities $\pi_{i,j}$ would be incorrect,
because the co-occurrence probabilities are now unknown/random and not fixed by
design. The two (usual) approaches for trying to estimate subpopulation variance
estimates in-spite of these difficulties are
[linearization](https://en.wikipedia.org/wiki/Linearization#:~:text=In%20mathematics%2C%20linearization%20is%20finding,around%20the%20point%20of%20interest.)
and replicate weighting. The computation is straightforward for replicate
weighting since the non-subpopulation entities can simply be discarded in the
computation. For linearization the computation is less straightforward as the
extra entities still have to be included as "0"s in the final computation --
this is according to Lumley's arguments in his Appendix.

Examples below demonstrating this idea estimate the number of teachers with
emergency, `emer`, training amongst California schools using the `api` dataset.


::: {.cell}

```{.r .cell-code}
emerg_high <- subset(strat_design, emer > 20)
emerg_low <- subset(strat_design, emer == 0)
svymean(~ api00 + api99, emerg_high)
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
api00 558.52 21.708
api99 523.99 21.584
```
:::
:::

::: {.cell}

```{.r .cell-code}
svymean(~ api00 + api00, emerg_low)
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
api00 749.09 17.516
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~enroll, emerg_high)
```

::: {.cell-output .cell-output-stdout}
```
        total     SE
enroll 762132 128674
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~enroll, emerg_low)
```

::: {.cell-output .cell-output-stdout}
```
        total    SE
enroll 461690 75813
```
:::
:::


In general, if replicate weights are available, domain estimation is much
easier.


::: {.cell}

```{.r .cell-code}
bys <- svyby(~bmi_p, ~ srsex + racehpr, svymean,
  design = chis,
  keep.names = FALSE
)
print(bys, digits = 3)
```

::: {.cell-output .cell-output-stdout}
```
    srsex                        racehpr bmi_p     se
1    MALE                         LATINO  28.2 0.1447
2  FEMALE                         LATINO  27.5 0.1443
3    MALE               PACIFIC ISLANDER  29.7 0.7055
4  FEMALE               PACIFIC ISLANDER  27.8 0.9746
5    MALE AMERICAN INDIAN/ALASKAN NATIVE  28.8 0.5461
6  FEMALE AMERICAN INDIAN/ALASKAN NATIVE  27.0 0.4212
7    MALE                          ASIAN  24.9 0.1406
8  FEMALE                          ASIAN  23.0 0.1112
9    MALE               AFRICAN AMERICAN  28.0 0.2663
10 FEMALE               AFRICAN AMERICAN  28.4 0.2417
11   MALE                          WHITE  27.0 0.0598
12 FEMALE                          WHITE  25.6 0.0680
13   MALE     OTHER SINGLE/MULTIPLE RACE  26.9 0.3742
14 FEMALE     OTHER SINGLE/MULTIPLE RACE  26.7 0.3158
```
:::
:::

::: {.cell}

```{.r .cell-code}
# This is the code from the book but it didn't work for me
# because of issues in the survey R package, I reproduce the
# first result using the srvyr package below
# medians <- svyby(~bmi_p, ~ srsex + racehpr, svyquantile,
#   design = chis,
#   covmat = TRUE,
#   quantiles = 0.5
# )
# svycontrast(medians, quote(MALE.LATINO/FEMALE.LATINO))

medians <- chis %>%
  as_survey() %>%
  group_by(srsex, racehpr) %>%
  summarize(Median_BMI = survey_median(bmi_p, vartype = "ci"))

medians
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 14 × 5
# Groups:   srsex [2]
   srsex  racehpr                       Median_BMI Median_BMI_low Median_BMI_upp
   <fct>  <fct>                              <dbl>          <dbl>          <dbl>
 1 MALE   LATINO                              27.4           27.3           27.8
 2 MALE   PACIFIC ISLANDER                    28.9           27.8           30.7
 3 MALE   AMERICAN INDIAN/ALASKAN NATI…       28.4           26.9           29.3
 4 MALE   ASIAN                               24.4           24.2           24.8
 5 MALE   AFRICAN AMERICAN                    27.4           26.6           28.1
 6 MALE   WHITE                               26.4           26.3           26.5
 7 MALE   OTHER SINGLE/MULTIPLE RACE          26.6           25.9           27.5
 8 FEMALE LATINO                              26.3           25.8           26.5
 9 FEMALE PACIFIC ISLANDER                    27.3           25.6           28.3
10 FEMALE AMERICAN INDIAN/ALASKAN NATI…       25.1           24.5           26.5
11 FEMALE ASIAN                               22.1           22.0           22.4
12 FEMALE AFRICAN AMERICAN                    27.2           26.6           27.5
13 FEMALE WHITE                               24.3           24.2           24.4
14 FEMALE OTHER SINGLE/MULTIPLE RACE          25.7           25.1           26.5
```
:::
:::


## Design of Stratified Samples

How to pick the sample size for each strata? Well it depends on the goals of the
analysis. If the goal is to estimate a total across the whole population, the
formula for the variance of a total can be used to gain insights about optimal
allocation. Since the variance of the total is dependent (via sum) of the strata
specific variances, more sample size would want to be dedicated to more
heterogeneous and/or larger strata.

This general approach means that the sample size for strata $k$, $n_k$ should be
proportional to the population strata size $N_k$ and strata variance
$\sigma^{2}_k$, $n_k \propto \sqrt{N^2_k \sigma^2_k} = N_k \sigma_k$. Lumley
notes that while this expression satisfies some theoretical optimality criteria,
it is often the case that different strata have different costs associated with
their sampling and so the expression can be modified in order to take into
account this cost as follows:

$$
n_k \propto \frac{ N_k \sigma_k}{\sqrt{\text{cost}_k}},
$$

where cost$_k$ is the cost of sampling for strata $k$.

## Exercises

1.You are conducting a survey of emergency preparedness at a large HMO, where
you want to estimate what proportion of medical staff would be able to get to
work after an earthquake.

  * You can either send out a single questionnaire to all staff, or send out a
  questionnaire to about 10% of the staff and make follow-up phone calls for those 
  that don't respond. What are the disadvantages of each approach?

This comes down to a discussion of cost for sampling and what missing data
mechanism may be at play. As a simple starting point, if we were to assume the
resulting data were MCAR and the non response rate was equivalent between both
sampling strategies, the single questionnaire would be preferred because it
would result in a higher overall sample size. These assumptions are probably not
likely however, and we may expect that non-response is associated with other
meaningful factors, by choosing a the follow-up phone call we might minimize
non-response to both reduce bias and improve precision.

Additional relevant concerns would be the possible response or lack of response
of certain strata --- certain physicians, technicians or other kinds of staff's
response would likely be worth knowing yet these groups may be less well
represented in a 10% simple random sample of the population.

  * You choose to survey just a sample. What would be useful variables to 
  stratify the sampling, and why?

The aforementioned job title would be useful to stratify on. This would likely
be most useful to conduct within each department. Further, if the HMO has more
than one site or clinic, that would be worth stratifying on as well for
substantive reasons just as much as statistical reasons.

  * The survey was conducted with just two strata: physicians and other staff.
The HMO has 900 physicians and 9000 other staff. You sample 450 physicians and
450 other staff. What are the sampling probabilities in each stratum?

Physician strata sampling probabilities are
$\frac{n}{N_k} = \frac{450}{900} = \frac{1}{2}$, while the "other staff"
probabilities are $\frac{450}{9000} = \frac{1}{20}$.

  * 300 physicians and 150 other staff say they would be able to get to work 
  after an earthquake. Give unbiased estimates of the proportion in each stratum 
  and the total proportion.

The physician strata estimate would be $\frac{300}{450} = \frac{2}{3}$. The
staff strata would be $\frac{150}{450} = \frac{1}{3}$ The total proportion would
be $\frac{2 \times 300 + 20 \times 150}{9900}$. This value can be recreated
below with the `survey` package as follows.


::: {.cell}

```{.r .cell-code}
df <- tibble(
  id = 1:900,
  job = c(rep("MD", 450), rep("staff", 450)),
  prep = c(rep(1, 300), rep(0, 150), rep(1, 150), rep(0, 300)),
  weights = c(rep(2, 450), rep(20, 450))
)
hmo_design <- svydesign(strata = ~job, ids = ~0, weights = ~weights, data = df)
hmo_design
```

::: {.cell-output .cell-output-stdout}
```
Stratified Independent Sampling design (with replacement)
svydesign(strata = ~job, ids = ~0, weights = ~weights, data = df)
```
:::
:::

::: {.cell}

```{.r .cell-code}
svymean(~prep, hmo_design)
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
prep 0.36364 0.0203
```
:::
:::


  * How would you explain to the managers that commissioned the study how the 
  estimate was computed and why it wasn't just the number who said "yes" divided 
  by the total number surveyed?

We sampled from the total population using the strata because we though these
two groups would respond differently and indeed, they did. Physicians have are
twice as likely to be able to make it to the hospital in the event of an
emergency as general staff. However, physicians make up a much smaller
proportion of the overall hospital workforce and so we need to down weight their
responses, relative to general staff in order to ensure their response reflects
their distribution in the total population, thus the total estimate of the HMO's
emergency preparedness is much closer to the "general" staff's strata estimate
of $\frac{1}{3}$.

2.You are conducting a survey of commuting time and means of transport for a
large university. What variables are likely to be available and useful for
stratifying the sampling?

Probably worth stratifying on "role" at university --- student vs. staff vs.
professor. Each of these have varying amounts of income available and would
likely determine their different means and, consequently, commute time of
getting to campus. It might also be worth stratifying on the department of
employment for staff and professors, as there can be a wide variability in these
measures, again, by department.

3.-4. Skip because of CHIS data issues

5.  In the Academic Performance Index data we saw large gains in precision from
    stratification on school type when estimating mean or total school size, and
    no gain when estimating mean Academic performance Index. Would you expect a
    large or small gain from the following variables: `mobility`, `emer`,
    `meals`, `pcttest`? Compare your expectations with the actual results.

The general principle here, is which of these variables do we expect to have
some association with the school type. The greater association the more the
benefit from stratifying.

6.  For estimating total school enrollment in the Academic Performance Index
    population, what is the optimal allocation of a total sample size of 200
    stratified by school size? Draw a sample with this optimal allocation and
    compare the standard errors to the stratified sample in Figure 2.5 for:
    total enrollment, mean 2000 API, mean `meals`, mean `ell`.

A first point worth noting is that school size is an integer valued variable and
so some grouping will have to be created to define the strata from which schools
are then drawn. One possible option is to define the strata as above and below
the median school enrollment. Since this divides the population exactly in half
the strata are equally sized and the only differentiating factor is the
variability of the enrollment.


::: {.cell}

```{.r .cell-code}
as_tibble(apipop) %>%
  transmute(
    enroll = enroll,
    strat = if_else(enroll > median(enroll, na.rm = TRUE), "Above", "Below")
  ) %>%
  group_by(strat) %>%
  summarize(sd_enroll = sd(enroll))
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 3 × 2
  strat sd_enroll
  <chr>     <dbl>
1 Above     504. 
2 Below      89.8
3 <NA>       NA  
```
:::
:::


Since the variability of the school enrollment sizes in the above median size
schools is roughly five times that in the below median size schools, we'd sample
from the two strata at a 5:1 ratio, respectively. Or, explicitly, we'd sample
160 schools from the above median school size and 40 schools from the below
median school size.


::: {.cell}

```{.r .cell-code}
api_opt_strat <- as_tibble(apipop) %>%
  filter(!is.na(enroll)) %>%
  transmute(
    enroll = enroll,
    strat = if_else(enroll > median(enroll, na.rm = TRUE), "Above", "Below")
  )

above <- api_opt_strat %>%
  filter(strat == "Above") %>%
  mutate(fpc = n()) %>%
  slice_sample(n = 160)

below <- api_opt_strat %>%
  filter(strat == "Below") %>%
  mutate(fpc = n()) %>%
  slice_sample(n = 40)

opt_strat_design <- svydesign(
  ids = ~1, strata = ~strat, fpc = ~fpc,
  data = rbind(above, below),
)

svytotal(~enroll, opt_strat_design)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 3780158 113922
```
:::
:::


The mean estimate is slightly closer to the true value,
3811472 but the standard error is slightly larger (119K vs
114K) compared to the previous stratified design. However, the other design
sampled 1000s of schools per strata so this is remarkably efficient.

I won't perform the other comparisons, but one would expect this design to be
much less efficient at estimating the other variables if they are not well
correlated with enrollment size.

7.  Figure 2.1 shows that the mean school size (enroll) in simple random samples
    of size 200 from the Academic Performance Index population has close to a
    Normal distribution.

  * Construct similar graphs for SRS of size 200, 50, 25, 10.

I've already done something like this at the start of the chapter. We'd expect
the central limit theorem (the normality of the sample means) to be better for
the larger sample sizes listed. Or rather, we might not trust the standard error
for the sample size of 10 to really represent the variability of the sample
mean.

  * Repeat for median school size.


::: {.cell}

```{.r .cell-code}
pop_median <- median(apipop$enroll)
p1 <- apipop %>%
  ggplot(aes(x = enroll)) +
  geom_histogram() +
  xlab("Student Enrollment") +
  geom_vline(xintercept = pop_median, linetype = 2, color = "red") +
  ggtitle("Distribution of School Enrollment",
    subtitle = "Median Enrollment Identified"
  )

p <- replicate(n = 500, {
  apipop %>%
    sample_n(200) %>%
    pull(enroll) %>%
    median(., na.rm = TRUE)
})

p <- tibble(sample_ix = 1:500, sample_median = p) %>%
  ggplot(aes(x = sample_median)) +
  geom_histogram() +
  xlab("Student Enrollment Medians") +
  geom_vline(
    xintercept = pop_median,
    linetype = 2, color = "red"
  ) +
  ggtitle("Distribution of Sample Medians")
p1 + p
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/sample_median_CLT-1.png){width=672}
:::
:::


  * Repeat for mean school size in stratified samples of size 100, 52, 24, 12
    using the same stratification proportions (50% elementary, 25% middle
    schools, 25% high schools) as in the built-in stratified sample.

The same result holds since the samples are independent within and across strata
as well.


::: {.cell}

```{.r .cell-code}
ns <- c(100, 52, 24, 12)
simdf <- map_dfr(1:200, function(sim_ix) {
  map_dfr(ns, function(n) {
    apipop %>%
      filter(stype == "E") %>%
      mutate(fpc = n()) %>%
      slice_sample(n = n * .5) %>%
      rbind(
        .,
        apipop %>%
          filter(stype != "E") %>%
          group_by(stype) %>%
          mutate(fpc = n()) %>%
          slice_sample(n = n * .25)
      ) %>%
      as_survey_design(strata = stype, fpc = fpc) %>%
      summarize(mean_enroll = survey_mean(enroll, na.rm = TRUE)) %>%
      mutate(sim_ix = sim_ix, n = n)
  })
})
simdf %>%
  ggplot(aes(x = mean_enroll, fill = factor(n))) +
  geom_histogram() +
  theme(legend.title = element_blank()) +
  ggtitle("CLT for stratified samples of different sizes.")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q3_7_c-1.png){width=672}
:::
:::


8.  In a design with just two strata write the sample sizes as $n_1$ and $n-n_1$
    so that there is only one quantity that can be varied. Differentiate the
    variance of the total with respect to $n_1$ to find the optimal allocation
    for two strata. Extend this to any number of strata by using the fact that
    an optimal allocation cannot be improved by moving samples from one stratum
    to another stratum.
    
$$
\hat{V}[\hat{T}] = V_1 + V_2 \\
V_1 = \frac{N_1 - n_1}{N_1} N_1^2 \frac{\sigma^2_1}{n_1} \\ 
V_2 = \frac{N_2 - (n - n_1)}{N_2} N_2^2 \frac{\sigma^2_2}{n_2}
$$
taking the derivative ...

$$
\frac{d\hat{V}[\hat{T}]}{dn_1} = \frac{dV_1}{dn_1} + \frac{dV_2}{dn_1} \\
= \frac{-N_1^2 \sigma^2_1}{n_1^2} + \frac{N_2^2 \sigma_2^2}{(n - n_1)^2}
$$
setting to zero and and solving for $n_1$ we get
$$
\frac{-N_1^2 \sigma^2_1}{n_1^2} + \frac{N_2^2 \sigma_2^2}{(n - n_1)^2} = 0 \\
\iff \\
\frac{N_2^2\sigma^2_2}{(n-n_1)^2} = \frac{N_1^2\sigma_1^2}{n_1^2}\\
\iff \\
\frac{n_1^2}{(n-n_1)^2} = \frac{N_1^2\sigma_1^2}{N_2^2\sigma^2_2} \\
\iff \\
\frac{n_1}{(n-n_1)} = \frac{N_1\sigma_1}{N_2\sigma}  \\
\iff \\ 
n_1 = \frac{nN_1\sigma_1}{N_2\sigma_2(1 + \frac{N_1\sigma_1}{N_2\sigma_2})} \\
 = \frac{nN_1\sigma_1}{N_2\sigma_2 + N_1\sigma_1}
$$

Where the square root is taken over variables constrained to be positive so
we only have the positive values as the solution to the equation.

Also, $N_1, N_2$ are taken to be the population strata sizes and 
$\sigma_1, \sigma_2$ are the strata standard deviations.

We can check the second derivative to ensure this is a global optima or we can
use the fact that the variance is a quadratic function and is therefore convex.
Consequently, there is only 
[one global optima](https://en.wikipedia.org/wiki/Convex_function#Properties).

Examining the expression - we see the optimal strata size is the
variance of each strata weighted by the size of the strata as a fraction of 
the same value added across all strata, i.e. the population variance.

Here we've derived the solution for the "first" strata, but this is arbitrary
and there would be a similar value for any strata, for a design with any number
of strata.

9.  Write an R function that takes inputs
    $n_1, n_2, N_1, N_2, \sigma^2_1, \sigma^2_2$ and computes the variance of
    the population total in a stratified sample. Choose some reasonable values
    of the population sizes and variances, and graph this function as $n_1$ and
    $n_2$ change, to find the optimum and to examine how sensitive the variance
    is the precise values of $n_1$ and $n_2$.


::: {.cell}

```{.r .cell-code}
strat_var_sample <- function(n_1, n_2, N_1, N_2, sigma_1, sigma_2) {
  one_strat_var <- function(n, N, sigma) {
    (N - n) / N * N^2 * sigma^2 / n
  }
  return(one_strat_var(n_1, N_1, sigma_1) + one_strat_var(n_2, N_2, sigma_2))
}
expand.grid(
  n_1 = seq(2, 100, 10), n_2 = seq(2, 100, 10),
  N_1 = 1E3, N_2 = 1E3, sigma_1 = 1, sigma_2 = 1
) %>%
  as_tibble() %>%
  mutate(
    var = map2_dbl(n_1, n_2, strat_var_sample, unique(N_1), unique(N_2), unique(sigma_1), unique(sigma_2)),
    n_2 = factor(n_2)
  ) %>%
  ggplot(aes(x = n_1, y = var, color = n_2)) +
  geom_point() +
  geom_line() +
  ylab("Variance") +
  ggtitle("Two Strata Design Variance", subtitle = "For Varying Strata Sizes")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q2_9-1.png){width=672}
:::
:::


I think the point Lumley is after is that the variance doesn't change 
dramatically after each sample size is roughly more than ten. This is the
quadratic behavior of the variance at play. Obviously the "optimal" variance
will be found at the highest $n_1$ and $n_2$ as those values will provide the 
lowest variance. In terms of where the greatest efficiency lies, however, 
we see the greatest change in the variance estimates after the two sample 
sizes are greater than 10, as stated previously.


10. Verify that equation 2.2 gives the HTE of variance for a SRS


  *  Show that when $i \neq j$ 
  
$$
\pi_{ij} = \frac{n}{N}n - 1N - 1
$$
It isn't completely clear from the above, but I think Lumley means for there
to be parentheses around the $n-1$ and $N-1$ terms, that is:
$$
\pi_{ij} = \frac{n}{N}(n-1)(N-1)
$$

This follows from combinatorics. We want the number of combinations in which
two (different) items are chosen from a sample of n together, divided by the 
total number of combinations of n samples from a population of size N. 

$$
\frac{n \choose 2}{N\choose n} = \frac{n!}{(n-2)!2!} \frac{(N-n)!n!}{N!} \\
=\frac{n(n-1)}{2} \times \frac{(N-n)!n!}{N!}
$$
I don't see yet how this reduces to the intended form but may return to this
later.

  * Compute $\pi_{ij} - \pi_i \pi_j$

Using the expression Lumley gives us for $\pi_{ij}$, and assuming a similar
combinatoric form for $\pi_i, \pi_j$,

$$
\pi_{ij} - \pi_i \pi_j =  \frac{n}{N}(n-1)(N-1) - \left(\frac{n (N-n)!n!}{N!}\right)^2.
$$

  * Show that the equation in exercise 1.10 (c) reduces to equation 2.2
  
I'm clearly on the wrong track here. My formulation of the combinatorics must
be throwing me off. Feel free to 
[file an issue](https://github.com/apeterson91/ComplexSurveys/issues) in the 
repo for these notes if you see what I'm missing.

  * Suppose instead each individual in the population is independently sampled
    with probability $\frac{n}{N}$, so that the sample size $n$ is *not* fixed.
    Show that the finite population correction disappears from equation 2.2 for
    this *Bernoulli sampling* design.
    
$$
V[\hat{T}_{HTE,Bern}] = V[\sum_{i=1}^{N} R_i \frac{Y_i}{\pi}] \\
\stackrel{ind}{=} \sum_{i=1}^{N} V[R_i \frac{Y_i}{\pi}] \\
= \sum_{i=1}^{N} \pi(1-\pi)\frac{Y_i^2}{\pi^2}\\
= \frac{(1-\pi)}{\pi}\sum_{i=1}^{N}Y_i^2 \\
= (\frac{1}{\pi} - 1) \sum_{i=1}^{N}Y_i^2 \\ 
= \frac{N -n}{n} \sum_{i=1}^{N} Y_i^2, \quad  \pi := \frac{n}{N}
$$
which can be understand as equivalent to equation 2.2 where --- assume without
loss of generality --- that the variable $Y$ has $E[Y]=0$ then it 
is easy to see that the equation reduces to
$$
(N-n) \times N \times \frac{V[Y]}{n} = (N^2 - Nn) \frac{V[Y]}{n}
$$

which replaces the finite population correction factor with the $N^2 - Nn$ term.

# Chapter 3: Cluster Sampling

## Why Clusters? The NHANES design

Why sample clusters? Because sometimes it's easier than sampling individuals.
Specifically, in cases where the *cost* of sampling individuals can be quite
high, sampling clusters can be more efficient. This is in spite of the fact that
within cluster correlation tends to be positive, reducing the information in the
sample. Lumley uses the NHANES survey to motivate this idea: moving mobile
examination centers all across the country to sample individuals is extremely
expensive. By sampling a large number of individuals within a census tract
aggregation area the NHANES survey is able to reduce the cost of their effort at
a reasonable expense in precision.

### Single-stage and multistage designs

Depending on the type of clusters involved it can be easy to sample the entire
cluster as classrooms, medical practice and workplaces are, however it is more
likely that some sub-sampling within clusters will be performed for the sake of
efficiency. As Lumley notes, clusters in the first stage are called *Primary
Sampling Units* or PSUs. "Stages" refer to the different levels at which
sampling occurs. E.g. Sampling individuals within sampled census tracts within a
state would involve sampling census tracts in the first stage and then
individuals in the second stage. The diagram below communicates this idea
graphically.


::: {.cell}
::: {.cell-output-display}
```{=html}
<div class="grViz html-widget html-fill-item" id="htmlwidget-7b5514f384172c7ab337" style="width:100%;height:464px;"></div>
<script type="application/json" data-for="htmlwidget-7b5514f384172c7ab337">{"x":{"diagram":"\ndigraph dot {\ngraph [layout = dot]\nnode [shape = circle,\nstyle = filled,\ncolor = grey]\n\nnode [fillcolor = red, label = \"US State (strata)\"]\na\n\nnode [fillcolor = green, label = \"census tract (PSU)\"]\nb\n\nnode [fillcolor = gold, label = \"households (SSU)\"]\n\nedge [color = grey]\na -> {b}\nb -> {c}\n}","config":{"engine":"dot","options":null}},"evals":[],"jsHooks":[]}</script>
```
:::
:::


Sampling weights are determined assuming independence across stages --- e.g. if
a cluster of houses is sampled with probability $\pi_1$ and a household is
sampled within that cluster with probability $\pi_2$ then the sampling
probability for that house is $\pi = \pi_1 \times \pi_2$ and it's weight is the
inverse of that probability. Note that this requires that clusters be mutually
exclusive - a sampled unit can belong only to one cluster and no others.
Further, note that we can still have biased sampling *within* a stage, as
independence is only required across stages to use to find probabilities via
their product.

Lumley goes on to describe how cluster sampling and individual sampling can be
mixed since each stratum of a survey can be thought of as a separate and
independent sample it is trivial to combine single stage sampling in one stratum
and multistage sampling in another; a stratified random sample can be used in
high density regions where measurement of multiple units is less costly and a
cluster sample can be taken in low density regions where the cost of each
additional unit is more costly.

The statistical rationale behind this strategy is fairly straightforward ---
since the variance of the sum is the sum of the variances of each stage
(assuming independence) each sampled cluster in a multistage sample can be
considered as a population for further sampling. Lumley uses the example of a
simplified NHANES design, where 64 regions are grouped into 32 strata. A simple
random sample of 440 individuals are then measured in each region. In Lumley's
words,

> The variance of an estimated total from this design can be partitioned across
> two sources: the variance of each estimated regional total around the true
> total of the region and the variance that would result if the true total for
> each of the 64 sampled regions were known exactly.

In my own words and understanding, I understand there to be variance that comes
from grouping the 64 regions into 32 strata --- so there is uncertainty across
region and then the uncertainty within that region that results from the sample
of *only* a subset of the population.

## Describing Multi Stage Designs to R

In order to specify a single stage cluster sample or a multistage sample treated
as a single stage sample with replacement, the main difference is that the PSU
identifier needs to be supplied to the `id` argument, as follows.


::: {.cell}

```{.r .cell-code}
# Data originally found at
# "https://github.com/cran/LogisticDx/blob/master/data/nhanes3.rda"
```
:::

::: {.cell}

```{.r .cell-code}
names3 <- load("Data/nhanes/nhanes3.rda")
as_tibble(nhanes3)
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 17,030 × 16
    SEQN SDPPSU6 SDPSTRA6 WTPFHX6 HSAGEIR HSSEX  DMARACER BMPWTLBS BMPHTIN
   <dbl>   <dbl>    <dbl>   <dbl>   <dbl> <fct>  <fct>       <dbl>   <dbl>
 1     3       1       44   1735.      21 male   white        180.    70.4
 2     4       1       43   1725.      32 female white        136.    63.9
 3     9       2       43  19452.      48 female white        150.    61.8
 4    10       1        6  27770.      35 male   white        204.    69.8
 5    11       2       40   1246.      48 male   white        155.    66.2
 6    19       1       35   3861.      44 male   black        190.    70.2
 7    34       1       13   5032.      42 female black        126.    62.6
 8    44       1        8  28149.      24 female white        123.    64.4
 9    45       1       22   4582.      67 female black        150.    64.3
10    48       1       24  26919.      56 female white        240.    67.6
# ℹ 17,020 more rows
# ℹ 7 more variables: PEPMNK1R <dbl>, PEPMNK5R <dbl>, HAR1 <fct>, HAR3 <fct>,
#   SMOKE <fct>, TCP <dbl>, HBP <fct>
```
:::
:::

::: {.cell}

```{.r .cell-code}
svydesign(
  id = ~SDPPSU6, strat = ~SDPSTRA6,
  weight = ~WTPFHX6,
  ## nest = TRUE indicates the PSU identifier is nested
  ## within stratum - repeated across strata
  nest = TRUE,
  data = nhanes3
)
```

::: {.cell-output .cell-output-stdout}
```
Stratified 1 - level Cluster Sampling design (with replacement)
With (98) clusters.
svydesign(id = ~SDPPSU6, strat = ~SDPSTRA6, weight = ~WTPFHX6, 
    nest = TRUE, data = nhanes3)
```
:::
:::


SDPPSU6 is the pseudo PSU variable, and SDPSTRA6 is the stratum identifier
defined for the single stage analysis.

For example, a two stage design for the API population that samples 40 school
districts, then five schools within each district , the design has population
size 757 at the first stage for the number of school districts in CA and the
number of schools within each district for the second stage. The weights need
not be supplied if they can be worked out from the other arguments.


::: {.cell}

```{.r .cell-code}
data(api)
as_tibble(apiclus2)
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 126 × 40
   cds       stype name  sname  snum dname  dnum cname  cnum  flag pcttest api00
   <chr>     <fct> <chr> <chr> <dbl> <chr> <int> <chr> <int> <int>   <int> <int>
 1 31667796… E     Alta… Alta…  3269 Alta…    15 Plac…    30    NA     100   821
 2 55751846… E     Tena… Tena…  5979 Big …    63 Tuol…    54    NA     100   773
 3 41688746… E     Pano… Pano…  4958 Bris…    83 San …    40    NA      98   600
 4 41688746… M     Lipm… Lipm…  4957 Bris…    83 San …    40    NA     100   740
 5 41688746… E     Bris… Bris…  4956 Bris…    83 San …    40    NA      98   716
 6 40687266… E     Cayu… Cayu…  4915 Cayu…   117 San …    39    NA     100   811
 7 20651936… E     Full… Full…  2548 Chow…   132 Made…    19    NA     100   472
 8 20651936… E     Fair… Fair…  2550 Chow…   132 Made…    19    NA     100   520
 9 20651936… M     Wils… Wils…  2549 Chow…   132 Made…    19    NA     100   568
10 06615980… H     Colu… Colu…   348 Colu…   152 Colu…     5    NA      96   591
# ℹ 116 more rows
# ℹ 28 more variables: api99 <int>, target <int>, growth <int>, sch.wide <fct>,
#   comp.imp <fct>, both <fct>, awards <fct>, meals <int>, ell <int>,
#   yr.rnd <fct>, mobility <int>, acs.k3 <int>, acs.46 <int>, acs.core <int>,
#   pct.resp <int>, not.hsg <int>, hsg <int>, some.col <int>, col.grad <int>,
#   grad.sch <int>, avg.ed <dbl>, full <int>, emer <int>, enroll <int>,
#   api.stu <int>, pw <dbl>, fpc1 <dbl>, fpc2 <int[1d]>
```
:::
:::

::: {.cell}

```{.r .cell-code}
## dnum = district id
## snum = school id
## fpc1 = school id number
clus1_design <- svydesign(id = ~dnum, fpc = ~fpc, data = apiclus1)
clus2_design <- svydesign(
  id = ~ dnum + snum, fpc = ~ fpc1 + fpc2,
  data = apiclus2
)
clus2_design
```

::: {.cell-output .cell-output-stdout}
```
2 - level Cluster Sampling design
With (40, 126) clusters.
svydesign(id = ~dnum + snum, fpc = ~fpc1 + fpc2, data = apiclus2)
```
:::
:::


## Strata with only one PSU

When only one PSU exists within a population stratum, the sampling fraction
*must* be 100%, since otherwise it would be 0%. In this case, the stratum does
not contribute to the first stage variance and it should be ignored in
calculating the first stage variance. Lumley argues that the best way to handle
a stratum with only one PSU is to combine it with another stratum, one that is
chosen to be similar based on *population* data available before the study was
done. The `survey` package has two different methods implemented to handle
"lonely" PSU's. Lumley has written further on this topic
[here](http://r-survey.r-forge.r-project.org/survey/exmample-lonely.html).

## How good is the single-stage approximation?

Here Lumley walks through an example detailing the trade-offs involved in using
the single stage approximation. I'll try to come up with a simulated example
later as the data is not listed on the book's
[website](https://r-survey.r-forge.r-project.org/svybook/) nor is it clear how
to reassemble his dataset from the files at the [NHIS
site](https://ftp.cdc.gov/pub/Health_Statistics/NCHS/Datasets/NHIS/1992/).

## Sampling by Size

> Why do white sheep eat more than black sheep? There are more white sheep than
> black sheep

A specific design theory, *Probability-proportional-to-size* (PPS), cluster
sampling is a sampling strategy that exploits the fact that for a simple random
sample of an unstratified population $\pi_i$ can be chosen such that it is
approximately proportional to $X_i$, the variable of interest, the resulting
variance of the estimate of the total
$V[\hat{T}] = \frac{N-n}{N} N^{2} \frac{V[X]}{n}$ can then be controlled to be
quite small. These are the same ideas I discussed at the start of the notes but
more discussion on this topic can be found in [@tille2006sampling;
@hanif1980sampling].


::: {.cell}

```{.r .cell-code}
data(election)
election <- as_tibble(election) %>%
  mutate(
    votes = Bush + Kerry + Nader,
    p = 40 * votes / sum(votes)
  )
election %>%
  ggplot(aes(x = Kerry, y = Bush)) +
  geom_point() +
  scale_y_log10() +
  scale_x_log10() +
  ggtitle("Correlation in Voting Totals from US 2004 Presidential Election",
    subtitle = "Both x and y axes are on log 10 scales."
  )
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/pps_election_demo-1.png){width=672}
:::
:::


When Lumley's book was written, only the single stage approximation of PPS could
be analyzed using the `survey` package. A demo is shown below using the voting
data, where a PPS sample is constructed and then analyzed.


::: {.cell}

```{.r .cell-code}
data(election)
election <- as_tibble(election) %>%
  mutate(
    votes = Bush + Kerry + Nader,
    p = 40 * votes / sum(votes)
  )
election
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 4,600 × 8
   County   TotPrecincts PrecinctsReporting   Bush Kerry Nader  votes       p
   <fct>           <int>              <int>  <int> <int> <int>  <int>   <dbl>
 1 Alaska            439                439 151876 86064  3890 241830 0.0832 
 2 Autauga            22                 22  15212  4774    74  20060 0.00691
 3 Baldwin            50                 50  52910 15579   371  68860 0.0237 
 4 Barbour            24                 24   5893  4826    26  10745 0.00370
 5 Bibb               16                 16   5471  2089    12   7572 0.00261
 6 Blount             26                 26  17364  3932    92  21388 0.00736
 7 Bullock            27                 27   1494  3210     3   4707 0.00162
 8 Butler             27                 27   4978  3409    13   8400 0.00289
 9 Calhoun            53                 53  29806 15076   182  45064 0.0155 
10 Chambers           24                 24   7618  5346    42  13006 0.00448
# ℹ 4,590 more rows
```
:::

```{.r .cell-code}
insample <- sampling::UPtille(election$p)
ppsample <- election[insample == 1, ]
ppsample$wt <- 1 / ppsample$p
pps_design <- svydesign(id = ~1, weight = ~wt, data = ppsample)
pps_design
```

::: {.cell-output .cell-output-stdout}
```
Independent Sampling design (with replacement)
svydesign(id = ~1, weight = ~wt, data = ppsample)
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~ Bush + Kerry + Nader, pps_design, deff = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
         total       SE   DEff
Bush  60203714  2711502 0.0118
Kerry 55617937  2695188 0.0052
Nader   377454    91396 0.1035
```
:::
:::


## Loss of information from sampling clusters

The loss of precision per observation from cluster sampling is given by the
design effect.

> "For a single-stage cluster sample with all clusters having the same number of
> individuals, $m$, the design effect is

$$
D_{eff} = 1 + (m-1)\rho,
$$

> where $\rho$ is the within-cluster correlation.

Lumley illustrates how design effects can illustrate the impact on inference
using the California school data set from before as well as the Behavioral Risk
Factor Surveillance System from 2007.


::: {.cell}

```{.r .cell-code}
svymean(~ api00 + meals + ell + enroll, clus1_design, deff = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
           mean       SE    DEff
api00  644.1694  23.5422  9.2531
meals   50.5355   6.2690 10.3479
ell     27.6120   2.0193  2.6711
enroll 549.7158  45.1914  2.7949
```
:::
:::


In the above, the variance is up to 10 times higher in the cluster sample as
compared to a simple random sample.


::: {.cell}

```{.r .cell-code}
## Lumley renames clus2_design to dclus2 from before. I maintain the same names.
svymean(~ api00 + meals + ell + enroll, clus2_design, deff = TRUE, na.rm = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
           mean       SE    DEff
api00  673.0943  31.0574  6.2833
meals   52.1547  10.8368 11.8585
ell     26.0128   5.9533  9.4751
enroll 526.2626  80.3410  6.1427
```
:::
:::


These values increase slightly for all measures except `api00` in the two stage
cluster sampling design. Lumley points out that these large design effects
demonstrate how variable the measures of interest are between cluster,
suggesting that the sampling of clusters, while efficient economically are not
as efficient statistically.

Similarly, when computing the proportion of individuals who have more than 5
servings of fruits and vegetables a day (X_FV5SRV = 2), as well as how often
individuals received a cholesterol test in the past 5 years (X_CHOLCHK = 1) from
the 2007 Behavioral Risk Factor Surveillance System data set, we see design
effects that reflect the geographic variability across the blocks of telephone
numbers that were sampled for the survey.


::: {.cell}

```{.r .cell-code}
brfss <- svydesign(
  id = ~X_PSU, strata = ~X_STATE, weight = ~X_FINALWT,
  data = "brfss", dbtype = "SQLite",
  dbname = "data/BRFSS/brfss07.db", nest = TRUE
)
brfss
```

::: {.cell-output .cell-output-stdout}
```
DB-backed Stratified Independent Sampling design (with replacement)
svydesign(id = ~X_PSU, strata = ~X_STATE, weight = ~X_FINALWT, 
    data = "brfss", dbtype = "SQLite", dbname = "data/BRFSS/brfss07.db", 
    nest = TRUE)
```
:::
:::

::: {.cell}

```{.r .cell-code}
food_labels <- c("Yes Veg", "No Veg")
chol_labels <- c("within 5 years", ">5 years", "never")
svymean(
  ~ factor(X_FV5SRV) +
    factor(X_CHOLCHK),
  brfss,
  deff = TRUE
)
```

::: {.cell-output .cell-output-stdout}
```
                         mean         SE   DEff
factor(X_FV5SRV)1  0.73096960 0.00153359 5.1632
factor(X_FV5SRV)2  0.23844253 0.00145234 5.0147
factor(X_FV5SRV)9  0.03058787 0.00069991 7.1323
factor(X_CHOLCHK)1 0.73870300 0.00168562 6.3550
factor(X_CHOLCHK)2 0.03230828 0.00058759 4.7676
factor(X_CHOLCHK)3 0.19989559 0.00162088 7.0918
factor(X_CHOLCHK)9 0.02909313 0.00055471 4.7029
```
:::
:::


## Repeated Measurements

Lumley notes that design based inference continues to differ from model based in
its analysis of repeated measurements. Where model based inference is careful to
account for modeling the -- for example -- within person or within household
correlation in a cohort study, no such adjustment is required in a designed
survey other than adjusting and using the appropriate weights - treating the
repeated measurement like another stage of clustering in the sampling.

Lumley illustrates this with the Survey of Income and Program Participation
(SIPP) panel survey.

> Each panel is followed for multiple years, with subsets of the panel
> participating in four month waves of follow-up... wave 1 of the 1996 panel,
> which followed 36,730 households with interviews every four months, starting
> in late 1995 or early 1996... The households were recruited in a two-stage
> sample. The first stage sampled 322 counties or groups of counties as PSUs;
> the second stage sampled households within these PSUs.

Lumley demonstrates how to estimate repeated measures with panel data using the
`survey` package via the code below. Five quantiles are estimated across the
population and across the 8 months. When Lumley mentions that there is no need
for adjusting for correlation in the block quote above, I believe he is referring
to the within-month point estimates. If we were to try and estimate the change
in, say, income as a function of other covariates I believe we would want to
adjust for correlation in order to get the appropriate standard errors. For the
point estimates Lumley points out that the weights are exactly as required for
the per-month estimate, but would need to be divided by the number of samples
when totaling across the number of measurements. Proportions or regressions are
invariant to this scaling factor so no adjustment is needed there.


::: {.cell}

```{.r .cell-code}
sipp_hh <- svydesign(
  id = ~ghlfsam, strata = ~gvarstr, nest = TRUE,
  weight = ~whfnwgt, data = "household", dbtype = "SQLite",
  dbname = "Data/SIPP/sipp.db"
)
sipp_hh <- update(sipp_hh,
  month = factor(rhcalmn,
    levels = c(12, 1, 2, 3, 4, 5, 6),
    labels = c(
      "Dec", "Jan", "Feb", "Mar",
      "Apr", "May", "Jun"
    )
  )
)
qinc <- svyby(~thtotinc, ~month, svyquantile,
  design = sipp_hh,
  quantiles = c(0.25, 0.5, 0.75, 0.9, 0.95), se = TRUE
)
pltdf <- as_tibble(qinc) %>%
  select(month, contains("thtotinc"), -contains("se")) %>%
  gather(everything(), -month, key = "quantile", value = "Total Income") %>%
  mutate(quantile = as.numeric(str_extract(quantile, "[0-9].[0-9]?[0-9]")) * 100)

se <- as_tibble(qinc) %>%
  select(month, contains("se")) %>%
  gather(everything(), -month, key = "quantile", value = "SE") %>%
  mutate(quantile = as.numeric(str_extract(quantile, "[0-9].[0-9]?[0-9]")) * 100)

pltdf <- pltdf %>%
  left_join(se) %>%
  mutate(
    lower = `Total Income` - 2 * SE,
    upper = `Total Income` + 2 * SE
  )
```
:::

::: {.cell}

```{.r .cell-code}
pltdf %>%
  mutate(quantile = factor(quantile)) %>%
  ggplot(aes(x = month, y = `Total Income`, color = quantile)) +
  geom_pointrange(aes(ymin = lower, ymax = upper)) +
  xlab("Month in 1995/1996") +
  ylab("Total Income (USD)") +
  ggtitle("Total Income Quantiles",
    subtitle = "Survey of Income and Program Participation"
  )
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/sipp_income_quantiles-1.png){width=672}
:::
:::


Lumley notes here that we're only talking about correlation at all here because
the same individuals are being measured across time. In a study like NHANES
that recruits different individuals each year, we can effectively assume the
samples are independent since the overall population is so big.

Estimation would be much more complicated if the samples overlapped in some way.


## Exercises

1.  The web site has data files demo_x.xpt of demographic data and bpx_c.xpt of
    blood pressure data from NHANES 2003-2004. Code to load and merge these data
    sets is in Appendix B, in Figure B.1

  * Construct a survey design object with these data.


::: {.cell}

```{.r .cell-code}
## data are from the book website:
# https://r-survey.r-forge.r-project.org/svybook/
# demographic data
ddf <- haven::read_xpt("data/nhanesxpt/demo_c.xpt")
# blood pressure data
bpdf <- haven::read_xpt("data/nhanesxpt/bpx_c.xpt")

bpdf <- merge(ddf, bpdf, by = "SEQN", all = FALSE) %>%
  mutate(
    sys_over_140 = (BPXSAR > 140) * 1,
    dia_over_90 = (BPXDAR > 90) * 1,
    hbp = (sys_over_140 + dia_over_90 == 2) * 1,
    age_group = cut(RIDAGEYR, c(0, 21, 65, Inf)),
    sex = factor(RIAGENDR, levels = 1:2, labels = c("Male", "Female"))
  )

bpdf_design <- bpdf %>%
  select(
    sys_over_140, dia_over_90, hbp, WTMEC2YR, SDMVPSU, SDMVSTRA,
    age_group, sex, RIDAGEYR, BPXSAR, BPXDAR, RIAGENDR, RIDAGEMN,
  ) %>%
  as_survey_design(
    weights = WTMEC2YR,
    id = SDMVPSU,
    strata = SDMVSTRA,
    nest = TRUE
  )
bpdf_design
```

::: {.cell-output .cell-output-stdout}
```
Stratified 1 - level Cluster Sampling design (with replacement)
With (30) clusters.
Called via srvyr
Sampling variables:
  - ids: SDMVPSU 
  - strata: SDMVSTRA 
  - weights: WTMEC2YR 
Data variables: 
  - sys_over_140 (dbl), dia_over_90 (dbl), hbp (dbl), WTMEC2YR (dbl), SDMVPSU
    (dbl), SDMVSTRA (dbl), age_group (fct), sex (fct), RIDAGEYR (dbl), BPXSAR
    (dbl), BPXDAR (dbl), RIAGENDR (dbl), RIDAGEMN (dbl)
```
:::
:::


  * Estimate the proportion of the US population with systolic blood pressure
    over 140 mm HG, with diastolic blood pressure over 90 mm Hg, and with both.
    
Lumley doesn't tell us which variable name corresponds to the measurements 
indicated and I didn't see any documentation in the files included so I just
guess here.

::: {.cell}

```{.r .cell-code}
bpdf_design %>%
  summarize(
    prop_hbp_one =
      survey_ratio(sys_over_140, n(), na.rm = TRUE, vartype = "ci", deff = TRUE),
    prop_hbp_two =
      survey_ratio(sys_over_140, n(), na.rm = TRUE, vartype = "ci", deff = TRUE),
    prob_hbp = survey_ratio(hbp, n(), na.rm = TRUE, vartype = "ci", deff = TRUE)
  ) %>%
  gather(everything(), key = "metric", value = "value")
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 12 × 2
   metric                 value
   <chr>                  <dbl>
 1 prop_hbp_one      0.0000119 
 2 prop_hbp_one_low  0.0000104 
 3 prop_hbp_one_upp  0.0000135 
 4 prop_hbp_one_deff 3.60      
 5 prop_hbp_two      0.0000119 
 6 prop_hbp_two_low  0.0000104 
 7 prop_hbp_two_upp  0.0000135 
 8 prop_hbp_two_deff 3.60      
 9 prob_hbp          0.00000217
10 prob_hbp_low      0.00000141
11 prob_hbp_upp      0.00000293
12 prob_hbp_deff     4.15      
```
:::
:::


  * Estimate the design effects for these proportions.

Included in the output above.

  * How do these proportions vary by age (`RIDAGEYR`) and gender (`RIAGENDR`)


::: {.cell}

```{.r .cell-code}
nhanes_ests <- bpdf_design %>%
  group_by(sex, age_group) %>%
  summarize(
    prop_hbp_one =
      survey_ratio(sys_over_140, n(), na.rm = TRUE, vartype = "ci", deff = TRUE),
    prop_hbp_two =
      survey_ratio(sys_over_140, n(), na.rm = TRUE, vartype = "ci", deff = TRUE),
    prob_hbp = survey_ratio(hbp, n(), na.rm = TRUE, vartype = "ci", deff = TRUE)
  ) %>%
  ungroup()
nhanes_ests
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 8 × 14
  sex    age_group  prop_hbp_one prop_hbp_one_low prop_hbp_one_upp
  <fct>  <fct>             <dbl>            <dbl>            <dbl>
1 Male   (0,21]      0.000000953     -0.000000551       0.00000246
2 Male   (21,65]     0.0000647        0.0000486         0.0000809 
3 Male   (65,Inf]    0.000478         0.000402          0.000554  
4 Male   <NA>      NaN              NaN               NaN         
5 Female (0,21]      0.000000660     -0.000000521       0.00000184
6 Female (21,65]     0.0000597        0.0000518         0.0000677 
7 Female (65,Inf]    0.000738         0.000655          0.000820  
8 Female <NA>      NaN              NaN               NaN         
# ℹ 9 more variables: prop_hbp_one_deff <dbl>, prop_hbp_two <dbl>,
#   prop_hbp_two_low <dbl>, prop_hbp_two_upp <dbl>, prop_hbp_two_deff <dbl>,
#   prob_hbp <dbl>, prob_hbp_low <dbl>, prob_hbp_upp <dbl>, prob_hbp_deff <dbl>
```
:::
:::


This produces a bunch of output, so I plot the high blood pressure result ---
both high systolic and diastolic blood pressure -- below.


::: {.cell}

```{.r .cell-code}
nhanes_ests %>%
  select(age_group, sex, prob_hbp, prob_hbp_low, prob_hbp_upp) %>%
  filter(!is.na(age_group)) %>%
  ggplot(aes(x = age_group, y = prob_hbp, color = sex)) +
  geom_pointrange(aes(ymin = prob_hbp_low, ymax = prob_hbp_upp),
    position = position_dodge(width = .25)
  ) +
  ylab("% US Population with High Blood Pressure") +
  xlab("Age Group") +
  ggtitle("Prevalence of High Blood Pressure in U.S. Across Age and Sex",
    subtitle = "Data from NHANES 2003-2004"
  ) +
  scale_y_continuous(labels = scales::percent)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q_3_1_compiled-1.png){width=672}
:::
:::


2. Repeat the sampling and estimation in Figure 3.4 1000 times.

FYI the `sampling::UPtille` function takes a while so running this simulation
1000 times takes a while.

::: {.cell}

```{.r .cell-code}
OneSimulation <- function() {
  insample <- sampling::UPtille(election$p)
  ppsample <- election[insample == 1, ]
  ppsample$wt <- 1 / ppsample$p
  pps_design <- svydesign(id = ~1, weight = ~wt, data = ppsample, pps = "brewer")
  total <- svytotal(~ Bush + Kerry + Nader, pps_design)
  std_err <- sqrt(diag(attr(total, "var")))
  ci <- confint(total)
  out <- cbind(total, std_err, ci)
}
total_sims <- replicate(10, OneSimulation())
```
:::


* Check that the mean of the estimated totals is close to the true population 
totals.


::: {.cell}

```{.r .cell-code}
estimated_totals <- as_tibble(t(rowMeans(total_sims[, 1, ]))) %>%
  mutate(label = "Estimate")

true_totals <- as_tibble(t(colSums(election[, c("Bush", "Kerry", "Nader")]))) %>%
  mutate(label = "Truth")

rbind(estimated_totals, true_totals) %>%
  gather(everything(), -label, key = "Candidate", value = "Vote Count") %>%
  spread(label, `Vote Count`) %>%
  gt::gt() %>%
  gt::fmt_scientific() %>%
  gt::tab_header("Simulated Total Comparison")
```

::: {.cell-output-display}
```{=html}
<div id="qmxhkjierr" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#qmxhkjierr table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#qmxhkjierr thead, #qmxhkjierr tbody, #qmxhkjierr tfoot, #qmxhkjierr tr, #qmxhkjierr td, #qmxhkjierr th {
  border-style: none;
}

#qmxhkjierr p {
  margin: 0;
  padding: 0;
}

#qmxhkjierr .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#qmxhkjierr .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#qmxhkjierr .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#qmxhkjierr .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#qmxhkjierr .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#qmxhkjierr .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#qmxhkjierr .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#qmxhkjierr .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#qmxhkjierr .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#qmxhkjierr .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#qmxhkjierr .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#qmxhkjierr .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#qmxhkjierr .gt_spanner_row {
  border-bottom-style: hidden;
}

#qmxhkjierr .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#qmxhkjierr .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#qmxhkjierr .gt_from_md > :first-child {
  margin-top: 0;
}

#qmxhkjierr .gt_from_md > :last-child {
  margin-bottom: 0;
}

#qmxhkjierr .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#qmxhkjierr .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#qmxhkjierr .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#qmxhkjierr .gt_row_group_first td {
  border-top-width: 2px;
}

#qmxhkjierr .gt_row_group_first th {
  border-top-width: 2px;
}

#qmxhkjierr .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#qmxhkjierr .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#qmxhkjierr .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#qmxhkjierr .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#qmxhkjierr .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#qmxhkjierr .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#qmxhkjierr .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#qmxhkjierr .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#qmxhkjierr .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#qmxhkjierr .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#qmxhkjierr .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#qmxhkjierr .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#qmxhkjierr .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#qmxhkjierr .gt_left {
  text-align: left;
}

#qmxhkjierr .gt_center {
  text-align: center;
}

#qmxhkjierr .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#qmxhkjierr .gt_font_normal {
  font-weight: normal;
}

#qmxhkjierr .gt_font_bold {
  font-weight: bold;
}

#qmxhkjierr .gt_font_italic {
  font-style: italic;
}

#qmxhkjierr .gt_super {
  font-size: 65%;
}

#qmxhkjierr .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#qmxhkjierr .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#qmxhkjierr .gt_indent_1 {
  text-indent: 5px;
}

#qmxhkjierr .gt_indent_2 {
  text-indent: 10px;
}

#qmxhkjierr .gt_indent_3 {
  text-indent: 15px;
}

#qmxhkjierr .gt_indent_4 {
  text-indent: 20px;
}

#qmxhkjierr .gt_indent_5 {
  text-indent: 25px;
}

#qmxhkjierr .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#qmxhkjierr div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_heading">
      <td colspan="3" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Simulated Total Comparison</td>
    </tr>
    
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1" scope="col" id="Candidate">Candidate</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Estimate">Estimate</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Truth">Truth</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="Candidate" class="gt_row gt_left">Bush</td>
<td headers="Estimate" class="gt_row gt_right">5.98&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td>
<td headers="Truth" class="gt_row gt_right">5.96&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td></tr>
    <tr><td headers="Candidate" class="gt_row gt_left">Kerry</td>
<td headers="Estimate" class="gt_row gt_right">5.61&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td>
<td headers="Truth" class="gt_row gt_right">5.61&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td></tr>
    <tr><td headers="Candidate" class="gt_row gt_left">Nader</td>
<td headers="Estimate" class="gt_row gt_right">3.82&nbsp;×&nbsp;10<sup style='font-size: 65%;'>5</sup></td>
<td headers="Truth" class="gt_row gt_right">4.04&nbsp;×&nbsp;10<sup style='font-size: 65%;'>5</sup></td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


These are quite close.

  * Compute the mean of the estimated standard errors and compare it to the
true simulation standard error, that is, the standard deviation of the 
estimated totals.


::: {.cell}

```{.r .cell-code}
estimated_stderrs <- as_tibble(t(rowMeans(total_sims[, 2, ]))) %>%
  mutate(label = "Estimate")

true_stderr <- as_tibble(apply(total_sims, 1, sd)) %>%
  mutate(label = "Truth", Candidate = c("Bush", "Kerry", "Nader")) %>%
  spread(Candidate, value)

rbind(estimated_stderrs, true_stderr) %>%
  gather(everything(), -label, key = "Candidate", value = "Vote Std.Err") %>%
  spread(label, `Vote Std.Err`) %>%
  gt::gt() %>%
  gt::fmt_scientific() %>%
  gt::tab_header("Simulated Standard Error Comparison")
```
:::


Same as above.

* Estimate 95% confidence intervals for the population totals and compute the 
proportion of intervals that contain the true population value.


::: {.cell}

```{.r .cell-code}
totals <- unlist(true_totals[, 1:3])
prop_contained <- rowMeans((total_sims[, 3, ] < totals &
  total_sims[, 4, ] > totals) * 1)
prop_contained
```
:::


We see that the proportion contained by the interval is near the 95% nominal 
value.


3.  The National Longitudinal Study of Youth is documented at 
www.nlsinfo.org/nlsy97/nlsdocs/nlsy97/maintoc.html

* What are the stages of sampling and the sampling units at each stage?

From the 
[website](https://www.nlsinfo.org/content/cohorts/nlsy97/intro-to-the-sample/sample-design-screening-process)
the sampling occurred in 2 phases (see image below).

![](https://www.nlsinfo.org/sites/default/files/attachments/sample-figure%201.JPG)

The first phase screened households and the second identified eligible
respondents. I think this would technically be called a two phase sample 
(discussed in chapter 8) but given the population sizes involved it may be that
there was no need to account for the potential dependence here. 

The sampling unit in the first stage came from NORC's 1990 sampling design which
used Standard Metropolitan Statistical Areas or non-metropolitan counties. That
is, according to the general social survey 
[documentation](https://gss.norc.org/Documents/codebook/GSS_Codebook_AppendixA.pdf)

Subsequent sampling units are segments, households, and then members of 
households.

  * What would the strata and PSUs be for the single stage approximation to 
  the design?

I don't know. It isn't clear how to reconstruct Lumley's example given that
the data is not available. Looking at the NHIS 
[website](https://www.cdc.gov/nchs/nhis/sudaan.htm) it looks like the 
strata and PSU information for a single stage approximation are given. 
When looking for the same information for the NLYS data I don't 
[see](https://www.nlsinfo.org/content/cohorts/nlsy97/other-documentation/errata/errata-nlsy97-round-15-release/calculating-design) equivalent instructions on how to construct a single stage 
approximation. His description in the book here isn't very forthcoming either 
but I'd guess that one could just concatenate the PSU and SSU labels and the 
same for the strata and then use those in a one stage design. This doesn't look 
exactly like what Lumley does in his example.

4.  The British Household Panel Survey (BHPS) is documented at 
http://www.iser.essex.ac.uk/survey/bhps (this website is no longer accurate) .

  * What are the stages of sampling, strata and the sampling units at each stage?

The website given is no longer accurate. Clicking through the documentation of 
the "Understanding Society" 
[website](https://www.understandingsociety.ac.uk/documentation/mainstage/user-guides/main-survey-user-guide/study-design/) 
it looks like the BHPS has been combined with other surveys. 
When I look at more documentation,
([1](https://www.understandingsociety.ac.uk/wp-content/uploads/documentation/user-guides/6614_main_survey_sample_design.pdf),
[2](https://repository.essex.ac.uk/21094/1/bhps-harmonised-user-guide.pdf#page=6.75),
[3](https://www.iser.essex.ac.uk/wp-content/uploads/bhps/documentation/volumes/5151userguide_vola.pdf)),
the last of these being the most pertinent, I see a variety of designs by wave.
It looks like the design was a two stage stratified sample where the sampling
frame was the Postcode Address File for Great Britian, excluding Northern
Ireland. 250 postcodes were chosen as the primary sampling unit, in the 
second stage, "delivery points" which are roughly equivalent to addresses were
then selected.

  * What would the strata and PSUs be for the single stage approximation to
  the design?

Again, my best guess is to concatenate the postcodes and delivery addresses
to construct the single stage identifiers, and the same for the strata.


5. Statistics New Zealand lists survey topics at 
http://www.stats.govt.nz/datasets/a-z-list.htm . Find the Household Labour Force
Survey.

* What are the stages of sampling and the sampling units at each stage?

Again, that site is no longer valid. The current help 
[page](https://www.stats.govt.nz/help-with-surveys/list-of-stats-nz-surveys/about-the-household-labour-force-survey/)
for the household labor survey simply says,

> Approximately fifteen thousand (15,000) households take part in this survey
every three months. A house is selected using a fair statistical method to
ensure the sample is an accurate representation of New Zealand. Every person 
aged 15 years or over living in a selected house needs to take part.

A different [doc](https://www.stats.govt.nz/assets/Uploads/Retirement-of-archive-website-project-files/Methods/Household-Labour-Force-Survey-sources-and-methods-2016/hlfs-sources-and-methods-2016.pdf#page=11.08)
I found on Google listed the 2016 survey design as a 2 stage 
design where samples are drawn from strata in the first stage. The PSU's 
are "meshblocks" which appear to be the NZ equivalent of a U.S. 
census block / tract.

  * What would the strata and PSUs be for the single stage approximation to
  the design?

Same answer here as previously.

6. This exercise uses the Washington State Crime data for 2004 as the 
population. The data consists of crime rates and population size for the
police districts (in cities/towns) and sheriffs' offices 
(in unincorporated areas), grouped by county.

I took the data from this 
[website](https://www.waspc.org/cjis-statistics---reports). Specifically,
[this excel sheet](https://waspc.memberclicks.net/index.php?option=com_content&view=article&id=121:crime-in-wa-archive-folder&catid=20:site-content). One of the tricky things about using this data was the fact 
that several police districts are reported as having zero associated population.
I removed these from the data set to make things simpler.


  * Take a simple random sample of 10 counties from the state and use all the
  data from the sampled counties. Estimate the total number of murders and
  burglaries in the state.


::: {.cell}

```{.r .cell-code}
# data from https://www.waspc.org/cjis-statistics---reports)
wa_crime_df <- readxl::read_xlsx("data/WA_crime/1984-2011.xlsx", skip = 4) %>%
  filter(Year == "2004", Population > 0) %>%
  mutate(
    murder = `Murder Total`,
    murder_and_crime = `Murder Total` + `Burglary Total`,
    violent_crime = `Violent Crime Total`,
    burglaries = `Burglary Total`,
    property_crime = `Property Crime Total`,
    state_pop = sum(Population),
    County = stringr::str_to_lower(County),
    num_counties = n_distinct(County),
  ) %>%
  group_by(County) %>%
  mutate(num_agencies = n_distinct(Agency)) %>%
  ungroup() %>%
  select(
    County, Agency, Population, murder_and_crime, murder, violent_crime,
    property_crime, burglaries, num_counties, num_agencies
  )

true_count <- sum(wa_crime_df$murder_and_crime)
county_list <- unique(wa_crime_df$County)
county_sample <- sample(county_list, 10)
part_a <- wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design(
    ids = c(County, Agency),
    fpc = c(num_counties, num_agencies)
  ) %>%
  summarize(total = survey_total(murder_and_crime)) %>%
  mutate(Q = "a")
part_a
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
   total total_se Q    
   <dbl>    <dbl> <chr>
1 52982.   26039. a    
```
:::
:::



  * Stratify the sampling so that King County is sampled with 100% probability
  together with a simple random sample of five other counties. Estimate the
  total number of of murders and burglaries in the state and compare to the
  previous estimates.


::: {.cell}

```{.r .cell-code}
county_sample <- sample(county_list[-which(county_list == "king")], 5)
county_sample <- c(county_sample, "king")
part_b <- wa_crime_df %>%
  filter(County %in% county_sample) %>%
  mutate(
    strata_label = if_else(County == "king", "strata 1", "strata 2"),
    num_counties = if_else(County == "king", 1, length(county_list) - 1)
  ) %>%
  as_survey_design(
    ids = c(County, Agency),
    fpc = c(num_counties, num_agencies),
    strata = strata_label
  ) %>%
  summarize(
    total = survey_total(murder_and_crime)
  ) %>%
  mutate(Q = "b")
part_b
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
   total total_se Q    
   <dbl>    <dbl> <chr>
1 92233.   55064. b    
```
:::
:::


  * Take simple random samples of five police districts from King County and
  five counties from the rest of the state. Estimate the total number of murders
  and burglaries in the state and compare to the previous estimates.

This is a stratified two-stage sample design with no uncertainty in the 
first stage in one (king county) strata, and no uncertainty in the second stage 
in the other (non-king counties) strata.

::: {.cell}

```{.r .cell-code}
king_districts <- wa_crime_df %>%
  filter(County == "king") %>%
  pull(Agency)
sampled_king_districts <- sample(king_districts, 5)
sampled_counties <- sample(county_list, 5)

part_c <- wa_crime_df %>%
  filter(County %in% sampled_counties | Agency %in% sampled_king_districts) %>%
  mutate(
    strata_label = if_else(County == "king", "King County", "WA Counties"),
    num_counties = if_else(County == "king", 1, length(county_list) - 1),
  ) %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties, num_agencies),
    strata = strata_label
  ) %>%
  summarize(total = survey_total(murder_and_crime, vartype = "se")) %>%
  mutate(Q = "c")
part_c
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
  total total_se Q    
  <dbl>    <dbl> <chr>
1 24160    2530. c    
```
:::
:::



  * Take a probability proportional to size (PPS) sample of 10 police districts.
  Estimate the total number of murders and burglaries in the state and compare
  to the previous estimates.


::: {.cell}

```{.r .cell-code}
pi <- sampling::inclusionprobabilities(a = wa_crime_df$Population, n = 10)
part_d <- wa_crime_df %>%
  mutate(
    pi = pi,
    in_sample = sampling::UPbrewer(pi)
  ) %>%
  filter(in_sample == 1) %>%
  as_survey_design(probs = pi) %>%
  summarize(total = survey_total(murder_and_crime)) %>%
  mutate(Q = "d")
part_d
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
   total total_se Q    
   <dbl>    <dbl> <chr>
1 47256.    5853. d    
```
:::
:::


This has the lowest standard error yet, likely since the sampling was
specifically designed to capture high density districts.

e) Take a simple random sample of counties and include all the police districts.
Estimate the total number of murders and burglaries in the state and
compare to the previous estimates.


::: {.cell}

```{.r .cell-code}
county_sample <- sample(county_list, 10)
part_e <- wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design(
    ids = c(County, Agency),
    fpc = c(num_counties, num_agencies)
  ) %>%
  summarize(total = survey_total(murder_and_crime)) %>%
  mutate(Q = "e")
part_e
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
   total total_se Q    
   <dbl>    <dbl> <chr>
1 38594.   16828. e    
```
:::
:::



I'll compare the standard error of all the estimates now.


::: {.cell}

```{.r .cell-code}
rbind(part_a, part_b, part_c, part_d, part_e) %>%
  select(Q, total_se) %>%
  gt::gt()
```

::: {.cell-output-display}
```{=html}
<div id="djxwwwogbv" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#djxwwwogbv table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#djxwwwogbv thead, #djxwwwogbv tbody, #djxwwwogbv tfoot, #djxwwwogbv tr, #djxwwwogbv td, #djxwwwogbv th {
  border-style: none;
}

#djxwwwogbv p {
  margin: 0;
  padding: 0;
}

#djxwwwogbv .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#djxwwwogbv .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#djxwwwogbv .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#djxwwwogbv .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#djxwwwogbv .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#djxwwwogbv .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#djxwwwogbv .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#djxwwwogbv .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#djxwwwogbv .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#djxwwwogbv .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#djxwwwogbv .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#djxwwwogbv .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#djxwwwogbv .gt_spanner_row {
  border-bottom-style: hidden;
}

#djxwwwogbv .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#djxwwwogbv .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#djxwwwogbv .gt_from_md > :first-child {
  margin-top: 0;
}

#djxwwwogbv .gt_from_md > :last-child {
  margin-bottom: 0;
}

#djxwwwogbv .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#djxwwwogbv .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#djxwwwogbv .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#djxwwwogbv .gt_row_group_first td {
  border-top-width: 2px;
}

#djxwwwogbv .gt_row_group_first th {
  border-top-width: 2px;
}

#djxwwwogbv .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#djxwwwogbv .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#djxwwwogbv .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#djxwwwogbv .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#djxwwwogbv .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#djxwwwogbv .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#djxwwwogbv .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#djxwwwogbv .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#djxwwwogbv .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#djxwwwogbv .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#djxwwwogbv .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#djxwwwogbv .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#djxwwwogbv .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#djxwwwogbv .gt_left {
  text-align: left;
}

#djxwwwogbv .gt_center {
  text-align: center;
}

#djxwwwogbv .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#djxwwwogbv .gt_font_normal {
  font-weight: normal;
}

#djxwwwogbv .gt_font_bold {
  font-weight: bold;
}

#djxwwwogbv .gt_font_italic {
  font-style: italic;
}

#djxwwwogbv .gt_super {
  font-size: 65%;
}

#djxwwwogbv .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#djxwwwogbv .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#djxwwwogbv .gt_indent_1 {
  text-indent: 5px;
}

#djxwwwogbv .gt_indent_2 {
  text-indent: 10px;
}

#djxwwwogbv .gt_indent_3 {
  text-indent: 15px;
}

#djxwwwogbv .gt_indent_4 {
  text-indent: 20px;
}

#djxwwwogbv .gt_indent_5 {
  text-indent: 25px;
}

#djxwwwogbv .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#djxwwwogbv div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1" scope="col" id="Q">Q</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="total_se">total_se</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="Q" class="gt_row gt_left">a</td>
<td headers="total_se" class="gt_row gt_right">26038.557</td></tr>
    <tr><td headers="Q" class="gt_row gt_left">b</td>
<td headers="total_se" class="gt_row gt_right">55064.174</td></tr>
    <tr><td headers="Q" class="gt_row gt_left">c</td>
<td headers="total_se" class="gt_row gt_right">2530.232</td></tr>
    <tr><td headers="Q" class="gt_row gt_left">d</td>
<td headers="total_se" class="gt_row gt_right">5852.756</td></tr>
    <tr><td headers="Q" class="gt_row gt_left">e</td>
<td headers="total_se" class="gt_row gt_right">16828.106</td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


We see the highest variance in estimates from the simple random samples - 
Q's a) and e). We see the lowest variance from part d) which used the 
proportional to size sampling scheme. Finally, we see the two 
stratified multi-stage samples have variances in between, all of which makes 
sense.

7. Use the household data from the 1966 SIP panel to estimate the 25th, 50th,
75th, 90th and 95th percentiles of income for households of different sizes
(`ehhnumpp`) averaged over the fourth months. You will want to re-code the large
values of `ehhnumpp` to a a single category. Describe the patterns you see.

For this question we effectively copy the code from the "Repeated Measures"
section, looking at quantile by a re-coded household size now, instead of
month.


::: {.cell}

```{.r .cell-code}
sipp_hh <- update(sipp_hh,
  household_size = factor(
    case_when(
      ehhnumpp <= 8 ~ as.character(ehhnumpp),
      TRUE ~ ">=9"
    ),
    levels = c(as.character(1:8), ">=9")
  )
)
qinc <- svyby(~thtotinc, ~household_size, svyquantile,
  design = sipp_hh,
  quantiles = c(0.25, 0.5, 0.75, 0.9, 0.95), se = TRUE
)
```
:::

::: {.cell}

```{.r .cell-code}
pltdf <- as_tibble(qinc) %>%
  select(household_size, contains("thtotinc"), -contains("se.")) %>%
  gather(everything(), -household_size, key = "quantile", value = "Total Income") %>%
  mutate(quantile = as.numeric(str_extract(quantile, "[0-9].[0-9]?[0-9]")) * 100)

se <- as_tibble(qinc) %>%
  select(household_size, contains("se.")) %>%
  gather(everything(), -household_size, key = "quantile", value = "SE") %>%
  mutate(quantile = as.numeric(str_extract(quantile, "[0-9].[0-9]?[0-9]")) * 100)

pltdf <- pltdf %>%
  left_join(se) %>%
  mutate(
    lower = `Total Income` - 2 * SE,
    upper = `Total Income` + 2 * SE
  )

pltdf %>%
  mutate(quantile = factor(quantile)) %>%
  ggplot(aes(x = household_size, y = `Total Income`, color = quantile)) +
  geom_pointrange(aes(ymin = lower, ymax = upper)) +
  xlab("Household Size") +
  ylab("Total Income (USD)") +
  ggtitle("Total Income Quantiles",
    subtitle = "Survey of Income and Program Participation"
  )
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q_3_7_2-1.png){width=672}
:::
:::


In the plot above we can see that the income variability gets wider as 
household size increases but then plateaus at around ~ 5. Additionally, there
are few samples with very large households so estimating the quantiles for those
groups is increasingly noisy.

8. In the data from the 1996 SIPP panel

  * What proportion of households received any "means-tested cash benefits"
  (`thtrninc`)? For those households that did receive benefits what mean
  proportion of their total income came from these benefits?



::: {.cell}

```{.r .cell-code}
db <- DBI::dbConnect(RSQLite::SQLite(), "Data/SIPP/sipp.db")
sipp_df <- tbl(db, sql("SELECT * FROM household")) %>%
  dplyr::collect() %>%
  select(ghlfsam, gvarstr, whfnwgt, thtotinc, thtrninc, tmthrnt) %>%
  mutate(
    benefit_recipient = I(thtrninc > 0) * 1,
    thtrninc = as.numeric(thtrninc),
    tmthrnt = as.numeric(tmthrnt)
  )
DBI::dbDisconnect(db)

sipp_hh_sub <- sipp_df %>%
  as_survey_design(
    id = ghlfsam, strata = gvarstr, nest = TRUE,
    weight = whfnwgt
  )
sipp_hh_sub %>%
  summarize(prop_benefit_recipients = survey_mean(benefit_recipient))
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  prop_benefit_recipients prop_benefit_recipients_se
                    <dbl>                      <dbl>
1                  0.0829                    0.00174
```
:::
:::

::: {.cell}

```{.r .cell-code}
sipp_hh_sub %>%
  filter(benefit_recipient == 1) %>%
  mutate(prop_benefit_income = thtrninc / thtotinc) %>%
  summarize(
    mn_prop_benefit_income = survey_mean(prop_benefit_income)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  mn_prop_benefit_income mn_prop_benefit_income_se
                   <dbl>                     <dbl>
1                  0.497                   0.00710
```
:::
:::



  * What proportion of households paid rent `tmthrnt`? What were the mean and the
  75th and 95th percentiles of the proportion of monthly income paid in rent?
  What proportion paid more than one third of their income in rent?


::: {.cell}

```{.r .cell-code}
sipp_hh_sub %>%
  mutate(
    pays_rent = I(as.numeric(tmthrnt) > 0) * 1,
    pct_income_rent = (tmthrnt / thtotinc) * 100,
    rent_pct_gt_thrd = (pct_income_rent > 0.33) * 1
  ) %>%
  # avoid divide by zero error
  filter(thtotinc > 0) %>%
  summarize(
    prop_pays_rent = survey_mean(pays_rent),
    mean_pct_income_rent = survey_mean(pct_income_rent, na.rm = TRUE),
    quantile_pct_income_rent = survey_quantile(pct_income_rent,
      na.rm = TRUE,
      quantiles = c(.75, .95)
    ),
    prop_rent_gt_thrd = survey_mean(rent_pct_gt_thrd, na.rm = TRUE)
  ) %>%
  pivot_longer(cols = everything(), names_to = "Rent", values_to = "Estimates")
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 10 × 2
   Rent                            Estimates
   <chr>                               <dbl>
 1 prop_pays_rent                    0.0401 
 2 prop_pays_rent_se                 0.00111
 3 mean_pct_income_rent              2.24   
 4 mean_pct_income_rent_se           0.633  
 5 quantile_pct_income_rent_q75      0      
 6 quantile_pct_income_rent_q95      0      
 7 quantile_pct_income_rent_q75_se   1.42   
 8 quantile_pct_income_rent_q95_se   1.42   
 9 prop_rent_gt_thrd                 0.0400 
10 prop_rent_gt_thrd_se              0.00111
```
:::
:::



9. By the time you read this, the `survey` package is likely to provide some
approximations for PPS designs that use the finite population size information.
Repeat 3.2 using these.


::: {.cell hash='ComplexSurveyNotes_cache/html/q_3_9_simulated_pps_f1e8d9039680678d9c41820f4ac216c4'}

```{.r .cell-code}
OneSimulation <- function() {
  insample <- sampling::UPtille(election$p)
  ppsample <- election[insample == 1, ]
  ppsample$wt <- 1 / ppsample$p
  pps_design <- svydesign(id = ~1, weight = ~wt, data = ppsample, pps = "brewer")
  total <- svytotal(~ Bush + Kerry + Nader, pps_design)
  std_err <- sqrt(diag(attr(total, "var")))
  ci <- confint(total)
  out <- cbind(total, std_err, ci)
}
total_sims <- replicate(100, OneSimulation())
```
:::


  * Check that the mean of the estimated totals is close to the true population 
  totals.


::: {.cell hash='ComplexSurveyNotes_cache/html/q_3_9_fpcpps_c51c26ec6b645e0e7d38a247bce57469'}

```{.r .cell-code}
OneSimulation <- function() {
  insample <- sampling::UPtille(election$p)
  ppsample <- election[insample == 1, ]
  ppsample$wt <- 1 / ppsample$p
  ppsample$fpc <- 40 / sum(election$votes)
  pps_design <- svydesign(id = ~1, weight = ~wt, fpc = ~fpc, data = ppsample, pps = "brewer")
  total <- svytotal(~ Bush + Kerry + Nader, pps_design)
  std_err <- sqrt(diag(attr(total, "var")))
  ci <- confint(total)
  out <- cbind(total, std_err, ci)
}
total_sims <- replicate(100, OneSimulation())
```
:::

::: {.cell hash='ComplexSurveyNotes_cache/html/q_3_9_fpc_pps_totals_d136e539c5e7ad3b5ecf71d6993acd9d'}

```{.r .cell-code}
estimated_totals <- as_tibble(t(rowMeans(total_sims[, 1, ]))) %>%
  mutate(label = "Estimate")

true_totals <- as_tibble(t(colSums(election[, c("Bush", "Kerry", "Nader")]))) %>%
  mutate(label = "Truth")

rbind(estimated_totals, true_totals) %>%
  gather(everything(), -label, key = "Candidate", value = "Vote Count") %>%
  spread(label, `Vote Count`) %>%
  gt::gt() %>%
  gt::fmt_scientific() %>%
  gt::tab_header("Simulated Total Comparison")
```

::: {.cell-output-display}
```{=html}
<div id="hxeisqgesw" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#hxeisqgesw table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#hxeisqgesw thead, #hxeisqgesw tbody, #hxeisqgesw tfoot, #hxeisqgesw tr, #hxeisqgesw td, #hxeisqgesw th {
  border-style: none;
}

#hxeisqgesw p {
  margin: 0;
  padding: 0;
}

#hxeisqgesw .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#hxeisqgesw .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#hxeisqgesw .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#hxeisqgesw .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#hxeisqgesw .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#hxeisqgesw .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#hxeisqgesw .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#hxeisqgesw .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#hxeisqgesw .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#hxeisqgesw .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#hxeisqgesw .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#hxeisqgesw .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#hxeisqgesw .gt_spanner_row {
  border-bottom-style: hidden;
}

#hxeisqgesw .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#hxeisqgesw .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#hxeisqgesw .gt_from_md > :first-child {
  margin-top: 0;
}

#hxeisqgesw .gt_from_md > :last-child {
  margin-bottom: 0;
}

#hxeisqgesw .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#hxeisqgesw .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#hxeisqgesw .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#hxeisqgesw .gt_row_group_first td {
  border-top-width: 2px;
}

#hxeisqgesw .gt_row_group_first th {
  border-top-width: 2px;
}

#hxeisqgesw .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#hxeisqgesw .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#hxeisqgesw .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#hxeisqgesw .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#hxeisqgesw .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#hxeisqgesw .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#hxeisqgesw .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#hxeisqgesw .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#hxeisqgesw .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#hxeisqgesw .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#hxeisqgesw .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#hxeisqgesw .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#hxeisqgesw .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#hxeisqgesw .gt_left {
  text-align: left;
}

#hxeisqgesw .gt_center {
  text-align: center;
}

#hxeisqgesw .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#hxeisqgesw .gt_font_normal {
  font-weight: normal;
}

#hxeisqgesw .gt_font_bold {
  font-weight: bold;
}

#hxeisqgesw .gt_font_italic {
  font-style: italic;
}

#hxeisqgesw .gt_super {
  font-size: 65%;
}

#hxeisqgesw .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#hxeisqgesw .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#hxeisqgesw .gt_indent_1 {
  text-indent: 5px;
}

#hxeisqgesw .gt_indent_2 {
  text-indent: 10px;
}

#hxeisqgesw .gt_indent_3 {
  text-indent: 15px;
}

#hxeisqgesw .gt_indent_4 {
  text-indent: 20px;
}

#hxeisqgesw .gt_indent_5 {
  text-indent: 25px;
}

#hxeisqgesw .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#hxeisqgesw div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_heading">
      <td colspan="3" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Simulated Total Comparison</td>
    </tr>
    
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1" scope="col" id="Candidate">Candidate</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Estimate">Estimate</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Truth">Truth</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="Candidate" class="gt_row gt_left">Bush</td>
<td headers="Estimate" class="gt_row gt_right">5.97&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td>
<td headers="Truth" class="gt_row gt_right">5.96&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td></tr>
    <tr><td headers="Candidate" class="gt_row gt_left">Kerry</td>
<td headers="Estimate" class="gt_row gt_right">5.61&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td>
<td headers="Truth" class="gt_row gt_right">5.61&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td></tr>
    <tr><td headers="Candidate" class="gt_row gt_left">Nader</td>
<td headers="Estimate" class="gt_row gt_right">4.06&nbsp;×&nbsp;10<sup style='font-size: 65%;'>5</sup></td>
<td headers="Truth" class="gt_row gt_right">4.04&nbsp;×&nbsp;10<sup style='font-size: 65%;'>5</sup></td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


These are quite close.

* Compute the mean of the estimated standard errors and compare it to the
true simulation standard error, that is, the standard deviation of the 
estimated totals.


::: {.cell hash='ComplexSurveyNotes_cache/html/q_3_9_pps_stderrs_e94f2b840a2ee4255aa6c07c8323863d'}

```{.r .cell-code}
estimated_stderrs <- as_tibble(t(rowMeans(total_sims[, 2, ]))) %>%
  mutate(label = "Estimate")

true_stderr <- as_tibble(apply(total_sims, 1, sd)) %>%
  mutate(label = "Truth", Candidate = c("Bush", "Kerry", "Nader")) %>%
  spread(Candidate, value)

rbind(estimated_stderrs, true_stderr) %>%
  gather(everything(), -label, key = "Candidate", value = "Vote Std.Err") %>%
  spread(label, `Vote Std.Err`) %>%
  gt::gt() %>%
  gt::fmt_scientific() %>%
  gt::tab_header("Simulated Standard Error Comparison")
```

::: {.cell-output-display}
```{=html}
<div id="gfxafpiffk" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#gfxafpiffk table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#gfxafpiffk thead, #gfxafpiffk tbody, #gfxafpiffk tfoot, #gfxafpiffk tr, #gfxafpiffk td, #gfxafpiffk th {
  border-style: none;
}

#gfxafpiffk p {
  margin: 0;
  padding: 0;
}

#gfxafpiffk .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#gfxafpiffk .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#gfxafpiffk .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#gfxafpiffk .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#gfxafpiffk .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#gfxafpiffk .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#gfxafpiffk .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#gfxafpiffk .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#gfxafpiffk .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#gfxafpiffk .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#gfxafpiffk .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#gfxafpiffk .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#gfxafpiffk .gt_spanner_row {
  border-bottom-style: hidden;
}

#gfxafpiffk .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#gfxafpiffk .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#gfxafpiffk .gt_from_md > :first-child {
  margin-top: 0;
}

#gfxafpiffk .gt_from_md > :last-child {
  margin-bottom: 0;
}

#gfxafpiffk .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#gfxafpiffk .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#gfxafpiffk .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#gfxafpiffk .gt_row_group_first td {
  border-top-width: 2px;
}

#gfxafpiffk .gt_row_group_first th {
  border-top-width: 2px;
}

#gfxafpiffk .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#gfxafpiffk .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#gfxafpiffk .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#gfxafpiffk .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#gfxafpiffk .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#gfxafpiffk .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#gfxafpiffk .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#gfxafpiffk .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#gfxafpiffk .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#gfxafpiffk .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#gfxafpiffk .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#gfxafpiffk .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#gfxafpiffk .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#gfxafpiffk .gt_left {
  text-align: left;
}

#gfxafpiffk .gt_center {
  text-align: center;
}

#gfxafpiffk .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#gfxafpiffk .gt_font_normal {
  font-weight: normal;
}

#gfxafpiffk .gt_font_bold {
  font-weight: bold;
}

#gfxafpiffk .gt_font_italic {
  font-style: italic;
}

#gfxafpiffk .gt_super {
  font-size: 65%;
}

#gfxafpiffk .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#gfxafpiffk .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#gfxafpiffk .gt_indent_1 {
  text-indent: 5px;
}

#gfxafpiffk .gt_indent_2 {
  text-indent: 10px;
}

#gfxafpiffk .gt_indent_3 {
  text-indent: 15px;
}

#gfxafpiffk .gt_indent_4 {
  text-indent: 20px;
}

#gfxafpiffk .gt_indent_5 {
  text-indent: 25px;
}

#gfxafpiffk .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#gfxafpiffk div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_heading">
      <td colspan="3" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Simulated Standard Error Comparison</td>
    </tr>
    
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1" scope="col" id="Candidate">Candidate</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Estimate">Estimate</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Truth">Truth</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="Candidate" class="gt_row gt_left">Bush</td>
<td headers="Estimate" class="gt_row gt_right">2.49&nbsp;×&nbsp;10<sup style='font-size: 65%;'>6</sup></td>
<td headers="Truth" class="gt_row gt_right">2.51&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td></tr>
    <tr><td headers="Candidate" class="gt_row gt_left">Kerry</td>
<td headers="Estimate" class="gt_row gt_right">2.48&nbsp;×&nbsp;10<sup style='font-size: 65%;'>6</sup></td>
<td headers="Truth" class="gt_row gt_right">2.36&nbsp;×&nbsp;10<sup style='font-size: 65%;'>7</sup></td></tr>
    <tr><td headers="Candidate" class="gt_row gt_left">Nader</td>
<td headers="Estimate" class="gt_row gt_right">8.37&nbsp;×&nbsp;10<sup style='font-size: 65%;'>4</sup></td>
<td headers="Truth" class="gt_row gt_right">1.96&nbsp;×&nbsp;10<sup style='font-size: 65%;'>5</sup></td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


* Estimate 95% confidence intervals for the population totals and compute the 
proportion of intervals that contain the true population value.


::: {.cell hash='ComplexSurveyNotes_cache/html/39_pps_cis_9949cab64a61abc349e56b0bf92e7bda'}

```{.r .cell-code}
totals <- unlist(true_totals[, 1:3])
prop_contained <- rowMeans((total_sims[, 3, ] < totals &
  total_sims[, 4, ] > totals) * 1)
prop_contained
```

::: {.cell-output .cell-output-stdout}
```
 Bush Kerry Nader 
 0.97  0.97  0.97 
```
:::
:::


10. Repeat 3.2 computing the Hartley-Rao approximation and the full 
Horvitz-Thompson estimate using the code and joint-probability data on the web
site. Compare the standard errors from these two approximations to the standard
errors from the single-stage with replacement approximation and to the true
simulation standard error.

Not clear to me where this code is when looking on his website.

11. Since 1999, NHANES has been conducting surveys continuously with data 
released on a 2 year cycle. Each data set includes a weight variable for
analyzing the two-year data; the weights add to the size of the US adult, 
civilian, non-institutionalized population.

  * What weight would be appropriate for estimating the number of diabetics in
  the population combining data from two two-year data sets?

Assuming these are non-overlapping samples --- which seems reasonable
--- and considering the target population we're generalizing to be 
roughly constant over the 4 years of interest, we could divide each of the 
previous weights in half. This would give us twice the sample size to estimate
the any target estimand considering the four year population as roughly
homogeneous.

* What weight would be appropriate if three two-year data sets were used?

Divide the weights by 3.

* What weights would be appropriate when estimating changes, comparing
the combined 1999-2000 and 2001-2002 data with the combined 2003-2004 and 
2005-2006 data?

You'd need to use the original weights and set these up as different strata.
Or at least, that's the most intuitive answer to me.

* How would the answers differ if the goal was to estimate a population 
proportion rather than a total?

If estimating a population total the weights would need to be adjusted further
such that the population total of each sample remained the same and the two
samples added up to the combined population from each sample -- this would be
more complicated than just dividing by two as suggested above. For proportions
this would matter less when using the simpler method above, assuming no 
dramatic change in the total population, but could still differ slightly 
if using the total adjusted weights.


# Chapter 4: Graphics

Lumley advocates for three principles in visualizing survey data:

1.  Base the graph on an estimated population distribution.

2.  Explicitly indicate weights on the graph.

3.  Draw a simple random sample from the estimated population distribution and
    graph this sample instead.

All three of these strategies are meant to counteract the difficulty in
visualizing survey data --- the data available do not represent the population
of interest without re-weighting.

Like other parts of the book, it isn't always clear how Lumley produces the 
charts in the chapter. I've done my best to reproduce the same charts, or a 
chart I'd consider appropriate for the question / survey at hand. That said, I 
don't always use base R as he does, though I 
do stick to the base R version of the survey plots he implemented in the 
survey package. There is, however, a 
[ggsurvey](https://cran.r-project.org/web/packages/ggsurvey/index.html) package
on CRAN that reproduces his same plots in the tidyverse friendly style fashion.


## Plotting a Table

Lumley recommends using bar charts, forest plots and fourfold plots to visualize
the data from a table. I would agree that all of these are "fine" in a certain
context but strongly prefer the forest plot and variations on it personally,
since it does more than all the other to include representations of uncertainty
about the tabulated estimates.


::: {.cell}

```{.r .cell-code}
medians %>%
  ggplot(aes(x = racehpr, y = Median_BMI, fill = srsex)) +
  geom_bar(stat = "identity", position = "dodge") +
  ylab("Median BMI (kg/m^2)") +
  xlab("") +
  theme(
    legend.title = element_blank(),
    axis.text.x = element_text(angle = 45, vjust = 0.25, hjust = .25)
  ) +
  ggtitle("Median BMI Across Race, Sex")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/barplot_medians-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
medians %>%
  ggplot(aes(x = Median_BMI, y = racehpr, color = srsex)) +
  geom_pointrange(aes(xmin = Median_BMI_low, xmax = Median_BMI_upp),
    position = position_dodge(width = .25)
  ) +
  geom_vline(aes(xintercept = 25), linetype = 2, color = "red") +
  xlab("BMI") +
  ylab("") +
  theme(legend.title = element_blank(), legend.position = "top") +
  ggtitle("Median BMI by Race/Ethnicity and gender, from CHIS") +
  labs(caption = str_c(
    "Line indicates CDC border between healthy and ",
    "overweight BMI."
  ))
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/forestplot_medians-1.png){width=672}
:::
:::


## One Continuous Variable

Lumley argues that the most straightforward graphical displays for a single
continuous variable are boxplots, cumulative distribution function (cdf) plots 
and survival curves. I think histograms are also useful but we'll get to that
shortly.

Below I reproduce Figure 4.6 from the text
in which Lumley demonstrates the difference between the weighted and unweighted
CDF's of the California school data demonstrating that, indeed, the design 
based estimate is closer to the population value than the unweighted, naive
estimate of the stratified design. This should all intuitively make sense.
My only complaint would be that the uncertainty of the cdf isn't visualized
when that should be just as easily accessible. I made a cursory investigation
to see if these data were available in the `svycdf` object but they were not. 



::: {.cell}

```{.r .cell-code}
cdf.est <- svycdf(~ enroll + api00 + api99, dstrata)

cdf.pop <- ecdf(apipop$enroll)

cdf.samp <- ecdf(apistrat$enroll)

par(mar = c(4.1, 4.1, 1.1, 2.1))

plot(cdf.pop,
  do.points = FALSE, xlab = "enrollment",
  ylab = "Cumulative probability",
  main = "Cumulative Distribution of California School Size",
  sub = "Reproduces Lumley Fig 4.6", lwd = 1
)

lines(cdf.samp, do.points = FALSE, lwd = 2)
lines(cdf.est[[1]],
  lwd = 2, col.vert = "grey60", col.hor = "grey60",
  do.points = FALSE
)
legend("bottomright",
  lwd = c(1, 2, 2), bty = "n",
  col = c("black", "grey60", "black"),
  legend = c("Population", "Weighted estimate", "Unweighted Estimate")
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/svycdf-1.png){width=672}
:::
:::


Next Lumley plots an adjusted 
[kaplan meier](https://en.wikipedia.org/wiki/Kaplan%E2%80%93Meier_estimator) 
curve using `svykm()` on the National Wilms Tumor Study data. Unfortunately
he doesn't show how to create the design object and I find no mention of it
elsewhere in the book. I made a guess and it seems to be accurate. Note
that I obtained the national wilms tumor study data from the [`survival`](https://cran.r-project.org/web/packages/survival/index.html) 
package.


::: {.cell}

```{.r .cell-code}
dcchs <- svydesign(ids = ~1, data = nwtco)
scurves <- svykm(Surv(edrel / 365.25, rel) ~ histol, dcchs)
plot(scurves)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/svykm-1.png){width=672}
:::
:::



The interpretation of this graph is that relapse (into cancer) appears to 
occur within about three years, if at all. The top line shows the survival
for those with a favorable histology classification with an even better
survival rate.

Lumley makes a note here describing how the sampling weights make a 
large difference in the estimate because the design is a case cohort sample.
That being said, I don't really see the difference when I compare his figure
to mine created with an equal probability sample... An error here perhaps.


#### Boxplots

Lumley describes the typical box plots as

> based on the quartiles of the data: the box shows the median and first and 
third quartiles, the whiskers extend out to the last observation within 1.5
interquartile ranges of the box ends, and all points beyond the whiskers are 
shown individually.


This is as good a summary as any, I'll only note that the median is the line 
within the interior of the box and the first and third quartile define the 
boundaries of the box. It's (somewhat) easy to image how Lumley estimates
these values, as he has already introduced the `svyquantile()` function ---
this should be a straightforward application of that function.

I reproduce Figure 4.10 from the text using the nhanes data and design object
created in answering the questions for chapter 3. His code is equivalent.


::: {.cell}

```{.r .cell-code}
## Lumley's code
# nhanes <- svydesign(id = ~SDMVPSU, strat = ~SDMVSTRA,
#                    weights = ~WTMEC2YR, data = both, nest = TRUE)

## I use the bpdf design from the chapter 3 questions
nhanes <- bpdf_design %>%
  mutate(
    age_group = cut(RIDAGEYR, c(0, 20, 30, 40, 50, 60, 80, Inf)),
  )
svyboxplot(BPXSAR ~ age_group, subset(nhanes, BPXSAR > 0),
  col = "gray80",
  varwidth = TRUE, ylab = "Systolic BP (mmHg)", xlab = "Age"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/boxplots-1.png){width=672}
:::
:::

The plot shows the distribution in blood pressure across the different age 
groups. I already showed how I visualized this previously, but it is nice
to know how to do something multiple ways. In general, our observation shows
an increase in systolic blood pressure medians and third quartiles across age.


#### Graphs based on the density

Histograms and kernel density estimators both visualize the probability
distribution function(pdf) of a given data set. Confusingly, perhaps, there 
is no pdf to estimate in finite population inference because the sampling
is random instead of the data. Still, the analogous effort can be made where
a given proportion is estimated for some number of bins of the variable of 
interest. Instead of estimating a naive proportion, there's now a sample based
proportion. I reproduce Lumley's Figure 4.12 below demonstrating how this 
works.


::: {.cell}

```{.r .cell-code}
svyhist(~BPXSAR, subset(nhanes, RIDAGEYR > 20 & BPXSAR > 0),
  main = "",
  col = "grey80", xlab = "Systolic BP (mmHg)"
)
lines(svysmooth(~BPXSAR, nhanes,
  bandwidth = 5,
  subset = subset(nhanes, RIDAGEYR > 20 & BPXSAR > 0)
), lwd = 2)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/svyhist-1.png){width=672}
:::
:::


This looks considerably different then the naive estimate, again showcasing
the need for taking the design into account.


::: {.cell}

```{.r .cell-code}
hist(bpdf_design$variables$BPXSAR,
  xlab = "Systolic BP (mmHg)",
  main = "Naive Distribution of Blood Pressure"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/naive_hist-1.png){width=672}
:::
:::



#### Two Continuous Variables

Where the design could be used to adjust the raw sample summaries to adjust
for the design with one variable, two variable visualizations make this much
more difficult as there is no way to incorporate the design information into
the presented estimate. An additional dimension has to be used to illustrate
the design information.


#### Scatterplots


To that end, Lumley uses a bubble plot with the size of the "bubbles" 
proportional to the sampling weight. This way, the viewer can identify which 
points should be considered as "more" or less impactful in terms of representing 
the population. I reproduce part of Figures 4.13 - 4.15 from the text below, 
for those plots in which Lumley includes the code. plots the systolic and 
diastolic blood pressure from the NHANES design.



::: {.cell}

```{.r .cell-code}
# TODO(petersonadam): Try plotting the api data here too
par(mfrow = c(1, 2))
svyplot(BPXSAR ~ BPXDAR,
  design = nhanes, style = "bubble",
  xlab = "Diastolic pressure (mmHg)",
  ylab = "Systolic pressure (mmHg)"
)
svyplot(BPXSAR ~ BPXDAR,
  design = nhanes, style = "transparent",
  pch = 19, alpha = c(0, 0.5),
  xlab = "Diastolic pressure (mmHg)",
  ylab = "Systolic pressure (mmHg)"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/scatterplots_demo-1.png){width=672}
:::
:::


As Lumley notes, this is a case in which the sampling weights --- bubble size 
--- is not particularly informative, as we don't see any particularly obvious 
pattern relating the size of the bubbles to the two variables of interest. 
Turning to the shading plot, though we can now see more than just the "blob"
in the first plot, there's still no apparent pattern between the 
bubble size and the two variables to identify.

Below, I include Lumley's "sub-sample" method, which draws a simple random
sample from the target population by using the provided sampling probabilities.
Lumley recommends looking at 2 or three iterations of a sub-sample plot
in order to ensure that any features visualized are not "noise" from the 
sampling process.


::: {.cell}

```{.r .cell-code}
svyplot(BPXSAR ~ BPXDAR,
  design = nhanes, style = "subsample",
  sample.size = 1000,
  xlab = "Diastolic pressure (mmHg)",
  ylab = "Systolic pressure (mmHg)"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/subsample-1.png){width=672}
:::
:::


#### Aggregation and smoothing

While less of an issue than it was at the time of Lumley's writing, 
aggregating and smoothing across points makes it easier to condense the
number of sample points in a visualization to reduce the memory required to
visualize all the points in the sample. In a design setting it also provides
an opportunity to incorporate the design information into a visualization
specific estimate.


Lumley's first example of this is a hexplot - a plot where a grid of hexagons
is created and sized according to the number of points within the hex-bin.


::: {.cell}

```{.r .cell-code}
svyplot(BPXSAR ~ BPXDAR,
  design = nhanes, style = "hex", legend = 0,
  xlab = "Diastolic pressure (mmHg)",
  ylab = "Systolic pressure (mmHg)"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/hexplot-1.png){width=672}
:::
:::


This is perhaps the best visualization of the data thus far, where the "blob"
is still apparent but outliers are still visible and a more concise summary
of the data is easily visible.


#### Scatterplot smoothers

Lumley next describes how to take the discrete estimates examining blood
pressure as a function of age and smooth them when both variables are 
continuous. The code below shows how to do this by using the `svyplot()` 
function using quantile regression methods from the 
[`quantreg`](https://cran.r-project.org/web/packages/quantreg/index.html) 
package. Mean estimates can also be obtained by using `method = 'locpoly'`.



::: {.cell}

```{.r .cell-code}
adults <- subset(nhanes, !is.na(BPXSAR))
a25 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .25, df = 4
)
a50 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .5, df = 4
)
a75 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .75, df = 4
)
a10 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .1, df = 4
)
a90 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .9, df = 4
)
plot(BPXSAR ~ RIDAGEYR,
  data = nhanes, type = "n", ylim = c(80, 200),
  xlab = "Age", ylab = "Systolic Pressure"
)
lines(a50, lwd = 3)
lines(a25, lwd = 1)
lines(a75, lwd = 1)
lines(a10, lty = 3)
lines(a90, lty = 3)
legend("topleft",
  legend = c("10%,90%", "25%, 75%", "median"),
  lwd = c(1, 1, 3), lty = c(3, 1, 1), bty = "n"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/scatterplot_smoother-1.png){width=672}
:::
:::


#### Conditioning Plots

A conditioning plot is effectively a scatter plot with a third 
variable fixed. This third variable is then displayed in the facet title of the 
plot. In Lumley's text he shows how to do this using a call to the 
`svycoplot()` function to recreate the top half of Figure 4.20. Of note,
the additional `hexscale` argument can be fed the `"absolute"` argument to
make the scales comparable between panels - by default the scales are
facet specific, though the data (for a continuous facet) has a ~50% overlap so
the scales are not dramatically different.



::: {.cell}

```{.r .cell-code}
svycoplot(BPXSAR ~ BPXDAR | equal.count(RIDAGEMN),
  style = "hex", # or "transparent" for shaded hexs,
  # hexscale = "absolute" # for fixed scales across facets.
  design = subset(nhanes, BPXDAR > 0), xbins = 20,
  strip = strip.custom(var.name = "AGE"),
  xlab = "Diastolic pressure (mm Hg)",
  ylab = "Systolic pressure (mm Hg)"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/svycoplot-1.png){width=672}
:::
:::


## Maps

### Design and Estimation Issues

The final section in this chapter looks at how to visualize survey data 
spatially across a map. Since many surveys contain geographic information
in both their sampling design and the questions they seek to answer, it 
makes sense that one might want to visualize estimates at some geographic
scale.

Lumley uses the `maptools` R package to take estimates computed
using techniques/functions already demonstrated and visualize them on maps.

Since maptools isn't available on CRAN at the current time of writing, I'll
use the [sf package](https://r-spatial.github.io/sf/) and `ggplot2`.


Below I reproduce Figure 4.21 from the text using these packages with
the same design based estimates in the text. The data was procured from
Lumley's [website](https://r-survey.r-forge.r-project.org/svybook/) --- 
search for "BRFSS".


::: {.cell}

```{.r .cell-code}
states <- read_sf("Data/BRFSS2007/brfss_state_2007_download.shp") %>%
  arrange(ST_FIPS)
bys <- svyby(~X_FRTSERV, ~X_STATE, svymean,
  design = subset(brfss, X_FRTSERV < 99999)
)
state_fruit_servings <- states %>%
  select(ST_FIPS) %>%
  st_drop_geometry() %>%
  left_join(bys, by = c("ST_FIPS" = "X_STATE")) %>%
  mutate(geometry = states$geometry) %>%
  st_as_sf()

state_fruit_servings %>%
  ggplot() +
  geom_sf(aes(fill = X_FRTSERV / 100)) +
  theme_void() +
  theme(legend.title = element_blank()) +
  ggtitle("Servings of fruit per day, from BRFSS 2007")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/maps_demo-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
hlth <- brfss %>%
  as_survey_design() %>%
  mutate(
    agegp = cut(AGE, c(0, 35, 50, 65, Inf)),
    state = X_STATE,
    covered = (HLTHPLAN == 1) * 1
  ) %>%
  group_by(agegp, state) %>%
  summarize(
    health_coverage = survey_mean(covered)
  ) %>%
  ## Formatting
  mutate(Age = case_when(
    agegp == "(0,35]" ~ "<35",
    agegp == "(35,50]" ~ "35-50",
    agegp == "(50,65]" ~ "50-65",
    agegp == "(65,Inf]" ~ "65+"
  ))



insurance_coverage <- states %>%
  select(ST_FIPS, geometry) %>%
  left_join(hlth, by = c("ST_FIPS" = "state")) %>%
  st_as_sf()

insurance_coverage %>%
  ggplot(aes(fill = health_coverage)) +
  geom_sf() +
  facet_wrap(~Age) +
  theme_void() +
  theme(legend.title = element_blank())
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/brfss_fruit_age-1.png){width=672}
:::
:::


Lumley then shows two plots with insurance coverage at the state level. 
Looking at the 
[code](https://r-survey.r-forge.r-project.org/svybook/maps-webpage.R) he posted
on the web it isn't clear to me what he's estimating that's different here as
I don't see any calls to any of the survey functions. Consequently,
I've just created some fake simulated data and created a plot with the 
equivalent data.


::: {.cell}

```{.r .cell-code}
cities <- read_sf("Data/BRFSS2007/BRFSS_MMSA_2007.shp") %>%
  filter(NAME != "Columbus") %>%
  transmute(Insurance = cut(rbeta(n(), 1, 1), c(0, 0.25, .5, .75, 1)))

marginal_insurance <- brfss %>%
  as_survey_design() %>%
  mutate(
    covered = (HLTHPLAN == 1) * 1,
    state = X_STATE,
  ) %>%
  group_by(state) %>%
  summarize(
    health_coverage = survey_mean(covered)
  ) %>%
  ungroup()

map_data <- states %>%
  select(ST_FIPS, geometry) %>%
  left_join(marginal_insurance, by = c("ST_FIPS" = "state")) %>%
  st_as_sf()

ggplot() +
  geom_sf(data = map_data, aes(fill = health_coverage)) +
  geom_sf(data = cities, aes(color = Insurance)) +
  theme_void() +
  theme(legend.title = element_blank()) +
  ggtitle("Insurance Coverage - from BRFSS and Fake Data")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/city_data-1.png){width=672}
:::
:::


## Exercises

1. Draw box plots of body mass index by race/ethnicity and by sex using the 
CHIS 2005 data introduced in Chapter 2.


::: {.cell}

```{.r .cell-code}
svyboxplot(bmi_p ~ racehpr, chis, main = "BMI By Race/Ethnicity")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/chis_bmi_race-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
svyboxplot(bmi_p ~ srsex, chis, main = "BMI BY Sex")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/chis_bmi_sex-1.png){width=672}
:::
:::


2. Using the code in Figure 3.8 draw a bar plot of the quantiles of income
and compare it to the dot chart in Figure 3.9. What are some advantages and
disadvantages of each display.


::: {.cell}

```{.r .cell-code}
pltdf <- as_tibble(qinc) %>%
  select(household_size, contains("thtotinc"), -contains("se.")) %>%
  gather(everything(), -household_size, key = "quantile", value = "Total Income") %>%
  mutate(quantile = as.numeric(str_extract(quantile, "[0-9].[0-9]?[0-9]")) * 100)

se <- as_tibble(qinc) %>%
  select(household_size, contains("se.")) %>%
  gather(everything(), -household_size, key = "quantile", value = "SE") %>%
  mutate(quantile = as.numeric(str_extract(quantile, "[0-9].[0-9]?[0-9]")) * 100)

pltdf <- pltdf %>%
  left_join(se) %>%
  mutate(
    lower = `Total Income` - 2 * SE,
    upper = `Total Income` + 2 * SE
  )

pltdf %>%
  mutate(quantile = factor(quantile)) %>%
  ggplot(aes(x = household_size, y = `Total Income`, fill = quantile)) +
  geom_bar(stat = 'identity', position='dodge') +
  geom_errorbar(aes(ymin = lower, ymax = upper), position='dodge') +
  xlab("Household Size") +
  ylab("Total Income (USD)") +
  ggtitle("Total Income Quantiles",
    subtitle = "Survey of Income and Program Participation"
  )
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_2-1.png){width=672}
:::
:::


I prefer the dot plot as I feel the bar plot takes up more space to convey the 
same amount of information. However, I think the bar plot might make it easier 
to compare the income quantiles within household size as they are right next to
each other.

3. Use `svysmooth()` to draw a graph showing change in systolic and diastolic
blood pressure over time in the NHANES 2003-2004 data. Can you see the change
to isolated systolic hypertension in old age that is shown in Figure 4.5.


::: {.cell}

```{.r .cell-code}
plot(svysmooth(BPXSAR ~ RIDAGEYR, nhanes))
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_3-1.png){width=672}
:::
:::



4. With the data from the SIPP 1996 panel draw the cumulative distribution 
function  density function, a histogram and a box plot of total household 
income. Compare these graphs for their usefulness in showing the distribution of
income.


::: {.cell}

```{.r .cell-code}
sipp_hh <- svydesign(
  id = ~ghlfsam, strata = ~gvarstr, nest = TRUE,
  weight = ~whfnwgt, data = "household", dbtype = "SQLite",
  dbname = "Data/SIPP/sipp.db"
)
plot(svycdf(~thtotinc, sipp_hh))
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_4_cdf-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
svyhist(~thtotinc, sipp_hh)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_4_hist-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
svyboxplot(thtotinc~1, sipp_hh)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_4_boxplot-1.png){width=672}
:::
:::


I think I prefer the histogram the most out of these in terms of conveying the 
most information with the space available. Both the box plot and the cdf give
a good sense that the data is skewed, but gives very little sense of how much
density or probability mass can be found in the lower incomes. Only the 
histogram does that to my satisfaction.

5. With the data from the SIPP 1996 panel draw a graph showing amount of rent 
(`tmthrnt`)  and proportion of income paid to rent. You will want to exclude
some outlying households that report much higher rent than income.


::: {.cell}

```{.r .cell-code}
sipp_hh <- update(sipp_hh,
    pct_income_rent = (as.numeric(tmthrnt) / as.numeric(thtotinc) * 100),
  ) 
svyplot(tmthrnt ~ pct_income_rent, 
        subset(sipp_hh,
               pct_income_rent < Inf &
               tmthrnt < thtotinc &
               pct_income_rent < 100),
        xlab = "% Income Paid To Rent",
        ylab = "Rent (USD)") 
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_5-1.png){width=672}
:::
:::


6. Using data from CHIS 2005 (see section 2.3.1) examine how body mass index
varies with age as we did with blood pressure in this chapter.


::: {.cell}

```{.r .cell-code}
plot(svysmooth(bmi_p ~ srage_p, chis),
  xlab = "Age", ylab = "BMI",
  main = "BMI vs. Age (CHIS data)"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_6-1.png){width=672}
:::
:::


We see a very distinctive U-shape - with middle aged individuals - 50 to 60 year
olds - having the highest average BMI, but the young and old having lower BMI's.

7. The left-hand panel of Figure 3.3 shows an interesting two-lobed pattern. 
Can you find what makes these lobes?

As stated previously, I don't have access to this data.

8. Set up a survey design for the BRFSS 2007 data as in Figure 3.7. BRFSS
measured annual income (`income2`) in categories < \$10K, \$10-15k \$20-25k,
\$25-35k, \$35-50k, \$50-75k, > \$75k and race (`orace`): White, Black, Asian,
Native Hawaiian/Pacific Islander, Native American, other.

* Draw a graph of income by race.
* Draw maps showing the geographical distribution of income and of racial 
groups.
* Draw a set of maps examining whether the geographical distribution of income
differs by race.

I don't see the race or income variables listed as Lumley describes in my
version of the BRFSS data, and while there are certain race variables,
it isn't clear how they map to White, black asian, etc.

9. Explore the impact on the graphs in Figure 4.18 of changes in the amount of 
smoothing, by altering the `df` argument to the code in Figure 4.19


::: {.cell}

```{.r .cell-code}
a25 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .25, df = 7
)
a50 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .5, df = 7
)
a75 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .75, df = 7
)
a10 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .1, df = 7
)
a90 <- svysmooth(BPXSAR ~ RIDAGEYR,
  method = "quantreg", design = adults,
  quantile = .9, df = 7
)
plot(BPXSAR ~ RIDAGEYR,
  data = nhanes, type = "n", ylim = c(80, 200),
  xlab = "Age", ylab = "Systolic Pressure"
)
lines(a50, lwd = 3)
lines(a25, lwd = 1)
lines(a75, lwd = 1)
lines(a10, lty = 3)
lines(a90, lty = 3)
legend("topleft",
  legend = c("10%,90%", "25%, 75%", "median"),
  lwd = c(1, 1, 3), lty = c(3, 1, 1), bty = "n"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q4_9-1.png){width=672}
:::
:::


We see that increasing the degrees of smoothing increases the "wiggliness" of
the plot which makes sense since we're increasing the flexibility of the 
functional form estimated.

# Chapter 5: Ratios and Linear Regression

Lumley identifies two main uses for regression in analyzing complex surveys:

1.  Identifying relationships between variables --- similar to any other data.
2.  More accurate estimates of population means and totals.

In contrast to model based inference which typically discusses linear regression
in the context of distributional assumptions on the outcome variable, Lumley
notes that his discussion of regression is still within the design-based
philosophy and consequently no model based assumptions are needed in order to
compute valid 95% confidence intervals. However, model choice still matters
insofar as one's goal is to precisely estimate a relationship or population mean
or total.

In statistical parlance, Lumley is using 
[GEE](https://en.wikipedia.org/wiki/Generalized_estimating_equation) - 
or generalized estimating equations to fit models, which only require
assumptions about the moments of the underlying data distribution, rather than
the family of the distribution.

My take is that this approach also allows for more flexibility in how the 
variance of the model can be estimated. Since the variance associated with a 
complex design is a function of the design itself, GEE makes it easier
to structure the estimating equations in such a way that the variance
computation corresponds to the design. In fact, Lumley explicitly notes 
in section 5.2.1 that the heteroskedastic / 
[sandwhich based variance estimators](https://stats.stackexchange.com/questions/50778/sandwich-estimator-intuition)
which came out of GEE are used by the `survey` package, with special
handling for combining variance within/across strata.


## Ratio Estimation

Ratio estimation comes up first in this chapter because it is important in
estimating (1) a population mean/total, (2) a ratio directly or (3) a
subpopulation estimate of a mean.

Lumley illustrates how to estimate ratios in design methods by using the api
dataset and estimating the proportion of students who took the Academic
Performance Index exam.


::: {.cell}

```{.r .cell-code}
svyratio(~api.stu, ~enroll, strat_design)
```

::: {.cell-output .cell-output-stdout}
```
Ratio estimator: svyratio.survey.design2(~api.stu, ~enroll, strat_design)
Ratios=
           enroll
api.stu 0.8369569
SEs=
             enroll
api.stu 0.007757103
```
:::
:::


This estimate does a good job of estimating the true population total,
0.84, which we
happen to have access to for this example.

It's worth noting here as Lumley does that ratio estimates are not unbiased but
are classified as "approximately unbiased" since the bias decreases proportional
to the sample size and is consequently smaller than the standard error -- which
decreases proportional to $\frac{1}{\sqrt{n}}$.

### Ratios for subpopulation estimates

In the case where individual - not aggregate - data are
available the ratio being estimated is simply a proportion. This is the same
logic for which subpopulation estimates had been calculated previously in
[Chapter 2:Simple Random Sample] via the `svyby()` function. These estimates
require special handling - though in the survey package they can all be
calculated via `svymean()`, `svyratio()` and `svycontrast()` which I show below
using the same designe object from Chapter 2.

It is worth noting before doing so however, that the special handling needed
here follows from the fact that both the numerator and the denominator are
estimated. Lumley delves into this in the appendix which I'll reproduce here
alongside questions I have that I'll look to return to in the future.

#### A brief aside on ratio variance estimation

We'll define the subpopulation estimate of interest using the indicator function
$Y_i = X_iI(Z_I > 0)$, where $I(Z_i > 0) = 1$ for members of the subpopulation
and 0 otherwise, and $X_i$ is the measurement of interest.

The variance estimate using replicate weights can be calculated similar to the
typical variance estimate, those replicate weights belonging to sample
observations outside the subpopulation are simply not used - this again
highlights the utility of replicate weights.

For linearization - that is using a taylor series to estimate the variance - the
value becomes more complicated, following Lumley's appendix the HTE is defined
as:

$$
\hat{V}[\hat{T}_Y] = \sum_{i,j} \left(\frac{Y_i Y_j}{\pi_ij} - 
\frac{Y_i}{\pi_i} \frac{Y_j}{\pi_j} \right) \\
= \sum_{Z_i,Z_j > 0} \left(\frac{Y_i Y_j}{\pi_ij} - 
\frac{Y_i}{\pi_i} \frac{Y_j}{\pi_j} \right).
$$

Here however, Lumley states

> but the simplified computational formulas for special designs are not the
> same.

which I don't completely understand. I suppose he means for clustered or
multiphase designs things but it isn't clear as he goes onto say

> for example, the formula for the variance of a total under simple random
> sampling (equation 2.2)

$$
V[\hat{T}_X] = \frac{N-n}{N} \times N^2 \times \frac{V[X]}{n}
$$

> cannot be replaced by

$$
V[\hat{T}_Y] \stackrel{?}{=} \frac{N-n}{N} \times N^2 \times \frac{V[X]}{n}
$$

> or even, defining $n_D$ as the number sampled in the subpopulation, by

$$
\stackrel{?}{=} \frac{N-n_D}{N} \times N^2 \times \frac{V[X]}{n_D}
$$

> In order to use these simplified formulas it is necessary to work with the
> variable $Y$ and use

$$
V[\hat{T}_Y] = \frac{N-n}{N} \times N^2 \times \frac{V[Y]}{n}
$$

Its not clear in this last expression if we're simply back to the initial
expression that couldn't be used, or if we're using the smaller sample subset
again for variance computations but Lumley's next text suggests that's the case:

> Operationally, this means that variance estimation in a subset of a survey
> design object in R needs to involve the $n - n_D$ zero contributions to an
> estimation equation.

I hope to shed more light on what's going on here in the future but for now its
clear why this is in the appendix, but not exactly clear to me why observations
outside the subpopulation are *simply* zero'd in variance computation.

Lumley uses the following three function calls to illustrate three different
ways to estimate a ratio (1) A call to `svymean()`, (2) A call to `svyratio()`
and (3) \`


::: {.cell}

```{.r .cell-code}
svymean(~bmi_p, subset(chis, srsex == "MALE" & racehpr == "AFRICAN AMERICAN"))
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
bmi_p 28.019 0.2663
```
:::
:::

::: {.cell}

```{.r .cell-code}
chis <- update(chis,
  is_aamale = (srsex == "MALE" & racehpr == "AFRICAN AMERICAN")
)
svyratio(~ I(is_aamale * bmi_p), ~is_aamale, chis)
```

::: {.cell-output .cell-output-stdout}
```
Ratio estimator: svyratio.svyrep.design(~I(is_aamale * bmi_p), ~is_aamale, chis)
Ratios=
                     is_aamale
I(is_aamale * bmi_p)  28.01857
SEs=
         [,1]
[1,] 0.266306
```
:::
:::

::: {.cell}

```{.r .cell-code}
totals <- svytotal(~ I(bmi_p * is_aamale) + is_aamale, chis)
totals
```

::: {.cell-output .cell-output-stdout}
```
                        total       SE
I(bmi_p * is_aamale) 19348927 260401.7
is_aamaleFALSE       25697039   6432.3
is_aamaleTRUE          690575   6341.9
```
:::

```{.r .cell-code}
svycontrast(
  totals,
  quote(`I(bmi_p * is_aamale)` / `is_aamaleTRUE`)
)
```

::: {.cell-output .cell-output-stdout}
```
          nlcon     SE
contrast 28.019 0.2663
```
:::
:::


### Ratio estimators of totals

The third use Lumley lists for the use of ratio estimators is to construct more
accurate estimates of population means or totals. His motivating example is to
take the ratio estimate of individuals who took the API tests and then use that
to determine the approximate number of students who took the test, by
multiplying the ratio estimate by the number of students. This can be done by
hand or via the survey package `predict()` function used in conjunction with
`svyratio()`.


::: {.cell}

```{.r .cell-code}
r <- svyratio(~api.stu, ~enroll, strat_design)
predict(r, total = sum(apipop$enroll, na.rm = TRUE))
```

::: {.cell-output .cell-output-stdout}
```
$total
         enroll
api.stu 3190038

$se
          enroll
api.stu 29565.98
```
:::
:::


Lumley uses this as a jumping off point to discuss linear regression since 
one can imagine the relationship between the number of students taking the API
test as being roughly proportional to the number of students enrolled at the
schools; $E[tests_i] = \alpha \times \text{enrollment}_i + \epsilon_i$ where
$E[\epsilon] = 0$ (note that this is an assumption about the moment of the error
distribution and not the shape of the error distribution itself).

## Linear Regression

A quick review of the moment assumptions for linear regression - which is all
that are needed for designed based estimation. 

If we have random response variable $Y$ and explanatory variable $X$ then linear
regression is looking at the relationship between the expectation of $Y$ and 
$X$:
$$
E[Y] = \alpha + X \beta,
$$

Where $\alpha$ is a constant offset, also called the intercept and $\beta$ is
the slope coefficient that describes the change in $E[Y]$ per unit change in 
$X$. Here I'm referring to $X$ and $\beta$ as singular variables but they
could also be a matrix and vector of explanatory variables and slope 
coefficients, respectively. The variance of the response variable, $Y$ is 
assumed to be constant, i.e. $V[Y] = \sigma^2$, unless otherwise modeled.

Following the standard OLS estimation procedure, we'd normally 
minimize the squared error and Lumley notes that if we have the complete 
population data, we'd be finished at that:

$$
RSS = \sum_{i=1}^{N} (Y_i - \alpha X_i\beta)^{2}.
$$

Given that we're typically dealing with (complex) samples though, we have
to adjust the estimates to account for the weighting:


$$
\hat{RSS} = \sum_{i=1}^{n} \frac{1}{\pi_i}(Y_i - \alpha - X_i \beta)^2,
$$

so each error term is up weighted according to its sampling probability.
 

### Regression Estimation of Population Totals

Lumley's ratio estimator of a population total described previously derives 
from the linear regression model with a single predictor and no intercept. 
The *separate ratio estimator*, similar to the 
[cell means model](https://online.stat.psu.edu/stat502/lesson/4/4.3) 
estimates a ratio for each stratum and the estimate of the total is the 
sum of the denominator for all strata. Formally,

$$
E[Y_i] = \beta_k \times X_i \times \{i \in \text{ stratum } k\},
$$

where $\beta_k$ is the ratio for stratum $k$ estimates the given quantity. This
setup can provide more precise estimates than the single ratio estimator when
the sample size gets large and the strata are able to better explain the outcome
variable. However, if the strata don't have any correlation with the outcome
then the standard errors increase, due to the need to estimate the extra
parameters.


THIS EXAMPLE DOESN'T MAKE COMPLETE SENSE.
Lumley illustrates this latter phenomenon with the California school data - 
using the percentage of english language learners as a predictor of the 
overall number of students taking the api tests.



::: {.cell}

```{.r .cell-code}
sep <- svyratio(~api.stu, ~enroll, strat_design, separate = TRUE)
com <- svyratio(~api.stu, ~enroll, strat_design)
stratum_totals <- list(E = 1877350, H = 1013824, M = 920298)
predict(sep, total = stratum_totals)
```

::: {.cell-output .cell-output-stdout}
```
$total
         enroll
api.stu 3190022

$se
          enroll
api.stu 29756.44
```
:::
:::

::: {.cell}

```{.r .cell-code}
predict(com, total = sum(unlist(stratum_totals)))
```

::: {.cell-output .cell-output-stdout}
```
$total
         enroll
api.stu 3190038

$se
          enroll
api.stu 29565.98
```
:::
:::

We see the common ratio has a smaller standard error than the separate ratio
estimator.



::: {.cell}

```{.r .cell-code}
svyby(~api.stu, ~stype, design = dstrata, denom = ~enroll, svyratio)
```

::: {.cell-output .cell-output-stdout}
```
  stype api.stu/enroll se.api.stu/enroll
E     E      0.8558562       0.006034685
H     H      0.7543378       0.031470156
M     M      0.8331047       0.017694634
```
:::
:::


**Incomes in Scotland**

Lumley works through an example looking at household incomes across Scotland
from their national household survey. Unfortunately both the dataset subset he 
provides at his [website](https://r-survey.r-forge.r-project.org/svybook/) to 
and the full dataset he [links to](https://www.restore.ac.uk/PEAS/exemplar2.php)
don't have the variables he uses in his example code in Figure 5.7. For 
example the code he provides is filtered using an ADULTH variable which isn't 
found in either dataset.


::: {.cell}

```{.r .cell-code}
load("Data/SHS/shs.rda") # Lumley's website data
colnames(shs$variables)
```

::: {.cell-output .cell-output-stdout}
```
 [1] "psu"      "uniqid"   "ind_wt"   "shs_6cla" "council"  "rc5"     
 [7] "rc7e"     "rc7g"     "intuse"   "groupinc" "clust"    "stratum" 
[13] "age"      "sex"      "emp_sta"  "grosswt"  "groc"    
```
:::

```{.r .cell-code}
load("Data/SHS/ex2.RData") # PEAS "full Data" website
colnames(shs)
```

::: {.cell-output .cell-output-stdout}
```
 [1] "PSU"      "UNIQID"   "IND_WT"   "SHS_6CLA" "COUNCIL"  "RC5"     
 [7] "RC7E"     "RC7G"     "INTUSE"   "GROUPINC" "CLUST"    "STRATUM" 
[13] "AGE"      "SEX"      "EMP_STA"  "GROSSWT"  "GROC"    
```
:::
:::


Consequently, I won't reproduce this example here, except to say that he uses
the example to illustrate how oversampling certain strata (poorer households)
can improve the precision associated with the household income estimate and
that using the population information increased the precision of the weekly
household income estimate via linear regression -- for those sub-populations for
which population information is available.


**US Elections**

Similarly it doesn't look like the data set included in the current `survey` R
package has data for the 2008 election - I only see Bush / McCain vote totals.



::: {.cell}

```{.r .cell-code}
data(elections)
colnames(election)
```

::: {.cell-output .cell-output-stdout}
```
[1] "County"             "TotPrecincts"       "PrecinctsReporting"
[4] "Bush"               "Kerry"              "Nader"             
[7] "votes"              "p"                 
```
:::
:::


Consequently I can't do the analysis he shows predicting 2008 votes
using 2000 votes. It'll hopefully suffice to say that in theme with the
content for the chapter, that because these values are correlated ---
2000 vote % for republican candidate and 2008 vote % for republican candidate 
--- it stands to reason that we can reduce the variance of the resulting
estimate rather than using the 2008 data alone.


### Confouding and other criteria for model choice

Lumley describes three categories for describing why a predictor might be 
included in a regression, noting that this may help the model fit better and
thus aid in reducing bias from a probability sample that results from, say
non-response.

1. Exposure of interest: If we're interested in a specific variable's impact
on a variable, it makes sense to include that in a model to estimate the
relationship.

2. Confounding variables: A variable may not be of primary interest, but may
be associated with both the outcome variable and exposure of interest. 
Consequently, this will need to be *adjusted for* in order to isolate the 
effect of interest.

3. Precision variables: These are, again, associated with the outcome variable
of interest, but not associated with the exposure of interest. However, 
because of their association alone, they can increase the precision with
which the exposure effect is estimated.

Lumley goes on to describe methods for model selection which I'll leave for the
text.


### Linear models in the `survey` package


Example: Dietary sodium and potassium and blood pressure



::: {.cell}

```{.r .cell-code}
demo <- haven::read_xpt("data/nhanesxpt/demo_c.xpt")[, c(1:8, 28:31)]
bp <- haven::read_xpt("data/nhanesxpt/bpx_c.xpt")
bm <- haven::read_xpt("data/nhanesxpt/bmx_c.xpt")[, c("SEQN", "BMXBMI")]
diet <- haven::read_xpt("data/nhanesxpt/dr1tot_c.xpt")[, c(1:52, 63, 64)]
nhanes34 <- merge(demo, bp, by = "SEQN")
nhanes34 <- merge(nhanes34, bm, by = "SEQN")
nhanes34 <- merge(nhanes34, diet, by = "SEQN")

demo5 <- haven::read_xpt("data/nhanesxpt/demo_d.xpt")[, c(1:8, 39:42)]
bp5 <- haven::read_xpt("data/nhanesxpt/bpx_x.xpt")

bp5$BPXSAR <- rowMeans(bp5[, c("BPXSY1", "BPXSY2", "BPXSY3", "BPXSY4")],
  na.rm = TRUE
)
bp5$BPXDAR <- rowMeans(bp5[, c("BPXDI1", "BPXDI2", "BPXDI3", "BPXDI4")],
  na.rm = TRUE
)
bm5 <- haven::read_xpt("data/nhanesxpt/bmx_d.xpt")[, c("SEQN", "BMXBMI")]
diet5 <- haven::read_xpt("data/nhanesxpt/dr1tot_d.xpt")[, c(1:52, 64, 65)]


nhanes56 <- merge(demo5, bp5, by = "SEQN")
nhanes56 <- merge(nhanes56, bm5, by = "SEQN")
nhanes56 <- merge(nhanes56, diet5, by = "SEQN")

nhanes <- rbind(nhanes34, nhanes56)
nhanes$fouryearwt <- nhanes$WTDRD1 / 2
# I added the two lines below to make graphing the
# smooth plot easier
nhanes$sodium <- nhanes$DR1TSODI / 1000
nhanes$potassium <- nhanes$DR1TPOTA / 1000

des <- svydesign(
  id = ~SDMVPSU, strat = ~SDMVSTRA, weights = ~fouryearwt,
  nest = TRUE, data = subset(nhanes, !is.na(WTDRD1))
)

des <- update(des, sodium = DR1TSODI / 1000, potassium = DR1TPOTA / 1000)
des
```

::: {.cell-output .cell-output-stdout}
```
Stratified 1 - level Cluster Sampling design (with replacement)
With (60) clusters.
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000)
```
:::
:::


Lumley uses the following plot --- examining systolic blood pressure as a 
function of daily sodium intake --- to motivate the need to adjust for 
confounders. As we can see below in the reproduced Figure 5.10, there doesn't
appear to be much a relationship between sodium intake and average blood 
pressure. Lumley argues that we observe this simpler relationship because of 
the association between sodium and blood pressure is confounded by age.


::: {.cell}

```{.r .cell-code}
plot(BPXSAR ~ sodium, data = nhanes, type = "n")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/svysmoothplot-1.png){width=672}
:::

```{.r .cell-code}
points(svyplot(BPXSAR ~ sodium,
  design = des, style = "transparent", xlab =
    "Dietary Sodium (g/day)", ylab = "Systolic Blood Pressure (mm Hg)"
))
lines((svysmooth(BPXSAR ~ sodium, des)))
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/svysmoothplot-2.png){width=672}
:::
:::



To test this hypothesis, Lumley first visualizes the three variables using the
conditional plot demonstrated previously.



::: {.cell}

```{.r .cell-code}
svycoplot(BPXSAR ~ sodium | equal.count(RIDAGEYR), des,
  style = "hexbin",
  xlab = "Dietary Sodium (g/day)",
  ylab = "Systolic BP (mmHg)",
  strip = strip.custom(var.name = "Age")
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/nacl_coplot-1.png){width=672}
:::
:::


In the above plot we see a greater indication that as dietary sodium increases
, so too does systolic blood pressure. 

To more formally test the hypothesis, Lumley fits several models with
these variables included. The first two just include (1) sodium and potassium
and (2) sodium, potassium and Age.



::: {.cell}

```{.r .cell-code}
model0 <- svyglm(BPXSAR ~ sodium + potassium, design = des)
summary(model0)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = BPXSAR ~ sodium + potassium, design = des)

Survey design:
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000)

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept) 120.3899     0.7105 169.436  < 2e-16 ***
sodium       -0.6907     0.1658  -4.166 0.000268 ***
potassium     0.7750     0.2655   2.919 0.006853 ** 
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 382.6159)

Number of Fisher Scoring iterations: 2
```
:::
:::

::: {.cell}

```{.r .cell-code}
model1 <- svyglm(BPXSAR ~ sodium + potassium + RIDAGEYR, design = des)
summary(model1)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = BPXSAR ~ sodium + potassium + RIDAGEYR, design = des)

Survey design:
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000)

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept) 99.73535    0.79568 125.346  < 2e-16 ***
sodium       0.79846    0.14866   5.371 1.13e-05 ***
potassium   -0.91148    0.18994  -4.799 5.23e-05 ***
RIDAGEYR     0.49561    0.01169  42.404  < 2e-16 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 275.4651)

Number of Fisher Scoring iterations: 2
```
:::
:::


As we can see, the sodium and potassium coefficient signs in the first
model switch direction once age is included in the second model, demonstrating
that the two variables are associated with both age and systolic blood pressure.
The second model makes more sense intuitively, because we expect systolic 
blood pressure to increase, on average, as a function of sodium intake.

Lumley adds a few more possible confounders to `model2` and then tests 
to see whether the effects of daily dietary sodium and potassium on systolic 
blood pressure are significantly different than zero using the `regTermTest()` 
function. 


::: {.cell}

```{.r .cell-code}
model2 <- svyglm(BPXSAR ~ sodium + potassium + RIDAGEYR + RIAGENDR + BMXBMI,
  design = des
)
summary(model2)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = BPXSAR ~ sodium + potassium + RIDAGEYR + RIAGENDR + 
    BMXBMI, design = des)

Survey design:
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000)

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept) 97.32903    1.42668  68.220  < 2e-16 ***
sodium       0.43458    0.16164   2.689   0.0126 *  
potassium   -0.96119    0.17043  -5.640 7.19e-06 ***
RIDAGEYR     0.45791    0.01080  42.380  < 2e-16 ***
RIAGENDR    -3.38208    0.38403  -8.807 3.90e-09 ***
BMXBMI       0.38460    0.03797  10.129 2.48e-10 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 263.5461)

Number of Fisher Scoring iterations: 2
```
:::
:::

::: {.cell}

```{.r .cell-code}
regTermTest(model2, ~ potassium + sodium, df = NULL)
```

::: {.cell-output .cell-output-stdout}
```
Wald test for potassium sodium
 in svyglm(formula = BPXSAR ~ sodium + potassium + RIDAGEYR + RIAGENDR + 
    BMXBMI, design = des)
F =  15.98481  on  2  and  25  df: p= 3.3784e-05 
```
:::
:::


The test is formally examining whether a model with these terms is more 
likely, given the data, then one without, with the null hypothesis assuming
that the extra terms are unneccessary. As we can see, the model is 
very unlikely to fit so well with the two extra terms, so there's evidence to
support the association between the terms and blood pressure.


Lumley digs into further details of why the effect is so small --- 1 gram
of sodium (2.5 grams of salt) is a lot of salt required to increase systolic
blood presssure "only" .43 mmHg. Some explanations include measurement error,
missing data, and model misspecification. Lumley examines the last of these 
by displaying the model diagnostic plots. Model misspecification can be
examined by identifying any association between the partial residuals and
the observed sodium intake. I replicate Lumley's code below.


::: {.cell}

```{.r .cell-code}
par(mfrow = c(1, 2))
plot(as.vector(predict(model1)), resid(model1),
  xlab = "Fitted Values", ylab = "Residuals"
)
nonmissing <- des[-model1$na.action]
plot(nonmissing$variables$sodium,
  resid(model1, "partial")[, 1],
  xlab = "Sodium",
  ylab = "Partial Residuals"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/partial_residual_plot-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
nonmissing <- des[-model1$na.action]
par(mfrow = c(1, 2))
plot(model1, panel = make.panel.svysmooth(nonmissing))
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/nacl_termplot-1.png){width=672}
:::

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/nacl_termplot-2.png){width=672}
:::

```{.r .cell-code}
termplot(model1,
  data = model.frame(nonmissing),
  partial = TRUE, se = TRUE, smooth = make.panel.svysmooth(nonmissing)
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/nacl_termplot-3.png){width=672}
:::

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/nacl_termplot-4.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
int1 <- svyglm(BPXSAR ~ (sodium + potassium) * I(RIDAGEYR - 40) + RIAGENDR + BMXBMI,
  design = des
)
summary(int1)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = BPXSAR ~ (sodium + potassium) * I(RIDAGEYR - 
    40) + RIAGENDR + BMXBMI, design = des)

Survey design:
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000)

Coefficients:
                             Estimate Std. Error t value Pr(>|t|)    
(Intercept)                116.215191   1.240298  93.699  < 2e-16 ***
sodium                       0.309581   0.165746   1.868 0.074583 .  
potassium                   -0.975994   0.182598  -5.345 1.99e-05 ***
I(RIDAGEYR - 40)             0.605278   0.022047  27.455  < 2e-16 ***
RIAGENDR                    -3.516219   0.377502  -9.314 2.87e-09 ***
BMXBMI                       0.387957   0.038347  10.117 6.14e-10 ***
sodium:I(RIDAGEYR - 40)     -0.015707   0.008767  -1.792 0.086356 .  
potassium:I(RIDAGEYR - 40)  -0.039575   0.010121  -3.910 0.000703 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 261.658)

Number of Fisher Scoring iterations: 2
```
:::
:::

### Is Weighting Needed in a Regression Model ?

Lumley caps off this chapter asking whether we even need to bother with the
specialized weighting built into the `survey` package. His answer is worth
digging into:

> Since regression models use adjustment for confounders as a way of removing 
distorted associations between exposure and response, it is plausible that a
regression model might not need sampling weights.

A key assumption here is whether the population we're estimating is stable 
across the population(s) represented by our data set. The naive, biased 
population that our sample represents when unweighted, or the "target" 
population our sample represents when re-weighted. A follow-up question would
ask whether we could even estimate the relationship as desired, if the 
effect is heterogeneous across populations. Lumley cites [@dumouchel1983using]
for further discussion on this topic.

I have more to say here --- Thinking of this article [gelman2007struggles] ---, 
but it may not fit in these notes. For now I'll end with Lumley's two 
limitations for regression models when not using weights:

1. Some important variables used in constructing the weights may not be 
available,

2. Further, the important variables mentioned above may not be suitable for 
including in the model.

Lumley urges caution in this regard, advising that even a small amount of bias
introduced from not including the weights may make any potential increase in
precision that comes from *not* using the weights as poor trade-off.

## Exercises

1. This exercise uses the WA State crime data for 2004 as the population.
The data consists of crime rates and population sizes for the police districts
(in cities/towns) and sheriffs' offices (in unincorporated areas), grouped by
county.

* Take a simple random sample of ten counties from the state and use all the 
data from the sampled counties. Estimate the total number of murders and 
burglaries in the state.


::: {.cell}

```{.r .cell-code}
county_sample <- wa_crime_df %>%
  distinct(County) %>%
  slice_sample(n = 10) %>%
  pull(County)

wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design() %>%
  summarize(
    crime = survey_total(murder_and_crime)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  crime crime_se
  <dbl>    <dbl>
1  8644    2238.
```
:::
:::


* Use the population of each county as an auxiliary variable to estimate the
totals.


::: {.cell}

```{.r .cell-code}
ratio_estimate <- wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design() %>%
  svyratio(~murder_and_crime, ~Population, design = .)

predict(ratio_estimate, total = sum(wa_crime_df$Population))
```

::: {.cell-output .cell-output-stdout}
```
$total
                 Population
murder_and_crime   56678.22

$se
                 Population
murder_and_crime   5888.009
```
:::
:::


* Use the numbers of murders and burglaries in the previous year as auxiliary 
variables in a regression estimate of the totals (why can't we use a ratio
estimate here?)


::: {.cell}

```{.r .cell-code}
wa_crime_03_df <- readxl::read_xlsx("data/WA_crime/1984-2011.xlsx", skip = 4) %>%
  filter(Year == "2003", Population > 0) %>%
  mutate(
    murder_and_crime = `Murder Total` + `Burglary Total`,
    state_pop = sum(Population),
    County = stringr::str_to_lower(County),
    num_counties = n_distinct(County),
  ) %>%
  group_by(County) %>%
  mutate(num_agencies = n_distinct(Agency)) %>%
  ungroup() %>%
  select(
    County, Agency, Population, murder_and_crime,
    num_counties, num_agencies
  )

model_df <- wa_crime_03_df %>%
  mutate(year = "2003") %>%
  bind_rows(wa_crime_df %>% mutate(year = "2004")) %>%
  mutate(
    # We'll use 2004's numbers here for the fpc.
    num_counties = if_else(year == "2004", num_counties, 0),
    num_agencies = if_else(year == "2004", num_agencies, 0),
  ) %>%
  spread(year, murder_and_crime) %>%
  group_by(County, Agency) %>%
  summarize(
    num_counties = sum(num_counties),
    num_agencies = sum(num_agencies),
    `2003` = sum(replace_na(`2003`, 0)),
    `2004` = sum(replace_na(`2004`, 0))
  ) %>%
  ungroup() %>%
  filter(
    # Agencies were removed between 2003 and 2004.
    num_counties > 0, num_agencies > 0,
    County %in% county_sample
  )

model_design <- model_df %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties, num_agencies)
  )

fit <- svyglm(`2004` ~ `2003`, design = model_design)

total_matrix <- c(sum(wa_crime_03_df$murder_and_crime))
total_matrix <- as.data.frame(total_matrix)
names(total_matrix) <- "2003"

predict(fit, newdata = total_matrix)
```

::: {.cell-output .cell-output-stdout}
```
   link     SE
1 52874 1138.2
```
:::
:::


We can't use a ratio estimator here because we're not using the total population
of the year in question as the denominator, we're only looking at the total
number of murders and burglaries in the previous year and relating it to the
next, without measuring the total population, explicitly.

* Stratify the sampling so that King County is sampled with 100% probability 
 together with a simple random sample of five other counties. use population
 as an auxiliary variable to construct a common ratio estimate and a separate
 ratio estimate of the population totals.


::: {.cell}

```{.r .cell-code}
smaller_county_sample <- wa_crime_df %>%
  distinct(County) %>%
  filter(County != "king") %>%
  slice_sample(n = 5) %>%
  pull(County)

county_list <- unique(wa_crime_df$County)

design <- wa_crime_df %>%
  filter(County == "king" | County %in% smaller_county_sample) %>%
  mutate(
    strata_label = if_else(County == "king", "King County", "WA Counties"),
    num_counties = if_else(County == "king", 1, length(county_list) - 1)
  ) %>%
  as_survey_design(
    ids = c(County, Agency),
    fpc = c(num_counties, num_agencies),
    strata = strata_label
  )

strata_totals <- wa_crime_df %>%
  mutate(strata = if_else(County == "king", "King", "WA Counties")) %>%
  group_by(strata) %>%
  summarize(Population = sum(Population)) %>%
  spread(strata, Population) %>%
  as.matrix()

separate_estimator <- svyratio(~murder_and_crime, ~Population, design,
  separate = TRUE
)
common_estimator <- svyratio(~murder_and_crime, ~Population, design)
predict(separate_estimator, total = strata_totals)
```

::: {.cell-output .cell-output-stdout}
```
$total
                 Population
murder_and_crime    68697.7

$se
                 Population
murder_and_crime   2141.227
```
:::
:::

::: {.cell}

```{.r .cell-code}
predict(common_estimator, total = sum(wa_crime_df$Population))
```

::: {.cell-output .cell-output-stdout}
```
$total
                 Population
murder_and_crime   68915.53

$se
                 Population
murder_and_crime   3370.313
```
:::
:::

 
* Take simple random samples of five police districts from King County and 
five counties from the rest of the state. use population as an auxiliary
variable to construct a common ratio estimate and a separate ratio estimate of 
the population totals.


::: {.cell}

```{.r .cell-code}
king_districts <- wa_crime_df %>%
  filter(County == "king") %>%
  pull(Agency)
sampled_king_districts <- sample(king_districts, 5)
sampled_counties <- sample(county_list, 5)

design <- wa_crime_df %>%
  filter(County %in% sampled_counties | Agency %in% sampled_king_districts) %>%
  mutate(
    strata_label = if_else(County == "king", "King County", "WA Counties"),
    num_counties = if_else(County == "king", 1, length(county_list) - 1),
  ) %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties, num_agencies),
    strata = strata_label
  )


separate_estimator <- svyratio(~murder_and_crime, ~Population, design,
  separate = TRUE
)
common_estimator <- svyratio(~murder_and_crime, ~Population, design)
predict(separate_estimator, total = strata_totals)
```

::: {.cell-output .cell-output-stdout}
```
$total
                 Population
murder_and_crime   61185.56

$se
                 Population
murder_and_crime   5576.053
```
:::
:::

::: {.cell}

```{.r .cell-code}
predict(common_estimator, total = sum(wa_crime_df$Population))
```

::: {.cell-output .cell-output-stdout}
```
$total
                 Population
murder_and_crime   61727.29

$se
                 Population
murder_and_crime   8328.097
```
:::
:::


2. Using the WA state crime data as a population, take a stratified sample
of five police districts from King County and five counties from the rest of the
state. Estimate the ratio of violent crimes to non-violent crimes. Compare to
the population value.


::: {.cell}

```{.r .cell-code}
sampled_king_districts <- sample(king_districts, 5)
sampled_counties <- sample(county_list, 5)

wa_crime_df %>%
  filter(County %in% sampled_counties | Agency %in% sampled_king_districts) %>%
  mutate(
    strata_label = if_else(County == "king", "King County", "WA Counties"),
    num_counties = if_else(County == "king", 1, length(county_list) - 1),
  ) %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties, num_agencies),
    strata = strata_label
  ) %>%
  summarize(
    violent_non_violent = survey_ratio(violent_crime, property_crime)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  violent_non_violent violent_non_violent_se
                <dbl>                  <dbl>
1              0.0608                0.00749
```
:::
:::

::: {.cell}

```{.r .cell-code}
round(sum(wa_crime_df$violent_crime) / sum(wa_crime_df$property_crime), 2)
```

::: {.cell-output .cell-output-stdout}
```
[1] 0.07
```
:::
:::


We can see that the estimate is quite close to the population value

3. Using the data from Wave 1 of the 1996 SIPP panel (see Figure 3.8)

* Estimate the ratio of population totals for monthly rent (`tmthrnt`) and
total household income (`thtrninc`) over the whole population and over the
sub-population who pay rent.

I think Lumley may have an error here when he says that `thtrninc` is the 
total household monthly income - earlier we used `thtotinc` for this measure 
as he had in creating Figure 3.9. Consequently, I use `thtotinc` below.


::: {.cell}

```{.r .cell-code}
sipp_hh_sub %>%
  # Total population
  summarize(
    ratio_of_monthly_rent_to_household_income = survey_ratio(tmthrnt, thtotinc)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  ratio_of_monthly_rent_to_household_income ratio_of_monthly_rent_to_household…¹
                                      <dbl>                                <dbl>
1                                   0.00236                            0.0000800
# ℹ abbreviated name: ¹​ratio_of_monthly_rent_to_household_income_se
```
:::
:::

::: {.cell}

```{.r .cell-code}
sipp_hh_sub %>%
  filter(tmthrnt > 0) %>%
  # Rent paying subpopulation
  summarize(
    ratio_of_monthly_rent_to_household_income = survey_ratio(tmthrnt, thtotinc)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  ratio_of_monthly_rent_to_household_income ratio_of_monthly_rent_to_household…¹
                                      <dbl>                                <dbl>
1                                     0.209                              0.00506
# ℹ abbreviated name: ¹​ratio_of_monthly_rent_to_household_income_se
```
:::
:::

* Compute the individual-level ratio, i.e., the proportion of household income
paid in rent, and estimate the population mean over the whole population and 
over the sub-population who pay rent.


::: {.cell}

```{.r .cell-code}
# Full Population
sipp_hh_sub %>%
  mutate(
    # I also ran the numbers if we excluded those with 0 household rent
    # and the estimates are effectively the same.
    prop_income_rent = if_else(thtotinc == 0, 0, (tmthrnt / thtotinc)),
  ) %>%
  summarize(
    prop_income_rent_est = survey_mean(prop_income_rent)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  prop_income_rent_est prop_income_rent_est_se
                 <dbl>                   <dbl>
1               0.0221                 0.00624
```
:::
:::

::: {.cell}

```{.r .cell-code}
sipp_hh_sub %>%
  # Rent paying subpopulation
  filter(tmthrnt > 0) %>%
  mutate(
    # I also ran the numbers if we excluded those with 0 household rent
    # and the estimates are effectively the same.
    prop_income_rent = if_else(thtotinc == 0, 0, (tmthrnt / thtotinc)),
  ) %>%
  summarize(
    prop_income_rent_est = survey_mean(prop_income_rent)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
  prop_income_rent_est prop_income_rent_est_se
                 <dbl>                   <dbl>
1                0.548                   0.154
```
:::
:::


What are we to make of these estimates being different? Well that's because
they're estimating two different things. As Lumley points out at the start of
the chapter, one is a ratio of two population-level quantities, the other is 
the population estimate of a ratio measured at the individual level.

4. Use the stratified sample from the Academic Performance Index population
to examine whether the proportion of teachers with only emergency qualifications
(`emer`) affects academic performance (as measured by 2000 API).


* What confounding variables measuring socioeconomic status of students should be 
included in the model?

Going off the web 
[documentation](https://r-survey.r-forge.r-project.org/survey/html/api.html)
of the `api` dataset from the `survey` package, it looks like there are a 
number of possible confounding variables should be included in the model. 
Here is a list with a brief explanation:
  * `meals`: The % of students eligible for subsidized meals. This is a proxy
  for poverty and is likely correlated with the academic achievement measured
  by the test.
  
  * `hsg`: percent of parents who are high school graduates. Parental academic
  achievement is likely associated with their students' academic achievement.
  
  * `avg.ed`: Average parental education level - this might be able to combine
  the above variable along with those who have college or post-graduate 
  education.
  
  * `comp.imp`  refers to a school "growth" improvement targets that may be
  related to students' academic performance.
  
  * `acs.k3` average class size years K-3 - class size is often associated with
  academic performance There is a similar `acs.46` variable for grades 4-6.
  
  *  `ell` The percent of english language learners. Since most classes are 
  typically instructed in english, a non-native english speakers may struggle
  more with academic instruction.
  
  * `full` percent fully qualified teachers. A fully qualified teacher will
  presumably be more capable of teaching than one that's not fully qualified.
  
  * `enroll` Enrollment may also be associated with the API, if larger schools
  have access to greater resources, or inversely, worse teacher to student
  ratios.

* Should 1999 API be in the model (and why, or why not?)

  *  The value of including the 1999 API score would be that its very likely one
  of the most correlated variables with the 2000 score. The downside is that it
  is also likely correlated with all the other measures, including the `emer`
  measure, and may mask that variable's weaker impact. I'll leave it out in
  my estimate to try to avoid this problem.

* Do any of the confounding variables need to be transformed?

  * Several of the binary / categorical variables will need to be transformed 
  into 0 / 1 or cell means encoding. It could also be beneficial to
  center several of the continuous variables at the mean to offer an easier
  interpretation of the model.

* Does `emer` need to be transformed?

It doesn't look like it needs to be to me. I include two plots below that
visualize the `emer` distribution (in the target population) as well as its 
relationship with the `api00` measure. While there is a right skew in the
distribution, this doesn't strike me as problematic and I think a log 
transformation - an attempt to fix the skew - would not be helpful both because
of the difficulty in handling 0 values as well as the log-scale interpretation.

It could be worth centering the `emer` value by the estimated population mean 
to make the interpretation of the intercept more valueable but I don't think
that's necessary for an adequate model interpretation here.


::: {.cell}

```{.r .cell-code}
svyhist(~emer, strat_design)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/54_emer_eda1-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
svyplot(api00 ~ emer, strat_design,
  main = "CA 2000 API Scores vs. Teacher Emergency Training Preparedness",
  xlab = "Teachers with emergency training",
  ylab = "API 2000"
)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/54_emer_eda-1.png){width=672}
:::
:::


* What is the conclusion at the end?


::: {.cell}

```{.r .cell-code}
fit <- svyglm(
  api00 ~ emer + meals + hsg + avg.ed + comp.imp +
    acs.k3 + acs.46 + ell + full + enroll,
  design = strat_design
)
summary(fit)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = api00 ~ emer + meals + hsg + avg.ed + comp.imp + 
    acs.k3 + acs.46 + ell + full + enroll, design = strat_design)

Survey design:
svydesign(id = ~1, strata = ~stype, fpc = ~fpc, data = apistrat)

Coefficients:
              Estimate Std. Error t value Pr(>|t|)    
(Intercept) 652.826459 163.333076   3.997 0.000137 ***
emer         -0.726831   1.485803  -0.489 0.625986    
meals        -2.378649   0.483856  -4.916 4.32e-06 ***
hsg           0.424671   0.416138   1.021 0.310419    
avg.ed       48.006215  17.485877   2.745 0.007390 ** 
comp.impYes  15.075954  14.397161   1.047 0.298035    
acs.k3       -3.366084   4.984776  -0.675 0.501357    
acs.46        2.989147   1.389796   2.151 0.034367 *  
ell          -0.228954   0.407923  -0.561 0.576110    
full          0.006335   1.242588   0.005 0.995944    
enroll       -0.042131   0.035810  -1.176 0.242720    
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 3905.969)

Number of Fisher Scoring iterations: 2
```
:::
:::


From the output above, we see that the `emer` value isn't found to be 
significantly associated (at $\alpha = 0.05$) with the 2000 API measure after
adjusting for other variables. Indeed, `meals`, `avg.ed` and `acs.46` are the
only values for which there is evidence to support a relationship at this
level. 

5. Following on from the previous exercise, fit the same model to the whole
population (the data set `apipop`) using the `glm()` function.


::: {.cell}

```{.r .cell-code}
fit_pop <- glm(api00 ~ emer + meals + hsg + avg.ed + comp.imp +
  acs.k3 + acs.46 + ell + full + enroll, data = apipop)
summary(fit_pop)
```

::: {.cell-output .cell-output-stdout}
```

Call:
glm(formula = api00 ~ emer + meals + hsg + avg.ed + comp.imp + 
    acs.k3 + acs.46 + ell + full + enroll, data = apipop)

Coefficients:
              Estimate Std. Error t value Pr(>|t|)    
(Intercept) 460.895349  21.325334  21.613  < 2e-16 ***
emer          0.542863   0.171137   3.172  0.00152 ** 
meals        -2.180465   0.058205 -37.462  < 2e-16 ***
hsg           0.209642   0.078151   2.683  0.00734 ** 
avg.ed       50.288082   2.345033  21.445  < 2e-16 ***
comp.impYes  30.659677   2.103305  14.577  < 2e-16 ***
acs.k3        0.816373   0.556165   1.468  0.14222    
acs.46        0.863564   0.268385   3.218  0.00130 ** 
ell          -0.488302   0.064119  -7.616 3.25e-14 ***
full          1.560638   0.152078  10.262  < 2e-16 ***
enroll       -0.035925   0.004889  -7.348 2.43e-13 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 2748.517)

    Null deviance: 70739140  on 4070  degrees of freedom
Residual deviance: 11158979  on 4060  degrees of freedom
  (2123 observations deleted due to missingness)
AIC: 43803

Number of Fisher Scoring iterations: 2
```
:::
:::


* Do the sample estimates agree with the population data? Do your decisions
about transforming variables hold up in the population data?

*Some* of the sample estimates agree with the population data. It is very clear
that there is a whole lot more power to detect associations when the entire 
population is present. In brief, we see that the three variables for which
an association was detected previously --- `meals`, `acs.46` and `avg.edu` ---
all have similar (within the sample based estimate margin of error) estimates
on the full population data. Additionally, many other variables now 
have significant associations that did not previously. Notably the `emer` 
variable has a positive association with the `api00` such that we'd expect to
see a .5 gain in API score for every additional percent gain in teachers that
are emergency qualified.

* Fit the same model to 100 stratified samples from the population. Is the 
sampling distribution of the coefficients close to a Normal distribution?



::: {.cell}

```{.r .cell-code}
OneSimulation <- function() {
  coefs <- apipop %>%
    group_by(stype) %>%
    slice_sample(n = 66) %>%
    ungroup() %>%
    as_survey_design(
      strata = stype
    ) %>%
    svyglm(api00 ~ emer + meals + hsg + avg.ed + comp.imp +
      acs.k3 + acs.46 + ell + full + enroll, design = .) %>%
    coef(.)
  return(coefs)
}
coef_dist <- replicate(100, OneSimulation())
par(mfrow = c(1, 2))
hist(coef_dist[1, ], main = "Histogram of Intercept")
hist(coef_dist[2, ], main = "Histogram of emer")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/55_c-1.png){width=672}
:::
:::


Yes, as we'd expect the sampling distributions are roughly normal.


6. Using the blood pressure data from NHANES 2003 - 2006, investigate the 
effect of obesity on blood pressure using the Body Mass Index and blood pressure
data.

* What variables in the data set are potential confounders?

Given the discussion in the book's example, sodium and potassium intake are 
likely confounders alongside the usual, age, sex race and socioeconomic status.
That said, i don't know if the last of these two are available given that the
variable names in the dataset are not particularly descriptive.

* Are there important confounders that are not measured?

See above - race and socioeconomic status stand out as two confounding variables
that don't appear to be measured in this dataset.

* Fit one or more suitable regression models and summarize the output.

I'll fit the same model as `model2` in the text since Lumley explains what the 
variables are in that model.


::: {.cell}

```{.r .cell-code}
fit <- svyglm(BPXSAR ~ sodium + potassium + RIDAGEYR + RIAGENDR + BMXBMI,
  design = des
)
summary(fit)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = BPXSAR ~ sodium + potassium + RIDAGEYR + RIAGENDR + 
    BMXBMI, design = des)

Survey design:
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000)

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept) 97.32903    1.42668  68.220  < 2e-16 ***
sodium       0.43458    0.16164   2.689   0.0126 *  
potassium   -0.96119    0.17043  -5.640 7.19e-06 ***
RIDAGEYR     0.45791    0.01080  42.380  < 2e-16 ***
RIAGENDR    -3.38208    0.38403  -8.807 3.90e-09 ***
BMXBMI       0.38460    0.03797  10.129 2.48e-10 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 263.5461)

Number of Fisher Scoring iterations: 2
```
:::
:::


According to this model there is a .38 mm Hg expected increase in systolic
blood pressure for every one unit increase in BMI after adjusting for age, sex,
and daily sodium and potassium intake. In other words we'd expect a higher blood
pressure amongst those with higher BMIs. 

* Examine whether there is an interaction with age or sex.


::: {.cell}

```{.r .cell-code}
fit <- svyglm(BPXSAR ~ (RIDAGEYR + RIAGENDR) * I(BMXBMI - 25) + sodium + potassium,
  design = des
)
summary(fit)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = BPXSAR ~ (RIDAGEYR + RIAGENDR) * I(BMXBMI - 
    25) + sodium + potassium, design = des)

Survey design:
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000)

Coefficients:
                          Estimate Std. Error t value Pr(>|t|)    
(Intercept)             107.178639   1.200310  89.292  < 2e-16 ***
RIDAGEYR                  0.462853   0.011268  41.076  < 2e-16 ***
RIAGENDR                 -3.485444   0.353566  -9.858 1.00e-09 ***
I(BMXBMI - 25)            0.510054   0.104374   4.887 6.18e-05 ***
sodium                    0.438810   0.163223   2.688 0.013119 *  
potassium                -0.989246   0.169362  -5.841 5.95e-06 ***
RIDAGEYR:I(BMXBMI - 25)  -0.005031   0.001305  -3.855 0.000806 ***
RIAGENDR:I(BMXBMI - 25)   0.041993   0.059821   0.702 0.489736    
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 263.1093)

Number of Fisher Scoring iterations: 2
```
:::
:::

I center the BMI variable at 25 (roughly the marginal average) and fit the 
model with the age and sex interactions with the centered BMI. The model fit 
shows that there's a significant negative association with the BMI-age 
interaction and no association with the gender/sex - age interaction. This 
corresponds to a amplifying effect of age --- the older you are the lower your 
blood pressure is, on average, a higher BMI will then also lead to a lower 
expected blood pressure in addition to this affect.

7. Prove that an unweighted regression estimator is approximately unbiased when
the weights depend only on variables in the model. Specifically, if the true
population regression coefficients $\beta^*$ satisfy:

$$
\sum_{i=1}^{N} x_i(y_i - x_i\beta^*) = 0
$$

and $R_i$ indicates that observation $i$ is in the sample prove that

$$
E \left [ \sum_{i=1}^{N} R_ix_i (y_i - x_i \beta^*)\right ] = 0
$$

so that the un-weighted sample estimating equations are unbiased.


If we take the expectation over $R_i$ in the last formula we get
$$
 \sum_{i=1}^{N} \pi_ix_i (y_i - x_i \beta^*)  = 0,
$$

where $\pi_i = E[R_i]$, the probability of being included in the sample. At this
point it isn't immediately clear to me how to proceed. One easy result is that
if $\pi_i = \pi \forall i$, then we have 
$$
\pi \sum_i^N x_i(y_i - x_i\beta^*) = 0 \\
\iff 
\sum_i^N x_i(y_i - x_i\beta^*) = 0
$$

which proves the desired claim.

5.8 A rough approximation to the loss of efficiency from unnecessarily using
weights can be constructed by considering the variance of the residuals in 
weighted and un-weighted estimation. Assume as an approximation that the 
residuals $r_i$ are independent of the sampling weights $w_i$

* Show that 
$$ 
V[\sum_{i=1}^{n} w_i r_i] = E[w^2] V[r] + V[w]E[r^2]
$$


I show the proof below, though I think Lumley forgot a factor of $n$. This 
doesn't change anything substantial.

$$
V[\sum_{i=1}^{n} w_i r_i] \stackrel{ind}{=} \sum_{i=1}^{n} V[w_ir_i] \\
\stackrel{id}{=} n (E[(wr)^2] -  E[wr]^2) \\
\stackrel{ind}{=} n (E[w^2r^2] -  E[w]^2E[r]^2) \\
\stackrel{ind}{=} n (E[w^2]E[r^2] -  E[w]^2E[r]^2) \\
= n(E[w^2](E[r^2] - E[r]^2)  + (E[w^2]E[r]^2 - E[w]^2E[r]^2)) \\
= n(E[w^2V[r] + V[w]E[r^2]) \\
\blacksquare
$$

* Now assume that the mean of the residuals is zero, and show that the 
relative efficiency of the un-weighted estimate is $1 + cv(w)$ where $cv$ is the
coefficient of variation, the ratio of the standard deviation to the mean.

First note that $E[r] = 0 \implies V[r] = E[r^2]$. Then the expression from
the first part --- now omitting $n$ --- reduces to:

$$
E[w^2]E[r^2] + V[w]E[r^2] = E[r^2](E[w^2] + V[w])
$$
If we take the ratio of this to the variance of the un-weighted residuals we'd 
get 

$$
\frac{V[\sum_{i=1}^{n}r_iw_i]}{V[\sum_{i=1}^{n} r_i]} = \frac{E[r^2](E[w^2] + V[w])}{E[r^2] - E[r]^2} \\
= (E[w^2] + V[w]) \\
= 1 + \frac{V[w]}{E[w^2]}
$$

At this point I'd like to say that you can take the square root of the term on
the right, but that's not quite accurate and doesn't make sense algebraically.
To get to the mean we'd need to have $E[w]^2$ in the denominator which we don't
quite have.

# Chapter 6: Categorical Data Regression

Chapter 6 covers regression models for discrete outcome data --- binary or 
categorical data. The `survey` package also handles poisson and
binomial regression though neither of these are covered in this chapter.


## Logistic Regression

Lumley gives a brief overview of logistic regression which is roughly equivalent
to what can be found on 
[wikipedia](https://en.wikipedia.org/wiki/Logistic_regression), so I won't
reiterate his points here. The main take home is that while the interpretation
of the coefficients change from a linear association to an odds ratio 
association the only change to `svyglm()` is adding the 
`family = quasibinomial()` option.


### Example: Internet use in Scotland

To demonstrate how to analyze binary data, Lumley uses the Scottish Household
Survey data, examining internet use amongst the nation's populace.


::: {.cell}

```{.r .cell-code}
load("Data/SHS/shs.rda") # Lumley's website data
par(mfrow = c(2, 1))
bys <- svyby(~intuse, ~ age + sex, design = shs, svymean)
plot(
  svysmooth(intuse ~ age,
    design = subset(shs, sex == "male" & !is.na(age)),
    bandwidth = 5
  ),
  ylim = c(0, 0.8), ylab = "% Using Internet",
  xlab = "Age"
)
lines(svysmooth(intuse ~ age,
  design = subset(shs, sex == "female" & !is.na(age)),
  bandwidth = 5
), lwd = 2, lty = 3)
points(bys$age, bys$intuse, pch = ifelse(bys$sex == "male", 19, 1))
legend("topright",
  pch = c(19, 1), lty = c(1, 3), lwd = c(1, 2),
  legend = c("Male", "Female"), bty = "n"
)
byinc <- svyby(~intuse, ~ sex + groupinc, design = shs, svymean)
barplot(byinc, xlab = "Income", ylab = "% Using Internet")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/6_scotland_example_eda-1.png){width=1152}
:::
:::


Since binary data can't be easily visualized across a continuous variable
very easily, the plots above use both smoothing and binning --- computing
the proportion point estimate within some range of age --- to understand
how internet use changes across these continuous dimensions.

We can see in the plot that internet use is lower amongst the older respondents
in the survey and men ten to use the internet more then women, except perhaps 
the very youngest. My phrasing here is intentional, to demonstrate that
the phenomenon we're observing is likely a *cohort effect* since, as Lumley 
notes, its likely that born before the arrival of the internet are less likely
to use it.

We see this same pattern in the box plot showing internet use across income;
men using internet more than women. We also see that those with higher incomes 
tend to use the internet more than those with lower incomes. In contrast
to the cohort effect seen above, Lumley suggests that income may be more of
a real effect, and those who start earning more may be more likely to use the
internet.

Lumley then fits a series of models to quantify the relationships we're seeing 
in the plots formally. I'll summarize these briefly here and fit them below.

* Model 1 estimates the log odds of internet as a linear 
(on the log odds scale) function of age, sex and their interaction.

* Model 2 is the same as Model 1 except with two slopes for those younger and
older than age 35, respectively --- a low dimensional way to account for the 
nonlinear shape we observed previously.

* Model 3 is the same as Model 2 but adds an additional fixed effect term for 
income.

* Model 4 now adds an interaction between income and sex to account for the 
differences.
  
Lumley examines the output of all the models and for models 2 and 3 examines
what the two age-slopes for the reference group (women) is by using 
`svycontrast()`. Similarly with model 4, Lumley tests whether the 5 additional
parameters  added as the result of the income / sex interaction  lead to better
model fit. As we can see below, they do. 




::: {.cell}

```{.r .cell-code}
m <- svyglm(intuse ~ I(age - 18) * sex, design = shs, family = quasibinomial())

m2 <- svyglm(intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex,
  design = shs,
  family = quasibinomial()
)
summary(m)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = intuse ~ I(age - 18) * sex, design = shs, family = quasibinomial())

Survey design:
svydesign(id = ~psu, strata = ~stratum, weight = ~grosswt, data = ex2)

Coefficients:
                       Estimate Std. Error t value Pr(>|t|)    
(Intercept)            0.804113   0.047571  16.903  < 2e-16 ***
I(age - 18)           -0.044970   0.001382 -32.551  < 2e-16 ***
sexfemale             -0.116442   0.061748  -1.886   0.0594 .  
I(age - 18):sexfemale -0.010145   0.001864  -5.444 5.33e-08 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for quasibinomial family taken to be 0.950831)

Number of Fisher Scoring iterations: 4
```
:::
:::

::: {.cell}

```{.r .cell-code}
summary(m2)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex, 
    design = shs, family = quasibinomial())

Survey design:
svydesign(id = ~psu, strata = ~stratum, weight = ~grosswt, data = ex2)

Coefficients:
                         Estimate Std. Error t value Pr(>|t|)    
(Intercept)              2.152291   0.156772  13.729  < 2e-16 ***
pmin(age, 35)            0.014055   0.005456   2.576 0.010003 *  
pmax(age, 35)           -0.063366   0.001925 -32.922  < 2e-16 ***
sexfemale                0.606718   0.211516   2.868 0.004133 ** 
pmin(age, 35):sexfemale -0.017155   0.007294  -2.352 0.018691 *  
pmax(age, 35):sexfemale -0.009804   0.002587  -3.790 0.000151 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for quasibinomial family taken to be 0.9524217)

Number of Fisher Scoring iterations: 5
```
:::
:::

::: {.cell}

```{.r .cell-code}
svycontrast(m2, quote(`pmin(age, 35)` + `pmin(age, 35):sexfemale`))
```

::: {.cell-output .cell-output-stdout}
```
           nlcon     SE
contrast -0.0031 0.0049
```
:::
:::

::: {.cell}

```{.r .cell-code}
svycontrast(m2, quote(`pmax(age, 35)` + `pmax(age, 35):sexfemale`))
```

::: {.cell-output .cell-output-stdout}
```
            nlcon     SE
contrast -0.07317 0.0018
```
:::
:::

::: {.cell}

```{.r .cell-code}
shs <- update(shs, income = relevel(groupinc, ref = "under 10K"))
m3 <- svyglm(intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + income,
  design = shs, family = quasibinomial()
)
summary(m3)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + 
    income, design = shs, family = quasibinomial())

Survey design:
update(shs, income = relevel(groupinc, ref = "under 10K"))

Coefficients:
                         Estimate Std. Error t value Pr(>|t|)    
(Intercept)              1.275691   0.179902   7.091 1.41e-12 ***
pmin(age, 35)           -0.009041   0.006170  -1.465  0.14286    
pmax(age, 35)           -0.049408   0.002124 -23.259  < 2e-16 ***
sexfemale                0.758883   0.235975   3.216  0.00130 ** 
incomemissing            0.610892   0.117721   5.189 2.15e-07 ***
income10-20K             0.533093   0.048473  10.998  < 2e-16 ***
income20-30k             1.246396   0.052711  23.646  < 2e-16 ***
income30-50k             2.197628   0.063644  34.530  < 2e-16 ***
income50K+               2.797022   0.132077  21.177  < 2e-16 ***
pmin(age, 35):sexfemale -0.023225   0.008137  -2.854  0.00432 ** 
pmax(age, 35):sexfemale -0.008103   0.002858  -2.835  0.00459 ** 
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for quasibinomial family taken to be 0.9574657)

Number of Fisher Scoring iterations: 5
```
:::
:::

::: {.cell}

```{.r .cell-code}
m4 <- svyglm(intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + income * sex,
  design = shs, family = quasibinomial()
)
regTermTest(m4, ~ income:sex)
```

::: {.cell-output .cell-output-stdout}
```
Wald test for income:sex
 in svyglm(formula = intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + 
    income * sex, design = shs, family = quasibinomial())
F =  1.872811  on  5  and  11641  df: p= 0.095485 
```
:::
:::



Double check Lumley's assertions made on linear vs. logistic regression here.

### Relative Risk Regression

Logistic regression gets its name from the "link" function that determines 
the scale on which outcome mean is modeled and estimated. For a logistic
function the link is the "logit" function, which models the log odds. Other
link functions are also possible. In particular, using the `log()` link function
models the relative risk i.e. $\log(P(Y=1)) = X\beta$. 

Lumley goes into the details of how to fit these models via `svyglm` and the
potential difficulties of estimating the relative risk between the two
distributional families. Of note, because the binomial family is more 
restrictive, estimation is more sensitive to the fitting algorithm's
parameters starting values and 


::: {.cell}

```{.r .cell-code}
rr3 <- svyglm(intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + income,
  design = shs, family = quasibinomial(log),
  start = c(-0.5, rep(0, 10))
)
rr4 <- svyglm(intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + income,
  design = shs, family = quasipoisson(log)
)
summary(rr3)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + 
    income, design = shs, family = quasibinomial(log), start = c(-0.5, 
    rep(0, 10)))

Survey design:
update(shs, income = relevel(groupinc, ref = "under 10K"))

Coefficients:
                         Estimate Std. Error t value Pr(>|t|)    
(Intercept)             -0.455448   0.092857  -4.905 9.48e-07 ***
pmin(age, 35)            0.002853   0.003017   0.946    0.344    
pmax(age, 35)           -0.026658   0.001260 -21.149  < 2e-16 ***
sexfemale                0.623713   0.112658   5.536 3.16e-08 ***
incomemissing            0.489194   0.079296   6.169 7.09e-10 ***
income10-20K             0.416094   0.036710  11.335  < 2e-16 ***
income20-30k             0.820195   0.036838  22.265  < 2e-16 ***
income30-50k             1.124631   0.037188  30.241  < 2e-16 ***
income50K+               1.247014   0.044928  27.756  < 2e-16 ***
pmin(age, 35):sexfemale -0.007584   0.003801  -1.995    0.046 *  
pmax(age, 35):sexfemale -0.012816   0.001721  -7.446 1.03e-13 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for quasibinomial family taken to be 0.9256337)

Number of Fisher Scoring iterations: 16
```
:::
:::

::: {.cell}

```{.r .cell-code}
summary(rr4)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = intuse ~ (pmin(age, 35) + pmax(age, 35)) * sex + 
    income, design = shs, family = quasipoisson(log))

Survey design:
update(shs, income = relevel(groupinc, ref = "under 10K"))

Coefficients:
                          Estimate Std. Error t value Pr(>|t|)    
(Intercept)             -0.1679519  0.0846956  -1.983  0.04739 *  
pmin(age, 35)           -0.0001441  0.0024776  -0.058  0.95363    
pmax(age, 35)           -0.0312057  0.0012654 -24.660  < 2e-16 ***
sexfemale                0.5946038  0.1036091   5.739 9.77e-09 ***
incomemissing            0.4673470  0.0791934   5.901 3.71e-09 ***
income10-20K             0.4152093  0.0367406  11.301  < 2e-16 ***
income20-30k             0.8348862  0.0369330  22.605  < 2e-16 ***
income30-50k             1.2039269  0.0368608  32.661  < 2e-16 ***
income50K+               1.3601811  0.0429853  31.643  < 2e-16 ***
pmin(age, 35):sexfemale -0.0087552  0.0033861  -2.586  0.00973 ** 
pmax(age, 35):sexfemale -0.0116652  0.0017556  -6.645 3.18e-11 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for quasipoisson family taken to be 0.5969912)

Number of Fisher Scoring iterations: 5
```
:::
:::


### Ordinal Regression

The logit function introduced in the first section can be extended for use
from binomial outcome data to categorical -- typically ordinal, or ordered
categorical data. Unfortunately, the data that Lumley uses for this section is
not available so I'll skip any notes on this, noting that the model 
interpretation for logit ordinal regression is the same as for a typical
iid logit ordinal model.


::: {.cell}

```{.r .cell-code}
## Data isn't available on lumley's website - or at least the data that is
# there associated with "nhanes3" does not contain the variables below.
dhanes <- svydesign(
  id = ~SDPPSU6, strat = ~SDPSTRA6,
  weight = ~WTPFHX6,
  ## nest = TRUE indicates the PSU identifier is nested
  ## within stratum - repeated across strata
  nest = TRUE,
  data = subset(nhanes3, !is.na(WTPFHX6))
)
dhanes <- update(dhanes, fpg = ifelse(phpfast >= 8 & gip < 8E3, gip, NA))
dhanes <- update(dhanes,
  diab = cut(fpg, c(0, 110, 125, Inf)),
  diabi = cut(fpg, c(0, 100, 125, Inf))
)
dhanes <- update(dhanes,
  cadmium = ifelse(upd < 88880, udp, NA),
  creatinine = ifelse(urp < 88880, urp, NA)
)
dhanes <- update(dhanes,
  age = ifelse(hsaitmor > 1000, NA, hsaitmor / 12)
)

model0 <- svyolr(diab ~ cadmium + creatine, design = dhanes)
```
:::


### Other cumulative Link models

In this subsection Lumley discusses the log-log link function. The log log,
probit, and `cauchit` or inverse Cauchy link function are all supported by the
`svyolr` function  and are collectively referred to as `cumulative link models`.

## Loglinear Models

Lumley introduces log linear models in the context of the chi-square tests 
(via `svychisq()`) introduced previously testing association between count 
variables. These functions test a similar hypothesis using different methods and
I'd suggest looking at the function documentation closely to determine if
choosing between the two would matter in a particular circumstance.


::: {.cell}

```{.r .cell-code}
droplevels <- function(f) as.factor(as.character(f))
chis <- update(chis, smoking = droplevels(smoking), ins = droplevels(ins))
null <- svyloglin(~ smoking + ins, chis)
# Dot below includes all previous variables
saturated <- update(null, ~ . + smoking:ins)
anova(null, saturated)
```

::: {.cell-output .cell-output-stdout}
```
Analysis of Deviance Table
 Model 1: y ~ smoking + ins
Model 2: y ~ smoking + ins + smoking:ins 
Deviance= 659.1779 p= 1.850244e-82 
Score= 694.3208 p= 9.032412e-168 
```
:::
:::


The anova output above shows two p-values  according to two test statistics, 
one the deviance and the other the score. Again, the difference here is quite
"in the weeds" of how test statistic distributions are computed so I'd suggest
consulting a general reference on generalized linear models like 
[@dobson2018introduction] and/or the function documentation.


::: {.cell}

```{.r .cell-code}
svychisq(~ smoking + ins, chis)
```

::: {.cell-output .cell-output-stdout}
```

	Pearson's X^2: Rao & Scott adjustment

data:  svychisq(~smoking + ins, chis)
F = 130.11, ndf = 1.9923, ddf = 157.3884, p-value < 2.2e-16
```
:::
:::

::: {.cell}

```{.r .cell-code}
pf(130.1143, 1.992, 157.338, lower.tail = FALSE)
```

::: {.cell-output .cell-output-stdout}
```
[1] 5.37419e-34
```
:::
:::

::: {.cell}

```{.r .cell-code}
svychisq(~ smoking + ins, chis, statistic = "Chisq")
```

::: {.cell-output .cell-output-stdout}
```

	Pearson's X^2: Rao & Scott adjustment

data:  svychisq(~smoking + ins, chis, statistic = "Chisq")
X-squared = 694.32, df = 2, p-value < 2.2e-16
```
:::
:::

::: {.cell}

```{.r .cell-code}
summary(null)
```

::: {.cell-output .cell-output-stdout}
```
Loglinear model: svyloglin(~smoking + ins, chis)
               coef         se            p
smoking1 -0.6209497 0.01251103 0.000000e+00
smoking2 -0.1391308 0.01066059 6.275861e-39
ins1      0.8246480 0.01015826 0.000000e+00
```
:::
:::

::: {.cell}

```{.r .cell-code}
summary(saturated)
```

::: {.cell-output .cell-output-stdout}
```
Loglinear model: update(null, ~. + smoking:ins)
                    coef         se             p
smoking1      -0.4440871 0.01612925 7.065991e-167
smoking2      -0.3086129 0.01735214  9.187439e-71
ins1           0.8052182 0.01151648  0.000000e+00
smoking1:ins1 -0.2821710 0.01847008  1.085166e-52
smoking2:ins1  0.2510967 0.01875679  7.205759e-41
```
:::
:::

::: {.cell}

```{.r .cell-code}
model.matrix(saturated)
```

::: {.cell-output .cell-output-stdout}
```
  (Intercept) smoking1 smoking2 ins1 smoking1:ins1 smoking2:ins1
1           1        1        0    1             1             0
2           1        0        1    1             0             1
3           1       -1       -1    1            -1            -1
4           1        1        0   -1            -1             0
5           1        0        1   -1             0            -1
6           1       -1       -1   -1             1             1
attr(,"assign")
[1] 0 1 1 2 3 3
attr(,"contrasts")
attr(,"contrasts")$smoking
[1] "contr.sum"

attr(,"contrasts")$ins
[1] "contr.sum"
```
:::
:::


### Choosing Models

Lumley explores model selection beyond just statistical testing by discussing 
graphical and hierarchical loglinear models. My sense is that the discussion
here is not complete and I'd encourage any reader here to consult the 
references he points to for further reading on the topic.

**Example: neck and back pain in NHIS**

I couldn't find the data associated with Lumley's example on his website
or from a cursory Google search for NHIS data. Furthermore, I found it 
difficult to parse some of Lumley's words here. For example, for those reading 
the book, the two graphical models listed in Figure 6.11 look identical to me 
when the text states that the graphics are supposed to correspond to two 
different models.

### Linear Association Models

The log linear models can be used to test linear association of ordered 
categorical variables by coding the variables with ordered numbers and then
running the loglinear tests discussed previously. Lumley demonstrates with the
Scottish Household Survey data.

Lumley first starts by looking at internet use, now an ordinal categorical 
variable, `rc5` across income levels as before. Again, the similarity to
the chi-square model is obvious, as we're looking at models with fixed effects
only --- a "null" model --- and comparing it to a "saturated" model --- one with
interaction effects. The `anova()` call below, tests whether the additional
parameters estimated by the interaction model provide a greater model fit than
the model with the fixed effects alone. The idea being that if it does,
then there's evidence to support the hypothesis that internet use varies across
income levels, and since we're measuring internet use on an ordinal scale we
have evidence that more income corresponds to more internet use. Note that
below, only the models fit with the `as.numeric()` pieces are testing linear
association on the ordinal scale, while the others are a simple categorical
test.


::: {.cell}

```{.r .cell-code}
shs <- update(shs, income = ifelse(groupinc == "missing", NA, groupinc))
null <- svyloglin(~ rc5 + income, shs)
saturated <- update(null, ~ .^2)
anova(null, saturated)
```

::: {.cell-output .cell-output-stdout}
```
Analysis of Deviance Table
 Model 1: y ~ rc5 + income
Model 2: y ~ rc5 + income + rc5:income 
Deviance= 289.2165 p= 6.429997e-26 
Score= 282.5453 p= 0 
```
:::
:::


For the first model fit, we have a low p-value, so we reject the null model 
which make sense substantively. We've already seen that income is associated
with internet use in the previous analysis.



::: {.cell}

```{.r .cell-code}
lin <- update(null, ~ . + as.numeric(rc5):as.numeric(income))
anova(null, lin)
```

::: {.cell-output .cell-output-stdout}
```
Analysis of Deviance Table
 Model 1: y ~ rc5 + income
Model 2: y ~ rc5 + income + as.numeric(rc5):as.numeric(income) 
Deviance= 105.3609 p= 1.305691e-21 
Score= 105.6157 p= 1.152954e-21 
```
:::
:::

Now the model is updated with the ordinal categorical variable of internet
use and income, again we have a significant result.



::: {.cell}

```{.r .cell-code}
anova(lin, saturated)
```

::: {.cell-output .cell-output-stdout}
```
Analysis of Deviance Table
 Model 1: y ~ rc5 + income + as.numeric(rc5):as.numeric(income)
Model 2: y ~ rc5 + income + rc5:income 
Deviance= 183.8556 p= 4.346664e-14 
Score= 184.0606 p= 5.465958e-182 
```
:::
:::

Now we compare the ordinal model to the categorical, non-ordered model. 
It would make sense that the ordinal model provides a better fit than the 
categorical, and although that's what we see in the book it isn't what we
see when I run the code, even with the same Deviance and Score test 
statistics... I suspect this is a bug. The survey R package isn't hosted on
github but I'll try to raise an issue with Lumley via email.



::: {.cell}

```{.r .cell-code}
shs <- update(shs, agegp = cut(age, c(20, 40, 60, 100)))
null <- svyloglin(~ agegp + rc5, shs)
sat <- update(null, ~ .^2)
lin <- update(null, ~ . + as.numeric(rc5):as.numeric(agegp))
anova(null, lin)
```

::: {.cell-output .cell-output-stdout}
```
Analysis of Deviance Table
 Model 1: y ~ agegp + rc5
Model 2: y ~ agegp + rc5 + as.numeric(rc5):as.numeric(agegp) 
Deviance= 350.1315 p= 1.163786e-73 
Score= 340.592 p= 1.284355e-71 
```
:::
:::


Now we look at the association of internet use with age tested via the 
log linear models. Again, we find that the ordinal interaction model has 
a better fit to the data according to the Deviance and Score tests.



::: {.cell}

```{.r .cell-code}
anova(lin, sat)
```

::: {.cell-output .cell-output-stdout}
```
Analysis of Deviance Table
 Model 1: y ~ agegp + rc5 + as.numeric(rc5):as.numeric(agegp)
Model 2: y ~ agegp + rc5 + agegp:rc5 
Deviance= 45.60211 p= 0.02538278 
Score= 48.02927 p= 7.788495e-09 
```
:::
:::


Looking at the ordinal vs. categorical model comparison we again see a 
discrepancy between my results and Lumleys, with mine suggesting that the
ordinal model does not provide a better fit than the categorical model...



::: {.cell}

```{.r .cell-code}
# code for producing Table 6.1
null <- svyloglin(~ agegp + income + rc5, shs)
m1 <- update(null, ~ . + as.numeric(agegp):as.numeric(rc5))
m2 <- update(m1, ~ . + agegp:income)
m3 <- update(m2, ~ . + as.numeric(income):as.numeric(rc5))
m4 <- update(m2, ~ . + income:rc5)
full <- update(null, ~ .^2)
```
:::


As a final step in this demonstration, Lumley then looks at each possible
model that has both interactions, noting that an interaction between age
group and income isn't possible because of sparsity of data amongst some
income / age group categories.

Looking at Table 6.1 in the book, Lumley examines the ratio of deviance to 
degrees of freedom to provide a heuristic of determining whether the full
model fits well. I'm not fully convinced of his discussion here and there,
again, appears to be an error in his table.

## Exercises

1. Using the same data as in Section 5.2.4, define hypertension as systolic 
blood pressure greater than 140 mm Hg or diastolic blood pressure greater than
90 mmHg. Fit logistic regression models to investigate the association between
dietary sodium and potassium and hypertension.


::: {.cell}

```{.r .cell-code}
des <- svydesign(
  id = ~SDMVPSU, strat = ~SDMVSTRA, weights = ~fouryearwt,
  nest = TRUE, data = subset(nhanes, !is.na(WTDRD1))
)

des <- update(des,
  sodium = DR1TSODI / 1000, potassium = DR1TPOTA / 1000,
  hypertension = (BPXSAR > 140 | BPXDAR > 90) * 1
)
summary(svyglm(hypertension ~ sodium + potassium, design = des))
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = hypertension ~ sodium + potassium, design = des)

Survey design:
update(des, sodium = DR1TSODI/1000, potassium = DR1TPOTA/1000, 
    hypertension = (BPXSAR > 140 | BPXDAR > 90) * 1)

Coefficients:
             Estimate Std. Error t value Pr(>|t|)    
(Intercept)  0.162091   0.010856  14.931 7.33e-15 ***
sodium      -0.014148   0.003770  -3.753 0.000811 ***
potassium    0.010877   0.005121   2.124 0.042645 *  
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 0.1361541)

Number of Fisher Scoring iterations: 2
```
:::
:::


We see sodium intake is negatively associated with hypertension while
potassium is positively associated. Because we're not adjusting for age like we
saw previously, the sodium intake coefficient is biased up from its adjusted
rate.

2. This exercise uses the WA State Crime data for 2004 as the population. The
data consists of crime rates and population size for the police districts 
in cities/towns and sheriffs' offices in unincorporated areas, grouped by 
county.

 * Take a simple random sample of 10 counties from the state and use all the 
 data from the sampled counties. Estimate the total number of murders and 
 burglaries in the state.
 

::: {.cell}

```{.r .cell-code}
county_sample <- wa_crime_df %>%
  distinct(County) %>%
  slice_sample(n = 10) %>%
  pull(County)

wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design(
    ids = c(County, Agency),
    fpc = c(num_counties, num_agencies)
  ) %>%
  summarize(total = survey_total(murder_and_crime)) %>%
  mutate(Q = "a")
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
   total total_se Q    
   <dbl>    <dbl> <chr>
1 19566.    5414. a    
```
:::
:::

 
 * Fit a poisson regression (`family=quasipoisson`) to model the relationship
 between number of murders and population. Poisson regression fits a linear
 model to the logarithm of the mean of the outcome variable. If the murder rate
 were constant, the optimal transformation of the predictor variable would be 
 the logarithm of population, and its coefficient would be 1.0. Is this 
 supported by the data?
 

::: {.cell}

```{.r .cell-code}
srs_design <- wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties, num_agencies)
  )

poisson_fit <- svyglm(murder ~ I(log(Population)),
  family = quasipoisson,
  design = srs_design
)
summary(poisson_fit)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = murder ~ I(log(Population)), design = srs_design, 
    family = quasipoisson)

Survey design:
Called via srvyr

Coefficients:
                   Estimate Std. Error t value Pr(>|t|)   
(Intercept)         -9.8678     2.0628  -4.784  0.00138 **
I(log(Population))   0.9508     0.2092   4.546  0.00189 **
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for quasipoisson family taken to be 0.9839811)

Number of Fisher Scoring iterations: 6
```
:::
:::


Yes. We see that the estimate is 1.13 which is roughly 1 and within the 
standard error. Furthermore, the t-test shows that there is evidence this 
estimate is not 0 at $\alpha = 0.05$.
 
 * Predict the total number of murders in the state using the Poisson 
 regression model. Compare to a ratio estimator using population as the 
 auxiliary variable.
 

::: {.cell}

```{.r .cell-code}
true_murder_count <- sum(wa_crime_df$murder)
total_population <- sum(wa_crime_df$Population)
# doesn't work
# predict(poisson_fit, newdata = wa_crime_df, type = "response",
#        total = sum(wa_crime_df$Population))
# Works - for mean estimate, care needed for summing variance
sum(predict(poisson_fit, newdata = wa_crime_df, type = "response"))
```

::: {.cell-output .cell-output-stdout}
```
[1] 181.9975
```
:::
:::

::: {.cell}

```{.r .cell-code}
predict(
  svyratio(~murder, ~Population,
    design = srs_design
  ),
  total = sum(wa_crime_df$Population)
)
```

::: {.cell-output .cell-output-stdout}
```
$total
       Population
murder    193.091

$se
       Population
murder   30.04947
```
:::
:::



We see that while both estimates are reasonably close to the true state 2004 
murder count of 189, the ratio estimator offers a readily 
available standard error while its not clear how to extract the same estimate
from the `svyglm` output. Note that the same code arguments used extract totals
in Lumley's election example does not appear to work here...


 * Take simple random samples of five police districts from King County and five
 counties from the rest of the state. Fit a Poisson regression model with 
 population and stratum (King County vs elsewhere) as predictors. Predict the
 total number of murders in the state.
 

::: {.cell}

```{.r .cell-code}
agency_sample <- wa_crime_df %>%
  filter(County == "king") %>%
  distinct(Agency) %>%
  slice_sample(n = 10) %>%
  pull(Agency)

non_king_county_sample <- wa_crime_df %>%
  filter(County != "king") %>%
  distinct(County) %>%
  slice_sample(n = 10) %>%
  pull(County)

county_list <- unique(wa_crime_df$County)

strata_design <- wa_crime_df %>%
  filter(County %in% non_king_county_sample | Agency %in% agency_sample) %>%
  mutate(
    strata_label = if_else(County == "king", "strata 1", "strata 2"),
    num_counties = if_else(County == "king", 1, length(county_list) - 1)
  ) %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties, num_agencies),
    strata = strata_label
  )


poisson_fit <- svyglm(murder ~ I(log(Population)) + strata_label,
  design = strata_design,
  family = quasipoisson
)
# doesn't produce appropriate output
# predict(poisson_fit, total = total_population)
sum(predict(poisson_fit,
  newdata = wa_crime_df %>% mutate(strata_label = if_else(County == "king", "strata 1", "strata 2")),
  type = "response"
))
```

::: {.cell-output .cell-output-stdout}
```
[1] 186.9499
```
:::
:::

This estimate appears to again, make sense though we have no standard error
estimate readily available to compare with the previous. One would guess that
it would be more precise, given extra information that comes from including
the King County agencies.
 
3. The variable MISEFFRT asks "How often in the past 30 days did you feel that 
everything was an effort?", on a 1-5 scale with 1 meaning "All" and 5 meaning 
"none". Investigate whether this variable varies seasonally and whether it 
peaks in winter by defining predictor variables cos(IMONTH * 0.5236) and 
sin(IMONTH * 0.5236), which describe smooth anual cycles and fitting:

 * A logistic regression model with outcome `MISEFFRT = 5`
 * A linear regresion model with outcome MISEFFRT
 * What further modeling could you do to investigate whether sunlight intensity 
   was related to this seasonal variation.
   
It isn't clear to what dataset Lumley is referring to here... so I'll leave
this question unanswered.
   
4. Using the California Health Interview Study 2005 data,

 * Fit a logistic regression model to the relationship between the probability 
 of having health insurance (`ins`) and household annual income (`ak22_p`). 
 Important potential confounders include age (`srage_p`), sex (`srsex`) and 
 race (`racecen`) and interactions may also be important.
 

::: {.cell}

```{.r .cell-code}
fit <- svyglm(I((ins == 1) * 1) ~ ak22_p + srage_p + srsex,
  design = chis,
  family = quasibinomial
)
summary(fit)
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = I((ins == 1) * 1) ~ ak22_p + srage_p + srsex, 
    design = chis, family = quasibinomial)

Survey design:
update(chis, smoking = droplevels(smoking), ins = droplevels(ins))

Coefficients:
              Estimate Std. Error t value Pr(>|t|)    
(Intercept) -1.269e+00  7.360e-02  -17.24   <2e-16 ***
ak22_p       2.061e-05  1.032e-06   19.97   <2e-16 ***
srage_p      4.035e-02  1.150e-03   35.09   <2e-16 ***
srsexFEMALE  4.785e-01  4.368e-02   10.96   <2e-16 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for quasibinomial family taken to be 1.591744)

Number of Fisher Scoring iterations: 6
```
:::
:::


 * What would be the impact on the interpretation of the income coefficient of
 adding a variable to the model indicating whether an employer provided
 health insurance?
 
Presumably income is correlated with employee offered healthcare insurance, 
though I don't know this for a fact. If this were the case then we'd see that
the effect of the income coefficient would be attenuated (move towards zero)
since its effect would now be more precisely captured by the employee
offered health care measurement.
 
5. Using the California Health Interview Study 2005 data, fit a relative risk
regression model to the relationship between the probability of having health
insurance (`ins`) and household annual income (`ak22_p`), age (`srage_p`),
sex (`srsex`) and race (`racecen`) . Compare the coefficients to those from a
logistic regression.


::: {.cell}

```{.r .cell-code}
fit <- svyglm(I((ins == 1) * 1) ~ ak22_p,
  design = chis,
  family = quasibinomial(link = "log"),
  start = c(-.5, 0),
  control = list(maxit = 150)
)
summary(fit)
```
:::


Despite trying several different starting values, "max iterations", and model
specifications, I wasn't able to fit a model as specified without getting
warning messages or errors. In general if the model *can* be fit, the 
exponentiated intercept represents the adjusted probability of the event
occuring --- in this case having health insurance --- and each subsequent
exponentiated coefficient represents the increase in probability, or relative
risk of the event occuring, conditional on a one unit change in value of the
covariate.

6. Using the same data as in Section 5.2.4, create an ordinal blood pressure
variable based on systolic and diastolic pressure with categories "normal" 
(systolic < 120 and diastolic < 80), "prehypertension" (systolic < 140,
diastolic < 90), "hypertension stage 1" (systolic < 160, diastolic < 100),
and "hypertension stage 2" ( systolic at least 160 or diastolic at least 100). 
Fit proportional odds regression models to investigate the association between 
dietary sodium and potassium and hypertension.


::: {.cell}

```{.r .cell-code}
nhanes <- nhanes %>%
  mutate(
    ordinal_high_bp = factor(dplyr::case_when(
      BPXSAR < 120 & BPXDAR < 80 ~ "Normal",
      (BPXSAR < 140 & BPXSAR >= 120) & (BPXDAR >= 80 & BPXDAR < 90) ~
        "PreHypertension",
      (BPXSAR >= 140 & BPXSAR < 160) & (BPXDAR >= 90 & BPXDAR < 100) ~
        "Hypertension Stage 1",
      BPXSAR >= 160 & BPXDAR >= 100 ~ "Hypertension Stage 1"
    ), levels = c(
      "Normal", "PreHypertension", "Hypertension Stage 1",
      "Hypertension Stage 2"
    ))
  )

des <- svydesign(
  id = ~SDMVPSU, strat = ~SDMVSTRA, weights = ~fouryearwt,
  nest = TRUE, data = subset(nhanes, !is.na(WTDRD1))
)

fit <- svyolr(ordinal_high_bp ~ sodium + potassium, design = des)
summary(fit)
```

::: {.cell-output .cell-output-stdout}
```
Call:
svyolr(ordinal_high_bp ~ sodium + potassium, design = des)

Coefficients:
                Value Std. Error   t value
sodium    -0.02732083 0.03175951 -0.860241
potassium  0.14089855 0.05304160  2.656378

Intercepts:
                                          Value   Std. Error t value
Normal|PreHypertension                     2.1054  0.1284    16.3969
PreHypertension|Hypertension Stage 1       3.8628  0.2031    19.0148
Hypertension Stage 1|Hypertension Stage 2 12.0641  0.1381    87.3570
(9218 observations deleted due to missingness)
```
:::
:::


Because many of the individuals have blood pressure values that aren't covered
by the categories Lumley listed, we lose ~9000 observations leading to 
a decrease in the precision of our estimates. Notably, the sodium coefficient
is no now significant (|t-value| < 2). Because the missignness here is 
self-caused I wouldn't draw any strong conclusions from this model.

7. Using data for Florida (`X_STATE = 12`) from the 2007 BRFSS, fit a loglinear
model to associations between the following risk factors and behaviors:
perform vigorous physical activity (`VIGPACT`), eat five or more servings of
fruit or vegetables per day (`X_FV5SRV`), binge drinking of alcohol 
(`X_RFBING4`), ever had an HIV test (`HIVST5`), ever had Hepatitis B vaccine 
(`HEPBVAC`), age group (`X_AGE_G`) and sex, `X_SEXG`. All except age group are
binary. You will need to remove missing values, coded 7,8 or 9.


::: {.cell}

```{.r .cell-code}
brfss_sub <- subset(brfss, X_STATE == 12 & !any(VIGPACT %in% c(7, 8, 9)) &
  !any(X_FV5SRV %in% 7:9) & !any(X_RFBING4 %in% 7:9) &
  !any(HIVTST5 %in% 7:9) & !any(HEPBVAC %in% 7:9) &
  !any(X_AGE_G %in% 7:9) & !any(X_SEXG_ %in% 7:9))
loglin_fit <- svyloglin(~ VIGPACT + X_FV5SRV + X_RFBING4 + HIVTST5 + HEPBVAC +
  X_AGE_G + X_SEXG_, design = brfss_sub)
summary(loglin_fit)
```
:::


Despite all the subsetting I received the following error message when trying
to fit the model as suggested. I'm guessing the dimensionality of the 
model matrix is quite large and the underlying functions don't use a sparse
representation...

`Error: vector memory limit of 50.0 Gb reached, see mem.maxVSize()`

# Chapter 7: Post-Stratification, Raking, and Calibration

## Introduction - Motivation

Lumley motivates the need to explore the three titular topics by expanding on
the principle developed in the second chapter --- stratification. Similar as to
how making use of the extra information available in strata we can improve
estimates in straightforward estimation of totals and means, Lumley's focus in
this chapter is how to use the "auxiliary" information to adjust for
non-response bias and improve the precision of the estimates.

## Post-Stratification

Post-stratification is exactly what it sounds like - re-weighting estimates
according to strata totals *after* or apart from any initial strata that might
have been involved in the inital sampling design.

Consider a relatively straightforward design in which there's a population of
subjects of size $N$ that can be partitioned into $K$ mutually exclusive strata
from which any of the $N_k$ individuals in that strata can be sampled for $n_k$
strata samples. In this setting the sampling weights for each individual in
group $k$ is $\frac{N_k}{n_k}$ and $N_k$ is known without any uncertainty.

If the sampling were not stratified but $N_k$ were still known, the group sizes
would not be exactly correct by simple Horvitz-Thompson estimation, but they
could be corrected by re-weighting so that the sizes are correct as they would
be in stratified sampling.

Specifically, take each weight $w_i = \frac{1}{\pi_i}$ and construct new weights
$w_i^* = \frac{g_i}{\pi_i} = \frac{N_k}{\hat{N}_k} \times \frac{1}{\pi_i}$.

For estimating the group side of the kth group then, we'll have

$$
n_k \times \frac{g_i}{\pi_i} = n_k \times \frac{1}{\pi_i} \times
\frac{N_k}{\hat{N}_k} = n_k \times \frac{\hat{N}_k}{n_k} \times 
\frac{N_k}{\hat{N}_k} = N_k,
$$ where $\pi_i = \frac{n_k}{\hat{N}_k}$. The consequence of this re-weighting
means that the estimated sub group population is exactly correct and subsequent
estimates within or across these groups benefit from the extra information.

Of course, as Lumley notes, there's a problem if no entities were sampled in the
particular strata of interest - you can't re-weight the number 0. Still since
this is unlikely to happen for groups and samples of "reasonable" size
post-stratification is still a worthy strategy given the potential reductions in
variance that are possible.

### Illustration

Lumley's illustration of post-stratification looks at the two-stage sample drawn
from the API population, with 40 school districts sampled from California and
then up to 5 schools sampled from each district. Lumley uses this example to
illustrate how improvements to precision can be made via post-stratification --
or not.

We'll start with a reminder of the sample design used here: a two-stage sample.


::: {.cell}

```{.r .cell-code}
clus2_design
```

::: {.cell-output .cell-output-stdout}
```
2 - level Cluster Sampling design
With (40, 126) clusters.
svydesign(id = ~dnum + snum, fpc = ~fpc1 + fpc2, data = apiclus2)
```
:::
:::


Then information about the population group sizes is included in the call to
`postStratify()` as well as the variable/strata across which to post-stratify.


::: {.cell}

```{.r .cell-code}
pop.types <- data.frame(stype = c("E", "H", "M"), Freq = c(4421, 755, 1018))
ps_design <- postStratify(clus2_design, strata = ~stype, population = pop.types)
ps_design
```

::: {.cell-output .cell-output-stdout}
```
2 - level Cluster Sampling design
With (40, 126) clusters.
postStratify(clus2_design, strata = ~stype, population = pop.types)
```
:::
:::


Totals, and so on are then estimated in the usual fashion. In this example
there's a large difference in the variability of the estimated total when
comparing the naive and post-stratified estimates because much of the
variability in the number of students enrolled can be explained by the school
type - elementary schools are typically smaller than middle schools and
highschools. By including more information about the types of schools in the
overall population, the standard error is decreased by a factor of \~ 2.6.


::: {.cell}

```{.r .cell-code}
svytotal(~enroll, clus2_design, na.rm = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 2639273 799638
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~enroll, ps_design, na.rm = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 3074076 292584
```
:::
:::


In contrast, school type is not associated with the variability in the school
scores measured by the Academic Performance Index - denoted below by `api00`.


::: {.cell}

```{.r .cell-code}
svymean(~api00, clus2_design)
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
api00 670.81 30.099
```
:::
:::

::: {.cell}

```{.r .cell-code}
svymean(~api00, ps_design)
```

::: {.cell-output .cell-output-stdout}
```
      mean     SE
api00  673 28.832
```
:::
:::


Indeed, the score is specifically setup to be standardized across school types
and as such there's little variance reduction observed by using the
post-stratification information in this instance.

Lumley notes that if the api dataset were a real survey, non response might vary
as a function of school type and in which case post-stratification could help
reduce non-response bias.

## Raking

If one were to post-stratify using more than one variable would require the
complete joint distribution of both variables. This can be problematic 
because the population totals for the joint distribution - or cross 
classification as Lumley calls it - is not available. Raking is a method
that aims to overcome this problem.

> The process involves post-stratifying on each set of variables in turn, and 
repeating this process until the weights stop changing.

Lumley further highlights the connection between log-linear regression models
and raking:

> Raking could be considered a form of post-stratification where a log-linear 
model is used to smooth out the sample and population tables before the weights
are adjusted.



::: {.cell}

```{.r .cell-code}
load("Data/Family Resource Survey/frs.rda")
frs.des <- svydesign(ids = ~PSU, data = frs)
pop.ctband <- data.frame(
  CTBAND = 1:9,
  Freq = c(
    515672, 547548, 351599, 291425,
    266257, 147851, 87767, 9190, 19670
  )
)
pop.tenure <- data.frame(
  TENURE = 1:4,
  Freq = c(1459205, 493237, 128189, 156348)
)

frs.raked <- rake(frs.des,
  sample = list(~CTBAND, ~TENURE),
  population = list(pop.ctband, pop.tenure)
)
overall <- svymean(~HHINC, frs.raked)
with_children <- svymean(~HHINC, subset(frs.raked, DEPCHLDH > 0))
children_singleparent <- svymean(~HHINC, subset(frs.raked, DEPCHLDH > 0 & ADULTH == 1))
c(
  "Overall" = overall,
  "With Children" = with_children,
  "Single Parent" = children_singleparent
)
```

::: {.cell-output .cell-output-stdout}
```
      Overall.HHINC With Children.HHINC Single Parent.HHINC 
           475.2484            605.1928            282.7683 
```
:::
:::


## Generalized Raking, Greg Estimation, and Calibration

Lumley identifies two ways to understand post-stratification:

1. Post-stratification makes small changes to the sampling weights such that the 
estimated totals match the population totals.

2. Post-stratification is a regression estimator, where each post-stratum is 
an indicator in a regression model.

An extension of the first view leads to calibration estimators while the 
second leads to generalized regression estimators. The former requires 
correctly specifying how to change the weights, the second requires correctly
specifying the model. Both can lead to great increases in precision of the
target estimates.

Lumley's motivation of calibration starts with the regression estimate of a
population total: if we have the auxiliary variable $X_i$ available on all
units in the population and an estimated $\hat{\beta}$ from a sample, we can
estimate $\hat{T}$ as the sum of the predicted values from the regression.

$$
\hat{T}_{reg} = \sum_{i=1}^N X_i\hat{\beta}
$$

From this starting point, Lumley describes that calibration follows a similar 
form, with the unknown parameter $\hat{\beta}$ know a function of the 
calibration weights $g_i$ and original sampling weights $\pi_i$ defined in such
a way that the population total of $X$ is equal to the estimated total of $X$.
Lumley identifies this as the *calibration constraints*:

$$
T_x = \sum_{i=1}^{n} \frac{g_i}{\pi_i} X_i.
$$

However, this constraint alone will not uniquely identify the $g_i$. 
Consequently, the specification of calibration weights are completed by 
requiring that they be "close" to the sampling weights - minimizing a distance
function, while still satisfying the previous constraint. 

Calibration provides a unified view of post-stratification and raking. As Lumley 
states:

> In linear regression calibration, the calibration weights $g_i$ are a linear 
function of the auxiliary variables; in raking calibration the calibration
weights are a multiplicative function of the auxiliary variables.

Variance estimation proceeds in a similar fashion for calibration as it did
for post-stratification, by constructing an unbiased estimator for the residual
between $Y_i$ and the estimated mean, or population total.

## Calibration in R

**Linear regression calibration**

We're in a similar spot as before with regard to the election data example.

::: {.cell}

:::

::: {.cell}

```{.r .cell-code}
pop.size <- sum(pop.ctband$Freq)
pop.totals <- c(
  "(intercept)" = pop.size, pop.ctband$Freq[-1],
  pop.tenure$Freq[-1]
)
frs.cal <- calibrate(frs.des,
  formula = ~ factor(CTBAND) + factor(TENURE),
  population = pop.totals,
  calfun = "raking"
)
```

::: {.cell-output .cell-output-stdout}
```
Sample:  [1] "(Intercept)"     "factor(CTBAND)2" "factor(CTBAND)3" "factor(CTBAND)4"
 [5] "factor(CTBAND)5" "factor(CTBAND)6" "factor(CTBAND)7" "factor(CTBAND)8"
 [9] "factor(CTBAND)9" "factor(TENURE)2" "factor(TENURE)3" "factor(TENURE)4"
Popltn:  [1] "(intercept)" ""            ""            ""            ""           
 [6] ""            ""            ""            ""            ""           
[11] ""            ""           
```
:::

```{.r .cell-code}
svymean(~HHINC, frs.cal)
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
HHINC 475.25 5.7306
```
:::

```{.r .cell-code}
svymean(~HHINC, subset(frs.cal, DEPCHLDH > 0))
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
HHINC 605.19 11.221
```
:::

```{.r .cell-code}
svymean(~HHINC, subset(frs.cal, DEPCHLDH > 0 & ADULTH == 1))
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
HHINC 282.77 9.3403
```
:::
:::


#### Comparing calibration methods 


::: {.cell}

```{.r .cell-code}
clus1 <- svydesign(id = ~dnum, weights = ~pw, data = apiclus1, fpc = ~fpc)
logit_cal <- calibrate(clus1, ~ stype + api99,
  population = c(6194, 755, 1018, 3914069),
  calfun = "logit", bounds = c(0.7, 1.7)
)
svymean(~api00, clus1)
```

::: {.cell-output .cell-output-stdout}
```
        mean     SE
api00 644.17 23.542
```
:::
:::

::: {.cell}

```{.r .cell-code}
svymean(~api00, logit_cal)
```

::: {.cell-output .cell-output-stdout}
```
        mean   SE
api00 665.46 3.42
```
:::
:::

::: {.cell}

```{.r .cell-code}
m0 <- svyglm(api00 ~ ell + mobility + emer, clus1)
summary(svyglm(api00 ~ ell + mobility + emer, clus1))
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = api00 ~ ell + mobility + emer, design = clus1)

Survey design:
svydesign(id = ~dnum, weights = ~pw, data = apiclus1, fpc = ~fpc)

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept) 780.4595    30.0210  25.997 3.16e-11 ***
ell          -3.2979     0.4689  -7.033 2.17e-05 ***
mobility     -1.4454     0.7343  -1.968  0.07474 .  
emer         -1.8142     0.4234  -4.285  0.00129 ** 
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 6628.496)

Number of Fisher Scoring iterations: 2
```
:::
:::

::: {.cell}

```{.r .cell-code}
m1 <- svyglm(api00 ~ ell + mobility + emer, logit_cal)
summary(svyglm(api00 ~ ell + mobility + emer, logit_cal))
```

::: {.cell-output .cell-output-stdout}
```

Call:
svyglm(formula = api00 ~ ell + mobility + emer, design = logit_cal)

Survey design:
calibrate(clus1, ~stype + api99, population = c(6194, 755, 1018, 
    3914069), calfun = "logit", bounds = c(0.7, 1.7))

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept) 789.1015    17.7622  44.426 9.18e-14 ***
ell          -3.2425     0.4803  -6.751 3.15e-05 ***
mobility     -1.5140     0.6436  -2.352 0.038318 *  
emer         -1.7793     0.3824  -4.653 0.000702 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Dispersion parameter for gaussian family taken to be 7034.423)

Number of Fisher Scoring iterations: 2
```
:::
:::

::: {.cell}

```{.r .cell-code}
predict(m0, newdata = data.frame(ell = 5, mobility = 10, emer = 10))
```

::: {.cell-output .cell-output-stdout}
```
    link     SE
1 731.37 26.402
```
:::
:::

::: {.cell}

```{.r .cell-code}
predict(m1, newdata = data.frame(ell = 5, mobility = 10, emer = 10))
```

::: {.cell-output .cell-output-stdout}
```
    link    SE
1 739.96 15.02
```
:::
:::


#### Cluster-level weights

Although altering the weights can lead to higher precision of estimates 
sampling weights applied to clusters - which are identical - can then lead
to unintuitive results -- Lumley gives the example of mothers and infants 
sampled together but the number of infants not adding up to the number of
mothers when using the calibrated weights. Consequently, Lumley includes an
option in the `survey` package to force within-cluster weights to be identical 
when calibrated at a given stage --- denoted below by the `aggregate.stage` 
argument.


To illustrate Lumley uses the `clus2_design` from earlier in the chapter, where
the design was post-stratified on school type. I don't really follow Lumley's
argument here but he says that the there is a loss of precision because of the
added constraints, but that's not what we see below or in the text - the 
constrained calibration has a lower standard error than the post-stratified 
design... Its hard to inspect things too carefully here because Lumley
uses a `cal` variable that's assigned a calibration object, presumably, but
does not show how he assigns it in the text, so we're left wondering.

::: {.cell}

```{.r .cell-code}
cal2 <- calibrate(clus2_design, ~stype,
  pop = c(4421 + 755 + 1018, 755, 1018),
  aggregate.stage = 1
)
svytotal(~enroll, cal2, na.rm = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 3084777 246321
```
:::
:::

::: {.cell}

```{.r .cell-code}
svytotal(~enroll, ps_design, na.rm = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
         total     SE
enroll 3074076 292584
```
:::
:::

::: {.cell}

```{.r .cell-code}
range(weights(cal) / weights(clus2_design))
```
:::

::: {.cell}

```{.r .cell-code}
range(weights(cal2) / weights(clus2_design))
```

::: {.cell-output .cell-output-stdout}
```
[1] 0.6418424 1.6764125
```
:::
:::


### Basu's Elephants

Lumley walks through a classic story[@basu2011essay] in statistics about how  
poor use of auxiliary information can lead to unreasonable behavior of design-
based inference. Lumley then goes on to show how calibrated weights could make
this more efficient. I'll summarize briefly below.

Suppose a circus owner has 50 elephants and wants to estimate their 
total weight. The catch? The owner only wants to use **one** of the elephants
to construct the estimate.

One could construct an unbiased point estimate by taking a randomly sampled
elephant's weight and multiplying it by fifty. No estimate of the variance is
available of course, because there's no way to get an estimate of the 
variability in the elephants' weights.

Alternatively, a **model** based approach could depend on some auxiliary variable.
Suppose the circus owner knew one particular elephant named Sambo was about
average weight when all the elephants were weighed some years ago. The proposed
method here then would be to sample Sambo with 100% probability and multiply his
weight by 50. Lumley notes that this can't be considered a valid **design** 
based estimate because all the other elephants had 0% probability of being
sampled.

Basu then imagines a compromise design that conforms to the necessary
conditions required for design based inference, but that produces a nonsensical
estimate. If Sambo was sampled with high probility, say 99%, then the point
estimate would be $\frac{100}{99} \times$ Sambo's weight if Sambo is sampled and
5000 $\times$ the weight of any other elephant.

Obviously, this is not a good estimate, though Lumley notes that it will
fulfill the Horvitz Thompson Property of being unbiased when averaging over
repeated sampling. The problem of course being in the extreme variability of
the estimate.

Lumley uses this scenario as an opportunity to discuss how to best use the 
auxiliary information (Sambo's roughly average weight) in the context of setting
up a design based estimator, arguing that the use of the information and the
Horvitz Thompson Estimator are inappropriately used.

#### Using auxiliary information in design

The first point Lumley expands on is how to use auxiliary information. He argues
that a stratified sample would be a better way to use information that Sambo 
is roughly average --- splitting elephants according to whether they were 
small, middle or large looking. 
[Ranked Set Sampling](https://www.asc.ohio-state.edu/dean.9/papers/wolfe-ranked-set-sampling.pdf)
is also called out as a strategy that might be more useful in this setting.

#### Using Auxiliary Information in Analysis

The bigger point Lumley wants to make is that it would've been better to use
the population size information to calibrate the weights, i.e. estimate the
population total with weights $\frac{g_i}{\pi_i} := 50$, so that
the estimate would be, as in the first approach, 50 times a sampled elephant's 
weight.

Lumley also lays out two ratio and difference calibration based models that
use the previous elephant's weight as a way to more precisely estimate the 
change in elephants weight since last weighing as as a generalization to the
total. 


All-in-all, Lumley's point is to highlight how auxiliary information can be
more efficiently utilized and to demonstrate how the simple example fails to
realize the full power of a design based approach.


### Selecting Auxiliary Variables for Non-Response

All the methods discussed thus far this chapter can be used to reduce the bias
from missing data, specifically what Lumley calls *unit non-response*. The 
phenomenon when the sampled unit cannot be measured, for example a person 
sampled for a telephone interview does not pick up the phone.

Missing data is an interesting research topic in its own right with standard
fundamentals describing the different 
[mechanisms]((https://en.wikipedia.org/wiki/Missing_data#Types) or models of 
how missing data may emerge. Lumley's discussion of how auxiliary variables
for non-response can be selected is equivalent to the missing data mechanism
known as "Missing at Random", which implies that the fact that a
measurement is missing is independent of the measurement itself, conditional
on the other variables measured.

In the design-based view, this can mean one of several things, though Lumley
focuses mainly on the strata in post-stratification. That is, the 
"other variables measured" refers to the strata information. 

Lumley uses a telephone survey to illustrate this concept. If non-response
was higher for land line than cellphone users but within each group the 
probability of responding was independent of the measure itself, then
an analysis that combined these two groups together would be biased, but a 
post-stratified analysis would not be, since it estimated the strata specific
rates.


#### Direct Standardization 

Lumley identifies "Direct Standardization" as an application of 
post-stratification to extrapolate an estimate in a different population than
the sampled population, while adjusting for as many known confounders as
possible. This is a bit like comparing regression predictions.

####  Standard error estimation

Lumley makes a brief note here that the same approach for computing standard
errors with complete data is used for incomplete data. Though, he makes 
a note that "secondary analysis" of large-scale surveys is not completely 
possible --- I'm not clear what he means here, if this is dependent on some
sampling stage specific information or perhaps he is referring to "domain" or
sub-population estimation.

In any case, Lumley states that a conservative approach is used in this case
when the calibration estimates are used --- again, I'm guessing he's referring 
to the `survey` R package's implementation, though it isn't explicit. However,
if replicate weights are available, then a full estimate is available.

## Exercises

1. Using the WA State Crime population, take a stratified random sample of five
police districts from King County and five counties from the rest of the state.

Setting up the sample design similar to before...


::: {.cell}

```{.r .cell-code}
agency_sample <- wa_crime_df %>%
  filter(County == "king") %>%
  distinct(Agency) %>%
  slice_sample(n = 10) %>%
  pull(Agency)

non_king_county_sample <- wa_crime_df %>%
  filter(County != "king") %>%
  distinct(County) %>%
  slice_sample(n = 10) %>%
  pull(County)

county_list <- unique(wa_crime_df$County)

strata_design <- wa_crime_df %>%
  filter(County %in% non_king_county_sample | Agency %in% agency_sample) %>%
  mutate(
    strata_label = factor(if_else(County == "king", "strata 1", "strata 2")),
    num_counties = if_else(County == "king", 1, length(county_list) - 1)
  ) %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties, num_agencies),
    strata = strata_label
  )
```
:::


* a. Calibrate the sample using stratum and population as the auxiliary 
variables. Estimate the number of murders and number of burglaries in the state 
using the calibrated and un-calibrated sample.


::: {.cell}

```{.r .cell-code}
pop_size <- wa_crime_df %>%
  summarize(p = sum(Population)) %>%
  pull(p)
non_king_county_pop_size <- wa_crime_df %>%
  filter(County != "king") %>%
  summarize(p = sum(Population)) %>%
  pull(p)

pop.totals <- c(
  "(Intercept)" = pop_size,
  "strata_labelstrata 2" = non_king_county_pop_size
)
wa_crime_cal <- calibrate(strata_design,
  formula = ~strata_label,
  population = pop.totals
)
rbind(
  c("calibration", svytotal(~murder_and_crime, wa_crime_cal)),
  c("Sample-Based", svytotal(~murder_and_crime, strata_design))
)
```

::: {.cell-output .cell-output-stdout}
```
                    murder_and_crime  
[1,] "calibration"  "3106294463.83561"
[2,] "Sample-Based" "96464"           
```
:::
:::


The calibration numbers look *far* too high here, which is puzzling given that
the weights should be exactly calibrated to better reproduce the population
total.


* b. Convert the original survey design object to use jackknife replicate 
weights. Calibrate the replicate-weight design using the same auxiliary 
variables and estimate the number of burglaries and of murders in the state.


::: {.cell}

```{.r .cell-code}
rep_design <- survey::as.svrepdesign(strata_design)

wa_crime_rep_cal <- calibrate(rep_design,
  formula = ~strata_label,
  population = pop.totals
)
svytotal(~murder_and_crime, wa_crime_rep_cal)
```

::: {.cell-output .cell-output-stdout}
```
                      total        SE
murder_and_crime 3106294464 164778523
```
:::
:::


This produces the same, erroneous, number as before.

* c. Calibrate the sample using the population and number of burglaries in the
previous year as auxiliary variables, and estimate the number of burglaries and 
murders in the state.


::: {.cell}

```{.r .cell-code}
wa_crime_03_df <- readxl::read_xlsx("data/WA_crime/1984-2011.xlsx",
  skip = 4
) %>%
  filter(Year == "2003", Population > 0) %>%
  mutate(
    murder_and_crime = `Murder Total` + `Burglary Total`,
    violent_crime = `Violent Crime Total`,
    burglaries = `Burglary Total`,
    property_crime = `Property Crime Total`,
    state_pop = sum(Population),
    County = stringr::str_to_lower(County),
    num_counties = n_distinct(County),
  ) %>%
  group_by(County) %>%
  mutate(num_agencies = n_distinct(Agency)) %>%
  ungroup() %>%
  select(
    County, Agency, Population, murder_and_crime, burglaries, property_crime,
    violent_crime, num_counties, num_agencies
  )

cols_to_keep <- c(
  "County", "Agency", "Population", "burglaries",
  "num_counties", "num_agencies", "murder_and_crime", "violent_crime",
  "property_crime"
)
wa_crime_df_mod <- wa_crime_df %>%
  select(all_of(cols_to_keep)) %>%
  rename_if(is.numeric, function(x) str_c(x, "_04"))

sample_design <- wa_crime_03_df %>%
  select(all_of(cols_to_keep)) %>%
  rename_if(is.numeric, function(x) str_c(x, "_03")) %>%
  right_join(wa_crime_df_mod) %>%
  filter(
    (County %in% non_king_county_sample |
      Agency %in% agency_sample)
  ) %>%
  mutate(
    strata_label = factor(if_else(County == "king", "strata 1", "strata 2")),
    num_counties = if_else(County == "king", 1, length(county_list) - 1)
  ) %>%
  as_survey_design(
    id = c(County, Agency),
    fpc = c(num_counties_04, num_agencies_04),
    strata = strata_label
  )

pop_03 <- wa_crime_03_df %>%
  summarize(P = sum(Population)) %>%
  pull(P)

previous_year_calibration <- calibrate(sample_design,
  formula = ~burglaries_03,
  population = c(
    "(Intercept)" = pop_size,
    "burglaries_03" = pop_03
  )
)
svytotal(~burglaries_04, previous_year_calibration)
```
:::


Here I get an error message that one of the inner computations doesn't have
the correct dimensions. I suspect this is a bug in Lumley's code as the 
sampling design I constructed above isn't too complicated.

* d. Estimate the ratio of violent crimes to property crimes in the state, 
using the un-calibrated sample and the sample calibrated on population and 
number of burglaries.



::: {.cell}

```{.r .cell-code}
strata_design %>%
  summarize(
    ratio = survey_ratio(violent_crime, property_crime)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
   ratio ratio_se
   <dbl>    <dbl>
1 0.0684  0.00599
```
:::
:::

::: {.cell}

```{.r .cell-code}
svyratio(
  numerator = ~violent_crime_04,
  denominator = ~property_crime_04,
  design = previous_year_calibration
)
```
:::


Again, the sample based estimator makes sense here, while the calibrated 
estimator has issues. So far this is my best understanding of how to apply
Lumley's code but I'll try to revisit this in the future.

2. Write an R function that accepts a set of 50 elephant weights and simulates
repeatedly choosing a single elephant and computing the Horvitz-Thompson and
ratio estimators of the total weight, reporting the mean and variance over
the repeated simulations. Explore the behavior for several sets of elephant 
weights. Verify that the Horvitz-Thompson estimator is always unbiased, but 
usually further from the truth than the ratio estimator.


::: {.cell}

```{.r .cell-code}
BasuHTE <- function(elephant_weights) {
  N <- length(elephant_weights)
  sambo <- which.min(abs(elephant_weights - mean(elephant_weights)))
  sample_probs <- rep((1 - 0.99) / (N - 1), N)
  sample_probs[sambo] <- 0.99
  # Okay to use the base R sample implementation here because we're only
  # sampling 1 item.
  x_ix <- sample(1:N, size = 1, prob = sample_probs)
  hte_weight <- 1 / sample_probs[x_ix] * elephant_weights[x_ix]
  ratio_estimate <- N * elephant_weights[x_ix]
  return(c("hte" = hte_weight, "ratio" = ratio_estimate))
}
elephant_is_male <- rbinom(50, size = 1, prob = 0.5)
elephant_weights <- elephant_is_male * rnorm(50, mean = 1.3E4, sd = 2E3) +
  (1 - elephant_is_male) * rnorm(50, mean = 6.6E3, sd = 1E3)
results <- as_tibble(t(replicate(1000, BasuHTE(elephant_weights)))) %>%
  mutate(rep_ix = 1:n()) %>%
  gather(hte, ratio, key = "estimator", value = "estimate") %>%
  group_by(estimator) %>%
  mutate(mean = mean(estimate), sd = sd(estimate)) %>%
  ungroup()
```
:::

::: {.cell}

```{.r .cell-code}
true_total <- sum(elephant_weights)
results %>%
  ggplot(aes(x = estimate, fill = estimator)) +
  geom_histogram() +
  geom_vline(aes(xintercept = mean, color = estimator)) +
  geom_vline(aes(xintercept = true_total), linetype = 2, color = "red") +
  facet_wrap(~estimator, nrow = 2) +
  ggtitle("Illustrating Basu's Elephants")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/q7_2_results-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
results %>%
  filter(estimator == "hte") %>%
  summarize(average_estimate = mean(estimate)) %>%
  mutate(
    bias = average_estimate - true_total,
    truth = true_total
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
  average_estimate   bias   truth
             <dbl>  <dbl>   <dbl>
1          591775. 57162. 534613.
```
:::
:::


From the plot above we can see that the horvitz thompson estimator is more
variable than the ratio estimator though both provide the appropriate estimate
on average.

3. Write an R function that accepts a set of 50 elephant weights and performs
the ranked set sampling procedure on page 150 to choose three of them. By
simulation, compare the bias and variance of the estimated total from ranked-set
sample to estimated totals from a simple random sample of three elephants.


::: {.cell}

```{.r .cell-code}
RankedSetSampleComparison <- function(elephant_weights) {
  srs_sample <- sample(elephant_weights, size = 3)
  sampling_weight <- length(elephant_weights) / 3
  srs_total <- sum(srs_sample * length(elephant_weights) / 3)
  rs_sample_one <- sample(elephant_weights, size = 3)
  rs_sample_two <- sample(elephant_weights, size = 3)
  rs_sample_three <- sample(elephant_weights, size = 3)
  rs_total <- rs_sample_one[which.min(rank(rs_sample_one))] * sampling_weight +
    rs_sample_two[which(rank(rs_sample_two) == 2)] * sampling_weight +
    rs_sample_three[which.max(rank(rs_sample_three))] * sampling_weight
  return(c("Ranked Set Estimate" = rs_total, "SRS Total" = srs_total))
}
results <- as_tibble(t(replicate(
  1000,
  RankedSetSampleComparison(elephant_weights)
))) %>%
  mutate(rep_ix = 1:n()) %>%
  gather(everything(), -rep_ix, key = "estimator", value = "estimate") %>%
  group_by(estimator) %>%
  mutate(mean = mean(estimate), sd = sd(estimate))
```
:::

::: {.cell}

```{.r .cell-code}
results %>%
  ggplot(aes(x = estimate, fill = estimator)) +
  geom_histogram() +
  geom_vline(aes(xintercept = true_total), linetype = 2) +
  ggtitle("Ranked Set and Simple Random Sample Estimator Comparison")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/unnamed-chunk-7-1.png){width=672}
:::
:::


As seen above the ranked set estimator has much lower variability, while 
maintaining the same mean estimate as compared to the simple random sample.

4. Estimate the proportions of people in CA with normal weight, overweight and 
obesity using the BRFSS 2007 data (X_STATE = 6, X_BMI4CAT = BMI). Post-stratify
the CA data to have the same age and sex distribution as the data for FL 
(X_STATE = 12) and compute the directly standardized estimates based on CA
data to estimates form the data for FL to see if the differences in BMI 
between the states are explained by differences in age distribution.


::: {.cell}

```{.r .cell-code}
db <- DBI::dbConnect(RSQLite::SQLite(), "Data/BRFSS/brfss07.db")
brfss_ca <- tbl(db, sql("SELECT * FROM brfss")) %>%
  filter(X_STATE == 6) %>%
  collect() %>%
  mutate(
    BMI_Cat = case_when(
      X_BMI4CAT == 1 ~ "Normal BMI",
      X_BMI4CAT == 2 ~ "Overweight BMI",
      X_BMI4CAT == 3 ~ "Obese BMI",
      TRUE ~ NA
    )
  ) %>%
  as_survey_design(
    id = X_PSU, strata = X_STATE, weight = X_FINALWT,
    nest = TRUE
  )

brfss_ca %>%
  group_by(BMI_Cat) %>%
  summarize(
    proportion = survey_mean()
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 4 × 3
  BMI_Cat        proportion proportion_se
  <chr>               <dbl>         <dbl>
1 Normal BMI         0.392        0.00889
2 Obese BMI          0.223        0.00769
3 Overweight BMI     0.341        0.00856
4 <NA>               0.0446       0.00377
```
:::
:::

::: {.cell}

```{.r .cell-code}
brfss_fl <- tbl(db, sql("SELECT * FROM brfss WHERE X_STATE == 12")) %>%
  collect() %>%
  mutate(
    BMI_Cat = case_when(
      X_BMI4CAT == 1 ~ "Normal BMI",
      X_BMI4CAT == 2 ~ "Overweight BMI",
      X_BMI4CAT == 3 ~ "Obese BMI",
      TRUE ~ NA
    )
  ) %>%
  as_survey_design(
    id = X_PSU, strata = X_STATE, weight = X_FINALWT, nest = TRUE
  )

fl_sex_age_totals <- brfss_fl %>%
  group_by(SEX, X_AGE_G) %>%
  survey_count() %>%
  select(-n_se) %>%
  rename(Freq = n)

postStratify(
  design = brfss_ca, strata = ~ SEX + X_AGE_G,
  population = fl_sex_age_totals
) %>%
  svymean(~BMI_Cat, design = ., na.rm = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
                         mean     SE
BMI_CatNormal BMI     0.40323 0.0083
BMI_CatObese BMI      0.23407 0.0075
BMI_CatOverweight BMI 0.36269 0.0083
```
:::
:::

::: {.cell}

```{.r .cell-code}
DBI::dbDisconnect(db)
brfss_fl %>%
  group_by(BMI_Cat) %>%
  summarize(
    proportion = survey_mean()
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 4 × 3
  BMI_Cat        proportion proportion_se
  <chr>               <dbl>         <dbl>
1 Normal BMI         0.360        0.00585
2 Obese BMI          0.229        0.00505
3 Overweight BMI     0.361        0.00593
4 <NA>               0.0508       0.00300
```
:::
:::


In general we see that the two estimates - the post stratified estimate and the
"direct" estimates largely agree, for the proportion of Obese and Overweight
BMI estimates. The normal BMI Estimates appear to be substantially different 
which may be due to factors that aren't unaccounted for in the age and sex 
strata.

5. Consider a categorical post-stratification variable with $K$ categories
having as population counts $N_1, N_2,...N_K$. Suppose we are interested in 
estimating the total of a variable $Y$.

* a. Show that the post-stratified estimate is 
$$
\hat{T}_{ps} = \sum_{k=1}^{K} N_k \hat{\mu}_k,
$$

where $\hat{\mu}_k$ is the estimated mean of $Y$ in group $K$ before 
post-stratification.

We'll start with using the result derived in the text that shows when we 
post-stratify we scale the weights, $\pi_i^* = \frac{g_i}{\pi_i}$, such that
the estimated strata size is exactly equal to the known (post) strata size. 
This results in a $g_i = \frac{N_k}{\hat{N_k}}$ scaling weight.

For our post-stratified total estimate then we have
$$
\hat{T}_{ps} = \sum_{i=1}^{n} y_iI(y_i \in S_k) \frac{N_k}{\hat{N}_k} \\
= \sum_{k=1}^{K} \hat{\mu}_k N_k
$$
Where the mean estimate, $\hat{\mu}_k$ comes from dividing the $y_i$ in each 
$k$ group by the $\hat{N}_k$ denominator.


* b. Show that the regression estimate from a model with indicator variables
for each group is also 

$$
\hat{T}_{reg} = \sum_{k=1}^{K} N_k \hat{\mu}_k
$$

This follows in a straightforward fashion from a regression model set-up. If we
fit the following model:
$$
E[Y_i] = \beta_k I(y_i \in S_k)
$$
Then it follows that 
$$
\hat{\beta}_k = \hat{\mu}_k = \frac{1}{n_k} \sum_{i=1}^{n}y_iI(y_i \in S_k), k = 1,...,K
$$
Predicting a new total then with the new counts re-weighted amounts to a 
regression prediction 
$\hat{T}_{ps} = \sum_{k=1}^{K} N_k \hat{\beta}_k = \sum_{k=1}^{K} N_k \hat{\beta}_k$

# Chapter 8: Two-Phase Sampling

## MultiStage and Multiphase Sampling

In comparison to multistage sampling, where individuals or clusters were sampled
independently of one another across stages, multiphase sampling is a design
where individuals are sampled dependent upon information obtained in the first
sample. Consequently, instead of having two sampling probabilities which are
multiplied by each other $\pi_1 \times \pi_2$ we have conditional weights
$\pi_1$ and $\pi_{2|1}$ which describe the probability of an entity being
sampled in phase one and the the probability of a phase being sampled in stage
two, conditional on being sampled in phase one. Multiplying and differencing
these two probabilities together can be used to construct an estimator very
similar to the horvitz-thompson estimator, though is theoretically distinct.
Lumley notes here that his software and exposition only covers the simple cases 
of both types of sampling designs described here can be expanded to cover 
multiple phases and combinations of both types of designs.

## Sampling for Stratification

One common motivation for two-phase sampling is to measure some strata variable
on a sample of the populations. The first phase sample is a random sample of 
the general population on which the strata variable is then measured. The second
phase of sampling then stratifies on this variable for greater precision.

Lumley gives several examples of this setup including the NHANES and NHIS 
surveys. Though the data associated with these are not made publicly available,
unfortunately.


## The Case-Control Design

> A type of sampling for stratification is the case-control design
(or choice-based design in economics).

This design is used when the measure of interest is particularly sparse or rare
in the population. The basic setup is the same as described in the previous 
section where the rare disease is measured in the first phase sample from a 
hopefully well specified population. The second phase samples all individuals
with the disease (cases) and some proportion of the controls. Typically matching
$k$ controls to each case.

A few brief notes Lumley makes here:

1. If done properly, the design effect is very large for this design; this 
design is far more efficient than a simple random sample proportional to the 
sparsity of the disease.
2. Two competing interests in selecting controls that leads to criticism of this
design --- they have to be representative of the population, but also good
"matches" for the cases. They need to be comparable in some way.
3. Design based inference was not as frequently used for case control designs.
  -- Because the odds ratio estimate is independent of the sampling fraction and
the estimated standard errors are typically less, a model based approach is
often used. However, this requires the model to be specified correctly 
which is not always the case. 
  -- Lumley notes that there seems to be less hesitancy about using either
  design based approach or a model based approach as there once was. 

### Oesophageal cancer in Ille-et-Vilaine


Lumley re analyzes data published as part of a previous study 
[@breslow1980statistical] [@aj1977cancer] looking at esophageal cancer in 
northwest France as impacted by of alcohol and tobacco consumption.

Lumley gets a control sampling weight of 441 by digging around in the related
papers and uses this to expand the original dataset, which contains the 
aggregate numbers to construct a survey design object below.

::: {.cell}

```{.r .cell-code}
cases <- cbind(esoph[rep(1:88, esoph$ncases), ], case = 1, weight = 1)
controls <- cbind(esoph[rep(1:88, esoph$ncontrols), ], case = 0, weight = 441)
esoph.x <- rbind(cases, controls)
d_esoph <- svydesign(
  id = ~1, strata = ~case, weights = ~weight,
  data = esoph.x
)
unwtd <- glm(case ~ agegp + as.numeric(tobgp) + as.numeric(alcgp),
  data = esoph.x, family = binomial()
)
wtd <- svyglm(case ~ agegp + as.numeric(tobgp) + as.numeric(alcgp),
  design = d_esoph, family = quasibinomial
)
coef(unwtd)[7:8]
```

::: {.cell-output .cell-output-stdout}
```
as.numeric(tobgp) as.numeric(alcgp) 
        0.4395543         1.0676597 
```
:::

```{.r .cell-code}
coef(wtd)[7:8]
```

::: {.cell-output .cell-output-stdout}
```
as.numeric(tobgp) as.numeric(alcgp) 
        0.4453378         1.0448879 
```
:::

```{.r .cell-code}
SE(unwtd)[7:8]
```

::: {.cell-output .cell-output-stdout}
```
as.numeric(tobgp) as.numeric(alcgp) 
       0.09623424        0.10492518 
```
:::

```{.r .cell-code}
SE(wtd)[7:8]
```

::: {.cell-output .cell-output-stdout}
```
as.numeric(tobgp) as.numeric(alcgp) 
        0.1221853         0.1236996 
```
:::
:::

::: {.cell}

```{.r .cell-code}
tbl_merge(
  tbls = list(tbl_regression(unwtd), tbl_regression(wtd)),
  tab_spanner = c("**Unweighted**", "**Weighted**")
)
```

::: {.cell-output-display}
```{=html}
<div id="sjfujptyot" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#sjfujptyot table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#sjfujptyot thead, #sjfujptyot tbody, #sjfujptyot tfoot, #sjfujptyot tr, #sjfujptyot td, #sjfujptyot th {
  border-style: none;
}

#sjfujptyot p {
  margin: 0;
  padding: 0;
}

#sjfujptyot .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#sjfujptyot .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#sjfujptyot .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#sjfujptyot .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#sjfujptyot .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#sjfujptyot .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#sjfujptyot .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#sjfujptyot .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#sjfujptyot .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#sjfujptyot .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#sjfujptyot .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#sjfujptyot .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#sjfujptyot .gt_spanner_row {
  border-bottom-style: hidden;
}

#sjfujptyot .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#sjfujptyot .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#sjfujptyot .gt_from_md > :first-child {
  margin-top: 0;
}

#sjfujptyot .gt_from_md > :last-child {
  margin-bottom: 0;
}

#sjfujptyot .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#sjfujptyot .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#sjfujptyot .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#sjfujptyot .gt_row_group_first td {
  border-top-width: 2px;
}

#sjfujptyot .gt_row_group_first th {
  border-top-width: 2px;
}

#sjfujptyot .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#sjfujptyot .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#sjfujptyot .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#sjfujptyot .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#sjfujptyot .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#sjfujptyot .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#sjfujptyot .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#sjfujptyot .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#sjfujptyot .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#sjfujptyot .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#sjfujptyot .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#sjfujptyot .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#sjfujptyot .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#sjfujptyot .gt_left {
  text-align: left;
}

#sjfujptyot .gt_center {
  text-align: center;
}

#sjfujptyot .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#sjfujptyot .gt_font_normal {
  font-weight: normal;
}

#sjfujptyot .gt_font_bold {
  font-weight: bold;
}

#sjfujptyot .gt_font_italic {
  font-style: italic;
}

#sjfujptyot .gt_super {
  font-size: 65%;
}

#sjfujptyot .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#sjfujptyot .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#sjfujptyot .gt_indent_1 {
  text-indent: 5px;
}

#sjfujptyot .gt_indent_2 {
  text-indent: 10px;
}

#sjfujptyot .gt_indent_3 {
  text-indent: 15px;
}

#sjfujptyot .gt_indent_4 {
  text-indent: 20px;
}

#sjfujptyot .gt_indent_5 {
  text-indent: 25px;
}

#sjfujptyot .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#sjfujptyot div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipVbndlaWdodGVkKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Unweighted&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipVbndlaWdodGVkKio="><div class='gt_from_md'><p><strong>Unweighted</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipXZWlnaHRlZCoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Weighted&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipXZWlnaHRlZCoq"><div class='gt_from_md'><p><strong>Weighted</strong></p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">agegp</td>
<td headers="estimate_1" class="gt_row gt_center"><br /></td>
<td headers="conf.low_1" class="gt_row gt_center"><br /></td>
<td headers="p.value_1" class="gt_row gt_center"><br /></td>
<td headers="estimate_2" class="gt_row gt_center"><br /></td>
<td headers="conf.low_2" class="gt_row gt_center"><br /></td>
<td headers="p.value_2" class="gt_row gt_center"><br /></td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp.L</td>
<td headers="estimate_1" class="gt_row gt_center">3.8</td>
<td headers="conf.low_1" class="gt_row gt_center">2.7, 5.6</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">3.9</td>
<td headers="conf.low_2" class="gt_row gt_center">2.5, 5.2</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp.Q</td>
<td headers="estimate_1" class="gt_row gt_center">-1.5</td>
<td headers="conf.low_1" class="gt_row gt_center">-3.1, -0.51</td>
<td headers="p.value_1" class="gt_row gt_center">0.014</td>
<td headers="estimate_2" class="gt_row gt_center">-1.4</td>
<td headers="conf.low_2" class="gt_row gt_center">-2.6, -0.16</td>
<td headers="p.value_2" class="gt_row gt_center">0.026</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp.C</td>
<td headers="estimate_1" class="gt_row gt_center">0.08</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.73, 1.2</td>
<td headers="p.value_1" class="gt_row gt_center">0.9</td>
<td headers="estimate_2" class="gt_row gt_center">0.29</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.63, 1.2</td>
<td headers="p.value_2" class="gt_row gt_center">0.5</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp^4</td>
<td headers="estimate_1" class="gt_row gt_center">0.12</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.58, 0.74</td>
<td headers="p.value_1" class="gt_row gt_center">0.7</td>
<td headers="estimate_2" class="gt_row gt_center">0.25</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.40, 0.90</td>
<td headers="p.value_2" class="gt_row gt_center">0.4</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp^5</td>
<td headers="estimate_1" class="gt_row gt_center">-0.25</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.67, 0.17</td>
<td headers="p.value_1" class="gt_row gt_center">0.2</td>
<td headers="estimate_2" class="gt_row gt_center">-0.28</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.73, 0.17</td>
<td headers="p.value_2" class="gt_row gt_center">0.2</td></tr>
    <tr><td headers="label" class="gt_row gt_left">as.numeric(tobgp)</td>
<td headers="estimate_1" class="gt_row gt_center">0.44</td>
<td headers="conf.low_1" class="gt_row gt_center">0.25, 0.63</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">0.45</td>
<td headers="conf.low_2" class="gt_row gt_center">0.21, 0.69</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">as.numeric(alcgp)</td>
<td headers="estimate_1" class="gt_row gt_center">1.1</td>
<td headers="conf.low_1" class="gt_row gt_center">0.87, 1.3</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">1.0</td>
<td headers="conf.low_2" class="gt_row gt_center">0.80, 1.3</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="7"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="T1IgPSBPZGRzIFJhdGlvLCBDSSA9IENvbmZpZGVuY2UgSW50ZXJ2YWw="><div class='gt_from_md'><p>OR = Odds Ratio, CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::



Regardless of which model is used, both show that tobacco and alcohol use 
increases the odds of esophageal cancer, though the un-weighted model does have
slightly smaller confidence intervals for the coefficients of interest,
as we can see in the table above.

### Simulations: efficiency of the design-based estimator

In this section Lumley computes the relative efficiency of the weighted 
estimator relative to the weighted comparing different distributions of 
cases and controls. Lumley defines the relative efficiency as the 
number of observations needed to produce the same accuracy but it isn't clear
how accuracy is measured here (length of standard error?). Since he doesn't
show how he computes the resulting table, I've omitted his example from my 
notes.



### Frequency matching

> Many case-control designs use a further level of stratification and unequal 
sampling in the phase-two sample, a practice known in the epidemiology 
literature as *frequency matching*.  

The idea is to avoid wasting cases / controls in regions of low exposure. 
Lumley uses the example of the esophageal cancer study to illustrate. Bracket
text below is mine:

> ... very little information about the effects of alcohol and tobacco is 
present in the youngest age group, because there is only one case. If the 
associations with age had already been understood and the study had been 
designed to estimate the effect of alcohol and tobacco [only] this would be an
inefficient design that effectively wasted 116 controls. A more efficient
design would saple more controls at older ages and end up with five controls 
per case in each age group rather than five controls per case on average over 
all ages.

Lumley goes on to say that frequency matching isn't the most efficient way to
use the age information, but he doesn't say what would be. I'm guessing he's
thinking of further stratification or unequal sampling but I can't be sure. 


## Sampling from Existing Cohorts

Lumley notes in this section that it is often advantageous in larger
randomized control trials or cohort studies to sub-sample once certain 
measurements have been taken. Lumley gives an account of methods used on these
data --- previously nested case control designs, ignoring the phase one sample 
and cohort representativeness but more recently stratifying on both outcome and 
covariates in phase two and post-stratifying to the original cohort. 
The trade-offs between model and design based approaches are/were not clear
at the date of publication in this setting. My take is that the design based
approach seems better given how many design elements are involved in the
construction of the study.

### Logistic regression

|         | Exposed | Unexposed |
|---------|---------|-----------|
| Case    | a       | b         |
| Control | c       | d         |
| Total   | a+c     | b + d     |

Lumley's goal in this section is to show how case control designs can be 
optimised to reduce the variance associated with the odds ratio estimate, 
typically estimated via logistic regression. Starting from a 2 x 2 contigency 
table like that shown above the variance of the log of the  odds ratio can be
estimated as follows:

$$
V[\log \psi] = \frac{1}{a} + \frac{1}{b} + \frac{1}{c} + \frac{1}{d},
$$

where $\psi$ represents the odds of the disease in the exposed cases vs the
odds of disease in the exposed controls. From the expression above, we see that
if we increase any of the cell counts, we decrease the variance. However,
a classic case control design will only control the table row margins, meaning
there might be greater improvement in controlling the sampling from the 
exposure covariate(s) as well. 

However, in that setting the classic logistic regression model no longer gives
a valid analysis because the odds ratio will be impacted by the unequal 
sampling fractions amongst the exposure covariate. In that case, a design based
approach is necessary. 

Lumley uses the National Wilms Tumor Study group to illustrate this idea. 
Kidney histology was available for all members of the study. Although the 
initial histology had an appreciable error rate, it could still be used to 
sample from. Though Lumley does not give an exact procedure for how he does this
he shows a table of of biased sampling that includes 183 controls with 
unfavorable histology ratings, a higher number than what would be included with
random sampling matched case-control procedure.


### Two-phase case-control designs in R

Lumley illustrates how to fit a two-phase sample estimate using the national
wilms tumor study data. The code and model output are below.


::: {.cell}

```{.r .cell-code}
nwts <- addhazard::nwtsco
set.seed(1337)
subsample <- with(nwts, c(
  which(relaps == 1 | instit == 1),
  sample(which(relaps == 0 & instit == 0), 499)
))
nwts$in.subsample <- (1:nrow(nwts)) %in% subsample
nwts_design <- twophase(
  id = list(~1, ~1), subset = ~in.subsample,
  strata = list(NULL, ~ interaction(instit, relaps)),
  data = nwts
)
nwts_design
```

::: {.cell-output .cell-output-stdout}
```
Two-phase sparse-matrix design:
 twophase2(id = id, strata = strata, probs = probs, fpc = fpc, 
    subset = subset, data = data, pps = pps)
Phase 1:
Independent Sampling design (with replacement)
svydesign(ids = ~1)
Phase 2:
Stratified Independent Sampling design
svydesign(ids = ~1, strata = ~interaction(instit, relaps), fpc = `*phase1*`)
```
:::
:::

::: {.cell}

```{.r .cell-code}
set.seed(1337)
casectrl <- with(nwts, c(which(relaps == 1), sample(which(relaps == 0), 699)))
nwts$in.ccs <- (1:nrow(nwts)) %in% casectrl
ccs_design <- twophase(
  id = list(~1, ~1), subset = ~in.ccs,
  strata = list(NULL, ~relaps), data = nwts
)
m1 <- svyglm(relaps ~ histol * stage + age + tumdiam,
  design = nwts_design,
  family = quasibinomial()
)
m2 <- svyglm(relaps ~ histol * stage + age + tumdiam,
  design = ccs_design,
  family = quasibinomial()
)
m3 <- glm(relaps ~ histol * stage + age + tumdiam,
  data = nwts, subset = in.ccs,
  family = binomial()
)
m1a <- svyglm(relaps ~ histol * stage + age + tumdiam,
  design = nwts_design,
  family = quasipoisson(log)
)
m2a <- svyglm(relaps ~ histol * stage + age + tumdiam,
  design = ccs_design,
  family = quasipoisson(log)
)
```
:::

::: {.cell}

```{.r .cell-code}
tbl_merge(
  tbls = list(
    tbl_regression(m1, conf.int = FALSE),
    tbl_regression(m2, conf.int = FALSE),
    tbl_regression(m3, conf.int = FALSE),
    tbl_regression(m1a, conf.int = FALSE),
    tbl_regression(m2a, conf.int = FALSE)
  ),
  tab_spanner = c(
    "Two Phase Design based Odds Ratio",
    "CC Design Based Odds Ratio",
    "Model Based Odds Ratio",
    "Two Phase Design Based Relative Risk",
    "CC Design Based Relative Risk"
  )
)
```

::: {.cell-output-display}
```{=html}
<div id="rahtglogfb" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#rahtglogfb table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#rahtglogfb thead, #rahtglogfb tbody, #rahtglogfb tfoot, #rahtglogfb tr, #rahtglogfb td, #rahtglogfb th {
  border-style: none;
}

#rahtglogfb p {
  margin: 0;
  padding: 0;
}

#rahtglogfb .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#rahtglogfb .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#rahtglogfb .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#rahtglogfb .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#rahtglogfb .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#rahtglogfb .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#rahtglogfb .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#rahtglogfb .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#rahtglogfb .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#rahtglogfb .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#rahtglogfb .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#rahtglogfb .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#rahtglogfb .gt_spanner_row {
  border-bottom-style: hidden;
}

#rahtglogfb .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#rahtglogfb .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#rahtglogfb .gt_from_md > :first-child {
  margin-top: 0;
}

#rahtglogfb .gt_from_md > :last-child {
  margin-bottom: 0;
}

#rahtglogfb .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#rahtglogfb .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#rahtglogfb .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#rahtglogfb .gt_row_group_first td {
  border-top-width: 2px;
}

#rahtglogfb .gt_row_group_first th {
  border-top-width: 2px;
}

#rahtglogfb .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#rahtglogfb .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#rahtglogfb .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#rahtglogfb .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#rahtglogfb .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#rahtglogfb .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#rahtglogfb .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#rahtglogfb .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#rahtglogfb .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#rahtglogfb .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#rahtglogfb .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#rahtglogfb .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#rahtglogfb .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#rahtglogfb .gt_left {
  text-align: left;
}

#rahtglogfb .gt_center {
  text-align: center;
}

#rahtglogfb .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#rahtglogfb .gt_font_normal {
  font-weight: normal;
}

#rahtglogfb .gt_font_bold {
  font-weight: bold;
}

#rahtglogfb .gt_font_italic {
  font-style: italic;
}

#rahtglogfb .gt_super {
  font-size: 65%;
}

#rahtglogfb .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#rahtglogfb .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#rahtglogfb .gt_indent_1 {
  text-indent: 5px;
}

#rahtglogfb .gt_indent_2 {
  text-indent: 10px;
}

#rahtglogfb .gt_indent_3 {
  text-indent: 15px;
}

#rahtglogfb .gt_indent_4 {
  text-indent: 20px;
}

#rahtglogfb .gt_indent_5 {
  text-indent: 25px;
}

#rahtglogfb .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#rahtglogfb div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;VHdvIFBoYXNlIERlc2lnbiBiYXNlZCBPZGRzIFJhdGlv&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;Two Phase Design based Odds Ratio&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="VHdvIFBoYXNlIERlc2lnbiBiYXNlZCBPZGRzIFJhdGlv"><div class='gt_from_md'><p>Two Phase Design based Odds Ratio</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;Q0MgRGVzaWduIEJhc2VkIE9kZHMgUmF0aW8=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;CC Design Based Odds Ratio&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="Q0MgRGVzaWduIEJhc2VkIE9kZHMgUmF0aW8="><div class='gt_from_md'><p>CC Design Based Odds Ratio</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;TW9kZWwgQmFzZWQgT2RkcyBSYXRpbw==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;Model Based Odds Ratio&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="TW9kZWwgQmFzZWQgT2RkcyBSYXRpbw=="><div class='gt_from_md'><p>Model Based Odds Ratio</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;VHdvIFBoYXNlIERlc2lnbiBCYXNlZCBSZWxhdGl2ZSBSaXNr&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;Two Phase Design Based Relative Risk&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="VHdvIFBoYXNlIERlc2lnbiBCYXNlZCBSZWxhdGl2ZSBSaXNr"><div class='gt_from_md'><p>Two Phase Design Based Relative Risk</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;Q0MgRGVzaWduIEJhc2VkIFJlbGF0aXZlIFJpc2s=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;CC Design Based Relative Risk&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="Q0MgRGVzaWduIEJhc2VkIFJlbGF0aXZlIFJpc2s="><div class='gt_from_md'><p>CC Design Based Relative Risk</p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSVJSKSoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(IRR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSVJSKSoq"><div class='gt_from_md'><p><strong>log(IRR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSVJSKSoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(IRR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSVJSKSoq"><div class='gt_from_md'><p><strong>log(IRR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">histol</td>
<td headers="estimate_1" class="gt_row gt_center">0.36</td>
<td headers="p.value_1" class="gt_row gt_center">0.3</td>
<td headers="estimate_2" class="gt_row gt_center">-0.10</td>
<td headers="p.value_2" class="gt_row gt_center">0.8</td>
<td headers="estimate_3" class="gt_row gt_center">0.09</td>
<td headers="p.value_3" class="gt_row gt_center">0.8</td>
<td headers="estimate_4" class="gt_row gt_center">0.65</td>
<td headers="p.value_4" class="gt_row gt_center">0.009</td>
<td headers="estimate_5" class="gt_row gt_center">0.37</td>
<td headers="p.value_5" class="gt_row gt_center">0.2</td></tr>
    <tr><td headers="label" class="gt_row gt_left">stage</td>
<td headers="estimate_1" class="gt_row gt_center">0.26</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">0.18</td>
<td headers="p.value_2" class="gt_row gt_center">0.004</td>
<td headers="estimate_3" class="gt_row gt_center">0.18</td>
<td headers="p.value_3" class="gt_row gt_center">0.003</td>
<td headers="estimate_4" class="gt_row gt_center">0.23</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td>
<td headers="estimate_5" class="gt_row gt_center">0.16</td>
<td headers="p.value_5" class="gt_row gt_center">0.002</td></tr>
    <tr><td headers="label" class="gt_row gt_left">age</td>
<td headers="estimate_1" class="gt_row gt_center">0.05</td>
<td headers="p.value_1" class="gt_row gt_center">0.032</td>
<td headers="estimate_2" class="gt_row gt_center">0.10</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">0.10</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center">0.04</td>
<td headers="p.value_4" class="gt_row gt_center">0.028</td>
<td headers="estimate_5" class="gt_row gt_center">0.07</td>
<td headers="p.value_5" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">tumdiam</td>
<td headers="estimate_1" class="gt_row gt_center">0.01</td>
<td headers="p.value_1" class="gt_row gt_center">0.6</td>
<td headers="estimate_2" class="gt_row gt_center">0.00</td>
<td headers="p.value_2" class="gt_row gt_center">>0.9</td>
<td headers="estimate_3" class="gt_row gt_center">0.00</td>
<td headers="p.value_3" class="gt_row gt_center">0.8</td>
<td headers="estimate_4" class="gt_row gt_center">0.01</td>
<td headers="p.value_4" class="gt_row gt_center">0.6</td>
<td headers="estimate_5" class="gt_row gt_center">0.00</td>
<td headers="p.value_5" class="gt_row gt_center">>0.9</td></tr>
    <tr><td headers="label" class="gt_row gt_left">histol * stage</td>
<td headers="estimate_1" class="gt_row gt_center">0.50</td>
<td headers="p.value_1" class="gt_row gt_center">0.001</td>
<td headers="estimate_2" class="gt_row gt_center">0.68</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">0.62</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center">0.17</td>
<td headers="p.value_4" class="gt_row gt_center">0.048</td>
<td headers="estimate_5" class="gt_row gt_center">0.28</td>
<td headers="p.value_5" class="gt_row gt_center">0.004</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="11"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="T1IgPSBPZGRzIFJhdGlvLCBJUlIgPSBJbmNpZGVuY2UgUmF0ZSBSYXRpbw=="><div class='gt_from_md'><p>OR = Odds Ratio, IRR = Incidence Rate Ratio</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


Few quick things to note:

1. The estimates between the relative risk rate ratio and odds ratio vary 
substantially, as we'd expect.
2. There is an appreciable precision gain in the `histol` x `stage` estimates 
comparing the design based to the model based, with a slight gain from using 
the `instit` for sampling in the more general two-phased sampling model.
3. Lumley notes that model based analysis are also available for two-phased 
samples. In my training we learned how to use a conditional logistic regression
in this setting. You can see a similar example 
[here](https://www.bookdown.org/rwnahhas/RMPH/blr-conditional.html).


### Survival Analysis

In this section Lumley briefly introduces the concept of the case-cohort study,
as well as the topic of survival analysis. Notes on each below.


**Case-Cohort Study**

* A two phase sample in which the the second phase sample is a sub-cohort chosen
at the beginning in addition to all cases identified during follow-up.
* Lumley notes that this initial sub-cohort can be used for comparing against
multiple types of measured "cases" or events, in contrast to a case-control 
design which is constrained to analysis of the single type of case.


**Survival Analysis**

* Survival analysis is a large enough topic to be given its own book but here
Lumley focuses on the Cox proportional hazards model, which estimates
the hazard function, or instantaneous rate of an event (like cancer relapse) 
occuring as a function of some covariates.

* Notably, survival analysis incorporates assumptions for censoring, or the 
unobservance of the event occuring within the follow-up period.

#### Case Cohort Designs

Lumley uses the Wilms Tumor study again, to illustrate how a case-cohort 
analysis can be used to estimate patients survival from Wilms tumor as a 
function of the measured covariates. Here's the setup in Lumley's own words:

> For the classical case-cohort analysis we take a sample from the cohort at the
start of the followup and then add all the cases to it. If the expected event
rate is about 1 in 7, giving about 650 expected cases, and we want 650 non-cases,
this means sampling a subcohort of 650 x 7/6 or about 750... Under this 
sampling design there are two sampled strata: those in the subcohort and those
not in the subcohort. The sampling probabilities are $\pi_{(2|1)} = 750 / 3915$
for the subcohort and $\pi_{(2|1)} = 1$ for cases not in the subcohort.

The code is below.


::: {.cell}

```{.r .cell-code}
set.seed(1729)
subcohort <- with(nwts, sample(1:nrow(nwts), 750))
cases <- which(nwts$relaps == 1)
nwts$in.cchsample <- (1:nrow(nwts)) %in% c(subcohort, cases)
nwts$in.subcohort <- (1:nrow(nwts)) %in% subcohort
nwts$wts <- ifelse(nwts$in.subcohort, 3915 / 750, 1)
cch_design <- twophase(
  id = list(~1, ~1), subset = ~in.cchsample,
  strata = list(NULL, ~in.subcohort),
  weights = list(NULL, ~wts),
  method = "approx",
  data = nwts
)

s1 <- svycoxph(Surv(trel, relaps) ~ histol * stage + age + tumdiam,
  design = cch_design
)

cch_data <- subset(nwts, in.cchsample)
cch_data$id <- 1:nrow(cch_data)
# "classic" analysis based only on phase 2 data
s2 <- cch(Surv(trel, relaps) ~ histol * stage + age + tumdiam,
  id = ~id,
  data = cch_data, subcoh = ~in.subcohort, cohort.size = 3915
)
tbl_merge(
  tbls = list(tbl_regression(s1), tbl_regression(s2)),
  tab_spanner = c(
    "**Case-Cohort Analysis**",
    "**Classic Phase 2 Analysis**"
  )
)
```

::: {.cell-output .cell-output-stdout}
```
Two-phase design: twophase(id = list(~1, ~1), subset = ~in.cchsample, strata = list(NULL, 
    ~in.subcohort), weights = list(NULL, ~wts), method = "approx", 
    data = nwts)
Phase 1:
Independent Sampling design (with replacement)
svydesign(ids = ~1)
Phase 2:
Stratified Independent Sampling design
svydesign(ids = ~1, strata = ~in.subcohort, weights = ~wts, fpc = `*phase1*`)
```
:::

::: {.cell-output-display}
```{=html}
<div id="sgrlwxrbjq" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#sgrlwxrbjq table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#sgrlwxrbjq thead, #sgrlwxrbjq tbody, #sgrlwxrbjq tfoot, #sgrlwxrbjq tr, #sgrlwxrbjq td, #sgrlwxrbjq th {
  border-style: none;
}

#sgrlwxrbjq p {
  margin: 0;
  padding: 0;
}

#sgrlwxrbjq .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#sgrlwxrbjq .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#sgrlwxrbjq .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#sgrlwxrbjq .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#sgrlwxrbjq .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#sgrlwxrbjq .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#sgrlwxrbjq .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#sgrlwxrbjq .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#sgrlwxrbjq .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#sgrlwxrbjq .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#sgrlwxrbjq .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#sgrlwxrbjq .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#sgrlwxrbjq .gt_spanner_row {
  border-bottom-style: hidden;
}

#sgrlwxrbjq .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#sgrlwxrbjq .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#sgrlwxrbjq .gt_from_md > :first-child {
  margin-top: 0;
}

#sgrlwxrbjq .gt_from_md > :last-child {
  margin-bottom: 0;
}

#sgrlwxrbjq .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#sgrlwxrbjq .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#sgrlwxrbjq .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#sgrlwxrbjq .gt_row_group_first td {
  border-top-width: 2px;
}

#sgrlwxrbjq .gt_row_group_first th {
  border-top-width: 2px;
}

#sgrlwxrbjq .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#sgrlwxrbjq .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#sgrlwxrbjq .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#sgrlwxrbjq .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#sgrlwxrbjq .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#sgrlwxrbjq .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#sgrlwxrbjq .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#sgrlwxrbjq .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#sgrlwxrbjq .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#sgrlwxrbjq .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#sgrlwxrbjq .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#sgrlwxrbjq .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#sgrlwxrbjq .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#sgrlwxrbjq .gt_left {
  text-align: left;
}

#sgrlwxrbjq .gt_center {
  text-align: center;
}

#sgrlwxrbjq .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#sgrlwxrbjq .gt_font_normal {
  font-weight: normal;
}

#sgrlwxrbjq .gt_font_bold {
  font-weight: bold;
}

#sgrlwxrbjq .gt_font_italic {
  font-style: italic;
}

#sgrlwxrbjq .gt_super {
  font-size: 65%;
}

#sgrlwxrbjq .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#sgrlwxrbjq .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#sgrlwxrbjq .gt_indent_1 {
  text-indent: 5px;
}

#sgrlwxrbjq .gt_indent_2 {
  text-indent: 10px;
}

#sgrlwxrbjq .gt_indent_3 {
  text-indent: 15px;
}

#sgrlwxrbjq .gt_indent_4 {
  text-indent: 20px;
}

#sgrlwxrbjq .gt_indent_5 {
  text-indent: 25px;
}

#sgrlwxrbjq .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#sgrlwxrbjq div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipDYXNlLUNvaG9ydCBBbmFseXNpcyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Case-Cohort Analysis&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipDYXNlLUNvaG9ydCBBbmFseXNpcyoq"><div class='gt_from_md'><p><strong>Case-Cohort Analysis</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipDbGFzc2ljIFBoYXNlIDIgQW5hbHlzaXMqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Classic Phase 2 Analysis&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipDbGFzc2ljIFBoYXNlIDIgQW5hbHlzaXMqKg=="><div class='gt_from_md'><p><strong>Classic Phase 2 Analysis</strong></p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">histol</td>
<td headers="estimate_1" class="gt_row gt_center">0.75</td>
<td headers="conf.low_1" class="gt_row gt_center">0.06, 1.4</td>
<td headers="p.value_1" class="gt_row gt_center">0.033</td>
<td headers="estimate_2" class="gt_row gt_center">0.96</td>
<td headers="conf.low_2" class="gt_row gt_center">0.21, 1.7</td>
<td headers="p.value_2" class="gt_row gt_center">0.012</td></tr>
    <tr><td headers="label" class="gt_row gt_left">stage</td>
<td headers="estimate_1" class="gt_row gt_center">0.14</td>
<td headers="conf.low_1" class="gt_row gt_center">0.01, 0.26</td>
<td headers="p.value_1" class="gt_row gt_center">0.029</td>
<td headers="estimate_2" class="gt_row gt_center">0.21</td>
<td headers="conf.low_2" class="gt_row gt_center">0.10, 0.33</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">age</td>
<td headers="estimate_1" class="gt_row gt_center">0.06</td>
<td headers="conf.low_1" class="gt_row gt_center">0.01, 0.11</td>
<td headers="p.value_1" class="gt_row gt_center">0.015</td>
<td headers="estimate_2" class="gt_row gt_center">0.06</td>
<td headers="conf.low_2" class="gt_row gt_center">0.01, 0.11</td>
<td headers="p.value_2" class="gt_row gt_center">0.013</td></tr>
    <tr><td headers="label" class="gt_row gt_left">tumdiam</td>
<td headers="estimate_1" class="gt_row gt_center">0.01</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.02, 0.04</td>
<td headers="p.value_1" class="gt_row gt_center">0.6</td>
<td headers="estimate_2" class="gt_row gt_center">0.01</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.03, 0.04</td>
<td headers="p.value_2" class="gt_row gt_center">0.7</td></tr>
    <tr><td headers="label" class="gt_row gt_left">histol * stage</td>
<td headers="estimate_1" class="gt_row gt_center">0.24</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.02, 0.49</td>
<td headers="p.value_1" class="gt_row gt_center">0.067</td>
<td headers="estimate_2" class="gt_row gt_center">0.12</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.16, 0.40</td>
<td headers="p.value_2" class="gt_row gt_center">0.4</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="7"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="SFIgPSBIYXphcmQgUmF0aW8sIENJID0gQ29uZmlkZW5jZSBJbnRlcnZhbA=="><div class='gt_from_md'><p>HR = Hazard Ratio, CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


Looking at the table above, we can see that there's a decent amount of 
disagreement between the two estimates. Lumley argues this is because of the 
poor fit of the cox model. An assumption that's necessary for the Cox model
to fit well is that the hazards functions are proportional across time. 
As we see in the plot below and in the text, the hazard estimate for age is
lower earlier in the trial than later on.


::: {.cell}

```{.r .cell-code}
plot(cox.zph(s1), var = "age")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/cox_ph_check_plot-1.png){width=672}
:::
:::


Lumley notes that the case-cohort analysis could just as easily be framed as
a stratified sample on cases in the second phase. The code below performs this
analysis --- again using both case and case + institution for the stratifying
variables. The last bit of code extracts the variance estimates that's from
each phase of sampling and computes how much of the variance comes from the
first phase as opposed to the second phase for each of the estimated 
coefficients.


::: {.cell}

```{.r .cell-code}
scch_design <- twophase(
  id = list(~1, ~1), subset = ~in.cchsample,
  strata = list(NULL, ~relaps), data = nwts
)

s1 <- svycoxph(Surv(trel, relaps) ~ histol * stage + age + tumdiam,
  design = scch_design
)
nwts_design <- twophase(
  id = list(~1, ~1), subset = ~in.subsample,
  strata = list(NULL, ~ interaction(instit, relaps)),
  data = nwts
)
s2 <- svycoxph(Surv(trel, relaps) ~ histol * stage + age + tumdiam,
  design = nwts_design
)


v1 <- vcov(s1)
v2 <- vcov(s2)

rbind(
  diag(attr(v1, "phases")$phase2 / attr(v1, "phases")$phase1),
  diag(attr(v2, "phases")$phase2 / attr(v2, "phases")$phase1)
)
```

::: {.cell-output .cell-output-stdout}
```
          [,1]      [,2]      [,3]      [,4]      [,5]
[1,] 1.4892354 0.7446175 1.3303922 1.1322496 1.7652980
[2,] 0.6304098 0.7120240 0.7681475 0.5199323 0.7651971
```
:::
:::

::: {.cell}

```{.r .cell-code}
s3 <- coxph(Surv(trel, relaps) ~ histol * stage + age + tumdiam,
  data = nwts
)
tbl_merge(
  tbls = list(
    tbl_regression(s1),
    tbl_regression(s2),
    tbl_regression(s3)
  ),
  c(
    "**Stratified Case Cohort**",
    "**Stratified Case Cohort - Institution**",
    "**Full Data**"
  )
)
```

::: {.cell-output .cell-output-stdout}
```
Two-phase sparse-matrix design:
 twophase2(id = id, strata = strata, probs = probs, fpc = fpc, 
    subset = subset, data = data, pps = pps)
Phase 1:
Independent Sampling design (with replacement)
svydesign(ids = ~1)
Phase 2:
Stratified Independent Sampling design
svydesign(ids = ~1, strata = ~relaps, fpc = `*phase1*`)
Two-phase sparse-matrix design:
 twophase2(id = id, strata = strata, probs = probs, fpc = fpc, 
    subset = subset, data = data, pps = pps)
Phase 1:
Independent Sampling design (with replacement)
svydesign(ids = ~1)
Phase 2:
Stratified Independent Sampling design
svydesign(ids = ~1, strata = ~interaction(instit, relaps), fpc = `*phase1*`)
```
:::

::: {.cell-output-display}
```{=html}
<div id="mptjsrhztr" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#mptjsrhztr table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#mptjsrhztr thead, #mptjsrhztr tbody, #mptjsrhztr tfoot, #mptjsrhztr tr, #mptjsrhztr td, #mptjsrhztr th {
  border-style: none;
}

#mptjsrhztr p {
  margin: 0;
  padding: 0;
}

#mptjsrhztr .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#mptjsrhztr .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#mptjsrhztr .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#mptjsrhztr .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#mptjsrhztr .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#mptjsrhztr .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#mptjsrhztr .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#mptjsrhztr .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#mptjsrhztr .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#mptjsrhztr .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#mptjsrhztr .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#mptjsrhztr .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#mptjsrhztr .gt_spanner_row {
  border-bottom-style: hidden;
}

#mptjsrhztr .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#mptjsrhztr .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#mptjsrhztr .gt_from_md > :first-child {
  margin-top: 0;
}

#mptjsrhztr .gt_from_md > :last-child {
  margin-bottom: 0;
}

#mptjsrhztr .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#mptjsrhztr .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#mptjsrhztr .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#mptjsrhztr .gt_row_group_first td {
  border-top-width: 2px;
}

#mptjsrhztr .gt_row_group_first th {
  border-top-width: 2px;
}

#mptjsrhztr .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#mptjsrhztr .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#mptjsrhztr .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#mptjsrhztr .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#mptjsrhztr .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#mptjsrhztr .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#mptjsrhztr .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#mptjsrhztr .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#mptjsrhztr .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#mptjsrhztr .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#mptjsrhztr .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#mptjsrhztr .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#mptjsrhztr .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#mptjsrhztr .gt_left {
  text-align: left;
}

#mptjsrhztr .gt_center {
  text-align: center;
}

#mptjsrhztr .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#mptjsrhztr .gt_font_normal {
  font-weight: normal;
}

#mptjsrhztr .gt_font_bold {
  font-weight: bold;
}

#mptjsrhztr .gt_font_italic {
  font-style: italic;
}

#mptjsrhztr .gt_super {
  font-size: 65%;
}

#mptjsrhztr .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#mptjsrhztr .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#mptjsrhztr .gt_indent_1 {
  text-indent: 5px;
}

#mptjsrhztr .gt_indent_2 {
  text-indent: 10px;
}

#mptjsrhztr .gt_indent_3 {
  text-indent: 15px;
}

#mptjsrhztr .gt_indent_4 {
  text-indent: 20px;
}

#mptjsrhztr .gt_indent_5 {
  text-indent: 25px;
}

#mptjsrhztr .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#mptjsrhztr div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipTdHJhdGlmaWVkIENhc2UgQ29ob3J0Kio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Stratified Case Cohort&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipTdHJhdGlmaWVkIENhc2UgQ29ob3J0Kio="><div class='gt_from_md'><p><strong>Stratified Case Cohort</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipTdHJhdGlmaWVkIENhc2UgQ29ob3J0IC0gSW5zdGl0dXRpb24qKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Stratified Case Cohort - Institution&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipTdHJhdGlmaWVkIENhc2UgQ29ob3J0IC0gSW5zdGl0dXRpb24qKg=="><div class='gt_from_md'><p><strong>Stratified Case Cohort - Institution</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipGdWxsIERhdGEqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Full Data&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipGdWxsIERhdGEqKg=="><div class='gt_from_md'><p><strong>Full Data</strong></p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">histol</td>
<td headers="estimate_1" class="gt_row gt_center">0.89</td>
<td headers="conf.low_1" class="gt_row gt_center">0.16, 1.6</td>
<td headers="p.value_1" class="gt_row gt_center">0.017</td>
<td headers="estimate_2" class="gt_row gt_center">0.48</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.11, 1.1</td>
<td headers="p.value_2" class="gt_row gt_center">0.11</td>
<td headers="estimate_3" class="gt_row gt_center">0.59</td>
<td headers="conf.low_3" class="gt_row gt_center">0.13, 1.1</td>
<td headers="p.value_3" class="gt_row gt_center">0.013</td></tr>
    <tr><td headers="label" class="gt_row gt_left">stage</td>
<td headers="estimate_1" class="gt_row gt_center">0.20</td>
<td headers="conf.low_1" class="gt_row gt_center">0.08, 0.31</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">0.26</td>
<td headers="conf.low_2" class="gt_row gt_center">0.14, 0.38</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">0.22</td>
<td headers="conf.low_3" class="gt_row gt_center">0.13, 0.31</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">age</td>
<td headers="estimate_1" class="gt_row gt_center">0.05</td>
<td headers="conf.low_1" class="gt_row gt_center">0.00, 0.10</td>
<td headers="p.value_1" class="gt_row gt_center">0.039</td>
<td headers="estimate_2" class="gt_row gt_center">0.03</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.01, 0.07</td>
<td headers="p.value_2" class="gt_row gt_center">0.2</td>
<td headers="estimate_3" class="gt_row gt_center">0.06</td>
<td headers="conf.low_3" class="gt_row gt_center">0.03, 0.08</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">tumdiam</td>
<td headers="estimate_1" class="gt_row gt_center">0.01</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.02, 0.04</td>
<td headers="p.value_1" class="gt_row gt_center">0.5</td>
<td headers="estimate_2" class="gt_row gt_center">0.01</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.02, 0.03</td>
<td headers="p.value_2" class="gt_row gt_center">0.6</td>
<td headers="estimate_3" class="gt_row gt_center">0.02</td>
<td headers="conf.low_3" class="gt_row gt_center">0.00, 0.04</td>
<td headers="p.value_3" class="gt_row gt_center">0.081</td></tr>
    <tr><td headers="label" class="gt_row gt_left">histol * stage</td>
<td headers="estimate_1" class="gt_row gt_center">0.19</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.09, 0.47</td>
<td headers="p.value_1" class="gt_row gt_center">0.2</td>
<td headers="estimate_2" class="gt_row gt_center">0.35</td>
<td headers="conf.low_2" class="gt_row gt_center">0.12, 0.57</td>
<td headers="p.value_2" class="gt_row gt_center">0.003</td>
<td headers="estimate_3" class="gt_row gt_center">0.31</td>
<td headers="conf.low_3" class="gt_row gt_center">0.14, 0.47</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="10"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="SFIgPSBIYXphcmQgUmF0aW8sIENJID0gQ29uZmlkZW5jZSBJbnRlcnZhbA=="><div class='gt_from_md'><p>HR = Hazard Ratio, CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


The above table contains the data from both of the stratified case-cohort 
analyses as well as a model fit to the full data. The case-institution 
stratified analysis is more efficient than only the case stratified analysis.
Both design based analyses agree with the full data analysis, though, of course,
the full data analysis has less variability. Interestingly though, it is not
that much better than the stratified case-cohort-institution estimates.

## Using Auxiliary Information from Phase 1

In this section Lumley reviews how information recorded on individuals during
the first phase can be used to improve the precision of estimates in the second
phase via calibration. Lumley differentiates the use of auxiliary information
here as compared to chapter 7's review of post-stratification, raking 
and calibration by noting the following 3 differences:

1. The Phase one data includes individual level auxiliary variables.
2. Re-weighting can be customized to a particular analysis.
3. There may be --- when sampling from a cohort --- a large number of auxiliary 
variables available.

With this set-up Lumley discusses how to construct more effective auxiliary 
variables using influence functions.

### Population calibration for regression models

Lumley motivates his discussion of influence functions with the api data, where
we can reference the full population data.

Lumley's description of influence functions are intentionally simple, 
describing them as 

> "The influence function for an estimate $\hat{\beta}$ describes how the 
estimate changes when observations are added to or removed from the data."

For linear, glms, and cox models, these are $\Delta\beta$ deletion diagnostics
that are often taught as part of the statistics curriculum covering the 
subjects.  For example, a simple linear regression has the following influence
function for the slope parameter:
$$
\mathcal{I}(x_i, y_i; \beta) = \frac{1}{\pi_i}\frac{1}{V[X]}(x_i - \bar{x})(y_i - \mu_i(\beta)),
$$

where $\mu_i(\beta)$ is the fitted value for the $i$th observation. Lumley
reasons that 

> If an auxiliary variable $Z$ is highly correlated with $Y$ it will have a low 
correlation with $\mathcal{I}$, because the multiplier $(x_i - \bar{x}) can be
negative or positive with about equal probability.

I don't quite see how Lumley is drawing the connection that
$Cor(Y,Z) \approx 1 \implies Cor(Z,\mathcal{I}) \approx 0$ unless there is some
way that $Z$'s impact on $Y$ is through $X$ --- epidemiologists would then say
"The effect of $Z$ on $Y$ is mediated by $X$. I don't that's what Lumley's 
referring to here...

The next part makes sense - Lumley says that in order to construct new 
variables for use in calibration, we can use the influence functions from
linear regressions constructed from $Z \tilde X$. Though I should say, it 
isn't clear to me *why* we might choose $Z$ as the regression variable here...

In any case, Lumley demonstrates this with the api data, using the full
population data to estimate the influence functions.



::: {.cell}

```{.r .cell-code}
m0 <- svyglm(api00 ~ ell + mobility + emer, clus1_design)
var_cal <- calibrate(clus1_design,
  formula = ~ api99 + ell + mobility + emer,
  pop = c(6194, 3914069, 141685, 106054, 70366),
  bounds = c(0.1, 10)
)
m1 <- svyglm(api00 ~ ell + mobility + emer, design = var_cal)


popmodel <- glm(api99 ~ ell + mobility + emer,
  data = apipop,
  na.action = na.exclude
)

inffun <- dfbeta(popmodel)
index <- match(apiclus1$snum, apipop$snum)
clus1if <- update(clus1_design,
  ifint = inffun[index, 1],
  ifell = inffun[index, 2], ifmobility = inffun[index, 3],
  ifemer = inffun[index, 4]
)
if_cal <- calibrate(clus1if,
  formula = ~ ifint + ifell + ifmobility + ifemer,
  pop = c(6194, 0, 0, 0, 0)
)

m2 <- svyglm(api00 ~ ell + mobility + emer, design = if_cal)
tbl_merge(
  tbls = list(
    tbl_regression(m0, intercept = TRUE),
    tbl_regression(m1, intercept = TRUE),
    tbl_regression(m2, intercept = TRUE)
  ),
  tab_spanner = c("Cluster Design", "Calibrated", "Influence Calibrated")
)
```

::: {.cell-output-display}
```{=html}
<div id="hgjzqhzohj" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#hgjzqhzohj table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#hgjzqhzohj thead, #hgjzqhzohj tbody, #hgjzqhzohj tfoot, #hgjzqhzohj tr, #hgjzqhzohj td, #hgjzqhzohj th {
  border-style: none;
}

#hgjzqhzohj p {
  margin: 0;
  padding: 0;
}

#hgjzqhzohj .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#hgjzqhzohj .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#hgjzqhzohj .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#hgjzqhzohj .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#hgjzqhzohj .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#hgjzqhzohj .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#hgjzqhzohj .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#hgjzqhzohj .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#hgjzqhzohj .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#hgjzqhzohj .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#hgjzqhzohj .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#hgjzqhzohj .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#hgjzqhzohj .gt_spanner_row {
  border-bottom-style: hidden;
}

#hgjzqhzohj .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#hgjzqhzohj .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#hgjzqhzohj .gt_from_md > :first-child {
  margin-top: 0;
}

#hgjzqhzohj .gt_from_md > :last-child {
  margin-bottom: 0;
}

#hgjzqhzohj .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#hgjzqhzohj .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#hgjzqhzohj .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#hgjzqhzohj .gt_row_group_first td {
  border-top-width: 2px;
}

#hgjzqhzohj .gt_row_group_first th {
  border-top-width: 2px;
}

#hgjzqhzohj .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#hgjzqhzohj .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#hgjzqhzohj .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#hgjzqhzohj .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#hgjzqhzohj .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#hgjzqhzohj .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#hgjzqhzohj .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#hgjzqhzohj .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#hgjzqhzohj .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#hgjzqhzohj .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#hgjzqhzohj .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#hgjzqhzohj .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#hgjzqhzohj .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#hgjzqhzohj .gt_left {
  text-align: left;
}

#hgjzqhzohj .gt_center {
  text-align: center;
}

#hgjzqhzohj .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#hgjzqhzohj .gt_font_normal {
  font-weight: normal;
}

#hgjzqhzohj .gt_font_bold {
  font-weight: bold;
}

#hgjzqhzohj .gt_font_italic {
  font-style: italic;
}

#hgjzqhzohj .gt_super {
  font-size: 65%;
}

#hgjzqhzohj .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#hgjzqhzohj .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#hgjzqhzohj .gt_indent_1 {
  text-indent: 5px;
}

#hgjzqhzohj .gt_indent_2 {
  text-indent: 10px;
}

#hgjzqhzohj .gt_indent_3 {
  text-indent: 15px;
}

#hgjzqhzohj .gt_indent_4 {
  text-indent: 20px;
}

#hgjzqhzohj .gt_indent_5 {
  text-indent: 25px;
}

#hgjzqhzohj .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#hgjzqhzohj div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;Q2x1c3RlciBEZXNpZ24=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;Cluster Design&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="Q2x1c3RlciBEZXNpZ24="><div class='gt_from_md'><p>Cluster Design</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;Q2FsaWJyYXRlZA==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;Calibrated&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="Q2FsaWJyYXRlZA=="><div class='gt_from_md'><p>Calibrated</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;SW5mbHVlbmNlIENhbGlicmF0ZWQ=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;Influence Calibrated&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="SW5mbHVlbmNlIENhbGlicmF0ZWQ="><div class='gt_from_md'><p>Influence Calibrated</p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">(Intercept)</td>
<td headers="estimate_1" class="gt_row gt_center">780</td>
<td headers="conf.low_1" class="gt_row gt_center">714, 847</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">785</td>
<td headers="conf.low_2" class="gt_row gt_center">755, 816</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">791</td>
<td headers="conf.low_3" class="gt_row gt_center">778, 803</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">ell</td>
<td headers="estimate_1" class="gt_row gt_center">-3.3</td>
<td headers="conf.low_1" class="gt_row gt_center">-4.3, -2.3</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">-3.3</td>
<td headers="conf.low_2" class="gt_row gt_center">-4.6, -1.9</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">-3.3</td>
<td headers="conf.low_3" class="gt_row gt_center">-3.5, -3.0</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">mobility</td>
<td headers="estimate_1" class="gt_row gt_center">-1.4</td>
<td headers="conf.low_1" class="gt_row gt_center">-3.1, 0.17</td>
<td headers="p.value_1" class="gt_row gt_center">0.075</td>
<td headers="estimate_2" class="gt_row gt_center">-1.5</td>
<td headers="conf.low_2" class="gt_row gt_center">-2.9, 0.00</td>
<td headers="p.value_2" class="gt_row gt_center">0.050</td>
<td headers="estimate_3" class="gt_row gt_center">-1.4</td>
<td headers="conf.low_3" class="gt_row gt_center">-1.9, -0.91</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">emer</td>
<td headers="estimate_1" class="gt_row gt_center">-1.8</td>
<td headers="conf.low_1" class="gt_row gt_center">-2.7, -0.88</td>
<td headers="p.value_1" class="gt_row gt_center">0.001</td>
<td headers="estimate_2" class="gt_row gt_center">-1.7</td>
<td headers="conf.low_2" class="gt_row gt_center">-2.5, -0.85</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">-2.2</td>
<td headers="conf.low_3" class="gt_row gt_center">-2.7, -1.8</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="10"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="Q0kgPSBDb25maWRlbmNlIEludGVydmFs"><div class='gt_from_md'><p>CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


We can see the precision in the estimates gets better as we move from the 
sampled model to the influence calibrated model. 

These can obviously get much more complex for more complex models.
For more theory Lumley refers the reader to [@breslow1980statistical]. 

### Two-phase designs

The best case for using influence functions as above in a two-phase design is
when a phase-one variable is correlated with the variable of interest measured
in phase two. Lumley gives an example for when this might occur: Self-reported
smoking at phase one then followed up by a urinary test in phase two. In this
setting Lumley proposes the following approach for constructing auxiliary 
variables based on influence functions.

1. Build an imputation model to predict the phase-two variable from the 
phase-one variables.

2. Fit a model to all of phase one, using the imputed value for observations 
that are not in the phase-two sample.

3. Use the influence functions from this model as auxiliary variables in 
calibration.

In the API example, the imputation model step was replaced by using the previous
year's API test values.


#### Example: Wilms' tumor

Lumley walks through [@breslow2009using]'s analysis of the NWTS data 
illustrating this approach. Lumley walks through the analysis in the text which 
has a clear mapping to the three steps above so I won't repeat him. The code to
fit the imputation model and calibrations are shown below.


::: {.cell}

```{.r .cell-code}
impmodel <- glm(histol ~ instit + I(age > 10) + I(stage == 4) * study,
  data = nwts, subset = in.subsample, family = binomial()
)

nwts$imphist <- predict(impmodel, newdata = nwts, type = "response")

nwts$imphist[nwts$in.subsample] <- nwts$histol[nwts$in.subsample]

ifmodel <- coxph(Surv(trel, relaps) ~ imphist * age + I(stage > 2) * tumdiam,
  data = nwts
)

inffun <- resid(ifmodel, "dfbeta")

colnames(inffun) <- paste("if", 1:6, sep = "")
nwts_if <- cbind(nwts, inffun)
if_design <- twophase(
  id = list(~1, ~1), subset = ~in.subsample,
  strata = list(NULL, ~ interaction(instit, relaps)),
  data = nwts_if
)

if_cal <- calibrate(if_design,
  phase = 2, calfun = "raking",
  formula = ~ if1 + if2 + if3 + if4 + if5 + if6 +
    relaps * instit
)

m1 <- svycoxph(Surv(trel, relaps) ~ histol * age + I(stage > 2) * tumdiam,
  design = nwts_design
)

m2 <- svycoxph(Surv(trel, relaps) ~ histol * age + I(stage > 2) * tumdiam,
  design = if_cal
)
m3 <- coxph(Surv(trel, relaps) ~ imphist * age + I(stage > 2) * tumdiam,
  data = nwts
)
m4 <- coxph(Surv(trel, relaps) ~ histol * age + I(stage > 2) * tumdiam,
  data = nwts
)

tbl_merge(
  tbls = list(
    tbl_regression(m1),
    tbl_regression(m2),
    tbl_regression(m3),
    tbl_regression(m4)
  ),
  tab_spanner = c(
    "2-Phase Weighted",
    "2-Phase Raked",
    "2 Phase Imputed",
    "Full Data"
  )
)
```

::: {.cell-output .cell-output-stdout}
```
Two-phase sparse-matrix design:
 twophase2(id = id, strata = strata, probs = probs, fpc = fpc, 
    subset = subset, data = data, pps = pps)
Phase 1:
Independent Sampling design (with replacement)
svydesign(ids = ~1)
Phase 2:
Stratified Independent Sampling design
svydesign(ids = ~1, strata = ~interaction(instit, relaps), fpc = `*phase1*`)
Two-phase sparse-matrix design:
 calibrate(if_design, phase = 2, calfun = "raking", formula = ~if1 + 
    if2 + if3 + if4 + if5 + if6 + relaps * instit)
Phase 1:
Independent Sampling design (with replacement)
svydesign(ids = ~1)
Phase 2:
Stratified Independent Sampling design
calibrate(phase2, formula, population, calfun = calfun, ...)
```
:::

::: {.cell-output-display}
```{=html}
<div id="iarytainyy" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#iarytainyy table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#iarytainyy thead, #iarytainyy tbody, #iarytainyy tfoot, #iarytainyy tr, #iarytainyy td, #iarytainyy th {
  border-style: none;
}

#iarytainyy p {
  margin: 0;
  padding: 0;
}

#iarytainyy .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#iarytainyy .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#iarytainyy .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#iarytainyy .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#iarytainyy .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#iarytainyy .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#iarytainyy .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#iarytainyy .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#iarytainyy .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#iarytainyy .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#iarytainyy .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#iarytainyy .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#iarytainyy .gt_spanner_row {
  border-bottom-style: hidden;
}

#iarytainyy .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#iarytainyy .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#iarytainyy .gt_from_md > :first-child {
  margin-top: 0;
}

#iarytainyy .gt_from_md > :last-child {
  margin-bottom: 0;
}

#iarytainyy .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#iarytainyy .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#iarytainyy .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#iarytainyy .gt_row_group_first td {
  border-top-width: 2px;
}

#iarytainyy .gt_row_group_first th {
  border-top-width: 2px;
}

#iarytainyy .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#iarytainyy .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#iarytainyy .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#iarytainyy .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#iarytainyy .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#iarytainyy .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#iarytainyy .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#iarytainyy .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#iarytainyy .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#iarytainyy .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#iarytainyy .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#iarytainyy .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#iarytainyy .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#iarytainyy .gt_left {
  text-align: left;
}

#iarytainyy .gt_center {
  text-align: center;
}

#iarytainyy .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#iarytainyy .gt_font_normal {
  font-weight: normal;
}

#iarytainyy .gt_font_bold {
  font-weight: bold;
}

#iarytainyy .gt_font_italic {
  font-style: italic;
}

#iarytainyy .gt_super {
  font-size: 65%;
}

#iarytainyy .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#iarytainyy .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#iarytainyy .gt_indent_1 {
  text-indent: 5px;
}

#iarytainyy .gt_indent_2 {
  text-indent: 10px;
}

#iarytainyy .gt_indent_3 {
  text-indent: 15px;
}

#iarytainyy .gt_indent_4 {
  text-indent: 20px;
}

#iarytainyy .gt_indent_5 {
  text-indent: 25px;
}

#iarytainyy .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#iarytainyy div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;Mi1QaGFzZSBXZWlnaHRlZA==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;2-Phase Weighted&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="Mi1QaGFzZSBXZWlnaHRlZA=="><div class='gt_from_md'><p>2-Phase Weighted</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;Mi1QaGFzZSBSYWtlZA==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;2-Phase Raked&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="Mi1QaGFzZSBSYWtlZA=="><div class='gt_from_md'><p>2-Phase Raked</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;MiBQaGFzZSBJbXB1dGVk&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;2 Phase Imputed&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="MiBQaGFzZSBJbXB1dGVk"><div class='gt_from_md'><p>2 Phase Imputed</p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;RnVsbCBEYXRh&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;Full Data&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="RnVsbCBEYXRh"><div class='gt_from_md'><p>Full Data</p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coSFIpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(HR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coSFIpKio="><div class='gt_from_md'><p><strong>log(HR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">histol</td>
<td headers="estimate_1" class="gt_row gt_center">1.8</td>
<td headers="conf.low_1" class="gt_row gt_center">1.4, 2.2</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">2.1</td>
<td headers="conf.low_2" class="gt_row gt_center">1.8, 2.4</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center"><br /></td>
<td headers="conf.low_3" class="gt_row gt_center"><br /></td>
<td headers="p.value_3" class="gt_row gt_center"><br /></td>
<td headers="estimate_4" class="gt_row gt_center">1.9</td>
<td headers="conf.low_4" class="gt_row gt_center">1.6, 2.2</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">age</td>
<td headers="estimate_1" class="gt_row gt_center">0.06</td>
<td headers="conf.low_1" class="gt_row gt_center">0.02, 0.10</td>
<td headers="p.value_1" class="gt_row gt_center">0.006</td>
<td headers="estimate_2" class="gt_row gt_center">0.10</td>
<td headers="conf.low_2" class="gt_row gt_center">0.07, 0.13</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">0.10</td>
<td headers="conf.low_3" class="gt_row gt_center">0.07, 0.13</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center">0.10</td>
<td headers="conf.low_4" class="gt_row gt_center">0.06, 0.13</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">I(stage &gt; 2)</td>
<td headers="estimate_1" class="gt_row gt_center"><br /></td>
<td headers="conf.low_1" class="gt_row gt_center"><br /></td>
<td headers="p.value_1" class="gt_row gt_center"><br /></td>
<td headers="estimate_2" class="gt_row gt_center"><br /></td>
<td headers="conf.low_2" class="gt_row gt_center"><br /></td>
<td headers="p.value_2" class="gt_row gt_center"><br /></td>
<td headers="estimate_3" class="gt_row gt_center"><br /></td>
<td headers="conf.low_3" class="gt_row gt_center"><br /></td>
<td headers="p.value_3" class="gt_row gt_center"><br /></td>
<td headers="estimate_4" class="gt_row gt_center"><br /></td>
<td headers="conf.low_4" class="gt_row gt_center"><br /></td>
<td headers="p.value_4" class="gt_row gt_center"><br /></td></tr>
    <tr><td headers="label" class="gt_row gt_left">    FALSE</td>
<td headers="estimate_1" class="gt_row gt_center">—</td>
<td headers="conf.low_1" class="gt_row gt_center">—</td>
<td headers="p.value_1" class="gt_row gt_center"><br /></td>
<td headers="estimate_2" class="gt_row gt_center">—</td>
<td headers="conf.low_2" class="gt_row gt_center">—</td>
<td headers="p.value_2" class="gt_row gt_center"><br /></td>
<td headers="estimate_3" class="gt_row gt_center">—</td>
<td headers="conf.low_3" class="gt_row gt_center">—</td>
<td headers="p.value_3" class="gt_row gt_center"><br /></td>
<td headers="estimate_4" class="gt_row gt_center">—</td>
<td headers="conf.low_4" class="gt_row gt_center">—</td>
<td headers="p.value_4" class="gt_row gt_center"><br /></td></tr>
    <tr><td headers="label" class="gt_row gt_left">    TRUE</td>
<td headers="estimate_1" class="gt_row gt_center">1.4</td>
<td headers="conf.low_1" class="gt_row gt_center">0.70, 2.0</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">1.4</td>
<td headers="conf.low_2" class="gt_row gt_center">0.91, 2.0</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">1.4</td>
<td headers="conf.low_3" class="gt_row gt_center">0.94, 1.9</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center">1.4</td>
<td headers="conf.low_4" class="gt_row gt_center">0.90, 1.9</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">tumdiam</td>
<td headers="estimate_1" class="gt_row gt_center">0.04</td>
<td headers="conf.low_1" class="gt_row gt_center">0.00, 0.08</td>
<td headers="p.value_1" class="gt_row gt_center">0.038</td>
<td headers="estimate_2" class="gt_row gt_center">0.06</td>
<td headers="conf.low_2" class="gt_row gt_center">0.03, 0.09</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">0.06</td>
<td headers="conf.low_3" class="gt_row gt_center">0.03, 0.09</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center">0.06</td>
<td headers="conf.low_4" class="gt_row gt_center">0.03, 0.09</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">histol * age</td>
<td headers="estimate_1" class="gt_row gt_center">-0.12</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.22, -0.01</td>
<td headers="p.value_1" class="gt_row gt_center">0.027</td>
<td headers="estimate_2" class="gt_row gt_center">-0.16</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.23, -0.08</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center"><br /></td>
<td headers="conf.low_3" class="gt_row gt_center"><br /></td>
<td headers="p.value_3" class="gt_row gt_center"><br /></td>
<td headers="estimate_4" class="gt_row gt_center">-0.14</td>
<td headers="conf.low_4" class="gt_row gt_center">-0.21, -0.08</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">I(stage &gt; 2) * tumdiam</td>
<td headers="estimate_1" class="gt_row gt_center"><br /></td>
<td headers="conf.low_1" class="gt_row gt_center"><br /></td>
<td headers="p.value_1" class="gt_row gt_center"><br /></td>
<td headers="estimate_2" class="gt_row gt_center"><br /></td>
<td headers="conf.low_2" class="gt_row gt_center"><br /></td>
<td headers="p.value_2" class="gt_row gt_center"><br /></td>
<td headers="estimate_3" class="gt_row gt_center"><br /></td>
<td headers="conf.low_3" class="gt_row gt_center"><br /></td>
<td headers="p.value_3" class="gt_row gt_center"><br /></td>
<td headers="estimate_4" class="gt_row gt_center"><br /></td>
<td headers="conf.low_4" class="gt_row gt_center"><br /></td>
<td headers="p.value_4" class="gt_row gt_center"><br /></td></tr>
    <tr><td headers="label" class="gt_row gt_left">    TRUE * tumdiam</td>
<td headers="estimate_1" class="gt_row gt_center">-0.07</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.12, -0.01</td>
<td headers="p.value_1" class="gt_row gt_center">0.014</td>
<td headers="estimate_2" class="gt_row gt_center">-0.08</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.13, -0.04</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">-0.08</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.12, -0.04</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center">-0.08</td>
<td headers="conf.low_4" class="gt_row gt_center">-0.12, -0.04</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">imphist</td>
<td headers="estimate_1" class="gt_row gt_center"><br /></td>
<td headers="conf.low_1" class="gt_row gt_center"><br /></td>
<td headers="p.value_1" class="gt_row gt_center"><br /></td>
<td headers="estimate_2" class="gt_row gt_center"><br /></td>
<td headers="conf.low_2" class="gt_row gt_center"><br /></td>
<td headers="p.value_2" class="gt_row gt_center"><br /></td>
<td headers="estimate_3" class="gt_row gt_center">2.1</td>
<td headers="conf.low_3" class="gt_row gt_center">1.8, 2.4</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center"><br /></td>
<td headers="conf.low_4" class="gt_row gt_center"><br /></td>
<td headers="p.value_4" class="gt_row gt_center"><br /></td></tr>
    <tr><td headers="label" class="gt_row gt_left">imphist * age</td>
<td headers="estimate_1" class="gt_row gt_center"><br /></td>
<td headers="conf.low_1" class="gt_row gt_center"><br /></td>
<td headers="p.value_1" class="gt_row gt_center"><br /></td>
<td headers="estimate_2" class="gt_row gt_center"><br /></td>
<td headers="conf.low_2" class="gt_row gt_center"><br /></td>
<td headers="p.value_2" class="gt_row gt_center"><br /></td>
<td headers="estimate_3" class="gt_row gt_center">-0.16</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.24, -0.08</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center"><br /></td>
<td headers="conf.low_4" class="gt_row gt_center"><br /></td>
<td headers="p.value_4" class="gt_row gt_center"><br /></td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="13"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="SFIgPSBIYXphcmQgUmF0aW8sIENJID0gQ29uZmlkZW5jZSBJbnRlcnZhbA=="><div class='gt_from_md'><p>HR = Hazard Ratio, CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


As expected we see that the raking and imputation - influence model have
the lowest standard errors. Lumley notes that this is often the case when the
imputation model is very good, further stating that the raking approach has
the advantage of always being valid, while the calibration depends on correct
modeling.

### Some history of the two phase calibration estimator

Here Lumley gives a brief overview of the history of calibration and 
augmented inverse-probability model based estimators, which were developed in
parallel and have a large amount of overlap.

I'd like to raise some questions I found myself asking as I read this last
section on using auxiliary information from phase 1.

1. The first concerns uncertainty estimates. Lumley doesn't mention it, but the
use of an imputation model should incorporate greater uncertainty into our final
estimates. It doesn't look like his software makes any effort to incorporate
that uncertainty or how optimistic our final standard errors might be.


2. The use of influence functions remind me of, but are not exactly similar to
dimensionality reduction methods, like 
[principle components analysis](https://en.wikipedia.org/wiki/Principal_component_analysis)
I can't help but wonder if there's a way to reduce many of the influence functions
to a few variables, or if some penalization methods would be required in the 
case of constructing *many* influence functions

3. As I mentioned earlier, it isn't clear how we choose which variables should
be regressed upon the influence function set-up. Why impute histology as opposed
to tumor stage? Is it because it's the most strongly associated with the 
outcome? How is someone supposed to know this in practice or *a priori*?

Overall I'm glad that Lumley included this subject in his book as its very 
important but there were a lot of questions I feel still needed answering in 
this last section.


##  Exercises

1.  Suppose a phase one simple random sample of size $n$ is taken from a 
population of size $N$, to measure a variable $X$ with $G$ categories. Write 
$N_1, N_2, ..., N_G$ for the (unknown) number of individual in the population in
each category, and $n_1,...,n_g$ for the number in the phase one sample. The 
phase-two sample takes a fixed number $m$ from each category. Show that 
$\pi_i^*$, $\pi_{ij}^*$ for this design approach $\pi_i$ and $\pi_{ij}$ for a
stratified sample from the population as $n$ increases.

I'll start with defining the weights in stratified sample design for an 
arbitrary strata $g$. The sampling probability $\pi_i = \frac{m}{N_g}$ and
the co-sampling probability is $\pi_{ij} = \frac{m}{N_g}(m-1)(N_g - 1)$. 
These values can be found from Chapter 2 question 10.

From the beginning of chapter 8 we have that 
$\pi_{i}^* = \pi_{i1} \times \pi_{i(2|1)}$ and the same is true for the 
co-inclusion probabilities. In which case we have $\pi_{i1} = \frac{n}{N}$
and $\pi_{i(2|1)} = \frac{m}{n_g}$. Which gives us
$$
\pi_{i*} = \frac{n}{N}\frac{m}{n_g}
$$
I'm not sure my notation is set up exactly right to answer Lumley's question
but I can reason my way there.

As $\n \to N$ we'll have more and more of the population in our first phase 
sample or in other words the first stage sampling probability will be 1 ; 
$\frac{N}{N} = 1$. In which case we're sampling from a strata with known strata 
size, $N_g$ which is equivalent to the sampling probability we saw before.

For the co-inclusion probability we have a similar phenomenon where $n \to N$
results in a leading factor becoming $1$.

$$
\pi_{ij,1} = \frac{n}{N}(n-1)(N-1) \\ 
\pi_{ij,(2|1)} = \frac{m}{n_g}(m-1)(n_g-1) \\
\implies \pi_{ij}^* = \frac{n}{N}(n-1)(N-1)\frac{m}{n_g}(m-1)(n_g-1) \\
\implies \pi_{ij}^* \stackrel{n\to N}{=} 1 \times (N-1)^2 \frac{m}{N_g}(m-1)(N_g - 1) \\
= \frac{m}{N_g}(m-1)(N_g - 1) (N - 1)^2
$$
which is the same except for the $(N-1)^2$ constant. I'm wondering if this
sampling probability assumes sampling with replacement... otherwise the 
conclusion probability should simplify --- easily --- to 1, the same as for
$\pi_i$, the only way you'd have any probability of sampling two things together
being more than 1, is if you were sampling with replacement...


2. Construct a full two-phase data set for the Ille-et Vilaine case-control 
study.The additional phase-one observations are 430000-975 controls to make the
number up to the population size. Fit the logistic regression model using the
design produced by `twophase()` and compare the results to the weighted 
estimates in Figure 8.1.


::: {.cell}

```{.r .cell-code}
cases <- cbind(esoph[rep(1:88, esoph$ncases), ], case = 1, weight = 1)
controls <- cbind(esoph[rep(1:88, esoph$ncontrols * 441), ], case = 0, weight = 1)
esoph_full <- rbind(cases, controls) %>%
  mutate(
    ix = 1:n(),
    in_second_phase = if_else(case == 1 | case == 0 & ix %% 441 == 1, TRUE,
      FALSE
    )
  )

twop_design <- twophase(list(~1, ~1),
  strata = list(NULL, ~case),
  subset = ~in_second_phase,
  data = esoph_full
)

fit <- svyglm(case ~ agegp + as.numeric(tobgp) + as.numeric(alcgp),
  design = twop_design, family = quasibinomial()
)

tbl_merge(
  tbls = list(
    tbl_regression(unwtd), tbl_regression(wtd),
    tbl_regression(fit)
  ),
  tab_spanner = c("**Unweighted**", "**Weighted**", "**Two Phase**")
)
```

::: {.cell-output-display}
```{=html}
<div id="dlmpzyvcnq" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#dlmpzyvcnq table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#dlmpzyvcnq thead, #dlmpzyvcnq tbody, #dlmpzyvcnq tfoot, #dlmpzyvcnq tr, #dlmpzyvcnq td, #dlmpzyvcnq th {
  border-style: none;
}

#dlmpzyvcnq p {
  margin: 0;
  padding: 0;
}

#dlmpzyvcnq .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#dlmpzyvcnq .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#dlmpzyvcnq .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#dlmpzyvcnq .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#dlmpzyvcnq .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#dlmpzyvcnq .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#dlmpzyvcnq .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#dlmpzyvcnq .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#dlmpzyvcnq .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#dlmpzyvcnq .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#dlmpzyvcnq .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#dlmpzyvcnq .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#dlmpzyvcnq .gt_spanner_row {
  border-bottom-style: hidden;
}

#dlmpzyvcnq .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#dlmpzyvcnq .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#dlmpzyvcnq .gt_from_md > :first-child {
  margin-top: 0;
}

#dlmpzyvcnq .gt_from_md > :last-child {
  margin-bottom: 0;
}

#dlmpzyvcnq .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#dlmpzyvcnq .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#dlmpzyvcnq .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#dlmpzyvcnq .gt_row_group_first td {
  border-top-width: 2px;
}

#dlmpzyvcnq .gt_row_group_first th {
  border-top-width: 2px;
}

#dlmpzyvcnq .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#dlmpzyvcnq .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#dlmpzyvcnq .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#dlmpzyvcnq .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#dlmpzyvcnq .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#dlmpzyvcnq .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#dlmpzyvcnq .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#dlmpzyvcnq .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#dlmpzyvcnq .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#dlmpzyvcnq .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#dlmpzyvcnq .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#dlmpzyvcnq .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#dlmpzyvcnq .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#dlmpzyvcnq .gt_left {
  text-align: left;
}

#dlmpzyvcnq .gt_center {
  text-align: center;
}

#dlmpzyvcnq .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#dlmpzyvcnq .gt_font_normal {
  font-weight: normal;
}

#dlmpzyvcnq .gt_font_bold {
  font-weight: bold;
}

#dlmpzyvcnq .gt_font_italic {
  font-style: italic;
}

#dlmpzyvcnq .gt_super {
  font-size: 65%;
}

#dlmpzyvcnq .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#dlmpzyvcnq .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#dlmpzyvcnq .gt_indent_1 {
  text-indent: 5px;
}

#dlmpzyvcnq .gt_indent_2 {
  text-indent: 10px;
}

#dlmpzyvcnq .gt_indent_3 {
  text-indent: 15px;
}

#dlmpzyvcnq .gt_indent_4 {
  text-indent: 20px;
}

#dlmpzyvcnq .gt_indent_5 {
  text-indent: 25px;
}

#dlmpzyvcnq .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#dlmpzyvcnq div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipVbndlaWdodGVkKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Unweighted&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipVbndlaWdodGVkKio="><div class='gt_from_md'><p><strong>Unweighted</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipXZWlnaHRlZCoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Weighted&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipXZWlnaHRlZCoq"><div class='gt_from_md'><p><strong>Weighted</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipUd28gUGhhc2UqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Two Phase&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipUd28gUGhhc2UqKg=="><div class='gt_from_md'><p><strong>Two Phase</strong></p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kipsb2coT1IpKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;log(OR)&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kipsb2coT1IpKio="><div class='gt_from_md'><p><strong>log(OR)</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">agegp</td>
<td headers="estimate_1" class="gt_row gt_center"><br /></td>
<td headers="conf.low_1" class="gt_row gt_center"><br /></td>
<td headers="p.value_1" class="gt_row gt_center"><br /></td>
<td headers="estimate_2" class="gt_row gt_center"><br /></td>
<td headers="conf.low_2" class="gt_row gt_center"><br /></td>
<td headers="p.value_2" class="gt_row gt_center"><br /></td>
<td headers="estimate_3" class="gt_row gt_center"><br /></td>
<td headers="conf.low_3" class="gt_row gt_center"><br /></td>
<td headers="p.value_3" class="gt_row gt_center"><br /></td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp.L</td>
<td headers="estimate_1" class="gt_row gt_center">3.8</td>
<td headers="conf.low_1" class="gt_row gt_center">2.7, 5.6</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">3.9</td>
<td headers="conf.low_2" class="gt_row gt_center">2.5, 5.2</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">3.9</td>
<td headers="conf.low_3" class="gt_row gt_center">2.5, 5.2</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp.Q</td>
<td headers="estimate_1" class="gt_row gt_center">-1.5</td>
<td headers="conf.low_1" class="gt_row gt_center">-3.1, -0.51</td>
<td headers="p.value_1" class="gt_row gt_center">0.014</td>
<td headers="estimate_2" class="gt_row gt_center">-1.4</td>
<td headers="conf.low_2" class="gt_row gt_center">-2.6, -0.16</td>
<td headers="p.value_2" class="gt_row gt_center">0.026</td>
<td headers="estimate_3" class="gt_row gt_center">-1.4</td>
<td headers="conf.low_3" class="gt_row gt_center">-2.6, -0.17</td>
<td headers="p.value_3" class="gt_row gt_center">0.026</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp.C</td>
<td headers="estimate_1" class="gt_row gt_center">0.08</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.73, 1.2</td>
<td headers="p.value_1" class="gt_row gt_center">0.9</td>
<td headers="estimate_2" class="gt_row gt_center">0.29</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.63, 1.2</td>
<td headers="p.value_2" class="gt_row gt_center">0.5</td>
<td headers="estimate_3" class="gt_row gt_center">0.29</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.63, 1.2</td>
<td headers="p.value_3" class="gt_row gt_center">0.5</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp^4</td>
<td headers="estimate_1" class="gt_row gt_center">0.12</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.58, 0.74</td>
<td headers="p.value_1" class="gt_row gt_center">0.7</td>
<td headers="estimate_2" class="gt_row gt_center">0.25</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.40, 0.90</td>
<td headers="p.value_2" class="gt_row gt_center">0.4</td>
<td headers="estimate_3" class="gt_row gt_center">0.25</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.40, 0.90</td>
<td headers="p.value_3" class="gt_row gt_center">0.4</td></tr>
    <tr><td headers="label" class="gt_row gt_left">    agegp^5</td>
<td headers="estimate_1" class="gt_row gt_center">-0.25</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.67, 0.17</td>
<td headers="p.value_1" class="gt_row gt_center">0.2</td>
<td headers="estimate_2" class="gt_row gt_center">-0.28</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.73, 0.17</td>
<td headers="p.value_2" class="gt_row gt_center">0.2</td>
<td headers="estimate_3" class="gt_row gt_center">-0.28</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.73, 0.17</td>
<td headers="p.value_3" class="gt_row gt_center">0.2</td></tr>
    <tr><td headers="label" class="gt_row gt_left">as.numeric(tobgp)</td>
<td headers="estimate_1" class="gt_row gt_center">0.44</td>
<td headers="conf.low_1" class="gt_row gt_center">0.25, 0.63</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">0.45</td>
<td headers="conf.low_2" class="gt_row gt_center">0.21, 0.69</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">0.45</td>
<td headers="conf.low_3" class="gt_row gt_center">0.21, 0.68</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
    <tr><td headers="label" class="gt_row gt_left">as.numeric(alcgp)</td>
<td headers="estimate_1" class="gt_row gt_center">1.1</td>
<td headers="conf.low_1" class="gt_row gt_center">0.87, 1.3</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">1.0</td>
<td headers="conf.low_2" class="gt_row gt_center">0.80, 1.3</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">1.0</td>
<td headers="conf.low_3" class="gt_row gt_center">0.80, 1.3</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="10"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="T1IgPSBPZGRzIFJhdGlvLCBDSSA9IENvbmZpZGVuY2UgSW50ZXJ2YWw="><div class='gt_from_md'><p>OR = Odds Ratio, CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


We can see from the table above that the two phase and weighted results are
effectively the same. This makes sense, given the large size of the phase one
sample.

3. This exercise uses the WA State crime data for 2004 as the population. The 
data consist of crime rates and population size for the police districts and
sheriffs offices grouped by county.


::: {.cell}

```{.r .cell-code}
wa_crime_df <- readxl::read_xlsx("data/WA_crime/1984-2011.xlsx",
  skip = 4
) %>%
  filter(Year == "2004", Population > 0) %>%
  mutate(
    murder = `Murder Total`,
    murder_and_crime = `Murder Total` + `Burglary Total`,
    violent_crime = `Violent Crime Total`,
    burglaries = `Burglary Total`,
    property_crime = `Property Crime Total`,
    state_pop = sum(Population),
    County = stringr::str_to_lower(County),
    num_counties = n_distinct(County),
  ) %>%
  group_by(County) %>%
  mutate(num_agencies = n_distinct(Agency)) %>%
  ungroup() %>%
  select(
    County, Agency, Population, murder_and_crime, murder, violent_crime,
    property_crime, burglaries, num_counties, num_agencies
  )
```
:::


  * Take a simple random sample of 20 police districts from the state and use
  all the data from the sampled counties. Estimate the total number of murders
  and burglaries in the state
  
Not clear what Lumley means here, given that he first says to sample at the 
police district level but then says to use all the data at the county level...

Given this mix up I proceed assuming he meant that we should sample at the
county level and then use all the data for those counties.


::: {.cell}

```{.r .cell-code}
set.seed(35315)
county_sample <- wa_crime_df %>%
  distinct(County) %>%
  slice_sample(n = 20) %>%
  pull(County)

wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design(
    ids = c(County, Agency),
    fpc = c(num_counties, num_agencies)
  ) %>%
  summarize(
    total = survey_total(murder_and_crime, vartype = "ci", deff = TRUE)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 4
   total total_low total_upp total_deff
   <dbl>     <dbl>     <dbl>      <dbl>
1 49532.    23911.    75153.       2.60
```
:::
:::


  * Calibrate the sample in the previous question using population as the 
  auxiliary variable and estimate the total number of murders and burglaries in 
  the state.
  

::: {.cell}

```{.r .cell-code}
srs_design <- wa_crime_df %>%
  filter(County %in% county_sample) %>%
  as_survey_design(
    ids = c(County, Agency),
    fpc = c(num_counties, num_agencies)
  )

calibrated_srs_design <- calibrate(srs_design,
  formula = ~1,
  population = sum(wa_crime_df$Population)
)

svytotal(~murder_and_crime, calibrated_srs_design)
```

::: {.cell-output .cell-output-stdout}
```
                      total        SE
murder_and_crime 1599843753 232267644
```
:::
:::


My use of the `calibrate()` function continues to spit out ridiculous numbers.
I'll try to debug this later but for now all I can say is that my cursory look
through the documentation for the function doesn't show any egregious misuse.
  
  * Take a simple random sample of 100 police districts as phase one of a two
  phase design, and assume that population is the only variable available at
  phase one. Divide the sample into 10 strata with roughly equal total 
  population and sample two police districts from each stratum for phase two.
  Estimate the total number of murders and of burglaries in the state.
  

::: {.cell}

```{.r .cell-code}
set.seed(33531)
p1_sample <- wa_crime_df %>%
  distinct(Agency) %>%
  slice_sample(n = 100) %>%
  pull(Agency)

p1_pop <- wa_crime_df %>%
  filter(Agency %in% p1_sample) %>%
  pull(Population)
p1_pop_quantiles <- quantile(p1_pop, seq(from = 0, to = 1, length.out = 10))

p2_sample <- wa_crime_df %>%
  filter(Agency %in% p1_sample) %>%
  mutate(strata = cut(Population,
    breaks = p1_pop_quantiles,
    include.lowest = TRUE
  )) %>%
  select(Agency, strata) %>%
  group_by(strata) %>%
  slice_sample(n = 2) %>%
  ungroup() %>%
  pull(Agency)

twop_design <- wa_crime_df %>%
  filter(Agency %in% p1_sample) %>%
  mutate(
    strata = cut(Population, breaks = p1_pop_quantiles, include.lowest = TRUE),
    in_phase_two = Agency %in% p2_sample,
    fpc_one = 234,
  ) %>%
  group_by(strata) %>%
  mutate(fpc_two = n()) %>%
  ungroup() %>%
  as_survey_twophase(
    id = list(Agency, Agency),
    strata = list(NULL, strata),
    subset = in_phase_two,
    fpc = list(fpc_one, fpc_two)
  )

twop_design %>%
  summarize(
    total = survey_total(murder_and_crime, deff = TRUE)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 3
   total total_se total_deff
   <dbl>    <dbl>      <dbl>
1 45205.    9184.     0.0152
```
:::
:::


We see a much lower uncertainty around the total estimate of the number of
murders and crimes relative to the previous design. This is especially
exemplified in the design effect.
  
  * Calibrate the two phase sample using the phase-one population size data.
  Estimate the total number of murders and of burglaries in the state.


::: {.cell}

```{.r .cell-code}
twop_calibration <- calibrate(twop_design, ~Population,
  phase = "2"
)
svytotal(~murder_and_crime, twop_calibration, deff = TRUE)
```

::: {.cell-output .cell-output-stdout}
```
                 total    SE   DEff
murder_and_crime 64008 10268 0.0128
```
:::
:::


Here at least the number makes sense, though it looks like the standard error
has increased relative to the standard two phase design. This might be because
the population information is already encoded in the strata design of the 
second phase of sampling.

  
4. The sampling probabilities $\pi$ and $\pi^*$ in the NWTS two phase 
case-control study depend on the 2 x 2 table of relaps and instit. Suppose that 
the super population probabilities for the cells in the 2 x 2 table match those
in Table 8.3.

 * Write R code to simulate realizations of Table 8.3 and to compute the second
 phase sampling probabilities $\pi_{i(2|1)}$ for a two-phase design with sample
 size 1300 and cell counts as equal as possible. That is, sample everyone in a 
 cell that has fewer than 1300 people and then divide the remaining sample size 
 evenly over the remaining cells.
 
I see a lot of ways to conduct the second phase sampling that could be 
influential on the results. For example, do we try to ensure that the cell
counts are as equal as possible, even at the expense of throwing away sample 
size? Or should we take as many samples as we can get even if it results in
more unequal cell counts.
 

::: {.cell}

```{.r .cell-code}
probs <- c(
  "Relapse_Unfav" = 194 / 3915, "Relapse_Fav" = 475 / 3915,
  "Control_Unfav" = 245 / 3915, "Control_Fav" = 3001 / 3915
)
probs_df <- as_tibble(t(probs)) %>%
  gather(everything(), key = "Cell", value = "pi_one")
sims <- t(rmultinom(1000, size = 3915, prob = probs)) %>%
  as_tibble() %>%
  mutate(sim_ix = 1:n()) %>%
  gather(everything(), -sim_ix, key = "Cell", value = "Count") %>%
  arrange(sim_ix, Count) %>%
  group_by(sim_ix) %>%
  mutate(
    cell_rank = min_rank(Count),
    initial_sample = sum((cell_rank == 1) * Count),
    remaining_sample = (1300 - initial_sample) / 3,
    twop_sampling_probability = case_when(
      # sample everything from smallest cell with probability 1
      cell_rank == 1 ~ 1,
      remaining_sample > Count ~ 1,
      remaining_sample < Count ~ remaining_sample / Count
    )
  ) %>%
  left_join(probs_df) %>%
  mutate(pi_star = pi_one * twop_sampling_probability)
```
:::

 
 
 * Run 1000 simulations, compute $\pi_i$ as the average of $\pi_i^*$ for a given
 cell over the simulations, and compare $\pi_i$ to the distribution of 
 $\pi_i^*$.
 

::: {.cell}

```{.r .cell-code}
pi_i <- sims %>%
  group_by(Cell) %>%
  summarize(mean_p = mean(pi_star))
sims %>%
  ggplot(aes(x = pi_star, fill = Cell)) +
  geom_histogram() +
  geom_vline(aes(xintercept = pi_i$mean_p[1]), linetype = 2, color = "red") +
  geom_vline(aes(xintercept = pi_i$mean_p[2]), linetype = 2, color = "red") +
  geom_vline(aes(xintercept = pi_i$mean_p[3]), linetype = 2, color = "black") +
  geom_vline(aes(xintercept = pi_i$mean_p[4]), linetype = 2, color = "red") +
  ggtitle("Distribution of Product Probabilities",
    subtitle = "Dotted Lines Indicate Average of Cell Distribution"
  )
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/8q4_b-1.png){width=672}
:::
:::


As constructed, the $\pi_i$ are the average of the $\pi_i^*$ so, naturally, 
these appear in the center of the distributions. What is perhaps more relieving
is that the distributions either have low variance, when they have small 
probabilities of occurring, or are nicely normally distributed when they are,
which, again, makes sense.

5. This exercise uses the WA state crime data for 2004 as the population and the
data for 2003 as the auxiliary variables.

  *  Take a simple random sample of 20 police districts from the state and use 
  all the data from the sampled counties. Estimate the ratio of violent crimes
  to property crimes.
  

::: {.cell}

```{.r .cell-code}
wa_crime_df <- readxl::read_xlsx("data/WA_crime/1984-2011.xlsx",
  skip = 4
) %>%
  filter(Year %in% c("2003", "2004"), Population > 0) %>%
  mutate(
    murder = `Murder Total`,
    murder_and_crime = `Murder Total` + `Burglary Total`,
    violent_crime = `Violent Crime Total`,
    burglaries = `Burglary Total`,
    property_crime = `Property Crime Total`,
    state_pop = sum(Population),
    County = stringr::str_to_lower(County),
    num_counties = n_distinct(County),
  ) %>%
  group_by(County) %>%
  mutate(num_agencies = n_distinct(Agency)) %>%
  ungroup() %>%
  select(
    Year, County, Agency, Population, murder_and_crime, murder, violent_crime,
    property_crime, burglaries, num_counties, num_agencies
  )

district_sample <- wa_crime_df %>%
  filter(Year == "2004") %>%
  distinct(Agency) %>%
  slice_sample(n = 20) %>%
  pull(Agency)

srs_design <- wa_crime_df %>%
  filter(Year == "2004") %>%
  mutate(fpc = n_distinct(Agency)) %>%
  filter(Agency %in% district_sample) %>%
  select(Year, County, Agency, Population, violent_crime, property_crime, fpc) %>%
  left_join(wa_crime_df %>% filter(Year == "2003") %>%
    select(County, Agency, burglaries, Population) %>%
    rename_if(is.numeric, function(x) str_c(x, "_2003"))) %>%
  as_survey_design(
    ids = "Agency",
    fpc = fpc
  )

srs_design %>%
  summarize(
    survey_ratio(violent_crime, property_crime)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 2
    coef   `_se`
   <dbl>   <dbl>
1 0.0445 0.00768
```
:::
:::

  
  * Calibrate the sample in the previous question using population and number of
  burglaries as the auxiliary variables, and estimate the ratio of violent 
  crimes to property crimes.
  

::: {.cell}

```{.r .cell-code}
pop_totals <- wa_crime_df %>% 
  filter(Year == "2003") %>% 
  summarize(
    burglaries_2003 = sum(burglaries),
    Population_2003 = sum(Population))

srs_calibration <- calibrate(srs_design,
                             ~ burglaries_2003 + ~Population_2003 - 1,
  population = pop_totals
)
svyratio(~violent_crime, ~property_crime, srs_calibration)
```

::: {.cell-output .cell-output-stdout}
```
Ratio estimator: svyratio.survey.design2(~violent_crime, ~property_crime, srs_calibration)
Ratios=
              property_crime
violent_crime      0.0437048
SEs=
              property_crime
violent_crime    0.008199953
```
:::
:::


Our point estimate is about the same, but the standard 
error has increased ever so slightly.
  
  * The ratio of violent crimes to property crimes in the state in 2003 was 
  $21078 / 290945 = 0.0724$. Define an auxiliary variable 
  `infl = violent - 0.0724 x property`, the influence function for the ratio,
  and use it to calibrate to the state data. Estimate the ratio of violent
  crimes to property crimes.
  

::: {.cell}

```{.r .cell-code}
srs_design <- update(srs_design,
  infl = violent_crime - 0.0724 * property_crime
)

pop_total <- wa_crime_df %>% 
  filter(Year == "2003") %>% 
  mutate(infl = violent_crime - 0.0724 * property_crime) %>% 
  summarize(
    infl = sum(infl)
  )

srs_calibration <- calibrate(srs_design, ~infl -1,
  population = pop_total
)
svyratio(~violent_crime, ~property_crime, srs_calibration)
```

::: {.cell-output .cell-output-stdout}
```
Ratio estimator: svyratio.survey.design2(~violent_crime, ~property_crime, srs_calibration)
Ratios=
              property_crime
violent_crime     0.07456344
SEs=
              property_crime
violent_crime    0.001312338
```
:::
:::


Now we see a pretty much spot on estimate and the lowest standard error of them
all.
  
  * Take a simple random sample of 100 police districts as phase one of a 
  two-phase design, and assume that the 2003 crime data are available at phase
  one. Divide the sample into 10 strata with roughly equal total number of 
  burglaries in 2003 and sample two police districts from each stratum for
  phase two. Estimate the total number of murders, and of burglaries, in the 
  state.
  

::: {.cell}

```{.r .cell-code}
set.seed(35315)
district_sample <- wa_crime_df %>%
  filter(Year == "2004") %>%
  distinct(Agency) %>%
  slice_sample(n = 100) %>%
  pull(Agency)

burglary_quantiles <- wa_crime_df %>%
  filter(Year == "2003") %>%
  pull(burglaries) %>%
  quantile(., seq(from = 0, to = 1, length.out = 10))

second_phase_sample <- wa_crime_df %>%
  filter(Year == "2004", Agency %in% district_sample) %>%
  mutate(
    strata = cut(burglaries, burglary_quantiles, include.lowest = TRUE)
  ) %>%
  group_by(strata) %>%
  slice_sample(n = 2) %>%
  ungroup() %>%
  pull(Agency)


wa_crime_df %>%
  filter(Year == "2004") %>%
  mutate(
    fpc_one = n_distinct(Agency),
    strata = cut(burglaries, burglary_quantiles, include.lowest = TRUE)
  ) %>%
  filter(Agency %in% district_sample) %>%
  mutate(
    in_second_phase = (Agency %in% second_phase_sample),
    fpc_two = n(),
  ) %>%
  as_survey_twophase(
    id = list(Agency, Agency),
    strata = list(NULL, strata),
    subset = in_second_phase,
    fpc = list(fpc_one, fpc_two)
  ) %>%
  summarize(
    num_burglaries = survey_total(burglaries),
    num_murders = survey_total(murder)
  )
```

::: {.cell-output .cell-output-stdout}
```
# A tibble: 1 × 4
  num_burglaries num_burglaries_se num_murders num_murders_se
           <dbl>             <dbl>       <dbl>          <dbl>
1         650988           318526.        2808           773.
```
:::
:::


The results are very imprecise, especially for the number of burglaries for 
which we're not able to compute a standard error.

  * Calibrate the two-phase sample using the auxiliary variable `infl`. Estimate
  the total number of murders, and of burglaries, in the state.
  

::: {.cell}

```{.r .cell-code}
two_phase_crime_design <- wa_crime_df %>%
  filter(Year == "2004") %>%
  mutate(
    fpc_one = n_distinct(Agency),
    strata = cut(burglaries, burglary_quantiles, include.lowest = TRUE),
    infl = violent_crime - 0.0724 * property_crime,
  ) %>%
  filter(Agency %in% district_sample) %>%
  mutate(
    in_second_phase = (Agency %in% second_phase_sample),
    fpc_two = n(),
  ) %>%
  as_survey_twophase(
    id = list(Agency, Agency),
    strata = list(NULL, strata),
    subset = in_second_phase,
    fpc = list(fpc_one, fpc_two)
  )

total_infl <- wa_crime_df %>%
  filter(Year == "2004") %>%
  summarize(infl = sum(violent_crime - 0.0724 * property_crime)) %>%
  pull(infl)


calibrate_tp_cd <- calibrate(two_phase_crime_design, ~infl - 1,
  population = total_infl,
  phase = "2"
)
svytotal(~ burglaries + ~murder, design = calibrate_tp_cd)
```

::: {.cell-output .cell-output-stdout}
```
              total        SE
burglaries 56029.82 196962.95
murder       707.75    642.11
```
:::
:::



Still highly variable answers, even with the calibration!


6. What would the efficiency approximation from exercise 5.8 give as the loss of
efficiency from using weights in a case-control design?

TODO(apeterson)

::: {.cell}

:::


# Chapter 9: Missing Data

## Item Non-Response

Lumley opens this chapter by drawing a distinction between item and unit 
non-response. *Item non-response* refers to the case in which partial data
is available from a sampled respondent or observation, but not all. In contrast,
*unit non-response* occurs when no data at all is available from a sampled
respondent or unit of analysis.

Lumley then lays out two classes of approach to handle item non-response that
correspond to both design and modeling philosophies:

1. Design ---  Frame the non-response as part of the sampling mechanism in a
two phase design.

2. Imputation --- Model the non-response as a function of other measured 
covariates and estimate the missing values.

## Two Phase Estimation for Missing Data

As described above, if the pattern of missingness is relatively simple, such
that the sample can be divided into two groups: (1) Those with complete data
and (2) those with incomplete data, then the data can be modeled as a two
phase sample in which the first phase individuals have incomplete data and the
second phase have complete data. 

### Calibration for item non-response

Knowing the mechanism by which individuals' data went missing is typically not
feasible but Lumley recommends starting with a simple random sample weight
and then calibrating those weights according to other available covariates.


::: {.cell}

```{.r .cell-code}
set.seed(31631)
sigmoid <- binomial()$linkinv
pmar <- with(apistrat, sigmoid(-7 + api99 / 100 - emer / 10))
pnar <- with(apistrat, sigmoid( -7 + api00 / 100 - emer / 10))
mar <- rbinom(nrow(apistrat), 1, pmar)
nar <- rbinom(nrow(apistrat), 1, pnar)

stratmar <- apistrat
stratmar$api00[mar == 1] <- NA
stratmar$w2 <- nrow(apistrat) / sum(1 - mar) ## MCAR sampling weights
stratnar <- apistrat
stratnar$api00[nar == 1] <- NA
stratnar$w2 <- nrow(apistrat) / sum(1 - nar)

mar_des <- twophase(id = list(~1, ~1),
                    strata = list(~stype, ~stype),
                    subset = ~I(!is.na(api00)),
                    weights = list(~pw, ~w2),
                    # difference from original code
                    method = 'approx',
                    data = stratmar)

nar_des <- twophase(id = list(~1, ~1),
                    strata = list(~stype, ~stype),
                    subset = ~I(!is.na(api00)),
                    weights = list(~pw, ~w2),
                    method = "approx",
                    data = stratnar)

calmar1 <- calibrate(mar_des, phase = 2, calfun = "raking",
                     ~api99 + emer + stype + enroll)

calnar1 <- calibrate(nar_des, phase = 2, calfun = "raking",
                     ~api99 + emer + stype + enroll)

calmar2 <- calibrate(mar_des, phase = 2, calfun = "raking",
                    ~ ns(api99, 3) + emer + stype + enroll)

calnar2 <- calibrate(nar_des, phase = 2, calfun = "raking",
                     ~ns(api99, 3) + emer + stype + enroll)

dstrat <- svydesign(id = ~1, strata = ~stype, weights = ~pw,
                    data = apistrat, fpc = ~fpc)
dstrat_mar <- svydesign(id = ~1, strata = ~stype, weights = ~pw,
                    data = stratmar, fpc = ~fpc)
dstrat_nar <- svydesign(id = ~1, strata = ~stype, weights = ~pw,
                    data = stratnar, fpc = ~fpc)

# This is included in the book code, but I focus on the glm output
# for ease of illustration.
# svymean(~api99 + api00, dstrat)
fit_full <- svyglm(api00 ~ emer + ell + meals, dstrat)
fit_naive <- svyglm(api00 ~ emer + ell + meals, dstrat_mar)
fit_mar1 <- svyglm(api00 ~ emer + ell + meals, calmar1)
fit_mar2 <- svyglm(api00 ~ emer + ell + meals, calmar2)
fit_naive_n <- svyglm(api00 ~ emer + ell + meals, dstrat_nar)
fit_nar1 <- svyglm(api00 ~ emer + ell + meals, calnar1)
fit_nar2 <- svyglm(api00 ~ emer + ell + meals, calnar2)
tbl_merge(
  tbls = list(
    tbl_regression(fit_full, conf.int = FALSE),
    tbl_regression(fit_naive, conf.int = FALSE),
    tbl_regression(fit_mar1, conf.int = FALSE),
    tbl_regression(fit_mar2, conf.int = FALSE),
    tbl_regression(fit_naive_n, conf.int = FALSE),
    tbl_regression(fit_nar1, conf.int = FALSE),
    tbl_regression(fit_nar2, conf.int = FALSE)
  ),
  tab_spanner = c(
   "**Full Data**",
   "**Naive - MAR**",
   "**Linear - MAR**",
   "**NonLinear - MAR**",
   "**Naive - NAR**",
   "**Linear - NAR**",
   "**NonLinear - NAR**"
  )
)
```

::: {.cell-output-display}
```{=html}
<div id="lnvpvhlxsh" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#lnvpvhlxsh table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#lnvpvhlxsh thead, #lnvpvhlxsh tbody, #lnvpvhlxsh tfoot, #lnvpvhlxsh tr, #lnvpvhlxsh td, #lnvpvhlxsh th {
  border-style: none;
}

#lnvpvhlxsh p {
  margin: 0;
  padding: 0;
}

#lnvpvhlxsh .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#lnvpvhlxsh .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#lnvpvhlxsh .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#lnvpvhlxsh .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#lnvpvhlxsh .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#lnvpvhlxsh .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#lnvpvhlxsh .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#lnvpvhlxsh .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#lnvpvhlxsh .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#lnvpvhlxsh .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#lnvpvhlxsh .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#lnvpvhlxsh .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#lnvpvhlxsh .gt_spanner_row {
  border-bottom-style: hidden;
}

#lnvpvhlxsh .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#lnvpvhlxsh .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#lnvpvhlxsh .gt_from_md > :first-child {
  margin-top: 0;
}

#lnvpvhlxsh .gt_from_md > :last-child {
  margin-bottom: 0;
}

#lnvpvhlxsh .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#lnvpvhlxsh .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#lnvpvhlxsh .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#lnvpvhlxsh .gt_row_group_first td {
  border-top-width: 2px;
}

#lnvpvhlxsh .gt_row_group_first th {
  border-top-width: 2px;
}

#lnvpvhlxsh .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#lnvpvhlxsh .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#lnvpvhlxsh .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#lnvpvhlxsh .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#lnvpvhlxsh .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#lnvpvhlxsh .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#lnvpvhlxsh .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#lnvpvhlxsh .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#lnvpvhlxsh .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#lnvpvhlxsh .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#lnvpvhlxsh .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#lnvpvhlxsh .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#lnvpvhlxsh .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#lnvpvhlxsh .gt_left {
  text-align: left;
}

#lnvpvhlxsh .gt_center {
  text-align: center;
}

#lnvpvhlxsh .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#lnvpvhlxsh .gt_font_normal {
  font-weight: normal;
}

#lnvpvhlxsh .gt_font_bold {
  font-weight: bold;
}

#lnvpvhlxsh .gt_font_italic {
  font-style: italic;
}

#lnvpvhlxsh .gt_super {
  font-size: 65%;
}

#lnvpvhlxsh .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#lnvpvhlxsh .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#lnvpvhlxsh .gt_indent_1 {
  text-indent: 5px;
}

#lnvpvhlxsh .gt_indent_2 {
  text-indent: 10px;
}

#lnvpvhlxsh .gt_indent_3 {
  text-indent: 15px;
}

#lnvpvhlxsh .gt_indent_4 {
  text-indent: 20px;
}

#lnvpvhlxsh .gt_indent_5 {
  text-indent: 25px;
}

#lnvpvhlxsh .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#lnvpvhlxsh div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipGdWxsIERhdGEqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Full Data&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipGdWxsIERhdGEqKg=="><div class='gt_from_md'><p><strong>Full Data</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipOYWl2ZSAtIE1BUioq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Naive - MAR&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipOYWl2ZSAtIE1BUioq"><div class='gt_from_md'><p><strong>Naive - MAR</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipMaW5lYXIgLSBNQVIqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Linear - MAR&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipMaW5lYXIgLSBNQVIqKg=="><div class='gt_from_md'><p><strong>Linear - MAR</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipOb25MaW5lYXIgLSBNQVIqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;NonLinear - MAR&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipOb25MaW5lYXIgLSBNQVIqKg=="><div class='gt_from_md'><p><strong>NonLinear - MAR</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipOYWl2ZSAtIE5BUioq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Naive - NAR&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipOYWl2ZSAtIE5BUioq"><div class='gt_from_md'><p><strong>Naive - NAR</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipMaW5lYXIgLSBOQVIqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Linear - NAR&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipMaW5lYXIgLSBOQVIqKg=="><div class='gt_from_md'><p><strong>Linear - NAR</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="2" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipOb25MaW5lYXIgLSBOQVIqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;NonLinear - NAR&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipOb25MaW5lYXIgLSBOQVIqKg=="><div class='gt_from_md'><p><strong>NonLinear - NAR</strong></p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">emer</td>
<td headers="estimate_1" class="gt_row gt_center">-1.8</td>
<td headers="p.value_1" class="gt_row gt_center">0.006</td>
<td headers="estimate_2" class="gt_row gt_center">-1.6</td>
<td headers="p.value_2" class="gt_row gt_center">0.016</td>
<td headers="estimate_3" class="gt_row gt_center">-1.6</td>
<td headers="p.value_3" class="gt_row gt_center">0.030</td>
<td headers="estimate_4" class="gt_row gt_center">-2.3</td>
<td headers="p.value_4" class="gt_row gt_center">0.004</td>
<td headers="estimate_5" class="gt_row gt_center">-1.5</td>
<td headers="p.value_5" class="gt_row gt_center">0.019</td>
<td headers="estimate_6" class="gt_row gt_center">-1.7</td>
<td headers="p.value_6" class="gt_row gt_center">0.021</td>
<td headers="estimate_7" class="gt_row gt_center">-2.2</td>
<td headers="p.value_7" class="gt_row gt_center">0.006</td></tr>
    <tr><td headers="label" class="gt_row gt_left">ell</td>
<td headers="estimate_1" class="gt_row gt_center">-0.56</td>
<td headers="p.value_1" class="gt_row gt_center">0.12</td>
<td headers="estimate_2" class="gt_row gt_center">-0.69</td>
<td headers="p.value_2" class="gt_row gt_center">0.083</td>
<td headers="estimate_3" class="gt_row gt_center">-0.54</td>
<td headers="p.value_3" class="gt_row gt_center">0.2</td>
<td headers="estimate_4" class="gt_row gt_center">-0.62</td>
<td headers="p.value_4" class="gt_row gt_center">0.14</td>
<td headers="estimate_5" class="gt_row gt_center">-0.53</td>
<td headers="p.value_5" class="gt_row gt_center">0.2</td>
<td headers="estimate_6" class="gt_row gt_center">-0.34</td>
<td headers="p.value_6" class="gt_row gt_center">0.4</td>
<td headers="estimate_7" class="gt_row gt_center">-0.47</td>
<td headers="p.value_7" class="gt_row gt_center">0.3</td></tr>
    <tr><td headers="label" class="gt_row gt_left">meals</td>
<td headers="estimate_1" class="gt_row gt_center">-2.7</td>
<td headers="p.value_1" class="gt_row gt_center"><0.001</td>
<td headers="estimate_2" class="gt_row gt_center">-2.2</td>
<td headers="p.value_2" class="gt_row gt_center"><0.001</td>
<td headers="estimate_3" class="gt_row gt_center">-2.5</td>
<td headers="p.value_3" class="gt_row gt_center"><0.001</td>
<td headers="estimate_4" class="gt_row gt_center">-2.4</td>
<td headers="p.value_4" class="gt_row gt_center"><0.001</td>
<td headers="estimate_5" class="gt_row gt_center">-2.5</td>
<td headers="p.value_5" class="gt_row gt_center"><0.001</td>
<td headers="estimate_6" class="gt_row gt_center">-2.7</td>
<td headers="p.value_6" class="gt_row gt_center"><0.001</td>
<td headers="estimate_7" class="gt_row gt_center">-2.7</td>
<td headers="p.value_7" class="gt_row gt_center"><0.001</td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


Above I show one iteration of a missing at random analysis, though in the book
Lumley shows the % bias in a simulation study of missing data. In his analysis
he identifies a reduction in bias from using the (linear) calibration in both
the missing at random data and not missing at random data. The flexible
calibration does worse, he argues, because of the low sample size.

### Models for Response Probability

In addition to calibrating the sampling weights in the hypothetical two-phase
sample, we can also model the second phase conditional probabilities using a
(e.g.) logistic regression.

Lumley notes that this approach does not satisfy the same constraints that 
calibrating does - calibration guarantees estimated totals in phase one match
their true counterpart, while the regression guarantees the same for the 
second phase.

**Example NHANES III bone density**

This example looks at missing bone mineral density measurements from the NHANES
study.


::: {.cell}

```{.r .cell-code}
sqlite <- dbDriver("SQLite")
dbcon <- dbConnect(sqlite, dbname = "data/nhanes/imp.db")
nhanes <- dbGetQuery(dbcon, "select SDPPSU6, WTPFQX6, SDPSTRA6,
                     HSAGEU, HSAGEIR, DMARETHN, HSSEX, DMARETHN, DMPMETRO,
                     DMPCREGN, BMPWTMI, BMPWTIF, BDPFNDMI, BDPFNDIF
                     from set1")
dbDisconnect(dbcon)

nhanes <- subset(nhanes, BDPFNDIF > 0)
nhanes$hasbone <- 2 - nhanes$BDPFNDIF
nhanes$age <- with(nhanes, ifelse(HSAGEU == 1, HSAGEIR / 12, HSAGEIR))
nhanes <- subset(nhanes, age > 20)
nhanes$psu <- with(nhanes, SDPPSU6 + 10 * SDPSTRA6)
nhanes$agegp <- with(nhanes, cut(age, c(20, 40, 60 , Inf)))

model <- glm(hasbone ~ (BMPWTMI + age) * 
               (HSSEX * DMARETHN + DMPMETRO * DMPCREGN), family = binomial,
             data = nhanes)
nhanes$po <- mean(nhanes$hasbone)
nhanes$pi2 <- fitted(model)
design0 <- twophase(id = list(~psu, ~1), strata = list(~SDPSTRA6, NULL),
                    weights = list(~WTPFQX6, ~I(1 / po)), 
                    subset = ~I(hasbone == 1),
                    ## necessary for updated version of survey package
                    method = "approx",
                    data = nhanes)
design1 <- twophase(id = list(~psu, ~1), strata = list(~SDPSTRA6, NULL),
                    weights = list(~WTPFQX6, ~I(1 / pi2)), 
                    subset = ~I(hasbone == 1),
                    ## necessary for updated version of survey package
                    method = "approx",
                    data = nhanes)

design2 <- calibrate(design0, phase = 2, 
                     formula = ~(BMPWTMI + age) * 
                       (HSSEX * DMARETHN + DMPMETRO * DMPCREGN))
```
:::

::: {.cell}

```{.r .cell-code}
tibble(
  design = c("Constant Non-Response Probability",
             "Modeled Non-Response Probability",
             "Calibrated Non-Response"),
  estimate = c(svymean(~BDPFNDMI, design0),
               svymean(~BDPFNDMI, design1),
               svymean(~BDPFNDMI, design2)),
  lower = c(confint(svymean(~BDPFNDMI, design0))[1],
            confint(svymean(~BDPFNDMI, design1))[1],
            confint(svymean(~BDPFNDMI, design2))[1]),
  upper = c(confint(svymean(~BDPFNDMI, design0))[2],
            confint(svymean(~BDPFNDMI, design1))[2],
            confint(svymean(~BDPFNDMI, design2))[2]),
) %>%  
  mutate(
    fmt_estimate = str_c(round(estimate, 3),
                         "(", round(lower, 3),
                         ", ", round(upper, 3),
                         ")"),
  ) %>% 
  select(design, fmt_estimate) %>% 
  spread(design,fmt_estimate) %>% 
  gt::gt()
```

::: {.cell-output-display}
```{=html}
<div id="aipwdxutlp" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#aipwdxutlp table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#aipwdxutlp thead, #aipwdxutlp tbody, #aipwdxutlp tfoot, #aipwdxutlp tr, #aipwdxutlp td, #aipwdxutlp th {
  border-style: none;
}

#aipwdxutlp p {
  margin: 0;
  padding: 0;
}

#aipwdxutlp .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#aipwdxutlp .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#aipwdxutlp .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#aipwdxutlp .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#aipwdxutlp .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#aipwdxutlp .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#aipwdxutlp .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#aipwdxutlp .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#aipwdxutlp .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#aipwdxutlp .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#aipwdxutlp .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#aipwdxutlp .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#aipwdxutlp .gt_spanner_row {
  border-bottom-style: hidden;
}

#aipwdxutlp .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#aipwdxutlp .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#aipwdxutlp .gt_from_md > :first-child {
  margin-top: 0;
}

#aipwdxutlp .gt_from_md > :last-child {
  margin-bottom: 0;
}

#aipwdxutlp .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#aipwdxutlp .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#aipwdxutlp .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#aipwdxutlp .gt_row_group_first td {
  border-top-width: 2px;
}

#aipwdxutlp .gt_row_group_first th {
  border-top-width: 2px;
}

#aipwdxutlp .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#aipwdxutlp .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#aipwdxutlp .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#aipwdxutlp .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#aipwdxutlp .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#aipwdxutlp .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#aipwdxutlp .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#aipwdxutlp .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#aipwdxutlp .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#aipwdxutlp .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#aipwdxutlp .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#aipwdxutlp .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#aipwdxutlp .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#aipwdxutlp .gt_left {
  text-align: left;
}

#aipwdxutlp .gt_center {
  text-align: center;
}

#aipwdxutlp .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#aipwdxutlp .gt_font_normal {
  font-weight: normal;
}

#aipwdxutlp .gt_font_bold {
  font-weight: bold;
}

#aipwdxutlp .gt_font_italic {
  font-style: italic;
}

#aipwdxutlp .gt_super {
  font-size: 65%;
}

#aipwdxutlp .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#aipwdxutlp .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#aipwdxutlp .gt_indent_1 {
  text-indent: 5px;
}

#aipwdxutlp .gt_indent_2 {
  text-indent: 10px;
}

#aipwdxutlp .gt_indent_3 {
  text-indent: 15px;
}

#aipwdxutlp .gt_indent_4 {
  text-indent: 20px;
}

#aipwdxutlp .gt_indent_5 {
  text-indent: 25px;
}

#aipwdxutlp .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#aipwdxutlp div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Calibrated Non-Response">Calibrated Non-Response</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Constant Non-Response Probability">Constant Non-Response Probability</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="Modeled Non-Response Probability">Modeled Non-Response Probability</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="Calibrated Non-Response" class="gt_row gt_right">0.817(0.811, 0.823)</td>
<td headers="Constant Non-Response Probability" class="gt_row gt_right">0.826(0.82, 0.831)</td>
<td headers="Modeled Non-Response Probability" class="gt_row gt_right">0.818(0.812, 0.825)</td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


Lumley points out that the impact of modeling the non response --- a change in
the point estimate of around .1 is larger than the standard error and so has the
potential to impact the inference. Further, it isn't clear what kind of 
definitive conclusion can be drawn from this adjustment but it seems *likely* 
that those who did not respond had a lower bone mineral density than those that
did.

### Effect on Precision

In summarizing the impact of modeling or calibrating response probabilities on
the estimate's precision Lumley makes the following categorization about those
variables used to calibrate or model non-response:

1. Associated with non-response but not with variables analyzed.
   * Decrease precision, no impact on bias.
2. Associated with non-response & variables being analyzed.
   * Decrease bias, may increase or decrease precision.
3. Associated only with variables analyzed and *not* with non-response.
  * No impact on bias, increase precision.

### Doubly Robust Estimators

Both of the previous methods that incorporated estimating the non-response
into a two-phase sampling design hinged on getting the relationship between
$Y$ and non-response correct:

 * The two-phase estimator will be valid if the model for the response 
probabilities correctly models everything that affects both $Y$ --- the outcome
variable --- and non response. 

 * The calibration estimator will be valid if the working model $Y$ correctly
 models everything affects both $Y$ and non-response.
 
It is also possible to separate the two models for non-response *and* the 
variable $Y$ so that the estimator will be unbiased in large samples if 
*either* model is correct. Further, if the calibration is correct we can
get increased precision. This approach is called a "Double Robust Estimator"
and more can be found on it in [@bickel2001inference].


I'm not entirely sure why Lumley included this section in his book as there's no
way I see to easily construct a doubly robust estimator from his `survey` R
package.

## Imputation of Missing Data

Lumley gives a brief description of imputation here which I'll only note 
requires similar strong assumptions as the previous methods discussed --- 
modelling the relationship between $Y$ and non-response, but has the added
benefit of being useful for other, more general analyses, as a complete 
dataset(s) is(are) provided instead of a single analysis' specific weights.

As I alluded to above, typically multiple imputed data sets are provided
to capture the added variability and uncertainty that the missing data adds
to our analysis. I'll note that this was added uncertainty was also not present
in the previous methods described earlier. Lumley cites Rubin in 
identifying that the variability between $m$ imputed data sets represents the 
added variance attributable to our missing data.

$$
\hat{V}[T] = \frac{1}{m} \sum_{j=1}^{m} v_j + (\frac{m + 1}{m}) \frac{1}{m-1} \sum_{j=1}^{m} (\hat{T}_j - \bar{T})^2
$$

### Describing Multiple Imputations in R

I think the demo in the next section illustrates the point best, but the 
general idea Lumley gets at here is that each imputed data set needs to be
given its own design object and analysis and then a special function needs to
be called to ensure the results are combined appropriately.

### Example: NHANES Imputations

This example looks at the same bone mineral density data from NHANES as 
used previously in this chapter, only now we use the five imputed data sets 
provided by the CDC. A description of how these imputations were created can be
found in [@schafer2001multiple]. Recall, that creating the imputed data sets 
requires specialized knowledge, while using them requires only a knowledge of 
how to combine the imputed results together.



::: {.cell}

```{.r .cell-code}
impdata <- imputationList(c("set1", "set2", "set3", "set4", "set5"),
                          dbtype = "SQLite", dbname = "Data/nhanes/imp.db")
impdata
```

::: {.cell-output .cell-output-stdout}
```
MI data with 5 datasets
Call: imputationList(c("set1", "set2", "set3", "set4", "set5"), dbtype = "SQLite", 
    dbname = "Data/nhanes/imp.db")
```
:::
:::

::: {.cell}

```{.r .cell-code}
designs <- svydesign(id = ~SDPPSU6, strat = ~SDPSTRA6,
                     weight = ~WTPFQX6, data = impdata, nest = TRUE)

designs
```

::: {.cell-output .cell-output-stdout}
```
DB-backed Multiple (5) imputations: svydesign(id = ~SDPPSU6, strat = ~SDPSTRA6, weight = ~WTPFQX6, 
    data = impdata, nest = TRUE)
```
:::
:::

::: {.cell}

```{.r .cell-code}
designs <- update(designs, age = ifelse(HSAGEU == 1, HSAGEIR / 12, HSAGEIR))
designs <- update(designs, agegp = cut(age, c(20, 40, 60, Inf, right = FALSE)))
res <- with(subset(designs, age >= 20),
            svyby(~BDPFNDMI, ~agegp + HSSEX, svymean))
summary(MIcombine(res))
```

::: {.cell-output .cell-output-stdout}
```
Multiple imputation results:
      with(subset(designs, age >= 20), svyby(~BDPFNDMI, ~agegp + HSSEX, 
    svymean))
      MIcombine.default(res)
             results          se    (lower    upper) missInfo
(0,20].1   0.9649648 0.017813216 0.9296637 1.0002659     20 %
(20,40].1  0.9317077 0.003545463 0.9246253 0.9387901     27 %
(40,60].1  0.8362213 0.003859253 0.8285832 0.8438593     19 %
(60,Inf].1 0.7652223 0.004013359 0.7573199 0.7731246     13 %
(0,20].2   0.8671506 0.016246398 0.8344422 0.8998590     32 %
(20,40].2  0.8505263 0.003199152 0.8442018 0.8568507     18 %
(40,60].2  0.7776360 0.003620937 0.7705191 0.7847530     10 %
(60,Inf].2 0.6428459 0.004189769 0.6342868 0.6514050     41 %
```
:::
:::


The `summary(MIcombine(res))` call above uses the `mitools` R package methods 
for combining separate imputed estimates together to produce the output we see. 
Note in the output that the `missInfo` column describes the percentage of 
"missing information" - roughly the between imputations variance as a fraction
of the total variance - akin to the design effect

Lumley doesn't go into the substantive meaning of the results beyond comparing
it to a previous published analysis. Likely because these will be used in the 
exercises at the end of the chapter.

Lumley's next analysis uses replicate weights in a design-based 
logistic regression model. Lumley notes that the replicate weights are the same
for all imputed models and that they were constructed using "Fay's method" which
makes better adjustments for when there are few observations present at the 
intersection of some observed covariate. 

I'm guessing that the replicate weights weren't changed as the result of the
imputation --- if all original observations have imputed results. Still would
be nice to have some commentary from Lumley on this point. 

The code to perform the analysis is below. As before, Lumley offers no 
commentary on the results of the analysis.



::: {.cell}

```{.r .cell-code}
designs <- update(designs,
                  systolic = (PEP6G1MI + PEP6H1MI + PEP6I1MI) / 3)
qres <- with(subset(designs, age >= 20),
             svyby(~systolic + TCPMI, ~agegp + HSSEX, svyquantile, 
                   quantiles = c(0.5, 0.9), se = TRUE))
summary(MIcombine(qres), digits = 2)
```

::: {.cell-output .cell-output-stdout}
```
Multiple imputation results:
      with(subset(designs, age >= 20), svyby(~systolic + TCPMI, ~agegp + 
    HSSEX, svyquantile, quantiles = c(0.5, 0.9), se = TRUE))
      MIcombine.default(qres)
                        results    se (lower upper) missInfo
(0,20].1:systolic.0.5       115  1.37    112    117     21 %
(20,40].1:systolic.0.5      117  0.50    116    118      0 %
(40,60].1:systolic.0.5      123  0.60    121    124     33 %
(60,Inf].1:systolic.0.5     135  0.83    133    136      0 %
(0,20].2:systolic.0.5       105  2.14    101    109     15 %
(20,40].2:systolic.0.5      107  0.55    106    108     39 %
(40,60].2:systolic.0.5      118  0.60    117    119     33 %
(60,Inf].2:systolic.0.5     137  0.89    135    139     22 %
(0,20].1:systolic.0.9       126  2.79    121    131      0 %
(20,40].1:systolic.0.9      132  0.79    131    134     28 %
(40,60].1:systolic.0.9      145  1.05    143    148     10 %
(60,Inf].1:systolic.0.9     164  1.07    162    166      0 %
(0,20].2:systolic.0.9       116  1.61    113    119      0 %
(20,40].2:systolic.0.9      122  0.66    121    123      0 %
(40,60].2:systolic.0.9      144  1.24    141    146     11 %
(60,Inf].2:systolic.0.9     170  1.42    167    172     38 %
(0,20].1:TCPMI.0.5          165  5.55    154    176      3 %
(20,40].1:TCPMI.0.5         190  1.57    187    193     16 %
(40,60].1:TCPMI.0.5         211  1.30    208    214      0 %
(60,Inf].1:TCPMI.0.5        209  1.53    206    212     11 %
(0,20].2:TCPMI.0.5          170  6.46    158    183     23 %
(20,40].2:TCPMI.0.5         184  1.25    182    187     16 %
(40,60].2:TCPMI.0.5         210  1.76    207    214     30 %
(60,Inf].2:TCPMI.0.5        229  1.52    226    232     17 %
(0,20].1:TCPMI.0.9          206 10.60    184    227     34 %
(20,40].1:TCPMI.0.9         244  1.72    241    248      8 %
(40,60].1:TCPMI.0.9         265  3.12    259    271      9 %
(60,Inf].1:TCPMI.0.9        264  3.44    257    271      2 %
(0,20].2:TCPMI.0.9          219 16.84    185    252     27 %
(20,40].2:TCPMI.0.9         234  2.29    229    239     38 %
(40,60].2:TCPMI.0.9         274  1.86    270    277      7 %
(60,Inf].2:TCPMI.0.9        288  3.22    282    295     15 %
```
:::
:::

::: {.cell}

```{.r .cell-code}
sqlite <- dbDriver("SQLite")
dbcon <- dbConnect(sqlite, dbname = "data/nhanes/imp.db")
impload <- function(varlist, conn) {
  tables <- paste("set", 1:5, sep = "")
  vars <- paste(varlist, collapse = ",")
  query <- paste("select",vars,"from @tab@")
  data <- lapply(tables,
                 function(table) dbGetQuery(conn, sub("@tab@", table, query)))
  mitools::imputationList(data)
}

regdata <- impload(
  c("SDPPSU6", "WTPFQX6","SDPSTRA6", "HSAGEU", "HSAGEIR", "DMARETHN",
    "BMPWTMI", "BMPHTMI", "DMPPIRMI",
    "HAB1MI", "HAT28MI", "HSSEX"), dbcon)
wtnames <- paste(paste("WTPQRP", 1:52, sep = ""), collapse = ",")
regwts <- dbGetQuery(dbcon, paste("select", wtnames, "from core"))
designs <- svrepdesign(data = regdata, repweights = regwts,
                       type = "Fay", rho = 0.3, weights = ~WTPFQX6, 
                       combined = TRUE)
dbDisconnect(dbcon)

designs <- update(designs,
                  age = ifelse(HSAGEU == 1, HSAGEIR / 12, HSAGEIR))
adults <- subset(designs, age >= 20)
adults <- update(adults,
                 race = factor(ifelse(DMARETHN == 4, 1, DMARETHN)),
                 bmi =  BMPWTMI / (BMPHTMI / 100)^2,
                 poverty = DMPPIRMI <= 1,
                 evgfp = factor(HAB1MI),
                 activity = factor(HAT28MI, levels = c(3,1,2,-9)),
                 agegp = cut(age, c(0, 20, 40, 60, Inf), right = FALSE)
                 )
adults <- update(adults,
                 overwt = ifelse(HSSEX == 1, bmi >= 27.8, bmi >= 27.3))
models <- with(adults,
               svyglm(overwt ~ HSSEX + agegp + race + evgfp + activity + poverty,
                      family = quasibinomial))
summary(MIcombine(models), digits = 3)
```

::: {.cell-output .cell-output-stdout}
```
Multiple imputation results:
      with(adults, svyglm(overwt ~ HSSEX + agegp + race + evgfp + activity + 
    poverty, family = quasibinomial))
      MIcombine.default(models)
              results     se  (lower upper) missInfo
(Intercept)   -1.5902 0.1122 -1.8106 -1.370     10 %
HSSEX          0.0491 0.0507 -0.0504  0.149      5 %
agegp[40,60)   0.6964 0.0549  0.5885  0.804      9 %
agegp[60,Inf)  0.6065 0.0740  0.4614  0.752      4 %
race2          0.4481 0.0573  0.3358  0.560      2 %
race3          0.4103 0.0657  0.2815  0.539      3 %
evgfp2         0.4281 0.0702  0.2904  0.566      4 %
evgfp3         0.6852 0.0761  0.5359  0.834      6 %
evgfp4         0.7669 0.0825  0.6052  0.929      3 %
evgfp5         0.3757 0.1175  0.1453  0.606      6 %
activity1     -0.4014 0.0622 -0.5239 -0.279     14 %
activity2      0.2732 0.0564  0.1626  0.384      3 %
povertyTRUE   -0.0160 0.0637 -0.1421  0.110     18 %
```
:::
:::


##  Exercises

1. Fit regression models to examine how self-rated general health (HAB1MI) is 
related to age (HSAGEIR) and obesity (`bmi` as defined in Figure 9.7)

  * Using the NHANES III multiply imputed data, with replicate weights.
  
I'll use the same `adults` object defined in the previous example since it
has all the appropriate variables loaded.


::: {.cell}

```{.r .cell-code}
models <- with(adults,
     svyglm(HAB1MI ~ age))

summary(MIcombine(models), digits = 3)
```

::: {.cell-output .cell-output-stdout}
```
Multiple imputation results:
      with(adults, svyglm(HAB1MI ~ age))
      MIcombine.default(models)
            results       se (lower upper) missInfo
(Intercept)  1.8457 0.032645 1.7817 1.9097      0 %
age          0.0141 0.000731 0.0126 0.0155      0 %
```
:::
:::


In the above table we see a statistically significant positive relationship
between age and self-rated general health, even after accounting for the 
missing data.
  
  * Using just the complete data before imputation.
  

::: {.cell}

```{.r .cell-code}
sqlite <- dbDriver("SQLite")
dbcon <- dbConnect(sqlite, dbname = "data/nhanes/imp.db")
nhanes <- dbGetQuery(dbcon, "select SDPPSU6, WTPFQX6, SDPSTRA6, BMPWTMI, 
                     BMPHTMI, HAB1MI, HSAGEU, HSAGEIR from set1
                     WHERE HAB1MI > 0 AND BMPWTMI > 0 AND BMPHTMI > 0")
dbDisconnect(dbcon)
design <- svydesign(ids = ~SDPPSU6, weights = ~WTPFQX6, strata = ~SDPSTRA6,
          data = nhanes, nest = TRUE)
design <- update(design,
                 age = ifelse(HSAGEU == 1, HSAGEIR / 12, HSAGEIR),
                 bmi =  BMPWTMI / (BMPHTMI / 100)^2,
                 self_health = factor(HAB1MI))

model <- svyolr(self_health ~ age, design = design, subset = age > 20)
summary(model)
```

::: {.cell-output .cell-output-stdout}
```
Call:
svyolr(self_health ~ age, design = design, subset = age > 20)

Coefficients:
         Value  Std. Error  t value
age 0.02408857 0.001442298 16.70153

Intercepts:
    Value   Std. Error t value
1|2 -0.3025  0.0744    -4.0658
2|3  1.1232  0.0733    15.3167
3|4  2.7634  0.0791    34.9400
4|5  4.5148  0.0775    58.2186
```
:::
:::


Wee see a similar statistically significant relationship as before but the 
effect is slightly attenuated.

2. Repeat the logistic regression analysis in Figure 9.7 using linearization 
instead of replicate weights.


::: {.cell}

```{.r .cell-code}
sqlite <- dbDriver("SQLite")
dbcon <- dbConnect(sqlite, dbname = "data/nhanes/imp.db")
impload <- function(varlist, conn) {
  tables <- paste("set", 1:5, sep = "")
  vars <- paste(varlist, collapse = ",")
  query <- paste("select",vars,"from @tab@")
  data <- lapply(tables,
                 function(table) dbGetQuery(conn, sub("@tab@", table, query)))
  mitools::imputationList(data)
}
regdata <- impload(
  c("SDPPSU6", "WTPFQX6","SDPSTRA6", "HSAGEU", "HSAGEIR", "DMARETHN", 
    "DMPMETRO", "BMPWTMI", "BMPHTMI", "DMPPIRMI", "HAB1MI", 
    "HAT28MI","BDPFNDMI",
    "HSSEX"), dbcon)
dbDisconnect(dbcon)

designs <- svydesign(data = regdata, weights = ~WTPFQX6, 
                     strata = ~SDPSTRA6, id = ~SDPPSU6, nest = TRUE)
designs <- update(designs,
                  age = ifelse(HSAGEU == 1, HSAGEIR / 12, HSAGEIR))
adults <- subset(designs, age >= 20)
adults <- update(adults,
                 race = factor(ifelse(DMARETHN == 4, 1, DMARETHN)),
                 bmi =  BMPWTMI / (BMPHTMI / 100)^2,
                 poverty = DMPPIRMI <= 1,
                 evgfp = factor(HAB1MI),
                 activity = factor(HAT28MI, levels = c(3,1,2,-9)),
                 agegp = cut(age, c(0, 20, 40, 60, Inf), right = FALSE)
                 )
adults <- update(adults,
                 overwt = ifelse(HSSEX == 1, bmi >= 27.8, bmi >= 27.3))
models <- with(adults,
               svyglm(overwt ~ HSSEX + agegp + race + evgfp + activity + poverty,
                      family = quasibinomial))
summary(MIcombine(models), digits = 3)
```

::: {.cell-output .cell-output-stdout}
```
Multiple imputation results:
      with(adults, svyglm(overwt ~ HSSEX + agegp + race + evgfp + activity + 
    poverty, family = quasibinomial))
      MIcombine.default(models)
              results     se  (lower upper) missInfo
(Intercept)   -1.5902 0.1283 -1.8420 -1.338      8 %
HSSEX          0.0491 0.0557 -0.0601  0.158      4 %
agegp[40,60)   0.6964 0.0606  0.5774  0.815      8 %
agegp[60,Inf)  0.6065 0.0721  0.4651  0.748      4 %
race2          0.4481 0.0647  0.3212  0.575      2 %
race3          0.4103 0.0745  0.2642  0.556      2 %
evgfp2         0.4281 0.0828  0.2657  0.590      3 %
evgfp3         0.6852 0.0888  0.5111  0.859      4 %
evgfp4         0.7669 0.0884  0.5936  0.940      3 %
evgfp5         0.3757 0.1338  0.1134  0.638      4 %
activity1     -0.4014 0.0636 -0.5266 -0.276     13 %
activity2      0.2732 0.0702  0.1356  0.411      2 %
povertyTRUE   -0.0160 0.0703 -0.1547  0.123     15 %
```
:::
:::


The results are effectively the same as before though the variances --- and 
consequently the missing Information estimates --- have changed slightly.

3. In the NHANES III imputation data, estimate mean, median, and 90th percentile
of systolic blood pressure and total cholesterol broken down by rural/urban
location and region of the country.

Not clear to me which variables represent Systolic blood pressure, total
cholesterol or the rural/urban location. When I check online 
[documentation](https://wwwn.cdc.gov/nchs/nhanes/nhanes3/default.aspx) for
the appropriate variables, I don't see them in the data set that Lumley 
posted on his website. Leaving this question blank for now.

4. Fit regression models to examine how self-rated general health (HAB1MI) is 
related to obesity (`bmi` as defined in Figure 9.7)

  * Using just the complete data before imputation.
  

::: {.cell}

```{.r .cell-code}
sqlite <- dbDriver("SQLite")
dbcon <- dbConnect(sqlite, dbname = "data/nhanes/imp.db")
nhanes <- dbGetQuery(dbcon, "select SDPPSU6, WTPFQX6, SDPSTRA6,
                     BMPWTMI, HSAGEU, HSAGEIR, DMARETHN, BMPHTMI, HAB1MI,
                     DMPMETRO from set1
                     WHERE HAB1MI >0 AND BMPWTMI > 0 AND BMPHTMI > 0")
dbDisconnect(dbcon)
## I use the weights as is here, but they should be adjusted to account for
## the missingness. Not clear to me if that's part of the imputed data or not
## but I'm following Lumley's example.
design <- svydesign(ids = ~SDPPSU6, weights = ~WTPFQX6, strata = ~SDPSTRA6,
          data = nhanes, nest = TRUE)
design <- update(design,
                 age = ifelse(HSAGEU == 1, HSAGEIR / 12, HSAGEIR),
                 bmi =  BMPWTMI / (BMPHTMI / 100)^2,
                 urban = factor(DMPMETRO),
                 race = factor(ifelse(DMARETHN == 4, 1, DMARETHN)),
                 self_health = factor(HAB1MI))
summary(svyolr(self_health ~ bmi, design, subset = age > 20))
```

::: {.cell-output .cell-output-stdout}
```
Call:
svyolr(self_health ~ bmi, design, subset = age > 20)

Coefficients:
         Value  Std. Error  t value
bmi 0.05506785 0.004228902 13.02179

Intercepts:
    Value   Std. Error t value
1|2  0.1008  0.1353     0.7450
2|3  1.5184  0.1369    11.0941
3|4  3.1349  0.1441    21.7494
4|5  4.8627  0.1386    35.0893
```
:::
:::


I model the self reported health outcome as an ordinal outcome, for which we
see a positive and significant association with BMI.

Looking at a plot below this relationship can roughly be seen. We also see some
BMI values > 60 that look dubious. I'd probably want to check the imputation 
model or the data records for these if I were working with this data more 
seriously.

::: {.cell}

```{.r .cell-code}
svyplot(self_health ~ bmi, design)
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/unnamed-chunk-9-1.png){width=672}
:::
:::


  * Calibrating on age, sex, race, urban / rural location and region of the
  country.
  
  Again, I'm not sure about which of the NHANES variables correspond to race,
  urban / rural location or region of the country. Also not clear how we should
  get the population data for these values.

5. For the incomplete data sets defined in Figure 9.1, fit a linear model to
predict `api00` from the other variables and impute from this regression model
by taking the fitted value from the regression and adding a randomly selected 
residual.

   * Impute a single complete data set and do the two analyses (mean and 
 regression model) at the end of Figure 9.1. Compare the results to the true 
 population values.
 
Its worth noting that adding the right residual involves computing the 
standard deviation of the estimated variables. I show how to do this below.
 

::: {.cell}

```{.r .cell-code}
imputation_model_one <- lm(api00 ~ api99 + emer + stype, data = stratmar)
api00_imp <- predict(imputation_model_one, newdata = stratmar[is.na(stratmar$api00),])
X_miss <- model.matrix(~api99 + emer + stype, data = stratmar[is.na(stratmar$api00),])
resid_std <- diag(X_miss %*% 
                    vcov(imputation_model_one) %*% 
                    t(X_miss))

imputed_apistrat <- stratmar
imputed_apistrat$api00_imp <- imputed_apistrat$api00
imputed_apistrat$api00_imp[is.na(imputed_apistrat$api00)] <- api00_imp
imputed_apistrat$pw <- apistrat$pw

dstrat_mar <- svydesign(id = ~1, strata = ~stype, weights = ~pw,
                    data = imputed_apistrat, fpc = ~fpc)
svymean(~api00_imp, dstrat_mar)
```

::: {.cell-output .cell-output-stdout}
```
            mean     SE
api00_imp 664.69 9.6795
```
:::
:::


For the missing at random data set, the mean estimate is very close to the 
population value of 664.71. Though it is worth
noting that the variance almost certainly underestimates the "true" variance of
this estimator.


::: {.cell}

```{.r .cell-code}
imputation_model_two <- lm(api00 ~ api99 + emer + stype, data = stratnar)
api00_imp <- predict(imputation_model_two, newdata = stratnar[is.na(stratnar$api00),])
X_miss <- model.matrix(~api99 + emer + stype, data = stratnar[is.na(stratnar$api00),])
resid_std <- diag(X_miss %*% 
                    vcov(imputation_model_two) %*% 
                    t(X_miss))

imputed_apistrat <- stratnar
imputed_apistrat$api00_imp <- imputed_apistrat$api00
imputed_apistrat$api00_imp[is.na(imputed_apistrat$api00)] <- api00_imp + 
  rnorm(n = length(resid_std), sd = resid_std)
imputed_apistrat$pw <- apistrat$pw

dstrat_nar <- svydesign(id = ~1, strata = ~stype, weights = ~pw,
                    data = imputed_apistrat, fpc = ~fpc)
svymean(~api00_imp, dstrat_nar)
```

::: {.cell-output .cell-output-stdout}
```
            mean    SE
api00_imp 662.38 9.624
```
:::
:::



Here we see a slight increase in bias (downward) from the true population value.
Again, the variance is similar to before but certainly underestimated.

  * Create 10 multiply imputed data sets and repeat the two analyses. Compare
  the results to the true population values.


::: {.cell}

```{.r .cell-code}
MI_apistrat <- mitools::imputationList(lapply(1:10, function(ix) {
  imputed_apistrat <- stratnar
  imputed_apistrat$api00_imp <- imputed_apistrat$api00
  imputed_apistrat$api00_imp[is.na(imputed_apistrat$api00)] <- api00_imp + rnorm(n = length(resid_std), sd = resid_std)
  imputed_apistrat$pw <- apistrat$pw
  return(imputed_apistrat)
}))

dstrat_nar_MI <- svydesign(id = ~1, strata = ~stype, weights = ~pw,
                    data = MI_apistrat, fpc = ~fpc)

summary(mitools::MIcombine(with(dstrat_nar_MI,
     svymean(~api00_imp))))
```

::: {.cell-output .cell-output-stdout}
```
Multiple imputation results:
      with(dstrat_nar_MI, svymean(~api00_imp))
      MIcombine.default(with(dstrat_nar_MI, svymean(~api00_imp)))
          results       se   (lower  upper) missInfo
api00_imp 662.195 9.591989 643.3949 680.995      1 %
```
:::
:::


For the "Not Missing at Random" data set, we see a similar mean and standard
error as before, which implies that the missing mechanism must not have been
particularly impactful as determined by the imputed datasets.


::: {.cell}

```{.r .cell-code}
api00_imp <- predict(imputation_model_one, newdata = stratmar[is.na(stratmar$api00),])
X_miss <- model.matrix(~api99 + emer + stype, data = stratmar[is.na(stratmar$api00),])
resid_std <- diag(X_miss %*% 
                    vcov(imputation_model_one) %*% 
                    t(X_miss))
MI_apistrat <- mitools::imputationList(lapply(1:10, function(ix) {
  imputed_apistrat <- stratmar
  imputed_apistrat$api00_imp <- imputed_apistrat$api00
  imputed_apistrat$api00_imp[is.na(imputed_apistrat$api00)] <- api00_imp + rnorm(n = length(resid_std), sd = resid_std)
  imputed_apistrat$pw <- apistrat$pw
  return(imputed_apistrat)
}))

dstrat_mar_MI <- svydesign(id = ~1, strata = ~stype, weights = ~pw,
                    data = MI_apistrat, fpc = ~fpc)

summary(mitools::MIcombine(with(dstrat_mar_MI,
     svymean(~api00_imp))))
```

::: {.cell-output .cell-output-stdout}
```
Multiple imputation results:
      with(dstrat_mar_MI, svymean(~api00_imp))
      MIcombine.default(with(dstrat_mar_MI, svymean(~api00_imp)))
           results       se   (lower  upper) missInfo
api00_imp 664.4089 9.760303 645.2788 683.539      1 %
```
:::
:::


We see a similar result as above, though again the mean estimate is slightly
closer to the true population value. 

# Chapter 10: Causal Inference

The 10th and final chapter of Lumley's book concerns itself with causal 
inference in the context of survey data. Lumley starts by identifying an 
important link to 
[randomized trials](https://en.wikipedia.org/wiki/Randomized_experiment), noting
that design based inference is also called *randomization inference*. 

From this point Lumley gives a brief outline of the potential outcomes framework
also called the 
[Rubin causal model](https://en.wikipedia.org/wiki/Rubin_causal_model). 

In a conventional randomized trial sampling probabilities to one of two of the
realized outcomes happens with equal probability, $\pi = \frac{1}{2}$ and so
there's no need to discuss the ideas from unequal probability sampling. However,
in an observational setting, randomization to treatments does not occur. 
Instead, Lumley notes that similar to how missing data can be thought of as
occuring due to some sampling mechanism, exposure can be framed similarly.

Lumley gives the example of an individual in a data set having some probability
of smoking or not smoking and the outcome of interest being the average 
difference in some health outcome between smoking and non-smoking groups, 
weighted by this probability. Whereas the target population previously was some 
well defined finite set, now we're considering the target population defined by 
the sampling probabilities to be the population of potential outcomes between 
smoking and not smoking.

As Lumley notes, we can think about how this might tell us something about the
*causal* mechanism between smoking and a health outcome, and/or the groundwork
for a certain type of estimator more broadly.

## IPTW Estimators

> The Inverse Probability of Treatment Weighted (IPTW) estimator is the analog
for potential-outcomes inference of the Horvitz-Thompson estimator.


### Randomized trials and calibration

Similar to how a designed sample has *known* sampling probabilities a randomized
trial does as well. Typically $\pi_{i} = \frac{1}{2}$ for assignment for each
individual to the treatment or control group. In this case, no weighting is 
needed at all in the estimation, since the weights are the same for each
individual.

Lumley then goes on to give a brief discussion of the relationship between
calibration and regression when used in a randomized trial with some covariate 
$X$ that is meaningfully associated with the outcome. A few brief notes
here to summarize:

* For a difference in means, the calibration and linear estimates are the same.
* When estimating a hazard, log-odds, or rate ratios they are not
  * The calibration estimator is still equivalent to the un-adjusted estimator.
  * The regression estimator is now different. I believe it would be a 
    conditional average treatment effect estimate.
  * Lumley shows with simulation that the calibration estimate has higher 
  precision then the un-adjusted estimate, and equivalently high precision to the
  adjusted.
  
Personally, I'd keep in mind that the conditional/adjusted estimate may be more 
interpretable than the un-adjusted - though the un-adjusted is often what is 
sought after by a journal, for example. Deciding which is more useful is a 
philosophical discussion, highly dependent on the question being asked.

### Estimated Weights for IPTW

Analogous to how weights (or values) had to be estimated for missing data in the
previous chapter, sampling weights have to be estimated for observational data
when examining the impact of an exposure on an outcome of interest.

Lumley uses an example from a paper [@tager1979effect] examining the impact of
smoking habits on pulmonary function amongst a sample of 654 childrenn from 
Boston in the 1970's.  I can't find the data accessible through Lumley's book 
[website](https://r-survey.r-forge.r-project.org/svybook/) but I did find the 
data set via the 
[`mplot` package](https://search.r-project.org/CRAN/refmans/mplot/html/fev.html).
However, this dataset does not have the `siblings` or `family` variables that 
Lumley uses in his analysis to adjust for the cluster sampling and family size.
I proceed assuming a simple random sample without any cluster estimates.

Lumley plots lung function (forced expiratory volume or FEV) against age and
childrens' smoking status and identifies that although there's a naive 
difference between smoking and non smoking children
confounded by the fact that older children --- 
who have higher FEV by virtue of age --- are more likely to smoke than those 
who aren't.


::: {.cell}

```{.r .cell-code}
fev <- mplot::fev
t.test(fev ~ smoke, data = fev)
```

::: {.cell-output .cell-output-stdout}
```

	Welch Two Sample t-test

data:  fev by smoke
t = -7.1496, df = 83.273, p-value = 3.074e-10
alternative hypothesis: true difference in means between group 0 and group 1 is not equal to 0
95 percent confidence interval:
 -0.9084253 -0.5130126
sample estimates:
mean in group 0 mean in group 1 
       2.566143        3.276862 
```
:::
:::

::: {.cell}

```{.r .cell-code}
boxplot(fev ~ smoke, data = fev, horizontal = TRUE,
        xlab = "FEV (L/S)", 
        main = "Forced Expiratory Volume and Smoking Status for 654 children")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/iptw_demo_boxplot-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
as_tibble(fev) %>% 
  mutate(smoke = factor(smoke, labels = c("Non-Smoking",
                                          "Smoking"))) %>% 
  ggplot(aes(x = age, y = fev, color = smoke)) +
  geom_point() + 
  geom_jitter() +
  theme(legend.title = element_blank())
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/iptw_demo_age_plot-1.png){width=672}
:::
:::


An IPTW analysis proceeds by estimating the probability of exposure --- 
smoking in this case --- as a function of other covariates. I fit three models
below following Lumley's example as close as I can and setting up design 
objects with sampling probabilities equal to the estimated probability of 
smoking or not smoking according to the model and the observed smoking status.

It is worth highlighting here again, the analogy between sampling from a finite
population of fixed individuals used in previous chapters, and how that 
corresponds to our example here, where we're hypothetically sampling from a 
population according to smoking status and using that to estimate the impact
of smoking on FEV.

A further point, that I brought up in the previous chapter, is that we're not
incorporating any of our uncertainty about the probability of smoking into the
model estimating the effect of smoking on FEV. Consequently the variance in the
reported models is under-estimated.


::: {.cell}

```{.r .cell-code}
olderkids <- subset(fev, age > 12)
m1 <- glm(smoke ~ age*sex, data = olderkids, family = binomial())
m2 <- glm(smoke ~ height*sex, data = olderkids, family = binomial())
m3 <- glm(smoke ~ age*sex + height*sex, data = olderkids, family = binomial())
olderkids$pi1 <- ifelse(olderkids$smoke == 1, fitted(m1), 1 - fitted(m1))
olderkids$pi2 <- ifelse(olderkids$smoke == 1, fitted(m2), 1 - fitted(m2))
olderkids$pi3 <- ifelse(olderkids$smoke == 1, fitted(m3), 1 - fitted(m3))

d1 <- svydesign(id = ~1, prob = ~pi1, data = olderkids)
d2 <- svydesign(id = ~1, prob = ~pi2, data = olderkids)
d3 <- svydesign(id = ~1, prob = ~pi3, data = olderkids)
fit1 <- svyglm(fev~smoke, design = d1)
fit2 <- svyglm(fev~smoke, design = d2)
fit3 <- svyglm(fev~smoke, design = d3)

tbl_merge(
  tbls = list(
    tbl_regression(fit1),
    tbl_regression(fit2),
    tbl_regression(fit3)
  ),
  tab_spanner = c("**Model 1**", "**Model 2**","**Model 3**")
)
```

::: {.cell-output-display}
```{=html}
<div id="zdsvucynny" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#zdsvucynny table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#zdsvucynny thead, #zdsvucynny tbody, #zdsvucynny tfoot, #zdsvucynny tr, #zdsvucynny td, #zdsvucynny th {
  border-style: none;
}

#zdsvucynny p {
  margin: 0;
  padding: 0;
}

#zdsvucynny .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#zdsvucynny .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#zdsvucynny .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#zdsvucynny .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#zdsvucynny .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#zdsvucynny .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#zdsvucynny .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#zdsvucynny .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#zdsvucynny .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#zdsvucynny .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#zdsvucynny .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#zdsvucynny .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#zdsvucynny .gt_spanner_row {
  border-bottom-style: hidden;
}

#zdsvucynny .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#zdsvucynny .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#zdsvucynny .gt_from_md > :first-child {
  margin-top: 0;
}

#zdsvucynny .gt_from_md > :last-child {
  margin-bottom: 0;
}

#zdsvucynny .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#zdsvucynny .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#zdsvucynny .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#zdsvucynny .gt_row_group_first td {
  border-top-width: 2px;
}

#zdsvucynny .gt_row_group_first th {
  border-top-width: 2px;
}

#zdsvucynny .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#zdsvucynny .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#zdsvucynny .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#zdsvucynny .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#zdsvucynny .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#zdsvucynny .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#zdsvucynny .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#zdsvucynny .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#zdsvucynny .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#zdsvucynny .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#zdsvucynny .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#zdsvucynny .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#zdsvucynny .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#zdsvucynny .gt_left {
  text-align: left;
}

#zdsvucynny .gt_center {
  text-align: center;
}

#zdsvucynny .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#zdsvucynny .gt_font_normal {
  font-weight: normal;
}

#zdsvucynny .gt_font_bold {
  font-weight: bold;
}

#zdsvucynny .gt_font_italic {
  font-style: italic;
}

#zdsvucynny .gt_super {
  font-size: 65%;
}

#zdsvucynny .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#zdsvucynny .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#zdsvucynny .gt_indent_1 {
  text-indent: 5px;
}

#zdsvucynny .gt_indent_2 {
  text-indent: 10px;
}

#zdsvucynny .gt_indent_3 {
  text-indent: 15px;
}

#zdsvucynny .gt_indent_4 {
  text-indent: 20px;
}

#zdsvucynny .gt_indent_5 {
  text-indent: 25px;
}

#zdsvucynny .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#zdsvucynny div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipNb2RlbCAxKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Model 1&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipNb2RlbCAxKio="><div class='gt_from_md'><p><strong>Model 1</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipNb2RlbCAyKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Model 2&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipNb2RlbCAyKio="><div class='gt_from_md'><p><strong>Model 2</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipNb2RlbCAzKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Model 3&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipNb2RlbCAzKio="><div class='gt_from_md'><p><strong>Model 3</strong></p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">smoke</td>
<td headers="estimate_1" class="gt_row gt_center">-0.14</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.47, 0.20</td>
<td headers="p.value_1" class="gt_row gt_center">0.4</td>
<td headers="estimate_2" class="gt_row gt_center">-0.10</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.41, 0.21</td>
<td headers="p.value_2" class="gt_row gt_center">0.5</td>
<td headers="estimate_3" class="gt_row gt_center">-0.10</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.42, 0.23</td>
<td headers="p.value_3" class="gt_row gt_center">0.6</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="10"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="Q0kgPSBDb25maWRlbmNlIEludGVydmFs"><div class='gt_from_md'><p>CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


In the table above, we now see that mean FEV value is now negatively associated
with smoking, but none of the estimates are statistically significant. 

Lumley shows that there are similar results for an un-weighted analysis and 
argues that the IPTW analysis is valuable for providing some justification for
believing this effect more generally.

### Double robustness

Lumley brings up the idea of a double robust estimator again, this time 
showing the estimates from a double robust estimation analysis whereby the 
covariates used in the probability or propensity model are included in the 
second stage model as well. This allows us to include even younger children
in the final analysis and derive a similar conclusion as when they were
excluded.


::: {.cell}

```{.r .cell-code}
olderkids <- subset(fev, age > 10)
m1 <- glm(smoke ~ age*sex, data = olderkids, family = binomial())
m2 <- glm(smoke ~ height*sex, data = olderkids, family = binomial())
m3 <- glm(smoke ~ age*sex + height*sex, data = olderkids, family = binomial())
olderkids$pi1 <- ifelse(olderkids$smoke == 1, fitted(m1), 1 - fitted(m1))
olderkids$pi2 <- ifelse(olderkids$smoke == 1, fitted(m2), 1 - fitted(m2))
olderkids$pi3 <- ifelse(olderkids$smoke == 1, fitted(m3), 1 - fitted(m3))
fit1 <- svyglm(fev~smoke + height*sex + age*sex, design = d1)
fit2 <- svyglm(fev~smoke + height*sex + age*sex, design = d2)
fit3 <- svyglm(fev~smoke + height*sex + age*sex, design = d3)

tbl_merge(
  tbls = list(
    tbl_regression(fit1),
    tbl_regression(fit2),
    tbl_regression(fit3)
  ),
  tab_spanner = c("**DR Model 1**", "**DR Model 2**","**DR Model 3**")
)
```

::: {.cell-output-display}
```{=html}
<div id="oqlpayxcoj" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#oqlpayxcoj table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#oqlpayxcoj thead, #oqlpayxcoj tbody, #oqlpayxcoj tfoot, #oqlpayxcoj tr, #oqlpayxcoj td, #oqlpayxcoj th {
  border-style: none;
}

#oqlpayxcoj p {
  margin: 0;
  padding: 0;
}

#oqlpayxcoj .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#oqlpayxcoj .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#oqlpayxcoj .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#oqlpayxcoj .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#oqlpayxcoj .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#oqlpayxcoj .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#oqlpayxcoj .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#oqlpayxcoj .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#oqlpayxcoj .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#oqlpayxcoj .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#oqlpayxcoj .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#oqlpayxcoj .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#oqlpayxcoj .gt_spanner_row {
  border-bottom-style: hidden;
}

#oqlpayxcoj .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#oqlpayxcoj .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#oqlpayxcoj .gt_from_md > :first-child {
  margin-top: 0;
}

#oqlpayxcoj .gt_from_md > :last-child {
  margin-bottom: 0;
}

#oqlpayxcoj .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#oqlpayxcoj .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#oqlpayxcoj .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#oqlpayxcoj .gt_row_group_first td {
  border-top-width: 2px;
}

#oqlpayxcoj .gt_row_group_first th {
  border-top-width: 2px;
}

#oqlpayxcoj .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#oqlpayxcoj .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#oqlpayxcoj .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#oqlpayxcoj .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#oqlpayxcoj .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#oqlpayxcoj .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#oqlpayxcoj .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#oqlpayxcoj .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#oqlpayxcoj .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#oqlpayxcoj .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#oqlpayxcoj .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#oqlpayxcoj .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#oqlpayxcoj .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#oqlpayxcoj .gt_left {
  text-align: left;
}

#oqlpayxcoj .gt_center {
  text-align: center;
}

#oqlpayxcoj .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#oqlpayxcoj .gt_font_normal {
  font-weight: normal;
}

#oqlpayxcoj .gt_font_bold {
  font-weight: bold;
}

#oqlpayxcoj .gt_font_italic {
  font-style: italic;
}

#oqlpayxcoj .gt_super {
  font-size: 65%;
}

#oqlpayxcoj .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#oqlpayxcoj .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#oqlpayxcoj .gt_indent_1 {
  text-indent: 5px;
}

#oqlpayxcoj .gt_indent_2 {
  text-indent: 10px;
}

#oqlpayxcoj .gt_indent_3 {
  text-indent: 15px;
}

#oqlpayxcoj .gt_indent_4 {
  text-indent: 20px;
}

#oqlpayxcoj .gt_indent_5 {
  text-indent: 25px;
}

#oqlpayxcoj .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#oqlpayxcoj div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_col_headings gt_spanner_row">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="2" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipDaGFyYWN0ZXJpc3RpYyoq&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Characteristic&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipDaGFyYWN0ZXJpc3RpYyoq"><div class='gt_from_md'><p><strong>Characteristic</strong></p>
</div></div></th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipEUiBNb2RlbCAxKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;DR Model 1&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipEUiBNb2RlbCAxKio="><div class='gt_from_md'><p><strong>DR Model 1</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipEUiBNb2RlbCAyKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;DR Model 2&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipEUiBNb2RlbCAyKio="><div class='gt_from_md'><p><strong>DR Model 2</strong></p>
</div></div></span>
      </th>
      <th class="gt_center gt_columns_top_border gt_column_spanner_outer" rowspan="1" colspan="3" scope="colgroup" id="&lt;div data-qmd-base64=&quot;KipEUiBNb2RlbCAzKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;DR Model 3&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;">
        <span class="gt_column_spanner"><div data-qmd-base64="KipEUiBNb2RlbCAzKio="><div class='gt_from_md'><p><strong>DR Model 3</strong></p>
</div></div></span>
      </th>
    </tr>
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipCZXRhKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;Beta&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipCZXRhKio="><div class='gt_from_md'><p><strong>Beta</strong></p>
</div></div></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;Kio5NSUgQ0kqKg==&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;95% CI&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;&lt;span class=&quot;gt_footnote_marks&quot; style=&quot;white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;&quot;&gt;&lt;sup&gt;1&lt;/sup&gt;&lt;/span&gt;"><div data-qmd-base64="Kio5NSUgQ0kqKg=="><div class='gt_from_md'><p><strong>95% CI</strong></p>
</div></div><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span></th>
      <th class="gt_col_heading gt_columns_bottom_border gt_center" rowspan="1" colspan="1" scope="col" id="&lt;div data-qmd-base64=&quot;KipwLXZhbHVlKio=&quot;&gt;&lt;div class='gt_from_md'&gt;&lt;p&gt;&lt;strong&gt;p-value&lt;/strong&gt;&lt;/p&gt;&#10;&lt;/div&gt;&lt;/div&gt;"><div data-qmd-base64="KipwLXZhbHVlKio="><div class='gt_from_md'><p><strong>p-value</strong></p>
</div></div></th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="label" class="gt_row gt_left">smoke</td>
<td headers="estimate_1" class="gt_row gt_center">-0.08</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.29, 0.12</td>
<td headers="p.value_1" class="gt_row gt_center">0.4</td>
<td headers="estimate_2" class="gt_row gt_center">-0.10</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.30, 0.10</td>
<td headers="p.value_2" class="gt_row gt_center">0.3</td>
<td headers="estimate_3" class="gt_row gt_center">-0.08</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.29, 0.12</td>
<td headers="p.value_3" class="gt_row gt_center">0.4</td></tr>
    <tr><td headers="label" class="gt_row gt_left">height</td>
<td headers="estimate_1" class="gt_row gt_center">0.03</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.03, 0.08</td>
<td headers="p.value_1" class="gt_row gt_center">0.3</td>
<td headers="estimate_2" class="gt_row gt_center">0.03</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.02, 0.09</td>
<td headers="p.value_2" class="gt_row gt_center">0.3</td>
<td headers="estimate_3" class="gt_row gt_center">0.03</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.03, 0.08</td>
<td headers="p.value_3" class="gt_row gt_center">0.3</td></tr>
    <tr><td headers="label" class="gt_row gt_left">sex</td>
<td headers="estimate_1" class="gt_row gt_center">-7.5</td>
<td headers="conf.low_1" class="gt_row gt_center">-13, -1.5</td>
<td headers="p.value_1" class="gt_row gt_center">0.015</td>
<td headers="estimate_2" class="gt_row gt_center">-7.3</td>
<td headers="conf.low_2" class="gt_row gt_center">-13, -2.1</td>
<td headers="p.value_2" class="gt_row gt_center">0.007</td>
<td headers="estimate_3" class="gt_row gt_center">-7.1</td>
<td headers="conf.low_3" class="gt_row gt_center">-13, -1.5</td>
<td headers="p.value_3" class="gt_row gt_center">0.014</td></tr>
    <tr><td headers="label" class="gt_row gt_left">age</td>
<td headers="estimate_1" class="gt_row gt_center">0.01</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.05, 0.07</td>
<td headers="p.value_1" class="gt_row gt_center">0.8</td>
<td headers="estimate_2" class="gt_row gt_center">0.01</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.05, 0.07</td>
<td headers="p.value_2" class="gt_row gt_center">0.7</td>
<td headers="estimate_3" class="gt_row gt_center">0.01</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.05, 0.07</td>
<td headers="p.value_3" class="gt_row gt_center">0.8</td></tr>
    <tr><td headers="label" class="gt_row gt_left">height * sex</td>
<td headers="estimate_1" class="gt_row gt_center">0.12</td>
<td headers="conf.low_1" class="gt_row gt_center">0.03, 0.20</td>
<td headers="p.value_1" class="gt_row gt_center">0.008</td>
<td headers="estimate_2" class="gt_row gt_center">0.11</td>
<td headers="conf.low_2" class="gt_row gt_center">0.04, 0.19</td>
<td headers="p.value_2" class="gt_row gt_center">0.004</td>
<td headers="estimate_3" class="gt_row gt_center">0.11</td>
<td headers="conf.low_3" class="gt_row gt_center">0.03, 0.19</td>
<td headers="p.value_3" class="gt_row gt_center">0.008</td></tr>
    <tr><td headers="label" class="gt_row gt_left">sex * age</td>
<td headers="estimate_1" class="gt_row gt_center">0.03</td>
<td headers="conf.low_1" class="gt_row gt_center">-0.08, 0.14</td>
<td headers="p.value_1" class="gt_row gt_center">0.6</td>
<td headers="estimate_2" class="gt_row gt_center">0.03</td>
<td headers="conf.low_2" class="gt_row gt_center">-0.08, 0.14</td>
<td headers="p.value_2" class="gt_row gt_center">0.6</td>
<td headers="estimate_3" class="gt_row gt_center">0.03</td>
<td headers="conf.low_3" class="gt_row gt_center">-0.08, 0.14</td>
<td headers="p.value_3" class="gt_row gt_center">0.6</td></tr>
  </tbody>
  
  <tfoot class="gt_footnotes">
    <tr>
      <td class="gt_footnote" colspan="10"><span class="gt_footnote_marks" style="white-space:nowrap;font-style:italic;font-weight:normal;line-height: 0;"><sup>1</sup></span> <div data-qmd-base64="Q0kgPSBDb25maWRlbmNlIEludGVydmFs"><div class='gt_from_md'><p>CI = Confidence Interval</p>
</div></div></td>
    </tr>
  </tfoot>
</table>
</div>
```
:::
:::


## Marginal Structural Models

Lumley begins this section with an important point:

> For estimating effects of a single binary exposure there is not much to choose
between IPTW estimators and regression estimators. Both will be valid if there 
are no unmeasured confounders, and any measured confounder can equally well be 
used in adjustment, in reweighting, or both. 

However, when considering repeated measures over time, IPTW estimators have
a significant advantage in modeling the relationship between outcome and
exposure more explicitly. After all, exposure measured at time point 1 may well
impact the outcome at *all* subsequent time points.

Thankfully, using the IPTW approach, we can proceed similarly as before, 
estimating the potential outcome for each possible exposure pathway. The 
propensity scores or probabilities can then be multiplied to give the potential
outcomes sequence and the sampling probability used will be for the sequence
observed.

**Example: maternal stress and children's illness**

I obtained the `mscm` dataset from Lumley's 
[website](https://r-survey.r-forge.r-project.org/svybook/), who in turn obtained
it from Patrick Heagerty's 
[website](https://faculty.washington.edu/heagerty/Books/AnalysisLongitudinal/)
which includes 
[documentation](https://faculty.washington.edu/heagerty/Books/AnalysisLongitudinal/mscm.txt)
I would encourage looking over.

This data set comes from the The Mothers' and Children's Morbidity Study which
examined how maternal stress might impact childhood illness. Stress and illness
were measured for 30 days on 167 mother-child pairs on a 0-1 scale. 

I reproduce Lumley's analysis in the code below examining the effect of a 
mother's stress on her child's illness. 

Lumley begins by filtering the data to only include mother-child pairs with
28 non-missing observations and ~ 3 weeks of data.

I try to reproduce Lumley's plots below but I think he may have produced them
before filtering or with alternative filtering in place, as we see a similar
trend in both plots but different absolute estimates.


::: {.cell}

```{.r .cell-code}
mscm <- read_table("data/mscm.txt",col_names = c("id","day","stress", paste("s",1:6,sep=""),
                 "illness", paste("i",1:6,sep=""),
                 "married","education","employed","chlth","mhlth","race",
                 "csex","hsize","wk1illness","wk1stress"))
mscm$nna <- with(mscm, ave(stress + illness, id, FUN = function(x) sum(!is.na(x))))
mscm <- subset(mscm, nna == 28 & day > 6 & day < 29)
```
:::

::: {.cell}

```{.r .cell-code}
# I try to reproduce Figure 10.5 here. While it's not the same as Lumley's
# plot, we do see a similar trend.
unwt <- svydesign(id = ~id, data = mscm)
eda_fit <- svyglm(i3 ~ s1 + s2 + s3 + s4 + s5 + s6 - 1, family = quasibinomial,
               design = unwt)
broom::tidy(eda_fit, conf.int = TRUE) %>% 
  mutate(time_offset = as.integer(str_replace(term,"s","")) - 3) %>% 
  ggplot(aes(x = time_offset, y = estimate)) + 
  geom_pointrange(aes(ymin = conf.low, ymax = conf.high)) + 
  xlab("Time Offset") +
  ylab("Log Odds Ratio") + 
  geom_vline(aes(xintercept = 0), linetype = 2, color = 'red') +
  ggtitle("Childhood Illness and Mother's Stress",
          subtitle = "Log Odds Ratio of Illness vs. Stress Over Time")
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/msm_plot_one-1.png){width=672}
:::
:::

::: {.cell}

```{.r .cell-code}
# I try to reproduce Figure 10.6 here. Again, this isn't the same as Lumley's
# plot but we see a similar trend. I'm guessing he might not've filtered 
unwt <- svydesign(id = ~id, data = mscm)
ill_eda <- svyglm(illness ~ i1 + i2 + i3 + i4 + i5 + i6 - 1, family = quasibinomial,
               design = unwt)
stress_eda <- svyglm(stress ~ s1 + s2 + s3 + s4 + s5 + s6 - 1, family = quasibinomial,
               design = unwt)
broom::tidy(ill_eda, conf.int = TRUE) %>% 
  mutate(model = "illness") %>% 
  rbind(.,tidy(stress_eda, conf.int = TRUE) %>% 
          mutate(model = "stress")) %>% 
  mutate(time_lag = as.integer(str_replace(term,"i|s",""))) %>% 
  ggplot(aes(x = time_lag, y = estimate, color = model)) + 
  geom_pointrange(aes(ymin = conf.low, ymax = conf.high)) + 
  geom_line() +
  xlab("Time Lag") +
  ylab("Log Odds Ratio") + 
  ggtitle("Log Odds Ratios of Childhood Illness and Mother's Stress 
          for different time lags.") +
  theme(legend.title = element_blank())
```

::: {.cell-output-display}
![](ComplexSurveyNotes_files/figure-html/msm_plot_two-1.png){width=672}
:::
:::



To analyze the data, Lumley produces stabilized weights by averaging two 
model's respective probabilities of stress. A model (1) that includes the 
child's current and previous state of illness (which might cause subsequent 
stress on the mother's part), as well as the current day in the study and 
baselines stress and a second (2) that looks only at the baseline stress and 
change in mother's stress over time.

When Lumley talks about separating the exposure model from the outcome without
having to worry about self-caused confounding --- child's illness inducing 
subsequent maternal stress --- this is what he means. Using both probabilities 
increases the variability of the weights without inducing any potential 
confounding.

The two design objects include id as the sampling unit to account for 
within-person correlation. As a side note, I'm wondering how this approach 
compares to, say, a GEE or linear mixed effects model's adjustment for within
person correlation in measurement. Since we end up fitting regression models 
that are effectively GEE models I'm guessing those are roughly equivalent.

::: {.cell}

```{.r .cell-code}
model <- glm(stress ~ illness + i1 + s1 + day + wk1stress, family = binomial(),
             data = mscm, na.action = na.exclude)
mscm$pstress <- fitted(model)
basemodel <- glm(stress ~ day + wk1stress, family = binomial, 
                 data = mscm, na.action = na.exclude)
mscm$pbstress <- fitted(basemodel)
mscm$swit <- with(mscm, ifelse(stress == 1, pbstress / pstress,
                               ( 1 - pbstress) / (1-pstress)))
mscm$swi <- with(mscm, ave(swit, id, FUN = prod))

des <- svydesign(id = ~id, weights = ~swi, data = mscm)
unwt <- svydesign(id = ~id, data = mscm)
```
:::



Lumley then fits three outcome models. One uses the stabilized sampling weights,
one uses no weights, and then a third uses no weights, but includes the 
baseline variable adjustment. The estimates and standard errors are in the table
below.

::: {.cell}

```{.r .cell-code}
m_iptw <- svyglm(illness ~ s1 + s2 + s3 + s4, design = des,
                 family = quasibinomial)
m_raw <- svyglm(illness ~ s1 + s2 + s3 + s4, des = unwt,
                family = quasibinomial)

m_gee <- svyglm(illness ~ s1+s2+s3+s4+wk1stress+wk1illness, des = unwt,
                family = quasibinomial)

pred_data <- data.frame(s1 = 0:1, s2 = 0:1, s3 = 0:1, s4 = 0:1,
                        wk1stress = 0.16, wk1illness = 0.16)

pred_iptw <- predict(m_iptw, pred_data, vcov = TRUE, type = "response")
pred_raw <- predict(m_raw, pred_data, vcov = TRUE, type = "response")
pred_gee <- predict(m_gee, pred_data, vcov = TRUE, type = "response")

# Here Lumley's looking at the difference in log odds of a child being ill 
# if its mother was stressed the previous four days and had (I believe) average
# baseline stress and illness, and if the mother did *not* stress the previous
# four days, but still had the same baseline variables. i.e. Is the mother's
# stress associated with a change in the odds of her child's illness.
c_iptw <- svycontrast(pred_iptw, c(-1, 1))
c_raw <- svycontrast(pred_raw, c(-1, 1))
c_gee <- svycontrast(pred_gee, c(-1, 1))
tibble(model = c("IPTW", "Unadjusted", "Baseline Adjusted"),
       estimate = c(c_iptw, c_raw, c_gee),
       std_err = c(attr(c_iptw,"var"), attr(c_raw,"var"),
                   attr(c_gee,"var"))) %>% 
  gt::gt() %>% 
  gt::fmt_number(decimals = 3) %>% 
  gt::tab_header("Estimate of Childhood Illness from Mother's Stress")
```

::: {.cell-output-display}
```{=html}
<div id="ysdqfdafyn" style="padding-left:0px;padding-right:0px;padding-top:10px;padding-bottom:10px;overflow-x:auto;overflow-y:auto;width:auto;height:auto;">
<style>#ysdqfdafyn table {
  font-family: system-ui, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji', 'Segoe UI Symbol', 'Noto Color Emoji';
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

#ysdqfdafyn thead, #ysdqfdafyn tbody, #ysdqfdafyn tfoot, #ysdqfdafyn tr, #ysdqfdafyn td, #ysdqfdafyn th {
  border-style: none;
}

#ysdqfdafyn p {
  margin: 0;
  padding: 0;
}

#ysdqfdafyn .gt_table {
  display: table;
  border-collapse: collapse;
  line-height: normal;
  margin-left: auto;
  margin-right: auto;
  color: #333333;
  font-size: 16px;
  font-weight: normal;
  font-style: normal;
  background-color: #FFFFFF;
  width: auto;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #A8A8A8;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #A8A8A8;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
}

#ysdqfdafyn .gt_caption {
  padding-top: 4px;
  padding-bottom: 4px;
}

#ysdqfdafyn .gt_title {
  color: #333333;
  font-size: 125%;
  font-weight: initial;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-color: #FFFFFF;
  border-bottom-width: 0;
}

#ysdqfdafyn .gt_subtitle {
  color: #333333;
  font-size: 85%;
  font-weight: initial;
  padding-top: 3px;
  padding-bottom: 5px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-color: #FFFFFF;
  border-top-width: 0;
}

#ysdqfdafyn .gt_heading {
  background-color: #FFFFFF;
  text-align: center;
  border-bottom-color: #FFFFFF;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ysdqfdafyn .gt_bottom_border {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ysdqfdafyn .gt_col_headings {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
}

#ysdqfdafyn .gt_col_heading {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 6px;
  padding-left: 5px;
  padding-right: 5px;
  overflow-x: hidden;
}

#ysdqfdafyn .gt_column_spanner_outer {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: normal;
  text-transform: inherit;
  padding-top: 0;
  padding-bottom: 0;
  padding-left: 4px;
  padding-right: 4px;
}

#ysdqfdafyn .gt_column_spanner_outer:first-child {
  padding-left: 0;
}

#ysdqfdafyn .gt_column_spanner_outer:last-child {
  padding-right: 0;
}

#ysdqfdafyn .gt_column_spanner {
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: bottom;
  padding-top: 5px;
  padding-bottom: 5px;
  overflow-x: hidden;
  display: inline-block;
  width: 100%;
}

#ysdqfdafyn .gt_spanner_row {
  border-bottom-style: hidden;
}

#ysdqfdafyn .gt_group_heading {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  text-align: left;
}

#ysdqfdafyn .gt_empty_group_heading {
  padding: 0.5px;
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  vertical-align: middle;
}

#ysdqfdafyn .gt_from_md > :first-child {
  margin-top: 0;
}

#ysdqfdafyn .gt_from_md > :last-child {
  margin-bottom: 0;
}

#ysdqfdafyn .gt_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  margin: 10px;
  border-top-style: solid;
  border-top-width: 1px;
  border-top-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 1px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 1px;
  border-right-color: #D3D3D3;
  vertical-align: middle;
  overflow-x: hidden;
}

#ysdqfdafyn .gt_stub {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
}

#ysdqfdafyn .gt_stub_row_group {
  color: #333333;
  background-color: #FFFFFF;
  font-size: 100%;
  font-weight: initial;
  text-transform: inherit;
  border-right-style: solid;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
  padding-left: 5px;
  padding-right: 5px;
  vertical-align: top;
}

#ysdqfdafyn .gt_row_group_first td {
  border-top-width: 2px;
}

#ysdqfdafyn .gt_row_group_first th {
  border-top-width: 2px;
}

#ysdqfdafyn .gt_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ysdqfdafyn .gt_first_summary_row {
  border-top-style: solid;
  border-top-color: #D3D3D3;
}

#ysdqfdafyn .gt_first_summary_row.thick {
  border-top-width: 2px;
}

#ysdqfdafyn .gt_last_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ysdqfdafyn .gt_grand_summary_row {
  color: #333333;
  background-color: #FFFFFF;
  text-transform: inherit;
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
}

#ysdqfdafyn .gt_first_grand_summary_row {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-top-style: double;
  border-top-width: 6px;
  border-top-color: #D3D3D3;
}

#ysdqfdafyn .gt_last_grand_summary_row_top {
  padding-top: 8px;
  padding-bottom: 8px;
  padding-left: 5px;
  padding-right: 5px;
  border-bottom-style: double;
  border-bottom-width: 6px;
  border-bottom-color: #D3D3D3;
}

#ysdqfdafyn .gt_striped {
  background-color: rgba(128, 128, 128, 0.05);
}

#ysdqfdafyn .gt_table_body {
  border-top-style: solid;
  border-top-width: 2px;
  border-top-color: #D3D3D3;
  border-bottom-style: solid;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
}

#ysdqfdafyn .gt_footnotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ysdqfdafyn .gt_footnote {
  margin: 0px;
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ysdqfdafyn .gt_sourcenotes {
  color: #333333;
  background-color: #FFFFFF;
  border-bottom-style: none;
  border-bottom-width: 2px;
  border-bottom-color: #D3D3D3;
  border-left-style: none;
  border-left-width: 2px;
  border-left-color: #D3D3D3;
  border-right-style: none;
  border-right-width: 2px;
  border-right-color: #D3D3D3;
}

#ysdqfdafyn .gt_sourcenote {
  font-size: 90%;
  padding-top: 4px;
  padding-bottom: 4px;
  padding-left: 5px;
  padding-right: 5px;
}

#ysdqfdafyn .gt_left {
  text-align: left;
}

#ysdqfdafyn .gt_center {
  text-align: center;
}

#ysdqfdafyn .gt_right {
  text-align: right;
  font-variant-numeric: tabular-nums;
}

#ysdqfdafyn .gt_font_normal {
  font-weight: normal;
}

#ysdqfdafyn .gt_font_bold {
  font-weight: bold;
}

#ysdqfdafyn .gt_font_italic {
  font-style: italic;
}

#ysdqfdafyn .gt_super {
  font-size: 65%;
}

#ysdqfdafyn .gt_footnote_marks {
  font-size: 75%;
  vertical-align: 0.4em;
  position: initial;
}

#ysdqfdafyn .gt_asterisk {
  font-size: 100%;
  vertical-align: 0;
}

#ysdqfdafyn .gt_indent_1 {
  text-indent: 5px;
}

#ysdqfdafyn .gt_indent_2 {
  text-indent: 10px;
}

#ysdqfdafyn .gt_indent_3 {
  text-indent: 15px;
}

#ysdqfdafyn .gt_indent_4 {
  text-indent: 20px;
}

#ysdqfdafyn .gt_indent_5 {
  text-indent: 25px;
}

#ysdqfdafyn .katex-display {
  display: inline-flex !important;
  margin-bottom: 0.75em !important;
}

#ysdqfdafyn div.Reactable > div.rt-table > div.rt-thead > div.rt-tr.rt-tr-group-header > div.rt-th-group:after {
  height: 0px !important;
}
</style>
<table class="gt_table" data-quarto-disable-processing="false" data-quarto-bootstrap="false">
  <thead>
    <tr class="gt_heading">
      <td colspan="3" class="gt_heading gt_title gt_font_normal gt_bottom_border" style>Estimate of Childhood Illness from Mother's Stress</td>
    </tr>
    
    <tr class="gt_col_headings">
      <th class="gt_col_heading gt_columns_bottom_border gt_left" rowspan="1" colspan="1" scope="col" id="model">model</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="estimate">estimate</th>
      <th class="gt_col_heading gt_columns_bottom_border gt_right" rowspan="1" colspan="1" scope="col" id="std_err">std_err</th>
    </tr>
  </thead>
  <tbody class="gt_table_body">
    <tr><td headers="model" class="gt_row gt_left">IPTW</td>
<td headers="estimate" class="gt_row gt_right">0.095</td>
<td headers="std_err" class="gt_row gt_right">0.015</td></tr>
    <tr><td headers="model" class="gt_row gt_left">Unadjusted</td>
<td headers="estimate" class="gt_row gt_right">0.216</td>
<td headers="std_err" class="gt_row gt_right">0.009</td></tr>
    <tr><td headers="model" class="gt_row gt_left">Baseline Adjusted</td>
<td headers="estimate" class="gt_row gt_right">0.159</td>
<td headers="std_err" class="gt_row gt_right">0.007</td></tr>
  </tbody>
  
  
</table>
</div>
```
:::
:::


Lumley uses this data to demonstrate how the IPTW estimator is useful for 
adjusting for all variability in the exposure -- mother's stress --- that might 
lead to the child's illness.

Unfortunately with such a small data set and a likely small effect size, we're
not able to determine that this result is statistically different from zero.
Lumley notes that this is often the case for methods like these, as removing
so much confounding from observational data results in very small subsequent
effect typically.


This method is fairly sophisticated. Lumley cites the following papers that
analyzed this data and I'd also encourage the interested reader to check out 
Hernan and Robins book on Causal Inference [@hernan2010causal].

## Exercises

There are no exercises for this chapter.


# Session Info

<details>


::: {.cell}

```{.r .cell-code}
sessionInfo()
```

::: {.cell-output .cell-output-stdout}
```
R version 4.4.1 (2024-06-14)
Platform: aarch64-apple-darwin20
Running under: macOS 15.1.1

Matrix products: default
BLAS:   /Library/Frameworks/R.framework/Versions/4.4-arm64/Resources/lib/libRblas.0.dylib 
LAPACK: /Library/Frameworks/R.framework/Versions/4.4-arm64/Resources/lib/libRlapack.dylib;  LAPACK version 3.12.0

Random number generation:
 RNG:     Mersenne-Twister 
 Normal:  Inversion 
 Sample:  Rounding 
 
locale:
[1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8

time zone: America/Denver
tzcode source: internal

attached base packages:
[1] grid      splines   stats     graphics  grDevices utils     datasets 
[8] methods   base     

other attached packages:
 [1] addhazard_1.1.0   lubridate_1.9.3   forcats_1.0.0     dplyr_1.1.4      
 [5] purrr_1.0.2       readr_2.1.5       tidyr_1.3.1       tibble_3.2.1     
 [9] ggplot2_3.5.1     tidyverse_2.0.0   survey_4.4-2      survival_3.6-4   
[13] Matrix_1.7-0      stringr_1.5.1     srvyr_1.3.0.9000  sf_1.0-17        
[17] RSQLite_2.3.7     quantreg_5.98     SparseM_1.84-2    patchwork_1.2.0  
[21] mitools_2.4       lattice_0.22-6    haven_2.5.4       gtsummary_2.0.3  
[25] gt_0.11.0         DiagrammeR_1.0.11 broom_1.0.6      

loaded via a namespace (and not attached):
 [1] DBI_1.2.3            readxl_1.4.3         rlang_1.1.4         
 [4] magrittr_2.0.3       shinydashboard_0.7.2 e1071_1.7-14        
 [7] compiler_4.4.1       vctrs_0.6.5          pkgconfig_2.0.3     
[10] fastmap_1.2.0        backports_1.5.0      dbplyr_2.5.0        
[13] labeling_0.4.3       utf8_1.2.4           promises_1.3.0      
[16] rmarkdown_2.27       markdown_1.13        tzdb_0.4.0          
[19] MatrixModels_0.5-3   mplot_1.0.6          bit_4.0.5           
[22] xfun_0.45            cachem_1.1.0         labelled_2.13.0     
[25] jsonlite_1.8.8       blob_1.2.4           later_1.3.2         
[28] parallel_4.4.1       R6_2.5.1             stringi_1.8.4       
[31] RColorBrewer_1.1-3   ahaz_1.15            cellranger_1.1.0    
[34] Rcpp_1.0.12          iterators_1.0.14     knitr_1.47          
[37] base64enc_0.1-3      httpuv_1.6.15        timechange_0.3.0    
[40] tidyselect_1.2.1     rstudioapi_0.16.0    yaml_2.3.9          
[43] codetools_0.2-20     doRNG_1.8.6          shiny_1.8.1.1       
[46] withr_3.0.0          evaluate_0.24.0      units_0.8-5         
[49] proxy_0.4-27         xml2_1.3.6           lpSolve_5.6.20      
[52] pillar_1.9.0         rngtools_1.5.2       KernSmooth_2.23-24  
[55] foreach_1.5.2        generics_0.1.3       hms_1.1.3           
[58] munsell_0.5.1        commonmark_1.9.1     scales_1.3.0        
[61] xtable_1.8-4         sampling_2.10        class_7.3-22        
[64] glue_1.8.0           tools_4.4.1          hexbin_1.28.4       
[67] visNetwork_2.1.2     cards_0.3.0          colorspace_2.1-0    
[70] cli_3.6.3            fansi_1.0.6          broom.helpers_1.17.0
[73] gtable_0.3.5         sass_0.4.9           digest_0.6.36       
[76] classInt_0.4-10      htmlwidgets_1.6.4    farver_2.1.2        
[79] memoise_2.0.1        htmltools_0.5.8.1    lifecycle_1.0.4     
[82] mime_0.12            bit64_4.0.5          MASS_7.3-60.2       
```
:::
:::


</details>

# References

