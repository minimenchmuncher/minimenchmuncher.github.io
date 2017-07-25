---
title: A logical Extension to Using Classes in Your Analysis
date: 2017-07-25T22:00:00Z
layout: post
path: "/class-configs/"
tags:
  - R
---

### Introduction
If you've read a few posts on this blog before, you'll probably note the fact that I really like to configure things. And organize things. And keep my configurations out of my actual code. If the project is medium-sized, I might put configuration files in the `inst/` directory to package them in (and when they get big, I'll have a standalone database).

Before, I've talked about runtime settings, things like where to store paths to files, or urls for downloading data from. I've also talked about creating and using classes before. What happens when you combine the two? It's something magical, I promise! By that, I mean storing your class definitions in an outside settings file.

### Why would I ever want to do that?
Putting settings outside of your code helps organize your settings. It makes it so that your code is more about what you're trying to do than how you're trying to do it. It means that if you go refactor all of your code, you don't have to worry about these kinds of definitions. I'll illustrate with an example.

Say I want to create a class for a DTO called `foo`, and I'll keep it as an S3 class for now. Since S3 classes only have attributes<sup>[1](#fn1)</sup>, we'll say that this `foo` class has 2 attributes `x` and `z` that would let you call `foo$x` and `foo$z`, we'll say `x` should be a character, and `z` should be a numeric. And while R isn't strictly typed, I like to type my classes because it means that you can raise errors in your functions much earlier than you would be able to otherwise<sup>[2](#fn2)</sup>. I'll make a constructor function:

```R
new_foo <- function(..., .dots = list()) {
    X <- c(list(...), .dots)
    types <- list(x = "character", y = "numeric")
    ptype <- list(x = NA_character_, y = NA_real_)
    # maybe some prechecking the inputs to new_foo() here
    Xc <- modifyList(ptype, X)  # uses ptype as a template
    # post-checking
    checkmate::assert_class(Xc[[1]], types[[1]])
    checkmate::assert_class(Xc[[2]], types[[2]])
    class(Xc) <- c("foo", "dto", "list")
    return(Xc)
}
```

There's a lot going on in there. Firstly, I am defining the types I'm expecting for `x` and `z`. Then, I create a new list that contains *all* of the list elements that I was expecting (`Xc`). Then I make sure that those inputs are the right class, then set the class once I know everything looks right, and return the object. Okay maybe that's not actually a lot.

### A General Instantiation Function
Let's talk about generalizing those steps. Chances are, if you're bothering to do this with one class, you'll bother with several, and if you're doing that, you shouldn't be repeating the same functions in your code - you should generalize!

We'll go from the back to the front. Well, the setting of that class at the end, that's easy--- that's just knowing that `new_foo()` should yield a class of `foo`.

Checking the classes of the list elements-- you just need an `lapply`:

```R
lapply(1:length(Xc), function(i) {
    checkmate::assert_class(Xc[[i]], types[[i]])
    })
```

Then you don't really care how many elements are in your list.

The `modifyList()` call - that's always going to be the same. Really the only things that are different for different class instantiation functions are going to be the `types` and `ptype` lists. Well that's fine - we'll just store them elsewhere. Your generalized function may look like this:

```R
new_general <- function(..., .dots = list(), .cls = "") {
    X <- c(list(...), .dots)
    cls <- get_cls(.cls)
    Xc <- modifyList(lapply(cls$attributes, `[[`, 'default'), X)
    lapply(1:length(Xc), function(i) {
        checkmate::assert_class(Xc[[i]], lapply(cls$attributes, `[[`, 'class')[[i]])
    })
    checkmate::assert_names(names(Xc), permutation.of = names(cls$attributes))
    class(Xc) <- c(.cls, class(Xc))
    return(Xc)
}
```

Same things going on here. Only difference is that I have a new string input `.cls`. Same checks, same assignment. And a new function, `get_cls()`

### Where's my Class Definition?
You'll see above this function I threw in there, `get_cls()`. That function gets a class definition by name. The way it looks - that sort of depends on how you actually want to store your class information. Things get tricky here, but if you've used class factories, service locators, or dependency injection before in some other programming language (java, C#, python, javascript, etc.), this might look familiar to you.

At this point, unless I'm going way overboard, I'll put my class definitions in a yaml file called `inst/types.yml`<sup>[3](#fn3) [4](#fn4)</sup>. That might look, in this case, like this:

```yaml
foo:
    attributes:
        x:
            class: "character"
            default: NA_character_
        z:
            class: "numeric"
            default: NA_real_ 
```

That looks straightforward.

But now we need a function to load these and put it somewhere in the working environment. The most convenient place is to do this on package load, in the `.onLoad()` function, and the easiest is keep a `types` or `classes` or some other similarly-named list in the Global Environment.

```R
get_types <- function(types_file = system.file("types.yml", package = "yourpackage")) {
    yaml::yaml.load_file(types_file)
}

.onLoad <- function(libname, pkgname) {
    types <- get_types()
    for (i in 1:length(types)) {
        for (j in 1:length(types[[i]]$attributes)) {
            types[[i]]$attributes[[j]]$default <- eval(parse(text = types[[i]]$attributes[[j]]$default))
        }
    }
    rm(i, j)
}
```

That first function there, `get_types()` is just to load up your `yaml` file. I like splitting that out because it makes the `.onLoad()` independent of how you decide to organize your `types` file.

The second function loads that list. The double-nested `for` loop - all that's doing is evaluating your default values. The ones I like using are usually some kind of `NA` value, but sometimes also a number, or empty `list()` or possibly a `Date`. Chances are, if you're storing these types outside of an R file (recommended), R won't recognize these kinds of values as R values automatically and will load them as strings. The `eval(parse(text = "..."))` just evaluates those strings.

And don't forget to remove the iterator variables, as they'll get added to your Global Environment too if you don't.

You may choose to add the line at the bottom

```R
types <- list2env(types)
```

if you'd rather `types` were an environment instead of a list<sup>[5](#fn5)</sup>.

Now that we set up all of the infrastructure, we can finally return to the `get_cls()` funcition alluded to at the beginning of the section:

```R
get_cls <- function(cls) {
    types[['cls']]
}
```

### What about other kinds of R classes?

All the code given above assumes your classes are going to be S3 classes. What if you want to use a different system? I'll leave that up to the reader for now, but you can rest assured that most everything shouldn't change. For S4 for example, all of the logic that I gave in the instantiation function moves into a `setClass()` call, and the instantiation just becomes `new("foo", ...)`. Which, by the way, is why I presented all of this in the case of using S3 classes - it's more complicated and less precriptive, and you don't have to use `checkmate` so much to ensure your inputs are the correct types

R6 classes (and R5/RC classes, but I like R6 better) can be defined this way as well using their own class definition semantics, and I have been sucessful at doing so once or twice. However, the whole point of those kinds of classes is that they have mutable state and they have methods that belong to them.

I have dealt with this a couple of ways. One is establishing only from the beginning the attributes each class should have, but not the methods. I have also added names for methods in my `types` definition, setting the default for them as `function()`. In one case, I organized `types`, splitting `attributes` and `methods` as the top level items; in another case, I didn't bother. Both seemed to work for me.

### Concluding thoughts

You might be asking "Is this overkill?" The answer is, yes, probably. But I have my reasons.

The whole reason I started doing this sort of thing is that the semantics for defining S4 classes is terrible, especially if they have a large-ish number of slots (10 was my annoyance-inflection-point). I would always want to define my slots and prototypes together, because if I wasn't using named lists, I'd miscount how many slots I had, and if I *was* using named lists, I'd mistype something, and the error messages were usually no help. So lots of copying and pasting.

I moved from that to wanting a representation that was external to `setClass()` or to my S3 instantiation functions that was just organized better. I moved those kinds of definitions into lists and from there, moved those lists into files. Why the move from lists to files? Changing a file, where all of your definitions are stored, is easier.

And do I do this with all of my classes? It really really depends. If my project uses DTOs, and has more than 1 or 2 DTO classes, I'll use it at least for the DTOs.



<a name="fn1">1:</a> Okay - fine! I know you can put functions as elements in lists. Stop yelling at me! But seriously, why would you want to do that? Think long and hard about why you would want to organize your beautiful, functional code this way. If you actually have a good reason, you should be using an RC or R6 class, *not* S3. That's not what they're for.

<a name="fn2">2:</a> In case you were wondering, yes, pythonistas hate me.

<a name="fn3">3:</a> I like yaml because it's clean and easy - easier than json. If I go with an xml file, that's pretty overboard, but sometimes really useful - especially if you want to version your classes, tag them with guids, or anything like that. If this is blowing your mind right now, just know I didn't invent this way of thinking, it's common in production-level code; I'm just bringing it to R.

<a name="fn4">4:</a> Put it in `inst` so it'll get packaged in with your code, and accessible later on using `system.file()`.

<a name="fn5">5:</a> That shouldn't change the code chunks below. If you want to `attach` `types`, keep an eye out because that will mess around with how you access the class definitions, since you can't do `types$foo`, instead, you'd do `get("foo", "types")`.
