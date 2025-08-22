---
title: Why I don't hate MS Access
date: 2017-07-15T15:00:00Z
layout: post
path: /r-and-access/"
tags:
  - R
  - databases
  - sql
---

### R and SQLite
There are situations when I want to keep data around in a persistent way across R sessions. If I'm doing this on my own, I might gravitate toward using a database of some kind. Sure, you can save your R objects, but I usually don't do that -- standard data formats make me feel just more comfortable in case I'm also running something in Python that'll use the same data, or in case I just wanted to look at the data in sql in the command line.

SQLite is a natural choice. It's free and open source, and requires absolutely no set up (unlike more robust databases). If I'm doing this, I'll put my database setup *in code*. R's `DBI` and `RSQLite` packages make this pretty easy.

```R
library(DBI); library(RSQLite)
mydb <- dbConnect(RSQLite::SQLite(), "/tmp/testdb.sqlite")
```

This is, by the way, a perfect example of where you might want to consider using some kind of `settings` object - where the database connection is, and any login credentials for it.<sup>[1](#fn1)</sup>

Easy enough to make new tables:

```R
q <- 'CREATE TABLE temporary (id numeric, col1 numeric, col2 char)'
dbGetQuery(mydb, q)
```

And you can send data using a SQL `UPDATE` query the same way.

### Who's the boss?

Okay so what am I getting at here? When considering a larger project, I oftentimes question "Who's the boss?". What I mean by that is that I tend to think of database administration as its own set of tasks.

Data scientists tend to think of a process that goes:

1. start with your data
2. import and transform your data
3. analyze your data
4. communicate your results.

So much so that you see diagrams of this linear process. The [RStudio Sparklyr Cheet Sheet](https://github.com/rstudio/cheatsheets/raw/master/source/pdfs/sparklyr.pdf) basically lists this process verbatim. Following this process, you might think that you should create your database, your tables, everything, and deploy your database before moving onto the analysis.

And you'd be wrong. Sorry. It's a much better strategy to put all of your database setup inside your R code. Here are a few reasons why:

* **Versioning** If you put all of your database setup in code, that means that all of your code tools can be used for your database as well; things like git, unit testing, etc.
* **Deployment** This is especially true if you're working with other people, it's much easier to deploy your database on multiple machines for development purposes

It might seem like the tail wagging the dog a bit at first, especially if you're used to the development cycle that data scientists might more typically use. And writing your code around your database this way makes your code "the boss".

### This post is about Access, isn't it?

There are many good applications for a dedicated RDBS. Especially if you are analyzing a large volume of data, you have a lot of tables, they are very wide and very long. Then you might consider using Postgres, MySQL, MS SQL Server, Oracle, some SAAS database system on Amazon or Azure, it really just depends on your needs. One of the big benefits is that these are all client-server based which means that you can have lots of different queries from different clients running simultaneously.

But what about smaller projects? SQLite is a great solution for the already stated reasons. Another one you might consider is MS Access. Here are some cons:

* You have to use ODBC with the `RODBC` package. It's not *nearly* as good as the `DBI` package. `DBI` has a lot of great functions, for example, protecting against SQLi. Usually, if using Access, I'll use both packages, `RODBC` for the actual connections, but `DBI` for its extra add-on functions. Don't worry though, if you have a good DAL, it shouldn't matter too much.
* You have to make sure that if you have 32-bit Office installed, you need to be using 32-bit ODBC and 32-bit R. Same goes for 64-bit. That can mean problems with memory allocation if you're using modeling tools that take up a lot of RAM, like machine learning.
* Your code becomes a bit less portable, and harder to use on Mac and Linux. Developing it shouldn't be a big problem, but running it could be. Setting up ODBC on mac and linux is a chore.

Sure there are some downsides, but there are upsides as well. Otherwise I wouldn't even bring it up as a possibility. For one thing, if you have Access, chances are your coworkers do too. And if that's the case, they can run their own queries or "reports" on the data contained therein and create forms for data entry. Sure, all of these things you can do with any GUI. Doing this well and doing it right, yes, your data should go through a DAL. But if you want a quick and dirty solution to share your data with your coworkers, and if you want some quick forms, this is a solution.

Big downside is that your database here is dependent on your R code. Your forms and any IO app really ought to be too.

### Other Options

Of course, if that's what you're after, Access is only one such tool. Long-term, you may be better off with something else. You can use something like Shiny for creating forms even for setting up queries and reports. That has the downside of needing a web server. 


<a name="fn1">1</a>: There is actually an even cleverer idea here. If you have a lot of runtime settings, it may even be a good idea to stash them *all* in a database. For smaller projects, I like using a yaml file usually because they're quick to write and get working. But then organization can get challenging. Where do you keep it? (I usually package it in, in `pkg/inst/settings.yml` so I can use `system.file()` to get it.) If you have a lot of settings, how should they be organized? How should they be nested? Then suddenly your function to read your settings gets super complicated. It's worth weighing the pros and cons.
