---
layout: page
date: "2016-11-11T21:48:00Z"
layout: post
path: "/r-organizing-projects/"
tags:
  - R
  - Data Science
---

### Short Analyses

Whenever starting a new analysis project, I always think, at some point or
another, how I should organize my code. More often than not, it starts as a
single file, usually split into a couple of sections:

1. Libraries/Imports
2. Settings, constants, file paths, connection strings
3. Loading the data and getting it ready for analysis
4. The analysis itself
5. Reporting results by way of graphs, charts, statistical summaries, etc.

Data loading and analysis I'll usually organize into functions from the get-go,
even if they're one-liner functions like

```r
load_data <- function(x) read.csv(x)
```

Pretty much the silliest, straightforward function you could imagine for
loading data from a file `x`. More often than not, this isn't the case so it's
just better to be in the habit of coding this way.

### Larger Projects
Sometimes, you just can't fit all of these aspects into a single file, or you're
trying to do something a bit more creative and/or need a better organizational
structure for some reason. The best reasons that I have found for this are

1. Better code organization
2. Better documentation
3. Better deployment options
4. Testing
5. Code sharing

#### Code organization
R, unfortunately for us, doesn't have a particularly great way of splitting code
across multiple files. The best by far is organizing your code into packages.
Yes, that includes even if you aren't compiling any C/C++ code, and yes, even if
you don't plan on ever distributing it- purely from an organizational
standpoint.

The reason that I say that is that putting all of your code into a package is,
first and foremost, the only way to use code across multiple files and still
retain full use of the Rstudio debugger (at least, insofar as it works at all).
Effectively, you could use the `source` command to load up functions into
memory, but debugging them when called works much more poorly.

You might ask, why would you want to split your code across multiple files in
the first place? Well, you really don't have to, but doing so makes finding
things much easier. If you had a single file, you'd see code that looked like
this:

```r
DAL_a <- function(x) {...}
DAL_b <- function(x) {...}
DAL_c <- function(x) {...}

analysis_a <- function(x, y, z) {...}
analysis_b <- funcion(x, y, z) {...}

helper_1 <- function(x) {...}
helper_2 <- function(x) {...}

mymethod <- function(x, ...) UseMethod("mymethod", x)
mymethod.numeric <- function(x, ...)
```

That's a best-case scenario. Where should you actually be putting the code for
any methods that you're using? At the beginning or the end? What about your DAL
functions? at the beginning? what about the case that `DAL_c`, for example, is
saving the results of the analysis somewhere? Do you group functions by what
type of job they're doing or by when they usually happen chronologically?

It's much simpler to split this code across different files. My rule of thumb
is that functions each take their own file, unless they are closely related
and/or call each other. For example, if `helper_1` is called by `analysis_a`,
I'll generally keep them in the same file.

Similarly, when writing method definitions for a class, I'll put them all in
the same file.

Organizing your files these ways also lets you name them. For example, you'd
have a function called `DAL_a()`. That's what it is, not what it does. Give your
functions descriptive names, like `GetDataByIDFromDatabase()`. Verbose, maybe,
but descriptive. You can then sort these functions into files, thus putting this
DAL into a file called `DAL_mydatatype.R` or the like.

Usually I prepend such files names with what type of function it is. If it
contains DAL functions, I'll usually call it `DAL_something.R`, more analysis
or modeling functions, `model_something.R` or `analysis_something.R`. For
classes, I'll usually group functions together in a file, `class_something.R`.

One little limitation is that once you're committed to a package structure for
your code, you can't do much more organization than putting all of your R code
into the flat `R` directory (excepting platform specific `unix` and `windows`
directories). This is why I prepend my file names this way - at least they'll
come out organized alphabetically.

#### Documentation with Roxygen2

Another huge reason to organize your code as a package is to use roxygen2. This
handy tool is great for keeping on-top of your documentation. You may ask, if
I'm not planning on sharing my code ever, why bother document it? If this is a
one-time analysis, fine, don't document it. But in my experience there is *no 
such thing as a one-time analysis*.

A bit of backstory here: The company I, until recently, worked for
had a number of customers who wanted bespoke analyses that seemed to be one-time
asks. But then, after months, they would seek to make tweaks, update it with new
input data, those kinds of things. Sometimes after months following the initial
release. That's more than enough time for the analyst to forget what was going
on, how the analysis worked, or any of the code. It is so valuable the first
time around then, to just document your code and analysis. If you have never
used roxygen before, I highly recommend it since you don't have to bother
writing `.Rd` files by hand, or changing your `NAMESPACE` file. Check out [this
tutorial](http://kbroman.org/pkg_primer/pages/docs.html) for details.

#### Testing with Testthat

The hugest time-saver of all though, really is in writing tests. My rule of
thumb is to have at least one test per function. That way you can ensure that
each function is running properly, at leasts with some set of inputs. Oftentimes
I'll try to write two tests for each function, one that will expect an error and
the other to expect completion.

Why bother with tests, again, if this is just for you? Some analyses don't need
tests, I'll admit. However, without tests your process may start looking a lot
like this:

1. Write function
2. Get expected results
3. Change something, like adding edge cases, branching, etc.
4. Accidentally change how your code originally runs, invalidating the results
5. Fail to notice this change until it's too late

Tests can easily double the size of your codebase, but trust me, your boss will
thank you if step 5 never happens.

Don't expect miracles though. I've fought a lot with Testthat, and will report
more of my trials and tribulations in future posts.

#### Other Factors

I'm not going to talk about the obvious reasons to use packages, but they do
exist:

* Ability to use compiled code
* calls out dependencies more strongly
* Better deployment options (easy to install on other machines)
* Version control

If you need any of these things, I really don't have to convince you to be
packaging your code, do I.

### Last Thoughts

R packages aren't a panacea, they won't fix all of your problems; rather,
organizing your code this way can cause other problems.

For example, debugging your code can sometimes be much trickier. It's important to keep in
mind that if you build & reload your package, check your environments.
Sometimes when I'm developing code, I'll source a single file (building can
be slow, sourcing a single function is faster), but it'll remain loaded when
you reload, and be in the global namespace (whereas you'll want to be looking in
the package namespace for your new edits).
