---
layout: page
date: "2016-11-17T6:00:00Z"
layout: post
path: "/r-config-files/"
tags:
  - R
  - Data Science
---

Personally, I'm a big fan of functional programming. I find that the ability to
chain functions together and the idea of object immutability to be extremely
useful and easy to understand, especially when doing analysis on data. I think
that's why I like R for these kinds of applications.

I think also that for larger projects I'll arrange my code into an R package. Reasons
for doing this were in another post, but for now, I'll just say that it's
more shareable, better code organization, better documentation, better
deployment options, and many other reasons to do this, even if you aren't
compiling any C or C++ code.

Getting to the point.

Generally speaking, my *short* analyses look like this:

```r
library(lib1)
library(lib2)

const_1 <- "some value"
Pi <- 3.14159
some_file <- "/path/to/file.ext"

getData <- function(x, ...) {...}

doAnalysis <- function(x, ...) {...}

# and possibly some code at the bottom which calls the above functions
```

I've talked about how, when a project grows, your `getData()`-like functions
become your DAL. The part where you call your libraries-- that gets complicated.
But what of these constants here? I find that with smaller projects, putting
all of the constants together, at the top is useful. That way, if I want to
change anything, it's easy, and values in each of the functions are soft-coded.

These constants may include:

* file paths
* specifics about how to read these files, such as
    * number of lines to skip
    * which columns to read
* connection strings to databases
* URLs for web-hosted data
* S3 credentials
* and so on

If your analysis is growing in size and you're trying to package it, chances
are that a. you'll need some place central to put settings like this, and b.
you'll need more than a couple of these settings.

### settings.R

The way I approached this at first was keeping a file called `settings.R` in my
`R` directory. As you might guess, it looked a lot like this:

```r
const_1 <- "some value"
Pi <- 3.14159
some_file <- "/path/to/file.ext"
```

Which actually worked quite well, but lacked 2 things. The first is good organization.
For things like connection strings, it would be much better to pass in a single
object, for example, than several different ones for server URI, username,
password, etc. So I changed my settings files to look like this:

```r
settings <- list(
    const_1 = "some value"
    Pi = 3.14159
    some_file = "/path/to/file.ext"
)
```

Same basic idea, I could even nest it into sections if I wanted to.

### settings.???

The second problem was that there was no way to programmatically change the
settings file! This isn't a common problem for me, but it happened once without
having a great solution<sup>[2](#fn1)</sup>.

<a name="fn1">1</a>: *The solution, if you're curious was that I had a system
call to `sed` *
