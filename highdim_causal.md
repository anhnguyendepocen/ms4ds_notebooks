Causal ML
================
Joshua Loftus
5/7/2020

``` r
library(glmnet)
```

    ## Loading required package: Matrix

    ## Loaded glmnet 4.0

``` r
library(hdm)
library(crossEstimation)
```

Generate data satisfying PLM assumptions
----------------------------------------

``` r
correlated_gaussian_design <- function(n, p, rho) {
  x <- matrix(rnorm(n*p), nrow = n)
  if (rho == 0) return(x)
  z <- matrix(rep(t(rnorm(n)), p), nrow = n)
  sqrt(1-rho)*x + sqrt(rho)*z
}

n <- 100
p <- 200
alpha_sparsity <- 5
alpha <- 1
beta_sparsity <- 5
beta <- 1
theta <- 3.14
X <- correlated_gaussian_design(n, p, 0.05)
V <- rnorm(n)
al <- rep(0, p)
al[1:alpha_sparsity] <- alpha
Dprop <- V + X %*% al
Dprop <- exp(Dprop)/(1+exp(Dprop))
D <- rbinom(n, 1, Dprop)
Y <- theta * D + rnorm(n)  
be <- rep(0, p)
be[1:beta_sparsity] <- beta
Y <- Y + X %*% be
```

Try various methods

### hdm package using DML and PDS

``` r
DML_fit = rlassoEffect(X, Y, D, method = "partialling out")
summary(DML_fit)
```

    ## [1] "Estimates and significance testing of the effect of target variables"
    ##      Estimate. Std. Error t value Pr(>|t|)    
    ## [1,]    2.2524     0.2402   9.379   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

``` r
PDS_fit = rlassoEffect(X, Y, D, method = "double selection")
summary(PDS_fit)
```

    ## [1] "Estimates and significance testing of the effect of target variables"
    ##    Estimate. Std. Error t value Pr(>|t|)    
    ## d1    3.3032     0.2537   13.02   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

### Failing assumption of experimental methods

``` r
#CE_fit <- ate.glmnet(X, Y, D)
#CE_fit
```

``` r
ate.randomForest(X, Y, D)
```

    ## $tau
    ## [1] 5.399424
    ## 
    ## $var
    ## [1] 0.1520102
    ## 
    ## $conf.int
    ## [1] 4.758120 6.040727
    ## 
    ## $conf.level
    ## [1] 0.9

Generate data with randomized treatment
---------------------------------------

Now *D* is randomized, i.e. *m*(*X*)≡0. Note also that these methods assume *D* is binary, while

``` r
D <- rbinom(n, 1, .5)
Y <- theta * D + rnorm(n)  
Y <- Y + X %*% be
```

### crossEstimation package

``` r
#CE_fit <- ate.glmnet(X, Y, D)
#CE_fit
```

``` r
CE_fit <- ate.randomForest(X, Y, D)
CE_fit
```

    ## $tau
    ## [1] 2.515404
    ## 
    ## $var
    ## [1] 0.2117177
    ## 
    ## $conf.int
    ## [1] 1.758561 3.272247
    ## 
    ## $conf.level
    ## [1] 0.9

------------
============

``` r
hd_instance <- function(n = 100, p = 200, rho = 0, alpha_sparsity = 0, alpha = 0, theta = 0, beta_sparsity = 0, beta = 0) {
  X <- correlated_gaussian_design(n, p, rho)
  V <- rnorm(n)
  al <- rep(0, p)
  if (alpha_sparsity > 0) {
    al[1:alpha_sparsity] <- alpha
    D <- D + X %*% al
  }
  Dprop <- V + X %*% al
  Dprop <- exp(Dprop)/(1+exp(Dprop))
  D <- rbinom(n, 1, Dprop)
  Y <- theta * D + rnorm(n)  
  be <- rep(0, p)
  if (beta_sparsity > 0) {
    be[1:beta_sparsity] <- beta
  }
  Y <- Y + X %*% be
  
  DML_fit = rlassoEffect(X, Y, D, method = "partialling out")
  PDS_fit = rlassoEffect(X, Y, D, method = "double selection")
  CElasso_fit <- ate.glmnet(X, Y, D)
  CERF_fit <- ate.randomForest(X, Y, D)
  
  output <- c(
    DML_fit$coefficient,
    PDS_fit$coefficient,
    CElasso_fit$tau,
    CERF_fit$tau
  )
  names(output) <- c(
    "DML", "PDS", "CElasso", "CErf"
  )
  output
}

hd_MCMSE <- function(nsim = 100, n = 100, p = 200, rho = 0, alpha_sparsity = 0, alpha = 0, theta = 0, beta_sparsity = 0, beta = 0) {
  rowMeans( (replicate(nsim, hd_instance(n, p, rho, alpha_sparsity, alpha, theta, beta_sparsity, beta) - theta)^2 )) 
}
```

