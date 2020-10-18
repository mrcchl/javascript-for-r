# Exercise - Machine Learning {#v8-ml}

In this chapter we build a package that, via V8, wraps [ml.js](https://github.com/mljs/ml), a library which brings machine learning to JavaScript. It covers quite a few models, in this chapter we only include one: the linear regression. This is an interesting example because it reveals some proceedings that one is likely to run into when creating packages using V8.

```js
const x = [0.5, 1, 1.5, 2, 2.5];
const y = [0, 1, 2, 3, 4];

const regression = new ml.SimpleLinearRegression(x, y);
```

## Dependency {#v8-ml-dependency}

We start by creating a package and add the V8 package as dependency.

```r
usethis::create_package("ml")
usethis::use_package("V8")
```

Then we create the `inst` directory in which we place the ml.js file downloaded from the CDN.

```r
dir.create("inst")
download.file(
  "https://www.lactame.com/lib/ml/4.0.0/ml.min.js", 
  "inst/ml.min.js"
)
```

With the dependency downloaded one can start working on the R code. First, a new V8 context needs to be created and the ml.js file need to be imported in it.

```r
# zzz.R
ml <- NULL

.onLoad <- function(libname, pkgname){
  ml <<- V8::v8()
  mljs <- system.file("ml.min.js", package = "ml")
  ml$source(mljs)
}
```

## Simple Regression {#v8-ml-regression}

First model we make available is the ["simple linear regression"](https://github.com/mljs/regression-simple-linear), which consists of a simple function that takes two arrays. We can thus create a function that takes two vectors, `x`, and `y`, and runs the regression.

```r
#' @export 
ml_simple_lm <- function(y, x){
  # assign x and y
  ml$assign("x", x)
  ml$assign("y", y)

  # run regression
  ml$eval("var regression = new ML.SimpleLinearRegression(x, y);")
  
  # return results
  ml$get("regression")
}
```

Then we can document and load the model to the function.

```r
ml_simple_lm(cars$speed, cars$dist)
## $name
## [1] "simpleLinearRegression"
## 
## $slope
## [1] 0.1655676
## 
## $intercept
## [1] 8.283906
```

This works but has a few issues, namely running two or more regression internally will override the variable `regression` in the V8 context. Let us demonstrate by implementing a function to predict.

```r
#' @export
ml_predict <- function(x){
  ml$call("regression.predict", x)
}
```

We then document and load the functions to then run two regressions in a row and observe the issue. Unlike R the model we created only exists in JavaScript, unlike the `lm`, the function `ml_simple_lm` does not return the model. Therefore, `ml_simple_lm` does not distinguish between models, unlike `predict` which takes the model as first argument and runs the prediction on it.

This implementation of ml.js is indeed very dangerous.

```r
# first model
ml_simple_lm(cars$speed, cars$dist)
ml_predict(15)
## 25.18405

# overrides model
ml_simple_lm(1:10, runif(10))

# produces different predictions
ml_predict(15)
## 10.76742
```

The package ml currently under construction should emulate R in that respect, the `ml_simple_lm` should return the model which should be usable with the `predict` function. In order to do we are going to need to track regressions internally in V8 so the `ml_simple_lm` return a proxy of the model that `predict` can use to predict on the intended model.

In order to track and store regressions internally we are going to declare an empty array when the package is loaded.

```r
# zzz.R
ml <- NULL

.onLoad <- function(libname, pkgname){
  ml <<- V8::v8()
  mljs <- system.file("ml.min.js", package = "ml")
  ml$source(mljs)
  ml$eval("var regressions = [];")
}
```

Then, one can track regressions by creating an R object which is incremented every time `ml_simple_lm` runs, this can be used as variable name in the JavaScript `regressions` array declare in `.onLoad`. This variable name must be stored in the object we intend to return so the `predict` method we will create later on can access the model and run predictions. Finally, in order to declare a new method on the `predict` function we need to return an object bearing a unique class, below we use `mlSimpleRegression`.

```r
counter <- new.env(parent = emptyenv())
counter$regressions <- 0

#' @export 
ml_simple_lm <- function(y, x){
  counter$regressions <- counter$regressions + 1

  # assign variables
  ml$assign("x", x)
  ml$assign("y", y)

  # address
  address <- paste0("regressions['", counter$regressions, "']")

  # create regression
  code <- paste0(
    address, " = new ML.SimpleLinearRegression(x, y);"
  )
  ml$eval(code)

  # retrieve and append address
  regression <- ml$get(address)
  regression$address <- address

  # create object of new class
  structure(
    regression, 
    class = c("mlSimpleRegression", class(regression))
  )
}
```

Then on can implement the `predict` method for `mlSimpleRegression`. The function uses the `address` of the model to run the JavaScript `predict` method on that object.

```r
#' @export 
predict.mlSimpleRegression <- function(object, newdata, ...){
  code <- paste0(object$address, ".predict")
  ml$call(code, newdata)
}
```

We can then build and load the package to it in action.


```r
library(ml)

# first model
model_cars <- ml_simple_lm(cars$speed, cars$dist)

# second model
model_random <- ml_simple_lm(1:10, runif(10))

predict(model_random, 15)
#> [1] 10.11191
predict(model_cars, 15)
#> [1] 10.76742
```