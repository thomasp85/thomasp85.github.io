---
title: "Introducing reqres"
description: "A modern R interface to HTTP messages"
tags: [R, reqres, HTTP, webserver]
categories: [R]
large_thumb: true
img:
    thumb: "/assets/images/reqres_logo.png"
---

<img src="/assets/images/reqres_logo_small.png" align="right" style="width:50%"/>

I'm very happy to announce that `reqres` has been released on CRAN. `reqres` is
a new (in R context) approach to working with HTTP messages, that is, the 
**req**uests you send to a server and the **res**pons it returns. The 
uunderlying mechanics of a web server is seldom something that R users comes 
into contact with, indeed the most popular way of using R code for the web is 
[`Shiny`](http://shiny.rstudio.com) by RStudio and 
[`OpenCPU`](https://www.opencpu.org) by Jeroen Ooms, both of which abstracts the
actual HTTP messaging away in order to provide a more friendly and R-native
interface to building web apps and services. 

*So why bother with HTTP in R at all?*  
Both of the above frameworks favors *ease-of-use* over control, and sometimes 
you just want control. Maybe you don't need the overhead that comes with 
full-fledged web app frameworks, maybe the high abstraction level makes it 
difficult to achieve what you want, or maybe you're the developer of Shiny or
OpenCPU and wants to declutter your codebase 😉. I'm of course
here to tell you to take a look at [`fiery`](http://github.com/thomasp85/fiery)
(which I'm developing) if you can give a nod to the two former reasons. I'm not
going to spend more time on how and why to build a web server (that will happen
in another series of blog posts), but simply state that there are very valid 
reasons for working directly with HTTP messaging in R and that `reqres` is here
to soothe the pain of it, should you ever be in that position.

## An overview of *reqres*
There are two main objects in `reqres` that the developer should know about. The
`Request` class and the `Response` class. Both of these are build on 
[`R6`](https://github.com/r-lib/R6) and heavily inspired by the request and 
response classes in [Express.js](http://expressjs.com) (a web server framework
for [Node.js](https://nodejs.org/en/)) to the point of seeming very familiar if
you've ever worked with Express.

### The *Request* class
When a HTTP request is recieved on the server the most likely way it ends up in
R is through the [`httpuv`](https://github.com/rstudio/httpuv) package, a 
minimal web server build upon libuv (which were developed for Node - we're 
beginning to owe the JavaScript crowd a beer). `httpuv` passes the request on as
a Rook compliant environment ([Rook](https://github.com/jeffreyhorner/Rook) 
being an earlier web server specification developed by Jeffrey Horner) and this
is the point where `reqres` can intercept it and make your life easier. 

In order to showcase the `Request` class we need a HTTP request. Thankfully 
`fiery` provides a function for mocking Rook requests, so we can play around 
with `reqres` without building up a true server.


{% highlight r %}
library(reqres)
rook <- fiery::fake_request(
    'http://www.data-imaginist.com/reqres/demo?user=Thomas+Lin+Pedersen&id=123',
    method = 'get',
    content = '{"numbers":[1,2,3],"letters":["a","b", "c"]}',
    headers = list(
        Content_Type = 'application/json',
        Accept = 'application/json, application/xml, text/*',
        Accept_Encoding = 'gzip',
        Cookie = 'session=321'
    )
)

request <- Request$new(rook)
request
{% endhighlight %}



{% highlight text %}
## A HTTP request
## ==============
## Trusted: No
##  Method: get
##     URL: http://www.data-imaginist.com:80/reqres/demo?user=Thomas+Lin+Pedersen&id=123
{% endhighlight %}

We now have a supercharged `Request` object. Being an `R6` class it uses 
reference semantics and there will thus only exist one version of this request
no matter how many times we reassign it to a new variable. We can ask the 
request all sorts on things about itself, such as the method, full url, path 
(the part of the url following the host address), the query (the optional part
of the url following the `?`).


{% highlight r %}
request$method
{% endhighlight %}



{% highlight text %}
## [1] "get"
{% endhighlight %}



{% highlight r %}
request$url
{% endhighlight %}



{% highlight text %}
## [1] "http://www.data-imaginist.com:80/reqres/demo?user=Thomas+Lin+Pedersen&id=123"
{% endhighlight %}



{% highlight r %}
request$path
{% endhighlight %}



{% highlight text %}
## [1] "/reqres/demo"
{% endhighlight %}



{% highlight r %}
request$query
{% endhighlight %}



{% highlight text %}
## $user
## [1] "Thomas Lin Pedersen"
## 
## $id
## [1] 123
{% endhighlight %}

As can be seen the query gets parsed automatically into a list. The same is true
for cookies. One surprise might come when we try to look at the body.


{% highlight r %}
request$body
{% endhighlight %}



{% highlight text %}
## NULL
{% endhighlight %}

The body is only parsed on request. The reason for this is that the request body
can be in all sorts of formats, some not even understandable by R. The format of
the body is advertised in the `Content-Type` header (here `application/json`)
and we can ask the request whether it is of a certain format.


{% highlight r %}
request$is('html')
{% endhighlight %}



{% highlight text %}
## [1] FALSE
{% endhighlight %}



{% highlight r %}
request$is('json')
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

This can be used to determine the correct approach to reading the body. An 
easier way is through the `parser()` method that takes multiple different 
parsing functions and chooses the correct (if present) and fills in the body. 
`reqres` already comes with a list of parsers for the standard formats so often
this is a very easy task.


{% highlight r %}
request$parse(default_parsers)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
request$body
{% endhighlight %}



{% highlight text %}
## $numbers
## [1] 1 2 3
## 
## $letters
## [1] "a" "b" "c"
{% endhighlight %}

The method returns `TRUE` if successful and `FALSE` otherwise. One little magic
feature is that the body is automatically decompressed if compressed (e.g. 
gzipped).

Arbitrary headers can be extracted with the `get_header()` method.


{% highlight r %}
request$get_header('Accept-Encoding')
{% endhighlight %}



{% highlight text %}
## [1] "gzip"
{% endhighlight %}

The last major feature of the `Request` class is content negotiation. It is 
expected that the client informs the server what format, encoding, etc it 
understands and prefers and the server then choses the best one it can do (this
is communicated through the `Accept(-*)` headers). The `Request` class has a 
range of methods that helps you chose the correct format of the response body.


{% highlight r %}
request$accepts(c('html', 'text/plain'))
{% endhighlight %}



{% highlight text %}
## [1] "html"
{% endhighlight %}

While the content negotiation seems relatively simple in our little contrived
example, it can easily end up being hairy as each format can be weighted by a
priority score and wildcards should be prioritized less. The `accepts()` method
takes care of all of this for you and simply returns the prefered choice out of
the given.

### The *Response* class
While the `Request` class is mainly meant for parsing and reading the intend of
the client, `Response` class is meant for manipulation, ultimately resulting in
an answer to the `Request`. A response is always linked to a request and cannot 
exist in solitude. While it can be created using the standard `R6` 
`Response$new()` ideom, it is recommended to create one from the request 
instead.


{% highlight r %}
response <- request$respond()
response
{% endhighlight %}



{% highlight text %}
## A HTTP response
## ===============
##         Status: 404 - Not Found
##   Content type: text/plain
## 
## In response to: http://www.data-imaginist.com:80/reqres/demo?user=Thomas+Lin+Pedersen&id=123
{% endhighlight %}

The reason why this is recommended is that the `respond()` method will always
return a response, either creating one or returning the one already exisiting.
`Response$new()` will throw an error if a response has already been created for
the request.


{% highlight r %}
try(Response$new(request))
identical(response, request$respond())
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

Responses are initialised to `404- Not Found` with an empty body but it is often 
desirable to change this (unless, of course, the requested ressource is not 
found). Status codes can be manipulated with the `status` field or the 
`status_with_text()` method which will also update the body to contain the name
of the status code, e.g.


{% highlight r %}
response$status_with_text(416)
response$body
{% endhighlight %}



{% highlight text %}
## [1] "Range Not Satisfiable"
{% endhighlight %}

Headers can be set with the `set_header()` and `append_header()` method and 
retrieved with `get_header()`.


{% highlight r %}
response$set_header('Server', 'fiery-0.2.3')
response$get_header('Server')
{% endhighlight %}



{% highlight text %}
## [1] "fiery-0.2.3"
{% endhighlight %}

The `set_header()` method is the lowest common denominator when it comes to 
adding headers to your response. In addition there are a range of helpers for 
specific common headers.


{% highlight r %}
response$timestamp()
response$get_header('Date')
{% endhighlight %}



{% highlight text %}
## [1] "Thu, 10 Aug 2017 09:15:20 GMT"
{% endhighlight %}



{% highlight r %}
response$type <- 'json'
response$get_header('Content-Type')
{% endhighlight %}



{% highlight text %}
## [1] "application/json"
{% endhighlight %}



{% highlight r %}
response$set_links(alternative = '/feed')
response$get_header('Link')
{% endhighlight %}



{% highlight text %}
## [1] "</feed>; rel=\"alternative\""
{% endhighlight %}

Furtermore, there's a special method for setting cookies. While cookies are set
with the `Set-Cookie` header, they live in a separate container until the 
response is ready to be send in order to facilitate lookup by cookie name.


{% highlight r %}
response$set_cookie('session', uuid::UUIDgenerate(), max_age = 9000, secure = TRUE)
response$get_header('Set-Cookie')
{% endhighlight %}



{% highlight text %}
## NULL
{% endhighlight %}



{% highlight r %}
response$has_cookie('session')
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
response$as_list()$headers[['Set-Cookie']]
{% endhighlight %}



{% highlight text %}
## [1] "session=776396a3-556e-4bef-88a5-228965ecfb3c; Max-Age=9000; Secure"
{% endhighlight %}

Apart from the headers, the each response also contains their own data store 
that can contain any data. This facilitate communication between different 
middleware (code that modify the HTTP messages on the server). The data store is
used pretty much like the headers.


{% highlight r %}
response$set_data('alphabet', letters)
response$has_data('alphabet')
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
response$get_data('alphabet')
{% endhighlight %}



{% highlight text %}
##  [1] "a" "b" "c" "d" "e" "f" "g" "h" "i" "j" "k" "l" "m" "n" "o" "p" "q"
## [18] "r" "s" "t" "u" "v" "w" "x" "y" "z"
{% endhighlight %}



{% highlight r %}
response$remove_data('alphabet')
{% endhighlight %}

The data contained in the data store will never become part of the actual 
response so anything can be added here safely.

Often the server responds with a file, e.g. a HTML file defining the web page 
the client requested. Files are easily added with the `file` field, which will
take care of setting the `Content-Type` and `Last-Modified` headers as well as
checking that the file actually exists.


{% highlight r %}
response$file <- system.file('NEWS.md', package = 'reqres')
response$type
{% endhighlight %}



{% highlight text %}
## [1] "text/markdown"
{% endhighlight %}



{% highlight r %}
response$get_header('Last-Modified')
{% endhighlight %}



{% highlight text %}
## [1] "Thu, 10 Aug 2017 08:33:54 GMT"
{% endhighlight %}

If the file is meant for download rather than display the `attach()` method will
set the correct headers to indicate to the browser that it should initiate a 
download.


{% highlight r %}
response$attach(system.file('help', 'figures', 'reqres_logo.png', package = 'reqres'))
response$type
{% endhighlight %}



{% highlight text %}
## [1] "image/png"
{% endhighlight %}



{% highlight r %}
response$get_header('Content-Disposition')
{% endhighlight %}



{% highlight text %}
## [1] "attachment; filename=reqres_logo.png"
{% endhighlight %}

Lastly, the response body is accessible in the `body` field. It can be 
absolutely anything you wish as until the response is send of, but should be
formatted to either a raw vector or a string prior to handing the response of to
e.g. `httpuv`. Thankfully there's a parallel to `Request$parse()` in the form of
`Response$format` that performs content negotiation based on the supplied 
formatters, chooses the prefered one and applies it, finally applying 
compression if the client permits it. To make life easier the standard 
formatters have been collected in `default_formatters` so this step is 
easy-peasy (headers will of course be set for you).


{% highlight r %}
response$body <- mtcars
head(response$body)
{% endhighlight %}



{% highlight text %}
##                    mpg cyl disp  hp drat    wt  qsec vs am gear carb
## Mazda RX4         21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## Mazda RX4 Wag     21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## Datsun 710        22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## Hornet 4 Drive    21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## Hornet Sportabout 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
## Valiant           18.1   6  225 105 2.76 3.460 20.22  1  0    3    1
{% endhighlight %}



{% highlight r %}
response$format(default_formatters, compress = FALSE)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
response$body
{% endhighlight %}



{% highlight text %}
## [{"mpg":21,"cyl":6,"disp":160,"hp":110,"drat":3.9,"wt":2.62,"qsec":16.46,"vs":0,"am":1,"gear":4,"carb":4,"_row":"Mazda RX4"},{"mpg":21,"cyl":6,"disp":160,"hp":110,"drat":3.9,"wt":2.875,"qsec":17.02,"vs":0,"am":1,"gear":4,"carb":4,"_row":"Mazda RX4 Wag"},{"mpg":22.8,"cyl":4,"disp":108,"hp":93,"drat":3.85,"wt":2.32,"qsec":18.61,"vs":1,"am":1,"gear":4,"carb":1,"_row":"Datsun 710"},{"mpg":21.4,"cyl":6,"disp":258,"hp":110,"drat":3.08,"wt":3.215,"qsec":19.44,"vs":1,"am":0,"gear":3,"carb":1,"_row":"Hornet 4 Drive"},{"mpg":18.7,"cyl":8,"disp":360,"hp":175,"drat":3.15,"wt":3.44,"qsec":17.02,"vs":0,"am":0,"gear":3,"carb":2,"_row":"Hornet Sportabout"},{"mpg":18.1,"cyl":6,"disp":225,"hp":105,"drat":2.76,"wt":3.46,"qsec":20.22,"vs":1,"am":0,"gear":3,"carb":1,"_row":"Valiant"},{"mpg":14.3,"cyl":8,"disp":360,"hp":245,"drat":3.21,"wt":3.57,"qsec":15.84,"vs":0,"am":0,"gear":3,"carb":4,"_row":"Duster 360"},{"mpg":24.4,"cyl":4,"disp":146.7,"hp":62,"drat":3.69,"wt":3.19,"qsec":20,"vs":1,"am":0,"gear":4,"carb":2,"_row":"Merc 240D"},{"mpg":22.8,"cyl":4,"disp":140.8,"hp":95,"drat":3.92,"wt":3.15,"qsec":22.9,"vs":1,"am":0,"gear":4,"carb":2,"_row":"Merc 230"},{"mpg":19.2,"cyl":6,"disp":167.6,"hp":123,"drat":3.92,"wt":3.44,"qsec":18.3,"vs":1,"am":0,"gear":4,"carb":4,"_row":"Merc 280"},{"mpg":17.8,"cyl":6,"disp":167.6,"hp":123,"drat":3.92,"wt":3.44,"qsec":18.9,"vs":1,"am":0,"gear":4,"carb":4,"_row":"Merc 280C"},{"mpg":16.4,"cyl":8,"disp":275.8,"hp":180,"drat":3.07,"wt":4.07,"qsec":17.4,"vs":0,"am":0,"gear":3,"carb":3,"_row":"Merc 450SE"},{"mpg":17.3,"cyl":8,"disp":275.8,"hp":180,"drat":3.07,"wt":3.73,"qsec":17.6,"vs":0,"am":0,"gear":3,"carb":3,"_row":"Merc 450SL"},{"mpg":15.2,"cyl":8,"disp":275.8,"hp":180,"drat":3.07,"wt":3.78,"qsec":18,"vs":0,"am":0,"gear":3,"carb":3,"_row":"Merc 450SLC"},{"mpg":10.4,"cyl":8,"disp":472,"hp":205,"drat":2.93,"wt":5.25,"qsec":17.98,"vs":0,"am":0,"gear":3,"carb":4,"_row":"Cadillac Fleetwood"},{"mpg":10.4,"cyl":8,"disp":460,"hp":215,"drat":3,"wt":5.424,"qsec":17.82,"vs":0,"am":0,"gear":3,"carb":4,"_row":"Lincoln Continental"},{"mpg":14.7,"cyl":8,"disp":440,"hp":230,"drat":3.23,"wt":5.345,"qsec":17.42,"vs":0,"am":0,"gear":3,"carb":4,"_row":"Chrysler Imperial"},{"mpg":32.4,"cyl":4,"disp":78.7,"hp":66,"drat":4.08,"wt":2.2,"qsec":19.47,"vs":1,"am":1,"gear":4,"carb":1,"_row":"Fiat 128"},{"mpg":30.4,"cyl":4,"disp":75.7,"hp":52,"drat":4.93,"wt":1.615,"qsec":18.52,"vs":1,"am":1,"gear":4,"carb":2,"_row":"Honda Civic"},{"mpg":33.9,"cyl":4,"disp":71.1,"hp":65,"drat":4.22,"wt":1.835,"qsec":19.9,"vs":1,"am":1,"gear":4,"carb":1,"_row":"Toyota Corolla"},{"mpg":21.5,"cyl":4,"disp":120.1,"hp":97,"drat":3.7,"wt":2.465,"qsec":20.01,"vs":1,"am":0,"gear":3,"carb":1,"_row":"Toyota Corona"},{"mpg":15.5,"cyl":8,"disp":318,"hp":150,"drat":2.76,"wt":3.52,"qsec":16.87,"vs":0,"am":0,"gear":3,"carb":2,"_row":"Dodge Challenger"},{"mpg":15.2,"cyl":8,"disp":304,"hp":150,"drat":3.15,"wt":3.435,"qsec":17.3,"vs":0,"am":0,"gear":3,"carb":2,"_row":"AMC Javelin"},{"mpg":13.3,"cyl":8,"disp":350,"hp":245,"drat":3.73,"wt":3.84,"qsec":15.41,"vs":0,"am":0,"gear":3,"carb":4,"_row":"Camaro Z28"},{"mpg":19.2,"cyl":8,"disp":400,"hp":175,"drat":3.08,"wt":3.845,"qsec":17.05,"vs":0,"am":0,"gear":3,"carb":2,"_row":"Pontiac Firebird"},{"mpg":27.3,"cyl":4,"disp":79,"hp":66,"drat":4.08,"wt":1.935,"qsec":18.9,"vs":1,"am":1,"gear":4,"carb":1,"_row":"Fiat X1-9"},{"mpg":26,"cyl":4,"disp":120.3,"hp":91,"drat":4.43,"wt":2.14,"qsec":16.7,"vs":0,"am":1,"gear":5,"carb":2,"_row":"Porsche 914-2"},{"mpg":30.4,"cyl":4,"disp":95.1,"hp":113,"drat":3.77,"wt":1.513,"qsec":16.9,"vs":1,"am":1,"gear":5,"carb":2,"_row":"Lotus Europa"},{"mpg":15.8,"cyl":8,"disp":351,"hp":264,"drat":4.22,"wt":3.17,"qsec":14.5,"vs":0,"am":1,"gear":5,"carb":4,"_row":"Ford Pantera L"},{"mpg":19.7,"cyl":6,"disp":145,"hp":175,"drat":3.62,"wt":2.77,"qsec":15.5,"vs":0,"am":1,"gear":5,"carb":6,"_row":"Ferrari Dino"},{"mpg":15,"cyl":8,"disp":301,"hp":335,"drat":3.54,"wt":3.57,"qsec":14.6,"vs":0,"am":1,"gear":5,"carb":8,"_row":"Maserati Bora"},{"mpg":21.4,"cyl":4,"disp":121,"hp":109,"drat":4.11,"wt":2.78,"qsec":18.6,"vs":1,"am":1,"gear":4,"carb":2,"_row":"Volvo 142E"}]
{% endhighlight %}



{% highlight r %}
response$compress()
head(response$body, 30)
{% endhighlight %}



{% highlight text %}
##  [1] 1f 8b 08 00 00 00 00 00 00 03 9d 97 dd 6e e3 36 10 85 5f 85 d0 b5 57
## [24] 10 c9 a1 48 f9 ae 6b
{% endhighlight %}



{% highlight r %}
response$type
{% endhighlight %}



{% highlight text %}
## [1] "application/json"
{% endhighlight %}



{% highlight r %}
response$get_header('Content-Encoding')
{% endhighlight %}



{% highlight text %}
## [1] "gzip"
{% endhighlight %}

## Wrapping up
As I hope I've made clear, working directly with HTTP messages does not need to
be a drag. Sure, there's a sea of conventions around headers and status codes 
that can seem daunting, but `reqres` takes care of the minimum requirements for
you, letting you focus on the server logic instead.

### Whats next
I obviously hope `reqres` will become pervasive in the world of R web 
technologies as I think it will make everyones life easier. `fiery` already uses
it in the development version on GitHub, as does `routr` so if you're going to
use either of these packages you'll automatically become aquainted. Furthermore
I've heard expression of interest from other developers so hopefully it will be
adopted beyond my own packages.
