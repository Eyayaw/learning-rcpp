My main goal in this educational endeavor is to be able to use the [MLPACK](http://www.mlpack.org/) library in R, with the hope of being able to include its Naive Bayes classifier in [MLPUGS](https://cran.r-project.org/package=MLPUGS). Now, there is a [RcppMLPACK](https://cran.r-project.org/package=RcppMLPACK), but that one apparently uses version 1 of MLPACK (which is now in version 2) and doesn't include any supervised learning methods, just unsupervised learning methods.

-   [Setup](#setup)
    -   [Software Libraries](#software-libraries)
    -   [Mac OS X](#mac-os-x)
    -   [Ubuntu/Debian](#ubuntudebian)
    -   [R Packages](#r-packages)
-   [Rcpp](#rcpp)
    -   [Basics](#basics)
    -   [Using Libraries](#using-libraries)
    -   [Modules](#modules)
-   [References](#references)

Setup
=====

Software Libraries
------------------

Mac OS X
--------

``` bash
## To install Homebrew:
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
## Then:
brew tap homebrew/versions && brew tap homebrew/science && brew update
# brew install gcc --enable-cxx && brew link --overwrite gcc && brew link cmake
brew install boost --c++11
# Installing cmake may require: sudo chown -R `whoami` /usr/local/share/man/man7
brew install mlpack
```

Ubuntu/Debian
-------------

``` bash
sudo apt-get install libmlpack
```

R Packages
----------

``` r
install.packages(c("BH", "Rcpp", "RcppArmadillo", "microbenchmark"))
```

If you get "ld: library not found for -lgfortran" error when trying to install RcppArmadillo, run:

``` bash
curl -O http://r.research.att.com/libs/gfortran-4.8.2-darwin13.tar.bz2
sudo tar fvxz gfortran-4.8.2-darwin13.tar.bz2 -C /
```

See "[Rcpp, RcppArmadillo and OS X Mavericks "-lgfortran" and "-lquadmath" error](http://thecoatlessprofessor.com/programming/rcpp-rcpparmadillo-and-os-x-mavericks-lgfortran-and-lquadmath-error/)"" for more info.

Rcpp
====

See [this section](http://rmarkdown.rstudio.com/authoring_knitr_engines.html#rcpp) in [RMarkdown documentation](http://rmarkdown.rstudio.com/) for details on Rcpp chunks.

``` r
library(magrittr)
library(Rcpp)
library(RcppArmadillo)
library(microbenchmark)
```

Basics
------

``` cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
NumericVector cumSum(NumericVector x) {
  int n = x.size();
  NumericVector out(n);
  out[0] = x[0];
  for (int i = 1; i < n; ++i) {
    out[i] = out[i-1] + x[i];
  }
  return out;
}
```

``` r
x <- 1:1000
microbenchmark(
  native = cumsum(x),
  loop = (function(x) {
    output <- numeric(length(x))
    output[1] <- x[1]
    for (i in 2:length(x)) {
      output[i] <- output[i-1] + x[i]
    }
    return(output)
  })(x),
  Rcpp = cumSum(x)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr   |     min|      lq|    mean|  median|      uq|     max|  neval|
|:-------|-------:|-------:|-------:|-------:|-------:|-------:|------:|
| native |  0.0025|  0.0030|  0.0051|  0.0041|  0.0055|  0.0271|    100|
| loop   |  0.8690|  1.0000|  1.3774|  1.2232|  1.6609|  2.9292|    100|
| Rcpp   |  0.0043|  0.0069|  0.0149|  0.0118|  0.0174|  0.0944|    100|

Using Libraries
---------------

### Armadillo vs RcppArmadillo

Use the **depends** attribute to bring in [RcppArmadillo](https://cran.r-project.org/package=RcppArmadillo), which is an Rcpp integration of the templated linear algebra library [Armadillo](http://arma.sourceforge.net/). The code below is an example of a fast linear model from Dirk Eddelbuettel.

``` cpp
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadillo.h>
using namespace Rcpp;
using namespace arma;
// [[Rcpp::export]]
List fastLm(const colvec& y, const mat& X) {
  int n = X.n_rows, k = X.n_cols;
  colvec coef = solve(X, y);
  colvec resid = y - X*coef;
  double sig2 = as_scalar(trans(resid) * resid/(n-k));
  colvec stderrest = sqrt(sig2 * diagvec( inv(trans(X)*X)) );
  return List::create(_["coefficients"] = coef,
                      _["stderr"]       = stderrest,
                      _["df.residual"]  = n - k );
}
```

``` r
data("mtcars", package = "datasets")
microbenchmark(
  lm = lm(mpg ~ wt + disp + cyl + hp, data = mtcars),
  fastLm = fastLm(mtcars$mpg, cbind(1, as.matrix(mtcars[, c("wt", "disp", "cyl", "hp")]))),
  RcppArm = fastLmPure(cbind(1, as.matrix(mtcars[, c("wt", "disp", "cyl", "hp")])), mtcars$mpg)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr    |     min|      lq|    mean|  median|      uq|     max|  neval|
|:--------|-------:|-------:|-------:|-------:|-------:|-------:|------:|
| lm      |  0.8361|  1.0069|  1.6627|  1.3751|  1.9665|  7.4787|    100|
| fastLm  |  0.0939|  0.1145|  0.1815|  0.1378|  0.2163|  0.7773|    100|
| RcppArm |  0.1199|  0.1441|  0.2150|  0.1838|  0.2396|  1.0945|    100|

### Fast K-Means

Unfortunately, [RcppMLPACK](https://cran.r-project.org/package=RcppMLPACK) uses version 1 of [MLPACK](http://www.mlpack.org/) (now in version 2) and only makes the unsupervised learning methods accessible. (Supervised methods would require returning a trained classifier object to R, which is actually a really difficult problem.)

Okay, let's try to get a fast version of <span title="Bradley, P. S., &amp; Fayyad, U. M. (1998). Refining Initial Points for K-Means Clustering. Icml." style="font-weight: bold;">k-means</span>.

First, install the MLPACK library (see [§ Software Libraries](#software-libraries)), then:

``` r
# Thanks to Kevin Ushey for suggesting Rcpp plugins (e.g. Rcpp:::.plugins$openmp)
registerPlugin("mlpack11", function() {
  return(list(env = list(
    USE_CXX1X = "yes",
    CXX1XSTD="-std=c++11",
    PKG_LIBS = "-lmlpack"
  )))
})
```

We refer to the documentation for [KMeans](http://www.mlpack.org/docs/mlpack-1.0.6/doxygen.php?doc=kmtutorial.html#kmeans_kmtut) shows, although it seems to incorrectly use `arma::Col<size_t>` for cluster assigments while in practice the cluster assignments are returned as an `arma::Row<size_t>` object.

``` cpp
// [[Rcpp::plugins(mlpack11)]]
// [[Rcpp::depends(RcppArmadillo)]]

#include <RcppArmadillo.h>
using namespace Rcpp;
#include <mlpack/core/util/log.hpp>
#include <mlpack/methods/kmeans/kmeans.hpp>
using namespace mlpack::kmeans;
using namespace arma;

// [[Rcpp::export]]
NumericVector fastKm(const arma::mat& data, const size_t& clusters) {
  Row<size_t> assignments;
  KMeans<> k;
  k.Cluster(data, clusters, assignments);
  // Let's change the format of the output to be a little nicer:
  NumericVector clust(data.n_cols);
  for (int i = 0; i < assignments.n_cols; i++) {
    clust[i] = assignments(i) + 1; // cluster assignments are 0-based
  }
  return clust;
}
```

(Alternatively: `sourceCpp("`[src/fastKm.cpp](src/fastKm.cpp)`")`)

``` r
data(trees, package = "datasets"); data(faithful, package = "datasets")
microbenchmark(
  kmeans_trees = kmeans(trees, 3),
  fastKm_trees = fastKm(t(trees), 3),
  kmeans_faithful = kmeans(faithful, 2),
  fastKm_faithful = fastKm(t(faithful), 2)
) %>% summary(unit = "ms") %>% knitr::kable(format = "markdown")
```

| expr             |     min|      lq|    mean|  median|      uq|     max|  neval|
|:-----------------|-------:|-------:|-------:|-------:|-------:|-------:|------:|
| kmeans\_trees    |  0.2301|  0.2515|  0.3552|  0.2841|  0.3655|  1.1422|    100|
| fastKm\_trees    |  0.0646|  0.0827|  0.1223|  0.0972|  0.1156|  0.8988|    100|
| kmeans\_faithful |  0.2766|  0.2949|  0.4182|  0.3270|  0.4627|  1.8186|    100|
| fastKm\_faithful |  0.1605|  0.2145|  0.2557|  0.2298|  0.2808|  0.4853|    100|

Modules
-------

### Simple

References
==========

-   Eddelbuettel, D. (2013). Seamless R and C++ Integration with Rcpp. New York, NY: Springer Science & Business Media. <http://doi.org/10.1007/978-1-4614-6868-4>
-   Wickham, H. A. (2014). Advanced R. Chapman and Hall/CRC. <http://doi.org/10.1201/b17487>
