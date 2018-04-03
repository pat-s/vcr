vcr
===





[![Project Status: WIP – Initial development is in progress, but there has not yet been a stable, usable release suitable for the public.](http://www.repostatus.org/badges/latest/wip.svg)](http://www.repostatus.org/#wip)
[![Build Status](https://travis-ci.org/ropensci/vcr.svg)](https://travis-ci.org/ropensci/vcr)
[![codecov](https://codecov.io/gh/ropensci/vcr/branch/master/graph/badge.svg)](https://codecov.io/gh/ropensci/vcr)
[![rstudio mirror downloads](http://cranlogs.r-pkg.org/badges/vcr)](https://github.com/metacran/cranlogs.app)
[![cran version](https://www.r-pkg.org/badges/version/vcr)](https://cran.r-project.org/package=vcr)

An R port of the Ruby gem [vcr](https://github.com/vcr/vcr) (i.e., a translation, there's no Ruby here :))

## Overview/Terminology

* Cassette: A _thing_ to record HTTP interactions to. Right now the only option is file system, but in the future could be other things, e.g. a key-value store
* Persisters: defines how to save requests - currently only option is the file system
* Serializers: defines how to serialize the HTTP response - currently only option is YAML; other options in the future could include e.g. json
* _insert cassette_: aka, create a cassette
* _eject cassette_: aka, eject the cassette (no longer recording to that cassette)
* _replay_: refers to using a cached result of an http request that was recorded earlier
* How `vcr` matches: By default it matches on the HTTP method and the URI, but you can tweak this using the `match_requests_on` option.

<details> <summary><strong>How it works in lots of detail</strong></summary> <p>

The very very short version is: `vcr` helps you stub HTTP requests so you 
don't have to repeat yourself.

**The Steps**

1. Use either `vcr::use_cassette` or `vcr::insert_cassette`
  a. If you use `vcr::insert_cassette`, make sure to run `vcr::eject_cassette` when you're done to stop recording
2. When you first run a request with `vcr` there's no cached data to use, so we allow HTTP requests until you're request is done. 
3. Before we run the real HTTP request, we "stub" the request with `webmockr` so that future requests will match the stub.
4. After the stub is made, we run the real HTTP request. 
5. We then disallow HTTP requests so that if the request is done again we use the cached response

When you run that request again using `vcr::use_cassette` or `vcr::insert_cassette`:

* We use `webmockr` to match the request to cached requests, and since we stubbed the request the first time we used the cached response.

Of course if you do a different request, even slightly (but depending on which matching format you decided to use), then 
the request will have no matching stub and no cached resposne, and then a real HTTP request is done, we cache it, then subsequent requests will pull from that cached response.


</p></details>

### Just want to mock and not store on disk?

You're looking for [webmockr](https://github.com/ropensci/webmockr)

<br>

## Best practices

### vcr for tests

* Add `webmockr` and `vcr` to `Suggests` in your package
* Make a file in your `tests/testthat/` directory called `helper-yourpackage.R` (or skip if as similar file already exists). In that file use the following lines to setup your path for storing cassettes (change path to whatever you want):

```r
library("vcr")
invisible(vcr::vcr_configure(dir = "../fixtures/vcr_cassettes"))
```

* In your tests, for whichever tests you want to use `vcr`, wrap them in a `vcr::use_cassette()` call like:

```r
library(testthat)
test_that("my test", {
  vcr::use_cassette("rl_citation", {
    aa <- rl_citation()

    expect_is(aa, "character")
    expect_match(aa, "IUCN")
    expect_match(aa, "www.iucnredlist.org")
  })
})
```

### vcr in your R project

You can use `vcr` in an R project as well. 

* Load `vcr` in your project
* Similar to the above example, use `use_cassette` to run code that does HTTP requests. 
* The first time a real request is done, and after that the cached response will be used.


## Installation


```r
install.packages("vcr")
```


```r
devtools::install_github("ropensci/vcr")
```


```r
library("vcr")
library("crul")
```

## Configuration

Without the user doing anything, we set a number of defaults for easy usage:

* `dir` = "~/vcr/vcr_cassettes"
* `record` = "once"
* `match_requests_on` = `c("method", "uri")`
* `allow_unused_http_interactions` = `TRUE`
* `serialize_with` = "yaml"
* `persist_with` = "FileSystem"
* `ignore_hosts` = `NULL`
* `ignore_localhost` = `FALSE`
* `ignore_request` = `NULL`
* `uri_parser` = `httr::parse_url`
* `preserve_exact_body_bytes` = `FALSE`
* `preserve_exact_body_bytes_for` = `FALSE`
* `turned_off` = `FALSE`
* `ignore_cassettes` = `FALSE`
* `cassettes` = `list()` # empty set
* `linked_context` = `NULL`
* `vcr_logging` = "vcr.log"


You can get the defaults programatically with 


```r
vcr_config_defaults()
```

However, you can change all the above defaults via calling 
`vcr_configure()`


```r
vcr_configure(
  dir = "fixtures/vcr_cassettes",
  record = "once"
)
#> <vcr configuration>
#>   Cassette Dir: fixtures/vcr_cassettes
#>   Record: once
#>   URI Parser: crul::url_parse
#>   Match Requests on: method, uri
#>   Preserve Bytes?: FALSE
```

Calling `vcr_configuration()` gives you some of the more important defaults in a nice tidy print out


```r
vcr_configuration()
#> <vcr configuration>
#>   Cassette Dir: fixtures/vcr_cassettes
#>   Record: once
#>   URI Parser: crul::url_parse
#>   Match Requests on: method, uri
#>   Preserve Bytes?: FALSE
```



## Basic usage




```r
cli <- crul::HttpClient$new(url = "https://httpbin.org")
system.time(
  use_cassette(name = "helloworld", {
    cli$get("get")
  })
)
#>    user  system elapsed 
#>   0.190   0.013   0.573
```

The request gets recorded, and all subsequent requests of the same form used the cached HTTP response, and so are much faster


```r
system.time(
  use_cassette(name = "helloworld", {
    cli$get("get")
  })
)
#>    user  system elapsed 
#>   0.097   0.003   0.105
```



`use_cassette()` is an easier approach. An alternative is to use 
`insert_cassett()` + `eject_cassette()`. 

`use_cassette()` does both insert and eject operations for you, but 
you can instead do them manually by using the above functions. You do have
to eject the cassette after using insert.

## Matchers

`vcr` looks for similarity in your HTTP requests to cached requests. You 
can set what is examined about the request with one or more of the 
following options:

* `body`
* `headers`
* `host`
* `method`
* `path`
* `query`
* `uri`

By default, we use `method` (HTTP method, e.g., `GET`) and `uri` (test for exact match against URI). 

You can set your own options like:




```r
use_cassette(name = "one", {
    cli$post("post", body = list(a = 5))
  }, 
  match_requests_on = c('method', 'headers', 'body')
)
```

## vcr in other languages

The canonical `vcr` (in Ruby) lists ports in other languages at <https://github.com/vcr/vcr>

## TODO

* Logging
* Provide toggling a re-record interval so you can say e.g., after 6 hrs, re-record a real response, updating the cached response
* ...

## Meta

* Please [report any issues or bugs](https://github.com/ropensci/vcr/issues)
* License: MIT
* Get citation information for `vcr` in R doing `citation(package = 'vcr')`
* Please note that this project is released with a [Contributor Code of Conduct](CODE_OF_CONDUCT.md). By participating in this project you agree to abide by its terms.

[![ropensci_footer](https://ropensci.org/public_images/github_footer.png)](https://ropensci.org)
