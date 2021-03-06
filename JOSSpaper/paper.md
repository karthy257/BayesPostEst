---
title: 'BayesPostEst: An R Package to Generate Postestimation Quantities for Bayesian MCMC Estimation'
tags:
  - MCMC
  - Bayesian methods
  - Visualization
  - ROC curves
  - Precision-Recall curves
  - Region of Practical Equivalence
authors:
  - name: Shana Scogin
    orcid: 0000-0002-7801-853X
    affiliation: 1
  - name: Johannes Karreth
    orcid: 0000-0003-4586-7153
    affiliation: 2
  - name: Andreas Beger
    orcid: 0000-0003-1883-3169
    affiliation: 3
  - name: Rob Williams
    orcid: 0000-0001-9259-3883
    affiliation: 4    
affiliations:
  - name: University of Notre Dame, South Bend, IN, USA
    index: 1
  - name: Ursinus College, Collegeville, PA, USA
    index: 2
  - name: Predictive Heuristics, Bellevue, WA, USA
    index: 3
  - name: Washington University in St. Louis, St. Louis, MO, USA
    index: 4
date: 1 October 2019
bibliography: paper.bib
---

# Summary

BayesPostEst is an R [@R] package with convenience functions to generate and present quantities of interest after estimating Bayesian regression models fit using MCMC via JAGS [@jags2017], Stan [@rstan2019], MCMCpack [@MCMCpack], or other MCMC samplers. Quantities of interest include predicted probabilities and changes in probabilities in generalized linear models and analyses of model fit using ROC curves and precision-recall curves. The package also contains two functions to create publication-ready tables summarizing model results with an assessment of substantively meaningful effect sizes.

The package currently consists of seven functions:

- `mcmcTab`: Summarize Bayesian MCMC output in a table
- `mcmcReg`: Create regression tables for multiple Bayesian MCMC models using `texreg`
- `mcmcAveProb`: Calculate predicted probabilities using Bayesian MCMC estimates for the "Average Case"
- `mcmcObsProb`: Calculate predicted probabilities using Bayesian MCMC estimates using the "Observed Value" approach, calculating probabilities for the average of observed cases
- `mcmcFD`: Calculate first differences of a Bayesian logit or probit model
- `mcmcFDplot`: Plot first differences from MCMC output
- `mcmcRocPrc`: Generate ROC and precision-recall curves using Bayesian MCMC estimates

# Need and applications

A variety of existing packages offer outstanding functionalities to extract and visualize posterior distribution of estimates and some postestimation quantities; the final section of this paper lists these packages. BayesPostEst offers a further contribution by (1) providing convenient summaries of MCMC estimates as used by social scientists and (2) implementing methods for interpreting estimates in generalized linear models that are widely used in the social sciences. These methods are not currently available in this form within one package and workflow. BayesPostEst also brings the "region of practical equivalence" [@Kruschke2013; @Kruschke2018] to the widely used tool of calculating first differences after logit or probit models. Finally, BayesPostEst implements the popular model fit diagnostic of examining receiver-operating characteristic and precision-recall curves for Bayesian models. 

# General setup

All functions in BayesPostEst work with distributions of posterior draws of regression coefficients. For further processing by BayesPostEst functions, these posterior draws need to be converted into a matrix. All functions in the package do this automatically for posterior draws generated by JAGS, BUGS, MCMCpack, rstan, and rstanarm. For posterior draws generated by other tools, users must convert these objects into a matrix, where rows represent iterations and columns represent parameters. The functions in BayesPostEst then use these posterior draws to generate postestimation quantities of interest that include measures of uncertainty based on posterior distributions.

# Summarizing Bayesian MCMC output in tables

BayesPostEst provides two functions to summarize Bayesian model results in tables: `mcmcTab` for summaries of one regression model with additional quantities of interest describing the posterior estimates, and `mcmcReg` for publication-ready regression tables summarizing estimates from one or multiple models. 

