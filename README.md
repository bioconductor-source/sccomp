sccomp - Outlier-aware and count-based compositional analysis of
single-cell data
================

<!-- badges: start -->

[![Lifecycle:maturing](https://img.shields.io/badge/lifecycle-maturing-blue.svg)](https://www.tidyverse.org/lifecycle/#maturing)
[![R build
status](https://github.com/stemangiola/tidyseurat/workflows/R-CMD-check/badge.svg)](https://github.com/stemangiola/tidyseurat/actions/)
<!-- badges: end -->

# <img src="inst/logo-01.png" height="139px" width="120px" />

Single-cell transcriptomics allows the unbiased characterisation of the
cellular composition of tissues. The cellular composition can be
compared between biological or clinical conditions to identify potential
cellular drivers. This strategy has been critical to unveil drivers of
immune response in cancer and pathogen infection from single-cell data.
Developing a robust statistical method for differential composition
analyses from single-cell data is crucial for driving discoveries. The
compositional data from single-cell experiments has four main
properties. The data is in count form; counts underlie inversely
correlated proportions that sum to one; larger cell groups are more
variable across samples than small groups; real-world data is rich in
outlier observation. A model that covers more than two of these
properties is currently lacking. **Here, we present a robust and
outlier-aware method for testing differential tissue composition from
single-cell data. This model can also transfer knowledge from a large
set of integrated datasets to increase accuracy further. We present how
this model can be applied to identify novel compositional and
heterogeneity changes in existing studies.**

# Installation

## (simple) Suggested for single-cell and CyTOF analyses

**Bioconductor**

``` r
if (!requireNamespace("BiocManager")) {
   install.packages("BiocManager")
 }
 BiocManager::install("sccomp")
```

**Github**

``` r
devtools::install_github("stemangiola/sccomp")
```

## (more complex and efficient, until further optimisation of the default installation) Suggested for microbiomics

**Github**

``` r
install.packages("cmdstanr", repos = c("https://mc-stan.org/r-packages/", getOption("repos")))
check_cmdstan_toolchain()
install_cmdstan(cores = 2)
# Then, check the correct cmdstanr installation here
# https://mc-stan.org/cmdstanr/articles/cmdstanr.html

# Then install sccomp with the cmdstanr branch
devtools::install_github("stemangiola/sccomp@cmdstanr")
```

# Analysis

## From Seurat Object

``` r
res =
  seurat_obj |>
   formula_composition = ~ type, 
    formula_variability = ~ 1, 
    sample, 
    cell_group 
  )
```

## From SingleCellExperiment Object

``` r
res =
  sce_obj |>
    formula_composition = ~ type, 
    formula_variability = ~ 1, 
    sample, 
    cell_group 
  )
```

## From data.frame

``` r
res =
  seurat_obj[[]] |>
  sccomp_glm(
    formula_composition = ~ type, 
    formula_variability = ~ 1, 
    sample, 
    cell_group 
  )
```

## From counts

``` r
res =
  counts_obj |>
  sccomp_glm( 
    formula_composition = ~ type, 
    formula_variability = ~ 1, 
    .sample = sample,
    .cell_group = cell_group,
    .count = count
  )
```

    ## sccomp says: outlier identification first pass - step 1/3 [ETA: ~20s]

    ## sccomp says: outlier identification second pass - step 2/3 [ETA: ~60s]

    ## sccomp says: outlier-free model fitting - step 3/3 [ETA: ~20s]

    ## sccomp says: the composition design matrix has columns: (Intercept), typecancer

    ## sccomp says: the variability design matrix has columns: (Intercept)

``` r
res
```

    ## # A tibble: 72 × 9
    ##    cell_group parameter   covariate c_lower c_effect c_upper    c_pH0     c_FDR
    ##    <chr>      <chr>       <chr>       <dbl>    <dbl>   <dbl>    <dbl>     <dbl>
    ##  1 B1         (Intercept) <NA>       0.557     0.708  0.869  0        0        
    ##  2 B1         typecancer  type      -1.18     -0.890 -0.585  0.000250 0.0000834
    ##  3 B2         (Intercept) <NA>       0.177     0.412  0.651  0.0355   0.00253  
    ##  4 B2         typecancer  type      -1.20     -0.753 -0.309  0.00601  0.00163  
    ##  5 B3         (Intercept) <NA>      -0.661    -0.454 -0.241  0.0113   0.000510 
    ##  6 B3         typecancer  type      -0.735    -0.325  0.0723 0.268    0.0635   
    ##  7 BM         (Intercept) <NA>      -1.31     -1.10  -0.880  0        0        
    ##  8 BM         typecancer  type      -0.752    -0.335  0.0664 0.259    0.0521   
    ##  9 CD4 1      (Intercept) <NA>       0.356     0.488  0.626  0        0        
    ## 10 CD4 1      typecancer  type      -0.0899    0.161  0.421  0.632    0.197    
    ## # … with 62 more rows, and 1 more variable: count_data <list>

## Visualise data + inference

``` r
plots = plot_summary(res) 
```

    ## Joining, by = c("sample", "cell_group")
    ## Joining, by = c("cell_group", "type")

    ## Warning: Ignoring unknown aesthetics: label

Plot of group proportion, faceted by groups. The blue boxplots represent
the posterior predictive check. If the model is likely be descriptively
adequate to the data, the blue boxplot should roughly overlay with the
black boxplot, which represent the observed data. The outliers are
coloured in red.

``` r
plots$boxplot
```

    ## [[1]]

![](inst/figures/unnamed-chunk-11-1.png)<!-- -->

Plot of estimates of differential composition (c\_) on the x axis, and
differential variability (v\_) on the y axis. The error bars represent
95% credible intervals. The dashed lines represent the minimal effect
that the hypothesis test is based on. An effect is labelled as
significant if bigger than the minimal effect according to the 95%
credible interval. Facets represent the covariates in the model.

``` r
plots$credible_intervals_1D
```

![](inst/figures/unnamed-chunk-12-1.png)<!-- -->

## Visualisation of the MCMC chains from the posterior distribution

It is possible to directly evaluate the posterior distribution. In this
example we plot the Monte Carlo chain for the slope parameter of the
first cell type. We can see that has converged and is negative with
probability 1.

``` r
res %>% attr("fit") %>% rstan::traceplot("beta[2,1]")
```

![](inst/figures/unnamed-chunk-13-1.png)<!-- -->

## Differential variability

We can model the cell-group variability also dependent on type, and so
test differences in variability

``` r
res = 
  counts_obj |>
  sccomp_glm( 
    formula_composition = ~ type, 
    formula_variability = ~ type, 
    .sample = sample,
    .cell_group = cell_group,
    .count = count
  )
```

    ## sccomp says: outlier identification first pass - step 1/3 [ETA: ~20s]

    ## sccomp says: outlier identification second pass - step 2/3 [ETA: ~60s]

    ## sccomp says: outlier-free model fitting - step 3/3 [ETA: ~20s]

    ## sccomp says: the composition design matrix has columns: (Intercept), typecancer

    ## sccomp says: the variability design matrix has columns: (Intercept), typecancer

``` r
res
```

    ## # A tibble: 72 × 14
    ##    cell_group parameter   covariate c_lower c_effect c_upper  c_pH0   c_FDR
    ##    <chr>      <chr>       <chr>       <dbl>    <dbl>   <dbl>  <dbl>   <dbl>
    ##  1 B1         (Intercept) <NA>       0.593     0.769  0.953  0      0      
    ##  2 B1         typecancer  type      -1.26     -0.915 -0.540  0      0      
    ##  3 B2         (Intercept) <NA>       0.134     0.387  0.630  0.0658 0.00373
    ##  4 B2         typecancer  type      -1.03     -0.541 -0.0711 0.0773 0.0160 
    ##  5 B3         (Intercept) <NA>      -0.649    -0.427 -0.191  0.0275 0.00151
    ##  6 B3         typecancer  type      -0.661    -0.205  0.262  0.491  0.145  
    ##  7 BM         (Intercept) <NA>      -1.29     -1.07  -0.825  0      0      
    ##  8 BM         typecancer  type      -0.741    -0.291  0.184  0.343  0.0798 
    ##  9 CD4 1      (Intercept) <NA>       0.370     0.514  0.662  0      0      
    ## 10 CD4 1      typecancer  type      -0.0446    0.239  0.538  0.394  0.106  
    ## # … with 62 more rows, and 6 more variables: v_lower <dbl>, v_effect <dbl>,
    ## #   v_upper <dbl>, v_pH0 <dbl>, v_FDR <dbl>, count_data <list>

Plot 1D significance plot

``` r
plots = plot_summary(res)
```

    ## Joining, by = c("sample", "cell_group")
    ## Joining, by = c("cell_group", "type")

    ## Warning: Ignoring unknown aesthetics: label

``` r
plots$credible_intervals_1D
```

![](inst/figures/unnamed-chunk-15-1.png)<!-- -->

Plot 2D significance plot. This is possible if only differential
variability has been tested

``` r
plots$credible_intervals_2D
```

![](inst/figures/unnamed-chunk-16-1.png)<!-- -->
