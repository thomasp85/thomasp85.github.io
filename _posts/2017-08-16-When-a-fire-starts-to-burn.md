---
title: "When A Fire Starts to Burn - Fiery 1.0 released"
description: "The R webserver framework fiery has just been updated to v1.0"
tags: [R, fiery, webserver, http]
categories: [R]
large_thumb: true
img:
    thumb: "/assets/images/fiery_logo.png"
---

<img src="/assets/images/fiery_f.jpeg" align="right" style="width:50%"/>

I'm pleased to announce that `fiery` has been updated to version 1.0 and is now
available on CRAN. As the version bump suggests, this is a rather major update 
to the package, fixing and improving upon the framework based on my experience
with it, as well as introducing a number of breaking changes. Below, I will go
through the major points of the update, but also give an overview of the 
framework itself, as I did not have this blog when it was first released and 
information on the framework is thus scarce on the internet (except this 
[nice little post](https://rud.is/b/2016/07/05/a-simple-prediction-web-service-using-the-new-firery-package/) 
by Bob Rudis).

## Significant changes in v1.0
The new version of `fiery` introduces both small and large changes. I'll start
by listing the breaking changes that one should be aware of in existing `fiery`
servers, and then continue to describe other major changes.

### Embracing reqres *BREAKING*
My `reqres` package was 
[recently released]({% post_url 2017-08-13-Introducing-reqres %}) and has been
adopted by `fiery` as the interface for working with HTTP messaging. I have been
a bit torn on whether to build `reqres` into `fiery` or simply let 
[`routr`](https://github.com/thomasp85/routr) use it internally, but in the end
the benefits of a more powerful interface to HTTP requests and responses far
outweighed the added dependency and breaking change.

The change means that everywhere a request object is handed on to an event 
handler (e.g. handlers listening to the `request` event) it is no longer passing
a rook environment but a `Request` object. The easiest fix in existing code is 
to simply extract the rook environment from the `Request` object using the 
`origin` field (this, of course, will not allow you to experience the joy of 
`reqres`).

The change to `reqres` also brings other, smaller, changes to the code base. 
`header` event handlers are now expected to return either `TRUE` or `FALSE` to
indicate whether to proceed or terminate, respectively. Prior to v1.0 they were
expected to return either `NULL` or a rook complaint list response, but as 
responses are now linked to requests, returning them does not make sense. In the
same vein, the return values of `request` event handlers are ignored and the 
response is not passed to `after-request` event handlers as the response can be 
extracted directly from the request.

### Arguments from *before-request* and *before-message* event handlers *BREAKING*
The `before-request` and `before-message` events are fired prior to the actual
HTTP request and WebSocket message handling. The return values from any handler
is passed on as arguments to the `request` and `message` handlers respectively
and these events can thus be used to inject data into the main request and 
message handling. Prior to v1.0 these values were passed in directly as named
arguments, but will now be passed in as a list in the `arg_list` argument. This
is much easier and consistent to work with. An example of the change is:


{% highlight r %}
# Old interface
app <- Fire$new()
app$on('before-request', function(...) {
    list(arg1 = 'Hello', arg2 = 'World')
})
app$on('request', function(arg1, arg2, ...) {
    message(arg1, ' ', arg2)
})

# New interface
app <- Fire$new()
app$on('before-request', function(...) {
    list(arg1 = 'Hello', arg2 = 'World')
})
app$on('request', function(arg_list, ...) {
    message(arg_list$arg1, ' ', arg_list$arg2)
})
{% endhighlight %}

As can be seen the code ends up being a bit more verbose, but the argument list
will be much more predictable.

### Embrassing snake_case *BREAKING*
When I first started developing `fiery` I was young and confused 
(😜). Bottom line I don't think my 
naming scheme was very elegant. While consistent (snake_case for methods and
camelCase for fields), this mix is a bit foreign and I've decided to use this
major release to clean up in the naming and use snake_case consistently 
throughout `fiery`. This has the effect of renaming the `triggerDir` field to 
`trigger_dir` and `refreshRate` to `refresh_rate`. Furthermore this change is
taken to its conclusion by also changing the plugin interface and require 
plugins to expose an `on_attach()` method rather than an `onAttach()` method.

### Keeping the event cycle in non-blocking mode
`fiery` supports running the server in both a blocking and a non-blocking way 
(that is, whether control should be returned to the user after the server is 
started, or not). Before v1.0 the two modes were not equal in their life cycle 
events as only the blocking server had support for `cycle-start` and `cycle-end`
events as well as handling of timed, delayed, and async evaluation. This has
changed and the lifetime of an app running in the two different modes are now
the same. To achieve this `fiery` uses the 
[`later`](https://github.com/r-lib/later) package to continually schedule cycle
evaluation for execution. This means that no matter the timing, cycles will only
be executed if the R process is idle, and it also has the slight inconvenience
of not allowing to stop a server as part of a cycle event (Bug report here:
<https://github.com/rstudio/httpuv/issues/78>). Parallel to the refresh rate of
a blocking server, the refresh rate of a non-blocking server can be set using 
the `refresh_rate_nb` field. By default it is longer than that of a blocking 
server, to give the R process more room to receive instructions from the 
console.

### Mounting a server
With v1.0 it is now possible to specify the root of a `fiery` server. The root
is the part of the URL path that is stripped from the path before sending 
requests on to the handler. This means that it is possible to create sub-app in
`fiery` that do not care at which location they are run. If e.g. the root is set
to `/demo/app` then requests made for `/demo/app/...` will look like `/...` 
internally, and switching the location of the app does not require any change in
the underlying app logic or routing. The root defaults to `''` (nothing), but
can be changed with the `root` field.

### Package documentation
Documentation can never by too good. The state of affairs for documenting 
classes based on reference semantics is not perfect in R, and I still struggle
with the best setup. Still, the current iteration of the documentation is a vast
improvement, compared to the previous release. Notable changes include separate
entries for documentation of events and plugins.

### Grab bag
The host and port can now be set during construction using the `host` and `port`
arguments in `Fire$new()`. `Fire` objects now has a print method, making them
much nicer to look at. The host, port, and root is now advertised when a server
starts. WebSocket connections can now be closed from the server using the 
`close_ws_con` method.

## A Fiery Overview
As promised in the beginning, I'll end with giving an overview of how `fiery` is
used. I'll do this by updating Bob's prediction server to the bright future where
`routr` and `reqres` makes life easy for you:

We'll start by making our fancy AI-machine-learning model of linear 
regressiveness:


{% highlight r %}
set.seed(1492)
x <- rnorm(15)
y <- x + rnorm(15)
fit <- lm(y ~ x)
saveRDS(fit, "model.rds")
{% endhighlight %}

With this at our disposable, we can begin to build up our app:


{% highlight r %}
library(fiery)
library(routr)
app <- Fire$new()

# When the app starts, we'll load the model we saved. Instead of
# polluting our namespace we'll use the internal data store

app$on('start', function(server, ...) {
  server$set_data('model', readRDS('model.rds'))
  message('Model loaded')
})

# Just for show off, we'll make it so that the model is atomatically
# passed on to the request handlers

app$on('before-request', function(server, ...) {
    list(model = server$get_data('model'))
})

# Now comes the biggest deviation. We'll use routr to define our request
# logic, as this is much nicer
router <- RouteStack$new()
route <- Route$new()
router$add_route(route, 'main')

# We start with a catch-all route that provides a welcoming html page
route$add_handler('get', '*', function(request, response, keys, ...) {
    response$type <- 'html'
    response$status <- 200L
    response$body <- '<h1>All your AI are belong to us</h1>'
    TRUE
})
# Then on to the /info route
route$add_handler('get', '/info', function(request, response, keys, ...) {
    response$status <- 200L
    response$body <- structure(R.Version(), class = 'list')
    response$format(json = reqres::format_json())
    TRUE
})
# Lastly we add the /predict route
route$add_handler('get', '/predict', function(request, response, keys, arg_list, ...) {
    response$body <- predict(
        arg_list$model, 
        data.frame(x=as.numeric(request$query$val)),
        se.fit = TRUE
    )
    response$status <- 200L
    response$format(json = reqres::format_json())
    TRUE
})
# And just to show off reqres file handling, we'll add a route 
# for getting a model plot
route$add_handler('get', '/plot', function(request, response, keys, arg_list, ...) {
    f_path <- tempfile(fileext = '.png')
    png(f_path)
    plot(arg_list$model)
    dev.off()
    response$status <- 200L
    response$file <- f_path
    TRUE
})

# Finally we attach the router to the fiery server
app$attach(router)

app$ignite(block = FALSE)
{% endhighlight %}



{% highlight text %}
## Fire started at 127.0.0.1:8080
{% endhighlight %}



{% highlight text %}
## Model loaded
{% endhighlight %}

As can be seen, `routr` makes the request logic nice and compartmentalized, 
while `reqres` makes it easy to work with HTTP messages. What is less apparent is
the work that `fiery` is doing underneath, but that is exactly the point. While
it is possible to use a lot of the advanced features in `fiery`, you don't have 
to - often it is as simple as building up a router and attaching it to a `fiery`
instance. Even WebSocket messaging can be offloaded to the router if you so 
wish.

Of course a simple prediction service is easy to build up in most frameworks - 
it is the To-Do app of data science web server tutorials. I hope to get the time 
to create some more fully fledged example apps soon. Next up in the `fiery`
stack pipeline is getting `routr` on CRAN as well and then begin working on some
of the plugins that will facilitate security, authentication, data storage, etc. 