First, `mcmcTab` generates a table summarizing the posterior distributions of all parameters contained in the model object. This table can then be used to summarize parameter quantities for one model in detail. By default, `mcmcTab` generates a dataframe with one row per parameter and columns containing the median, standard deviation, and 95% credible interval of each parameter's posterior distribution. Users can add a column to the table that calculates the percent of posterior draws that have the same sign as the median of the posterior distribution. 

Users can also define a "region of practical equivalence" [ROPE, see @Kruschke2013; @Kruschke2018]. This region is a band of values around 0 that are "practically equivalent" to 0, or a "negligible" effect. As discussed in the online supplement to @Kruschke2018, in an example of analyzing individual responses of opinion polls, one can think of the ROPE capturing the range of a margin of error in an aggregate poll. If a common margin of error is $\pm0.03$, then for logistic regression coefficients in analyses of predictors of individual choice such a range of negligible coefficient values might be $[-0.06; 0.06]$.

```
mcmcTab(fit, 
	pars = c("female", "neuroticism", "extraversion"), 
	ROPE = c(-0.1, 0.1))
```

```
## This table contains an estimate for parameter values 
## outside of the region of practical equivalence (ROPE). 
## For this quantity to be meaningful, all parameters 
## must be on the same scale (e.g., standardized
## coefficients or first differences).
##
##       Variable Median    SD  Lower Upper PrOutROPE
## 1       female  0.237 0.111  0.023 0.462     0.886
## 2  neuroticism  0.055 0.108 -0.144 0.276     0.342
## 3 extraversion  0.518 0.112  0.296 0.733     1.000
```

To define a ROPE, it can be useful for all parameters (e.g., regression coefficients) to be on the same scale because `mcmcTab` accepts only one definition of ROPE for all parameters. Users can standardize continuous predictors to achieve this, for instance by dividing them by two standard deviations [@Gelman2008].

The output from `mcmcTab` can be exported to be inserted into a variety of document types using appropriate R packages, including flextable [@flextable], xtable [@xtable], or knitr [@knitr]. 

Second, `mcmcReg` generates tables summarizing estimates from *multiple* Bayesian models. The function then uses texreg [@texreg] to produce publication-ready tables in HTML or LaTeX format. This gives users a convenient and time-saving option to present estimates from different model specifications alongside each other. Beyond the arguments explicitly mentioned in the help file for `mcmcReg`, users can also supply any of the arguments that are part of the `texreg` function. The code below produces the following table:

```
mcmcReg(mod = list(fit1, fit2), 
        pars = c("(Intercept)", "female", "neuroticism", "extraversion"),
        pointest = "median",
        coefnames = list(
        	c("Intercept", "Female"), 
        	c("Intercept", "Female", "Neuroticism", "Extraversion")),
        ci = 0.95,
        format = "latex",
        caption = "Determinants of volunteering",
        caption.above = TRUE)
```        

![Table produced with the `mcmcReg` function.](mcmcReg_table.png)

# Predicted probabilities and first differences

BayesPostEst builds on estimates from a Bayesian generalized linear model with $k$ covariates $x$ and the inverse logit (or probit) link function, where 

$$
\text{Pr}(y = 1 | x_{k}) = \text{logit}^{-1}(\beta_1 + \beta_2 x_{k})
$$

or 

$$
\text{Pr}(y = 1 | x_{k}) = \Phi(\beta_1 + \beta_2 x_{k})
$$

To evaluate the relationship between covariates and a binary outcome, `mcmcAveProb` calculates the predicted probability ($\text{Pr}(y = 1)$) at pre-defined values of one covariate of interest ($x$), while all other covariates are held at a "typical" value. This follows suggestions outlined in @KingEtal2000 and elsewhere, which are commonly adopted by users of GLMs. The `mcmcAveProb` function by default calculates the median value of all covariates other than $x$ as "typical" values. Users can then access the full posterior distributions of the predicted probability at different values of $x$, or the median and a credible interval of the same quantity.

