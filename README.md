
<!-- README.md is generated from README.Rmd. Please edit that file -->

# armacmp

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
[![Travis build
status](https://travis-ci.org/dirkschumacher/armacmp.svg?branch=master)](https://travis-ci.org/dirkschumacher/armacmp)
[![Codecov test
coverage](https://codecov.io/gh/dirkschumacher/armacmp/branch/master/graph/badge.svg)](https://codecov.io/gh/dirkschumacher/armacmp?branch=master)
[![AppVeyor build
status](https://ci.appveyor.com/api/projects/status/github/dirkschumacher/armacmp?branch=master&svg=true)](https://ci.appveyor.com/project/dirkschumacher/armacmp)
<!-- badges: end -->

The goal of `armacmp` is to create a DSL to formulate linear algebra
code in R that is compiled to C++ using the Armadillo Template Library.

The scope of the package is linear algebra and Armadillo. It is not
meant to evolve into a general purpose R to C++ transpiler.

This is currently an *experimental prototype* with most certainly bugs
or unexpected behaviour. However I would be happy for any type of
feedback, alpha testers, feature requests and potential use cases.

Potential use cases:

  - Speed up your code :)
  - Quickly estimate `Rcpp` speedup gain for linear algebra code
  - Learn how R linear algebra code can be expressed in C++ using
    `armacmp_compile` and use the code as a starting point for further
    development.
  - …

## Installation

``` r
remotes::install_github("dirkschumacher/armacmp")
```

## Caveats and limitations

  - *speed*: R is already really fast when it comes to linear algebra
    operations. So simply compiling your code to C++ might not give you
    a *significant and relevant* speed boost. The best way to check is
    to measure it yourself and see for your specific use-case, if
    compiling your code to C++ justifies the additional complexity.
  - *NAs*: there is currently no NA handling. In fact everything is
    assumed to be double (if you use matrices/vectors).
  - *numerical stability*: Note that your C++ code might produce
    different results in certain situations. Always validate before you
    use it for important applications.

## Example

You can compile R like code to C++. Not all R functions are supported.

``` r
library(armacmp)
```

Takes a matrix and returns its transpose.

``` r
trans <- armacmp(function(X) {
  return(t(X))
})
trans(matrix(1:10))
#>      [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8] [,9] [,10]
#> [1,]    1    2    3    4    5    6    7    8    9    10
```

Or a slightly larger example using QR decomposition

``` r
# from Arnold, T., Kane, M., & Lewis, B. W. (2019). A Computational Approach to Statistical Learning. CRC Press.
lm_cpp <- armacmp(function(X, y = type_colvec()) {
  qr_res <- qr(X)
  qty <- t(qr.Q(qr_res)) %*% y
  beta_hat <- backsolve(qr.R(qr_res), qty)
  return(beta_hat, type = type_colvec())
})

# example from the R docs of lm.fit
n <- 70000 ; p <- 20
X <- matrix(rnorm(n * p), n, p) 
y <- rnorm(n)
all.equal(
  as.numeric(coef(lm.fit(X, y))),
  as.numeric(lm_cpp(X, y))
)
#> [1] TRUE
```

## API

`armacmp` always compiles functions. Every function needs to have a
`return` statement with an optional type argument.

``` r
my_fun <- armacmp(function(X, y = type_colvec())) {
  return(X %*% y, type = type_colvec())
}
```

A lot of linear algebra functions/operators are defined as well some
control flow (for loops and if/else). Please take a look at the
[function referece
article](https://dirkschumacher.github.io/armacmp/articles/function-reference.html)
for more details what can be expressed.

### A complex example where `armacmp` improves performance

Armadillo can combine linear algebra operations. For example the
addition of 4 matrices `A + B + C + D` can be done in a single for loop.
Armadillo can detect that and generates efficient code.

So whenever you combine many different operations, `armacmp` *might* be
helpful in speeding things up.

To showcase that I took an algorithm from a great book called “A
Computational Approach to Statistical Learning” that involves a lot of
linear algebra code in a for loop. The C++ version is significantly
faster.

``` r
# From Arnold, T., Kane, M., & Lewis, B. W. (2019). A Computational Approach to Statistical Learning. CRC Press.

# Logistic regression using the Newton-Raphson algorithm
log_reg <- armacmp(function(X, y = type_colvec()) {
  beta <- rep.int(0, ncol(X))
  for (i in seq_len(25)) {
    b_old <- beta
    alpha <- X %*% beta
    p <- 1 / (1 + exp(-alpha))
    W <- p * (1 - p)
    XtX <- crossprod(X, diag(W) %*% X)
    score <- t(X) %*% (y - p)
    delta <- solve(XtX, score)
    beta <- beta + delta
  }
  return(beta, type = type_colvec())
})

log_reg_r <- function(X, y) {
  beta <- rep.int(0, ncol(X))
  for (i in seq_len(25)) {
    b_old <- beta
    alpha <- X %*% beta
    p <- 1 / (1 + exp(-alpha))
    W <- as.numeric(p * (1 - p))
    XtX <- crossprod(X, diag(W) %*% X)
    score <- t(X) %*% (y - p)
    delta <- solve(XtX, score)
    beta <- beta + delta
  }
  return(beta)
}

set.seed(10)
n <- 1000 ; p <- 50
true_beta <- rnorm(p)
X <- cbind(1, matrix(rnorm(n * (p - 1)), ncol = p - 1))
y <- runif(n) < plogis(X %*% true_beta)

# to see that it actually does logistic regression
# there can be a small difference to the glm.fit result
all.equal(
  as.numeric(log_reg(X, y)),
  as.numeric(log_reg_r(X, y)),
  coef(glm.fit(X, y, family = binomial()))
)
#> [1] TRUE

bench::mark(
  log_reg(X, y),
  log_reg_r(X, y)
)
#> Warning: Some expressions had a GC in every iteration; so filtering is
#> disabled.
#> # A tibble: 2 x 6
#>   expression           min   median `itr/sec` mem_alloc `gc/sec`
#>   <bch:expr>      <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#> 1 log_reg(X, y)    74.66ms  98.77ms    10.7      10.8KB     0   
#> 2 log_reg_r(X, y)    1.67s    1.67s     0.598   211.8MB     3.59
```

### Related projects

  - [nCompiler](https://github.com/nimble-dev/nCompiler) - Code-generate
    C++ from R. Inspired the approach to compile R functions directly
    instead of just a code block as in the initial version.

### Contribute

`armacmp` is experimental and has a volatile codebase. The best way to
contribute is to write issues/report bugs/propose features and test the
package with your specific use-case.

### Code of conduct

Please note that the ‘armacmp’ project is released with a [Contributor
Code of Conduct](CODE_OF_CONDUCT.md). By contributing to this project,
you agree to abide by its terms.
