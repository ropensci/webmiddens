webmiddens
==========



[![Project Status: WIP – Initial development is in progress, but there has not yet been a stable, usable release suitable for the public.](https://www.repostatus.org/badges/latest/wip.svg)](https://www.repostatus.org/#wip)
[![Build Status](https://travis-ci.com/ropenscilabs/webmiddens.svg?branch=master)](https://travis-ci.com/ropenscilabs/webmiddens)
[![Build status](https://ci.appveyor.com/api/projects/status/m0rfrp3eb285bwfd?svg=true)](https://ci.appveyor.com/project/sckott/webmiddens)
[![codecov](https://codecov.io/gh/ropenscilabs/webmiddens/branch/master/graph/badge.svg)](https://codecov.io/gh/ropenscilabs/webmiddens)

simple caching of HTTP requests/responses, hooking into webmockr (https://github.com/ropensci/webmockr)
for the HTTP request matching

A midden is a debris pile constructed by a woodrat/pack rat (https://en.wikipedia.org/wiki/Pack_rat#Midden)

### the need

- `vcr` is meant really for testing, or script use. i don't think it fits
well into a use case where another pkg wants to cache responses
- `memoise` seems close-ish but doesn't fit needs, e.g., no expiry, not specific
to HTTP requests, etc.
- we need something specific to HTTP requests, that allows expiration handling, a few different caching location options, works across HTTP clients, etc
- caching just the http responses means the rest of the code in the function can change, and the response can still be cached
    - the downside, vs. memoise, is that we're only caching the http response, so if there's still a lot of time spent processing the response, then the function will still be quite slow - BUT, if the HTTP response processing code is within a function, you could memoise that function
- memoise is great, but since it caches the whole function call, you don't benefit from individually caching each http request, which we do here. if you cache each http request, then any time you do that same http request, it's response is already cached

### brainstorming

- use `webmockr` to match requests (works with `crul` and `httr`, maybe `curl` soon)
- possibly match on, and expire based on headers: Cache-Control, Age, Last-Modified,
ETag, Expires (see Ruby's faraday-http-cache (https://github.com/plataformatec/faraday-http-cache#what-gets-cached))
- caching backends: probably all binary to save disk space since most likely
we don't need users to be able to look at plain text of caches
- expiration: set a time to expire. if set to `2019-03-08 00:00:00` and it's
`2019-03-07 23:00:00`, then 1 hr from now the cache will expire, and a new real HTTP
request will need to be made (i.e., the cache will be deleted whenever the next
HTTP request is made)

### http libraries

right now we only support `crul`, but `httr` support should arrive soon

### installation


```r
remotes::install_github("ropenscilabs/webmiddens")
```

### use_midden()


```r
library(webmiddens)
library(crul)
```

Let's say you have some function `http_request()` that does an HTTP request that
you re-use in various parts of your project or package


```r
http_request <- function(...) {
  x <- crul::HttpClient$new("https://httpbin.org", opts = list(...))
  x$get("get")
}
```

And you have a function `some_fxn()` that uses `http_request()` to do the HTTP 
request, then proces the results to a data.frame or list, etc. This is a super
common pattern in a project or R package that deals with web resources.


```r
some_fxn <- function(...) {
  res <- http_request(...)
  jsonlite::fromJSON(res$parse("UTF-8"))
}
```

Without `webmiddens` the HTTP request happens as usual and all is good


```r
some_fxn()
#> $args
#> named list()
#> 
#> $headers
#> $headers$Accept
#> [1] "application/json, text/xml, application/xml, */*"
#> 
#> $headers$`Accept-Encoding`
#> [1] "gzip, deflate"
#> 
#> $headers$Host
#> [1] "httpbin.org"
#> 
#> $headers$`User-Agent`
#> [1] "libcurl/7.64.1 r-curl/4.3 crul/0.9.2.91"
#> 
#> $headers$`X-Amzn-Trace-Id`
#> [1] "Root=1-5e7e9427-609c99fcf96ba880177b5946"
#> 
#> 
#> $origin
#> [1] "24.21.229.59"
#> 
#> $url
#> [1] "https://httpbin.org/get"
```

Now, with `webmiddens`

run `wm_configuration()` first to set the path where HTTP requests will be cached


```r
wm_configuration("foo1")
```


```
#> configuring midden from $path
```

first request is a real HTTP request


```r
res1 <- use_midden(some_fxn())
res1
#> $args
#> named list()
#> 
#> $headers
#> $headers$Accept
#> [1] "application/json, text/xml, application/xml, */*"
#> 
#> $headers$`Accept-Encoding`
#> [1] "gzip, deflate"
#> 
#> $headers$Host
#> [1] "httpbin.org"
#> 
#> $headers$`User-Agent`
#> [1] "libcurl/7.64.1 r-curl/4.3 crul/0.9.2.91"
#> 
#> $headers$`X-Amzn-Trace-Id`
#> [1] "Root=1-5e7e9427-0fae59facbad5688cba62304"
#> 
#> 
#> $origin
#> [1] "24.21.229.59"
#> 
#> $url
#> [1] "https://httpbin.org/get"
```

second request uses the cached response from the first request


```r
res2 <- use_midden(some_fxn())
res2
#> $args
#> named list()
#> 
#> $headers
#> $headers$Accept
#> [1] "application/json, text/xml, application/xml, */*"
#> 
#> $headers$`Accept-Encoding`
#> [1] "gzip, deflate"
#> 
#> $headers$Host
#> [1] "httpbin.org"
#> 
#> $headers$`User-Agent`
#> [1] "libcurl/7.64.1 r-curl/4.3 crul/0.9.2.91"
#> 
#> $headers$`X-Amzn-Trace-Id`
#> [1] "Root=1-5e7e9427-3cf7275293c69378ec9d42ee"
#> 
#> 
#> $origin
#> [1] "24.21.229.59"
#> 
#> $url
#> [1] "https://httpbin.org/get"
```

### the midden class


```r
x <- midden$new()
x # no path
#> <midden> 
#>   path: 
#>   expiry (sec): not set
# Run $init() to set the path
x$init(path = "forest")
x
#> <midden> 
#>   path: /Users/sckott/Library/Caches/R/forest
#>   expiry (sec): not set
```

The `cache` slot has a `hoardr` object which you can use to fiddle with
files, see `?hoardr::hoard`


```r
x$cache
#> <hoard> 
#>   path: forest
#>   cache path: /Users/sckott/Library/Caches/R/forest
```

Use `expire()` to set the expire time (in seconds). You can set it through
passing to `expire()` or through the environment variable `WEBMIDDENS_EXPIRY_SEC`


```r
x$expire()
#> NULL
x$expire(5)
#> [1] 5
x$expire()
#> [1] 5
x$expire(reset = TRUE)
#> NULL
x$expire()
#> NULL
Sys.setenv(WEBMIDDENS_EXPIRY_SEC = 35)
x$expire()
#> [1] 35
x$expire(reset = TRUE)
#> NULL
x$expire()
#> NULL
```

adfadf


```r
con <- crul::HttpClient$new("https://httpbin.org")
# first request is a real HTTP request
x$r(con$get("get", query = list(stuff = "bananas")))
#> <crul response> 
#>   url: https://httpbin.org/get?stuff=bananas
#>   request_headers: 
#>     User-Agent: libcurl/7.64.1 r-curl/4.3 crul/0.9.2.91
#>     Accept-Encoding: gzip, deflate
#>     Accept: application/json, text/xml, application/xml, */*
#>   response_headers: 
#>     status: HTTP/2 200 
#>     date: Sat, 28 Mar 2020 00:02:47 GMT
#>     content-type: application/json
#>     content-length: 408
#>     server: gunicorn/19.9.0
#>     access-control-allow-origin: *
#>     access-control-allow-credentials: true
#>   params: 
#>     stuff: bananas
#>   status: 200
# following requests use the cached response
x$r(con$get("get", query = list(stuff = "bananas")))
#> <crul response> 
#>   url: https://httpbin.org/get?stuff=bananas
#>   request_headers: 
#>     User-Agent: libcurl/7.64.1 r-curl/4.3 crul/0.9.2.91
#>     Accept-Encoding: gzip, deflate
#>     Accept: application/json, text/xml, application/xml, */*
#>   response_headers: 
#>     status: HTTP/2 200 
#>     date: Sat, 28 Mar 2020 00:02:47 GMT
#>     content-type: application/json
#>     content-length: 408
#>     server: gunicorn/19.9.0
#>     access-control-allow-origin: *
#>     access-control-allow-credentials: true
#>   params: 
#>     stuff: bananas
#>   status: 200
```

verbose output


```r
x <- midden$new(verbose = TRUE)
x$init(path = "rainforest")
x$r(con$get("get", query = list(stuff = "bananas")))
#> CrulAdapter enabled!
#> HttrAdapter enabled!
#> net connect allowed
#> 1 stubs loaded
#> <crul response> 
#>   url: https://httpbin.org/get?stuff=bananas
#>   request_headers: 
#>     User-Agent: libcurl/7.64.1 r-curl/4.3 crul/0.9.2.91
#>     Accept-Encoding: gzip, deflate
#>     Accept: application/json, text/xml, application/xml, */*
#>   response_headers: 
#>     status: HTTP/2 200 
#>     date: Fri, 27 Mar 2020 18:47:24 GMT
#>     content-type: application/json
#>     content-length: 408
#>     server: gunicorn/19.9.0
#>     access-control-allow-origin: *
#>     access-control-allow-credentials: true
#>   params: 
#>     stuff: bananas
#>   status: 200
```

set expiration time


```r
x <- midden$new()
x$init(path = "grass")
x$expire(3)
#> [1] 3
x
#> <midden> 
#>   path: /Users/sckott/Library/Caches/R/grass
#>   expiry (sec): 3
```

Delete all the files in your "midden" (the folder with cached files)


```r
x$cleanup()
#> no files found
```

Delete the "midden" (the folder with cached files)


```r
x$destroy()
```

## Meta

* Please [report any issues or bugs](https://github.com/ropensci/webmiddens/issues).
* License: MIT
* Get citation information for `webmiddens` in R doing `citation(package = 'webmiddens')`
* Please note that this project is released with a [Contributor Code of Conduct][coc].
By participating in this project you agree to abide by its terms.

[![ropensci_footer](https://ropensci.org/public_images/github_footer.png)](https://ropensci.org)

[coc]: https://github.com/ropenscilabs/webmiddens/blob/master/CODE_OF_CONDUCT.md