As an alternative to probabilities for "typical" cases, @HanmerKalkan2013 suggest to calculate predicted probabilities for all observed cases and then derive an "average effect". In their words, the goal of this postestimation "is to obtain an estimate of the average effect in the population ... rather than seeking to understand the effect for the average case." BayesPostEst allows users to calculate these quantities for binary and continuous predictors based on observed values of covariates, using the `mcmcObsProb` function. Users can then summarize or visualize these quantities as is common in the social sciences [@Long1997; @KingEtal2000; @HanmerKalkan2013] 

To summarize typical effects across covariates rather than full predicted probabilities, users can generate "first differences" [@Long1997; @KingEtal2000]. This quantity represents, for each covariate, the difference in predicted probabilities for cases with low and high values of the respective covariate. For each of these differences, all other variables are held constant at their median. To make use of the full posterior distribution of first differences, BayesPostEst provides a dedicated plotting function. `mcmcFDplot` returns a ggplot2 object that can be further customized. The function is modeled after Figure 1 in @Karreth2018. Users can specify a region of practical equivalence and print the percent of posterior draws to the right or left of the ROPE. If ROPE is not specified, the figure automatically prints the percent of posterior draws to the left or right of 0.

![Output from the `mcmcFDplot` function. See the package vignette for more details.](fd_full.png)

# Model fit analyses using ROC and Precision-Recall curves

One way to assess the fit of statistical models fit on binary outcomes is to calculate the area under the Receiver Operating Characteristic (ROC) and Precision-Recall curves. A short description of these curves and their utility for model assessment is provided in @Beger2016. Both methods assess the trade-off between true and false positives. ROC curves become less useful as outcomes of interest (or observed ones) become rare; precision-recall curves are a more suitable tool for such rare outcomes. The `mcmcRocPrc` function produces an object with four elements: the area under the ROC curve, the area under the PR curve, and two dataframes to plot each curve. While the area under each curve can be a useful summary of model fit, plotting the curves can serve to assess the model's trade-off in more detail.

# Comparison to other packages

The following packages provide functions similar (concretely or in spirit) to BayesPostEst. BayesPostEst is inspired by some of the functionality in these packages, but aims to combine tools commonly used in the social sciences in one workflow.

- tidybayes [@tidybayes] offers a tidy workflow for extracting posterior distributions of estimates, fits, and predictions.
- bayesplot [@bayesplot] offers various plotting options for posterior quantities, including posterior predictive checks.
- bayestable [@bayestable] generates a regression table from MCMC estimates that can be passed on to the texreg package [@texreg] for printing.
- The sjstats [@sjstats] and sjPlot [@sjPlot] suite of packages allows for a variety of postestimation commands, including predicted probabilities, marginal effects, and a function to evaluate estimates in relationship to a user-defined ROPE.
- Similarly, bayestestR [@bayestestR] offers a broad set of functions to analyze and describe posterior distributions of coefficients, but not other post-estimation quantities of interest.
- The ggmcmc [@ggmcmc] package contains a function to plot the ROC curve after a regression model for binary outcomes.
- The brms [@brms] package offers a variety of convenient postestimation commands, including predicted probabilities, for Bayesian models estimated directly in brms.

# Future developments

Plans for future work include extending existing methods in this package to a broader class of models, including generalized linear models with other link functions (e.g., models for ordered, categorical, and count outcomes) and multilevel/hierarchical generalized linear models. 

# Contributions

At the time of submission of this manuscript, the authors have contributed to the project as follows. S.S. created the package, improved functions, wrote function documentation, tested the package, and is the package maintainer. J.K. wrote most of the initial functions, the package vignette, and the JOSS manuscript. R.W. wrote the `mcmcReg` function and contributes to package development and optimization. A.B. rewrote the `mcmcRocPrc` function and reviewed code coverage.

# References