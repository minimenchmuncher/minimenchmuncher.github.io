---
title: Using Data Transfer objects
date: 2017-07-12T22:00:00Z
layout: post
path: "/using-dtos/"
tags:
 - R
---


### Data Abstraction Layers (DALs)
So I've talked plenty about DALs in the past. If you've never heard the phrase before, read (this) post. But the essential idea is that for larger projects, you should abstract away your actual data sources because the source may change, the formatting may change, and you don't want it screwing up your downstream analysis.
There are many points to using DALs, the essential function though is to read and write your data from its sources (files, databases, the internet, whatever) into and out of data transfer objects, or DTOs.

### DTOs as lists
Let's take the example of getting your data from a SQL table:

```sql
SELECT id, col1, col2 FROM table1;
```

Simple enough<sup>[1](#fn1)</sup>. Using R's `DBI` package<sup>[2](#fn2)</sup>, your function for getting the data might look something like this:

```R
# get data from a mysql database with the id x and a settings object
get_my_data <- function(x, settings) {
   conn <- DBI::dbConnect(  # connect to database
       drv = RMySQL::MySQL(), dbname = "dbname", host = "localhost",
       username = settings$username, password = settings$username)
   q <- "SELECT id, col1, col2 FROM table;"
   res <- DBI::dbSendQuery(conn, q)  # send query
   all_data <- DBI::dbFetch(res)  # fetch results
   DBI::dbClearResult(res)  # clear the result cache
   DBI::dbDisconnect(conn)  # disconnect from database
   one_row <- dplyr::filter(all_data, id == x)  # get just the row you want
   as.list(one_row)  # return a list instead of data.frame
}
```

If you assigned the result to a variable `mydat` the result will look something like

```R
mydat
$id
[1] 3

$col1
[1] 4

$col2
[1] "hello"
```

This DAL function has a few things to note:

1. Open database connection using credentials from a settings object
2. Fetch data from the database using a hard-coded query.
3. Close the database connection
4. Clear data and disconnect
5. return

Now, the data are contained in a list. When considering how to organize my code, and what flavor of DTO I want to use, I'll usually start here.

### Problems with using lists

The issue with using lists like the one above is that it isn't locked down. If you're using a strongly-typed object oriented language like C# or Java, your DTOs will have well-defined attributes, which makes them very easy to code and develop toward. Just using vanilla lists in R, you can't be assured that your functions will necessarily work with your DTOs, or that your DTOs have the right attributes<sup>[3](#fn3)</sup>.

The answer to that issue is some kind of validation. You really should be doing this somewhere in your code. In the above situation, your best option is to validate when that object gets used, ie, in your analysis functions. I like to

* check the length of each attribute (here they should all be 1)
* check the type of each attribute (here, maybe 2 ints or numerics, and a character)
* Sometimes check the values, to make sure they fall in a certain range, aren't `NULL` or `NA`, anything that would screw up your code execution later on.

The `checkmate` package has some good tools for doing just this. If you have to validate your inputs in your analysis code, I'd recommend doing that first, before anything else gets executed.

That all being said, I think (and I'm opinionated, but that I'm assuming if you're bothering to read this blog, you actually care what I think) that style of coding is sloppy. I like my analysis code to be nice and pristine. I like that when I write code that's doing calculations or analyzing my data, I like seeing the analysis, and generally assuming that the inputs are fine to use. I see validation as more of a data-related task, and it's good to to keep your analysis or your business logic separate from your data.

### S3 Classes

One way forward with this is to make your object a member of an S3 class. That gives you the benefit of moving your validation more on the data side of things. In the above exaple, you could simply say

```R
obj <- structure(mydat, class = c("myclass", "dto", "list"))
```

Firstly, why give it 3 classes? If I'm bothering to do do this at all, I'll call the first class the most specific name, the second will always be "dto" (we're talking about DTOs here), and the third I'll keep it as a  list, so that all of the list-like methods are still implemented for this `obj`

Then, I could define my analysis methods:

```R
analyze <- function(x) UseMethod("analyze")

analyze.default <- function(x) stop("that DTO is not recognized")

analyze.myclass <- function(x) {
   # your actual analysis code goes here.
}
```

That *ensures* that you can only run your analysis on if `x` is a member of `myclass`. Now the problem with that is that the first block of code in this section is wrong, and bad. Don't actually do that. The issue is that you can make *anything* into `myclass`. Instead, it's much better to have some constructor function, and have it be called by your `get_my_data` function. Looking up to the top of this post, let's change the last line of the code:

```R
   ...
   one_row <- dplyr::filter(all_data, id == x)  # get just the row you want
   myclass(one_row)  # return a list instead of data.frame
}
```
then elsewhere, you have a constructor function for creating `myclass` objects from lists:

```R
myclass <- function(x) {
   X <- list(
       id = unname(unlist(x['id']))
       col1 = unname(unlist(x['col1']))
       col2 = unname(unlist(x['col2']))
   )
   # some validation operations go here.
   class(X) <- c("myclass", "dto", "list")
   X
}
```

Ah, that's better. Generally, I won't "reclass" objects except in constructor functions. That way, if I can easily identify constructor functions when I see them, and, as usual, helps keep code better organized. Suddenly you're not coding with just functions, you're coding with different *types* of functions, that have different functionality in your code. You have now your analysis functions, or what it is you're actually trying to accomplish, your DAL types of functions for reading and writing data, and your class constructor functions for creating classes and validating<sup>[4](#fn4)</sup> your data.

You might be thinking, "That sounds great and all, but why can't this just be more explicit?" Well, it can be, if you so choose.

### S4 Classes
S4 is a system that's a lot like S3, but where everything is more explicit. Your constructor function will look more like this:

```R
setClass("myclass",
   representation(id = "numeric", col1 = "numeric", col2 = "character"))
```
You can set default values using `prototype` and have a validation function as well. I generally always have some validation, and will always use a `prototype` The default values are things like `numeric(0)`, `character(0)`, and other values that are hard to code around. Usually I'll replace all of these with `NA`; realizing that `NA` itself is a `logical`, so they might actually be `NA_real_`, `NA_integer`, or `NA_character_`. Yes, R will complain if you don't do this.

Going backwards, your constructor function will look something more like this:

```R
myclass <- function(x) {
   new("myclass",
   id = unname(unlist(x['id'])),
   col1 = unname(unlist(x['col1'])),
   col2 = unname(unlist(x['col2'])))
}
```

Very similar to before. If your class validation is good and doesn't allow null values, you can use `$` instead of `[` and save yourself the headache of `unname(unlist(...))`. Just like how I won't put `class()<-` in my code other than in constructor functions, I also won't use `new()` except here.

The one other change is in using generic functions and methods. You'll need a generic and a method:

```R
setGeneric("analyze", valueClass = "numeric", function(object) {
   standardGeneric("analyze")
})

setMethod("analyze", signature("myclass"), function(object) {
   # your analysis
})
```

If you've never used the S4 system before, don't worry, you'll end up with an `analyze()` function.

### Pros and Cons of each system

[Hadley](http://adv-r.had.co.nz/OO-essentials.html) has a pretty good comparison<sup>[5](#fn5)</sup>, his general takeaway is that S4 is more rigid and more verbose than S3. I've found that to be true. In the context of assigning DTOs to classes, verbosity isn't really a bad thing. Personally, I've used both for DTOs before (though not usually in the same project), but my code, because of the above issues, comes out looking pretty much the same.

The real differences are that:

* S4 classes have inheritence whereas S3 objects have an order to their classes
* S4 methods can have signatures for multiple methods.

That first bullet point means that depending on how your methods are written, and what classes you have assigned, you can achieve pretty equivalent functionality; however, I do find that the S4 system has inheritence, makes a bit more sense in this context. A lot of this is going to depend on what your analysis functions look like.

The second bullet point, what I mean is that S3 methods usually look at just the first input to determine what method to dispatch (doing otherwise is a bit more complicated). S4 methods can have multiple signatures, which means that it's a lot easier to specify the classes of the inputs. In keeping with DAL functions and DTOs, you can end up having functions that look like this:

```R
write_object(object = my_object, to = my_data_store) {...}
```

where `my_object` may be some instance of some class, the object you're trying to write, and `to = my_data_store` is where you're trying to write that object, which could be a database or a file somewhere. Of course, you just need to make sure that method you're calling is implemented for that function `write_object` -  R is great, but it's not magic.

Taking this to the extreme, you could write a single function that does everything in your code, it just picks up what it's supposed to do based on the inputs you give it. Obviously, that's not a good idea. Your function name should give you a good idea of what the function is doing, after all.

### Concluding thoughts

I strongly recommend thinking through the structure of your DTOs, if you're going to have more than, say, 3 or 4 different kinds. S3 or S4 both work great in these situations. As far as DTOs go though, I don't recommend mixing these object types in the same project. Think about your data ahead of time, and choose a system that will work for you.


<a name="fn1">1</a>: Typically, so long as my query doesn't yield too terribly much data, I generally won't filter it in the query; rather, I'll filter it in code. Usually easier to grab more than you need. This can take a hit on performance of your code. It's up to the reader to determine what works best for each individual situation.

<a name="fn2">2</a>: R has really one and a half good options for database connections. The `DBI` interface is the good one, the half is split using ODBC or JDBC using the `RODBC` or `RJDBC` packages. ODBC is a pain to set up, especially on macOS or linux (usually the other way around, isn't it?).

<a name="fn3">3</a>: This becomes a bigger problem as your code base grows. If you only have a a single DTO in your code, feel free to ignore all of this advice. Use `data.frame`s. Whatever.

<a name="fn4">4</a>: Look again at this `myclass` constructor function. One thing to note here was the use of the `[` extract function. Usually, I don't extract columns or list elements this way, I find that using `$` is a bit more expressive, and your coding tools like RStudio will actually pick up on them. The `$` (or `[[`) will return `NULL` if that column isn't found, whereas `[` yields an error. In constructor functions, errors are better. The downside to this, of course, is the fact that you have to `unlist` and `unname` your columns. You can write a quick wrapper for these two functions if you're sick of seeing this all over your code.

<a name="fn5">5</a>: He also talks about Reference Classes (RC). I'm generally not a big fan since RC objects work pretty differently than S3 or S4 classes, plus, are mutable, which kind of ruins the beauty of R. In my opinion.