``` r
# Observational
#hd_instance(100, 200, 0.1, 5, 1, 3.14, 5, 1)

# Experimental
#hd_instance(100, 200, 0.1, 0, 0, 3.14, 5, 1)
```

``` r
# Observational
#hd_MCMSE(100, 100, 200, 0.1, 5, 1, 3.14, 5, 1)

# Experimental
#hd_MCMSE(100, 100, 200, 0.1, 0, 0, 3.14, 5, 1)
```

More strongly correlated features

``` r
# Observational
#hd_instance(100, 200, 0.5, 5, 1, 3.14, 5, 1)

# Experimental
#hd_instance(100, 200, 0.5, 0, 0, 3.14, 5, 1)
```

``` r
# Observational
#hd_MCMSE(20, 100, 200, 0.5, 5, 1, 3.14, 5, 1)

# Experimental
#hd_MCMSE(20, 100, 200, 0.5, 0, 0, 3.14, 5, 1)
```

A less sparse scenario

``` r
# Observational
#hd_instance(100, 200, 0.1, 50, 1, 3.14, 20, 1)

# Experimental
#hd_instance(100, 200, 0.1, 0, 0, 3.14, 20, 1)
```

``` r
# Observational
#hd_MCMSE(20, 100, 200, 0.1, 50, 1, 3.14, 20, 1)

# Experimental
#hd_MCMSE(20, 100, 200, 0.1, 0, 0, 3.14, 20, 1)
```

Observational setting with omitted variable bias

"As good as random conditional on covariates" fails because our set of covariates doesn't include everything that's relevant

``` r
hd_instance <- function(n = 100, p = 200, rho = 0, alpha_sparsity = 0, alpha = 0, theta = 0, beta_sparsity = 0, beta = 0) {
  X <- correlated_gaussian_design(n, p, rho)
  V <- rnorm(n)
  al <- rep(0, p)
  if (alpha_sparsity > 0) {
    al[1:alpha_sparsity] <- alpha
    D <- D + X %*% al
  }
  Dprop <- V + X %*% al
  Dprop <- exp(Dprop)/(1+exp(Dprop))
  D <- rbinom(n, 1, Dprop)
  Y <- theta * D + rnorm(n)  
  be <- rep(0, p)
  if (beta_sparsity > 0) {
    be[1:beta_sparsity] <- beta
  }
  Y <- Y + X %*% be
  
  x <- X[, alpha_sparsity:ncol(X)]
  
  DML_fit = rlassoEffect(x, Y, D, method = "partialling out")
  PDS_fit = rlassoEffect(x, Y, D, method = "double selection")
  CElasso_fit <- ate.glmnet(x, Y, D)
  CERF_fit <- ate.randomForest(x, Y, D)
  
  output <- c(
    DML_fit$coefficient,
    PDS_fit$coefficient,
    CElasso_fit$tau,
    CERF_fit$tau
  )
  names(output) <- c(
    "DML", "PDS", "CElasso", "CErf"
  )
  output
}

hd_MCMSE <- function(nsim = 100, n = 100, p = 200, rho = 0, alpha_sparsity = 0, alpha = 0, theta = 0, beta_sparsity = 0, beta = 0) {
  rowMeans( (replicate(nsim, hd_instance(n, p, rho, alpha_sparsity, alpha, theta, beta_sparsity, beta) - theta)^2 )) 
}
```

``` r
# Observational
#hd_instance(100, 200, 0.1, 5, 1, 3.14, 5, 1)

# Experimental
#hd_instance(100, 200, 0.1, 0, 0, 3.14, 5, 1)
```

``` r
# Observational
#hd_MCMSE(30, 100, 200, 0.5, 10, 1, 3.14, 10, 1)

# Experimental
#hd_MCMSE(20, 100, 200, 0.1, 0, 0, 3.14, 5, 1)
```
