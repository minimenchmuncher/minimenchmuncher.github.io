---
title: Warning Catching Issues
date: "2016-08-29T13:06:00Z"
layout: post
path: "/suppress-warning/"
tags:
  - R
  - Data Science
---

R's built-in `suppressWarnings()` function has a bit of a frightful oversight, which is that you can't selectively suppress some warnings. Having that functionality would be key in certain situations. For example, you might write a function that checks to see if some variable is defined, if some file exists, etc. In those kinds of situations, if that variable *should* be defined or that file *should* exist, and doesn't, but the program can recover in some way, the function should issue a warning. Having lots of these checks could mean a bunch of possible warnings.

Another staple of good analysis is good tests (with `test_that`, for example). Now, to test your function with lots of potential warnings, R gives you more or less 3 options:

1. Ignore the warnings - the programmer would need to know to do this, and it's not a good idea.
2. Suppress the warnings using `supressWarnings()` wrapping around the function you're testing.
3. Changing the `options` so that warnings get ignored.

None of these options are particularly good. #2 would seem to be the best since it affects only that function; however, it still may cause some problems to be missed since it suppresses *all* of the warnings issued by a function. What if you just want to ignore one or two types of warnings, but not others? Here is an answer:

```r
suppressWarning <- function(expr, msgs) {
    withCallingHandlers(expr, warning = function(w) {
        if (any(sapply(msgs, grepl, x = w))) {
            invokeRestart("muffleWarning")
        }
    })
}
```

This compact function `suppressWarning` functions similarly to R's base `suppressWarnings` except that one supplies specific messages into `msgs` either as a single string or a string vector, and may contain regular expressions.
