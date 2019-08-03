
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

This is currently an *experimental prototype* with most certainly a lot
of bugs and the appropriate code quality ;). However I would be happy
for any type of feedback, alpha testers, feature requests and potential
use cases.

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

### Inputs

You can define your inputs using the standard function syntax. By
default paramters are of type `type_matrix`. But they can have other
types such as:

  - `type_colvec` - a column vector
  - `type_rowvec` - a row vector
  - `type_scalar_integer` - a single int value
  - `type_scalar_numeric` - a single double value

### Body

  - `<-` you can use assignments that cause a C++ copy. As most
    operations return armadillo expressions, this is often not a
    problem. Usually assignments create new matrix variables, unless all
    operands on the right hand side can be assumed to not be any matrix
    code. Then `armacmp` and the C++11 compiler will figure out the
    concrete type.

Take a look at the `Function reference` vignette to get an overview of
all supported functions and expressions.

### Return

All functions need to return a value using the `return` function.

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
