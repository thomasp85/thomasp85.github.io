---
title: "Announcing curry: Operator based currying and partial application"
description: "curry is a new package just released on CRAN, that allows you to perform currying and partial application of functions using a range of operators."
tags: [R, package, announcement]
categories: [R]
---



I am pleased to announce the release of `curry` - a small package I've developed 
as part of improving my meta-programming skills. `curry` is yet another attempt 
at providing a native currying/partial application mechanism in R. Other 
examples of implementations of this can be found in 
[`purrr`](https://CRAN.R-project.org/package=purrr) and 
[`functional`](https://CRAN.R-project.org/package=functional) (and probably 
others). `curry` sets itself apart in the manner it is used and in the functions 
it creates. `curry` is operator based and a partially applied function retains 
named arguments for easier autocomplete etc. `curry` provides three mechanisms 
for partial application: `%<%` (`curry()`), `%-<%` (`tail_curry()`), and `%><%` 
(`partial()`), as well as a true currying operator (`%<!%`) and a *"weak partial 
application"* (`%<?%`) - read on to see the differences.

## Example

### Currying
Currying is the reduction of the arity of a function by fixing the first 
argument, returning a new function lacking this ([no it's not - read on or click 
here](#real-currying)).


{% highlight r %}
# Equivalent to curry(`+`, 5)
add_5 <- `+` %<% 5
add_5(10)
{% endhighlight %}



{% highlight text %}
#> [1] 15
{% endhighlight %}



{% highlight r %}

# ellipsis are retained when currying
bind_5 <- cbind %<% 5
bind_5(1:10)
{% endhighlight %}



{% highlight text %}
#>       [,1] [,2]
#>  [1,]    5    1
#>  [2,]    5    2
#>  [3,]    5    3
#>  [4,]    5    4
#>  [5,]    5    5
#>  [6,]    5    6
#>  [7,]    5    7
#>  [8,]    5    8
#>  [9,]    5    9
#> [10,]    5   10
{% endhighlight %}

### Tail currying
Tail currying is just like currying except it reduces the arity of the function
from the other end by fixing the last argument.


{% highlight r %}
# Equivalent to tail_curry(`/`, 5)
divide_by_5 <- `/` %-<% 5
divide_by_5(10)
{% endhighlight %}



{% highlight text %}
#> [1] 2
{% endhighlight %}



{% highlight r %}

no_factors <- data.frame %-<% FALSE
df <- no_factors(x = letters[1:5])
class(df$x)
{% endhighlight %}



{% highlight text %}
#> [1] "character"
{% endhighlight %}

### Partial function application
When the argument you wish to fix is not in either end of the argument list it
is necessary to use a more generalised approach. Using `%><%` (or `partial()`)
it is possible to fix any (and multiple) arguments in a function using a list of
values to fix.


{% highlight r %}
dummy_lengths <- vapply %><% list(FUN = length, FUN.VALUE = integer(1))
test_list <- list(a = 1:5, b = 1:10)
dummy_lengths(test_list)
{% endhighlight %}



{% highlight text %}
#>  a  b 
#>  5 10
{% endhighlight %}

Other efforts in this has the drawback of returning a new function with just an
ellipsis, making argument checks and autocomplete impossible. With `curry` the
returned functions retains named arguments (minus the fixed ones).


{% highlight r %}
args(no_factors)
{% endhighlight %}



{% highlight text %}
#> function (..., row.names = NULL, check.rows = FALSE, check.names = TRUE, 
#>     fix.empty.names = TRUE) 
#> NULL
{% endhighlight %}



{% highlight r %}
args(dummy_lengths)
{% endhighlight %}



{% highlight text %}
#> function (X, ..., USE.NAMES = TRUE) 
#> NULL
{% endhighlight %}

### Real currying
The above uses a very loose (incorrect) definition of currying. The correct
definition is that currying a function returns a function taking a single 
argument (the first from the original function). Calling a curried function will
return a new function accepting the second argument of the original function and
so on. Once all arguments of the original function has been consumed it 
evaluates the call and returns the result. Thus:


{% highlight r %}
# Correct currying
foo(arg1)(arg2)(arg3)

# Incorrect currying
foo(arg1)(arg2, arg3)
{% endhighlight %}

True currying is less useful in R as it does not play nice with function 
containing `...` as the argument list will never be consumed. Still, it is 
available in `curry` using the `Curry()` function or the `%<!%` operator:


{% highlight r %}
testfun <- function(x = 10, y, z) {
  x + y + z
}
curriedfun <- Curry(testfun)
curriedfun(1)(2)(3)
{% endhighlight %}



{% highlight text %}
#> Error in do.call("testfun", args): could not find function "testfun"
{% endhighlight %}



{% highlight r %}

# Using the operator
testfun %<!% 1 %<!% 2 %<!% 3
{% endhighlight %}



{% highlight text %}
#> Error in do.call("testfun", args): could not find function "testfun"
{% endhighlight %}



{% highlight r %}

# The strict operator is only required for the first call
testfun %<!% 1 %<% 2 %<% 3
{% endhighlight %}



{% highlight text %}
#> Error in do.call("testfun", args): could not find function "testfun"
{% endhighlight %}

As with the partial application functionality the strict currying retains
argument names and defaults at each step of the currying sequence:


{% highlight r %}
args(curriedfun)
{% endhighlight %}



{% highlight text %}
#> function (x = 10) 
#> NULL
{% endhighlight %}

### Weak partial function application
The last functionality provided by `curry` is a *"weak"* partial function 
application in the sense that it sets (or changes) argument defaults. Thus, 
compared to partial application it returns a function with the same arguments,
but if the defaulted arguments are ignored it will be equivalent to a partial
application. Defaults can be set or changed using the `set_defaults()` function
or the `%<?%` operator:


{% highlight r %}
testfun <- function(x = 1, y = 2, z = 3) {
  x + y + z
}
testfun()
{% endhighlight %}



{% highlight text %}
#> [1] 6
{% endhighlight %}



{% highlight r %}
testfun2 <- testfun %<?% list(y = 10)
testfun3 <- testfun %><% list(y = 10)
testfun2()
{% endhighlight %}



{% highlight text %}
#> Error in do.call("testfun", args): could not find function "testfun"
{% endhighlight %}



{% highlight r %}
testfun3()
{% endhighlight %}



{% highlight text %}
#> Error in do.call("testfun", args): could not find function "testfun"
{% endhighlight %}



{% highlight r %}

testfun2(y = 20)
{% endhighlight %}



{% highlight text %}
#> Error in do.call("testfun", args): could not find function "testfun"
{% endhighlight %}



{% highlight r %}
testfun3(y = 20)
{% endhighlight %}



{% highlight text %}
#> Error in testfun3(y = 20): unused argument (y = 20)
{% endhighlight %}


## Installation
`curry` can be installed from CRAN:


{% highlight r %}
install.packages('curry')
{% endhighlight %}

but for the latest and greatest use GitHub:


{% highlight r %}
if (!require(devtools)) {
    install.packages(devtools)
}
devtools::install_github('thomasp85/curry')
{% endhighlight %}

## Performance

`curry` adds a layer around the manipulated functions adding some overhead, but 
the overhead is not accumulative (currying or partially applying multiple times 
does not add additional overhead). depending on the runtime of the function the 
overhead can seem large but as the complexity of the call increases, the effect 
of the overhead will decrease:


{% highlight r %}
library(microbenchmark)
meanP <- mean %><% list(na.rm = TRUE)

data <-  sample(1000)
microbenchmark(mean(data, na.rm = TRUE), meanP(data))
{% endhighlight %}



{% highlight text %}
#> Unit: microseconds
#>                      expr    min      lq      mean  median      uq
#>  mean(data, na.rm = TRUE) 15.867 21.0775  26.95604 28.4145 30.8590
#>               meanP(data) 61.474 76.8480 101.56067 86.1620 92.1305
#>       max neval
#>    52.100   100
#>  1531.636   100
{% endhighlight %}



{% highlight r %}

data <- sample(1e6)
microbenchmark(mean(data, na.rm = TRUE), meanP(data))
{% endhighlight %}



{% highlight text %}
#> Unit: milliseconds
#>                      expr      min       lq     mean   median       uq
#>  mean(data, na.rm = TRUE) 10.28008 11.76435 18.93964 12.15770 13.98587
#>               meanP(data) 12.42514 13.56954 25.53669 14.22797 50.61040
#>       max neval
#>  53.39197   100
#>  98.23340   100
{% endhighlight %}

Happy coding!
