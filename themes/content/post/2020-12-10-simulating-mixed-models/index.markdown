---
title: Simulating mixed models
author: Oliver P. Madsen
date: '2020-12-10'
slug: simulating-mixed-models
categories: ['Hierachical models', 'Mixed model', 'Statistics', 'Stackoverflow question', 'Data Science']
tags: ['Hierachical models', 'Mixed model', 'Statistics', 'Stackoverflow question', 'Data Science']
keywords:
  - R
  - Data science
  - Statistics
  - Mixed Models
math: true
---
# Simulating mixed models

A question on stackoverflow sparked an interesting problem.

<!--more-->
The [question](https://stackoverflow.com/q/65129483/10782538) itself was not anything out of the ordinary. The question itself was about how one can select random effects and why/when one should use `cbind` in the formula for binary/binomial outcome. My [answer](https://stackoverflow.com/a/65145859/10782538) gave a suggestion for the mixed effect structure and I told that `cbind` is used to binomial data while the outcome vector can be used for binary data, which is equivalent to using `cbind` for aggregate binary data. And I said the results using `cbind` are equivalent and it only provides stability in the estimation process. But following this the questioner told me in a comment, that he/she was observing varying results between `cbind` and the raw outcome vector. This is what I will investigate in my blog post here. 

# Data background

The way that I'll investigate this behaveour is to do a repeated simulation study using simulated data equivalent to the data described in the question. This means we have 

1. 4 variables (time, plot, subplot, cover) and 1 binary outcome (eaten).
   * cover is the percentage coverage of the specific plot/subplot at time t
   * time is a period of 9 years (here I'm assuming from 2012 to 2020)
   * plot is 1 of 90 plot areas which are repeated each year, possibly missing some years
   * subplot is 1 of 4 subplot within plot.
   * eaten is our outcome indicating whether the sapling planted on plot(i) and subplot(j) has been eaten during year(t).
1. For each plot there are (up to) 100 saplings measured. Eg. 25 per subplot. So we have `\(4 \cdot 25 \cdot 90 \cdot 9 = 81000\)` possible outcomes excluding cover %. 
   * According to the questioner they have approximately 55000 observations total.
   * It is not mentioned, but during their simulation cover % is replicated across years. So I'll assume that this is the case and the same cover % is used for every year.
   
   

For mixed structure we'll let time, plot and subplot be mixed effect with subplot nested in plot nested in time. Several candidates could be used, but given the background of the problem this one makes sense. One could both argue that plot is not nested in time, and that subplot is not a mixed effect (namely 1: it only has 4 effects, which contains the entire population of subplots and 2: we only have 4 observations, which is generally not enough to consistently estimate the variance parameters), but for this simulation it does not make a difference.

# Simulation example
To simulate our data we only have to worry about simulating 3 parts:

1. First we have to simulate our random effects. These follow a normal distribution according the GLMM definition followed by `lme4`.
2. We also have to simulate `cover %` for each plot.
3. Finaly we need to simulate our outcome variable eaten from the model definition `\(logit(eaten) = \eta\)` meaning `\(eaten~binary(eta)\)` and where `\(eta = X\beta+ Zb\)`. I'll use the slightly altered definition `\(eta=X\beta +Z\Lambda u\)` where `\(\Lambda=\Sigma^{1/2}\)` and `\(u~N_q(0,1)\)`. This is in line with the definition used by the `lme4` package and is simpler to simulate.

I'll assume that `\(\beta^\top = \{-2, .75\}\)` and `\(\Lambda_{ii}=\{.3, .7, .1\}^{1/2}_{i}\)`. Thus the variance parameters are `\(\delta_{\text{time}} = 1.0\)`, `\(\delta_{\text{time:plot}} = 0.7\)` and `\(\delta_{\text{time:plot:subplot}} = 0.3\)`. 

The simulation process is as follows:

1. Simulate 90 outcomes for cover, 1 for each plot. These are repeated for each year, plot, subplot and 25 saplings resulting in 81.000 observations.
1. From the grid of 81.000 observations a sample of 55.000 observations is taken. 
1. Using the model formulation I'll `\(q\)` random effects from the standard normal distribution.
1. Again using the model formulation we'll estimate `\(eta\)` and use this to simulate our binary outcome vector eaten using `rbinom`.
1. With our simulation of eaten we'll fit a model to eaten using both the raw outcome vector and 
success/failure with `cbind`. 
1. After fitting the models compare the fixed and random effects using `all.equal`. Do note that as of R 4.1.0 `all.equal` might return false where it previously returned true, as it will start comparing environments.

Below I've split the simulation process into data simulation of data, simulation of outcome and model comparison

```r
library(lme4)
```

```
## Loading required package: Matrix
```

```r
simulate_data <- function(){
  totalobs <- 4 * 9 * 90 * 25
  subplot <- rep(1:4, totalobs / 4)
  plot <- rep(1:90, totalobs / 90)
  time <- rep(2012:2020, totalobs / 9)
  cover <- rep(runif(90), totalobs / 90)
  # obs after sampling
  nobs <- 55000  
  # Extract data sample
  samp <- sample(nobs)
  data.frame(subplot, plot, time, cover)[samp, ]
}
simulate_outcome <- function(data){
  form <- out ~ cover + (1|time/plot/subplot)
  data$out <- 1
  glF <- glFormula(form, data = data)
  data$out <- NULL
  rtrms <- glF$reTrms
  Z <- t(rtrms$Zt)
  # Pre-choose beta, sigma and b
  beta <- c(-2, 0.75) # picked out of thin air
  # We need a correlation matrix for our random effects.
  # I go with the same definition as the one used by lme4
  delta <- sqrt(c(.3, .7, 1.0))
  lambda <- delta[rtrms$Lind] # crossprod(lambda) = sigma
  u <- rnorm(ncol(Z))
  eta <- glF$X %*% beta + Z %*% (lambda * u)
  expit <- function(x)
    1 / ( 1 + exp(-x) )
  # Simulate our outcome vector
  data$eaten <- rbinom(55000, 1, prob = as.numeric(expit(eta)))
  # Add not eaten for cbind method
  data$not_eaten <- 1 - data$eaten
  data
}
model <- function(data){
  fit_standard <- glmer(eaten ~ cover + (1|time/plot/subplot), data = data, family = binomial)
  fit_cbind <- glmer(cbind(eaten, not_eaten) ~ cover + (1|time/plot/subplot), data = data, family = binomial)
  c(fixef = all.equal(fixef(fit_standard), fixef(fit_cbind)), 
    ranef = all.equal(ranef(fit_standard),ranef(fit_cbind)))
}
```
So an example of a simulation can be done using:


```r
data <- simulate_data()
data <- simulate_outcome(data)
comparison <- model(data)
cat('are the estimates identical?\n')
```

```
## are the estimates identical?
```

```r
print(comparison)
```

```
## fixef ranef 
##  TRUE  TRUE
```

# Repeated simulation
We'll repeat the above simulation 20 times to ensure that our observation was not just a fluke.

```r
# Helper function
sim <- function(times = 20, seed = 4300){
  .seed <- .Random.seed
  set.seed(seed)
  on.exit(.Random.seed <- .seed)
  do.call(rbind, lapply(seq_len(times), function(...){
    data <- simulate_data()
    data <- simulate_outcome(data)
    model(data)
  }))
}
# Simulate and show the result in a simple html table
kableExtra::kable(sim())
```

<table>
 <thead>
  <tr>
   <th style="text-align:left;"> fixef </th>
   <th style="text-align:left;"> ranef </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
</tbody>
</table>

From our 20 simulations it is very clear, that the comment from the questioner is not replicated by these random simulations. I can think of a few potential reasons:

1. The most obvious seems to be an error by the questioner, that I cannot see from my side.
1. The parameters for the real dataset are ill defined. Maybe they are unstable in a manner that leads to unpredictable results.
   * I dont believe this is theoretically possible, but maybe a practical example exists (similar to the example shown in the article "sometimes mixed models are just bad").

there are likely other as well. 
