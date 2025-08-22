---
layout: page
date: "2016-11-13T15:30:00Z"
layout: post
path: "/using-dals/"
tags:
  - R
  - Data Science
---

One of the first things you'll need to do while doing an analysis is to load up
your data into memory. Sometimes this is as easy as reading a file. Sometimes
more complicated. It's a good idea to organize all of your data munging functions
into a data abstraction layer or DAL. The point of doing this is so that you
pass nicely organized data into your analysis functions.

Oftentimes, you'll start with a data loader function that looks like this:

```r
mydata <- read.csv(file_path)
```

If that works for you, that's great. But let me tell you something, it really
won't.

### 1. Use the data as it (they) comes (come)

One of the most important things that you can do for yourself is start with any
data as they come.

* If your data are coming from a file, the first step of your data loading needs
  to be loading that file. You should be *extretemly* careful when you
  preprocess your data. If for some reason you must, You should do so
  programmatically if at all possible, and package in your data preprocessing
  programs in your package's `inst` directory. The kicker here is that if you
  can simply do this, it's not a stretch to execute those procedures in your
  DAL either.
* If your data are coming from a database, I highly recommend against really
  fancy SQL queries in favor of simple ones. If your data are tabular,
  `dplyr` is a useful tool for dealing with them.
* In any case, it's usually a good idea to set column names in code instead of
  picking them up from a database or a file. This way, you know exactly what
  the column names are supposed to be when you pass your data into your
  analysis.

#### Example
The thing is, you don't want your analysis to stop working if your data source
changes formats. I'll give you an example pulling from my own experience.

[The Electric Reliability Council of Texas (ERCOT)](ercot.com) publishes, among
other things, their wholesale electricity prices on 5- and 15-minute timescales<sup>[1](#fn1)</sup>.
They offer these data at these regularly-spaced increments (depending on the
type of pricing), one file per time period, for each of the points on their
system (for example files, [go here](mis.ercot.com/misapp/GetReports.do?reportTypeId=12301)),
each of these (`.csv`) files have the following columns:

1. `DeliveryDate`
2. `DeliveryHour`
3. `DeliveryInterval`
4. `SettlementPointName`
5. `SettlementPointType`
6. `SettlementPointPrice`
7. `DSTFlag`

Some historical context though. ERCOT, in its current form, has only existed
since December 2010. Data from that date through March 13, 2011 was all in CST<sup>[2](#fn2)</sup>.
On March 13th though, one of their analysts must have noticed that on that day,
when the clocks get turned back, are going to have a repeated hour's worth of
data, and that it's terribly bad practice to not be able to uniquely identify
each row of a data table.

They came up with a solution of adding a `DSTFlag` column in their data, which
is `Y` for that repeated hour only, and `N` the other 8759 hours of the year.

The point is of this is that they added a column where there wasn't one before.
That means that if you designed an analysis around only the first six columns,
and did not use a robust DAL, your whole analysis will fail when this change was
made. Or worse, it will *NOT* fail, but you won't be able to pick up the time
change, and your analysis will be wrong half the time<sup>[3](#fn3)</sup>.

But, if you have a good robust DAL, when the data format changes, yes your
analysis may fail, but you'll know it happened in the DAL, which is a signal
to you that the data format changed, and allow you to comfortably make changes
to your DAL code to allow it to recognize and process the new format without
changing a single line of code in your actual analysis.

### 2. Multiple 'Get' or 'Push' Options
One of the big reasons why I like this coding structure is that it allows very
nicely for multiple functions for fetching and sending data. I'll tell you why
that might be a good idea and how to go about doing it.

[The US Energy Information Agency (EIA)](eia.gov) publishes a monthly history [(form 923)](https://www.eia.gov/electricity/data/eia923/)
of power plants in the US, their fuel consumptions, and power outputs. Say you
wanted to include the monthly power output data in your analysis in order to
compare states to each other. We already know we're going to want something that
looks like `getStateMonthlyTotals()` as a function. But how do we decide we'll
want to query these data? By state abbreviation? Full name? FIPS code?<sup>[4](#fn4)</sup>
You'll note these data include a `Plant State` column, which is a two-letter
upper-case abbreviation, so let's start there:

```r
library(dplyr)
library(readr)
getStateMonthlyTotals <- function(x, file.path) {
    read.csv(file.path, skip = 6) %>%
        filter(`Plant State` == x) %>%
        select(<the right columns>) %>%  # you can figure this part out
        summarise(jan = sum(jan), feb = sum(feb))  #etc.
}

```
All right, but it should probably stop if you use a bad state
abbreviation. And adding in state names isn't terribly difficult either:

```r
getStateMonthlyTotals <- function(x, file.path) {
    if (!(x %in% state.abb | x %in% state.name) {
        stop(sprintf("%s isn't a state!", x))
    }
    if (nchar(x) > 2) {
        state_abb <- state.abb[which(state.name == x)]
    } else {
        state_abb <- x
    }
    read_csv(file.path, skip = 6) %>%
        filter(`Plant State` == state_abb) %>%
        select(<the right columns>)
        summarise(jan = sum(jan), feb = sum(feb))  #etc.
}
```

Still something missing - what about using a FIPS code? That'll take a bit more
work still since you need a list of those from somewhere. But you don't *always*
need it- it just depends on how you're trying to query the data. Since those are
numeric, whereas names are character, we can use R's method dispatch easily here.

```r
getStateMonthlyTotals <- function(x, file.path) UseMethod("getStateMonthlyTotals", x)

getStateMonthlyTotals.default <- function(x, file.path) stop("This method isn't yet implemented")

getStateMonthlyTotals.numeric <- function(x, file.path) {
    # write code for converting FIPS to state abbreviations, and then:
    getStateMonthlyTotals.character(x, file.path)
}

getStateMonthlyTotals.character <- function(x, file.path) {
    # exactly what you had before
}
```

Two things:

1. You'll note that the function for the numeric class calls the one for
   characters. I do this a lot for these types of problems. You could avoid the
   method dispatch paradigm, but imagine getting the FIPS codes took a while.
   You wouldn't want it called all the time.
2. Why not combine all these into one function? You could, and put the FIPS part
   into a conditional. But that gets messy. Plus, this way, there's better,
   easier ways of adding in your own classes. A UUID class, for example.
   Or a `SpatialPointsDataFrame`.

### 3. DAL tests
There's one last benefit, and I'll be brief here. I'm all about testing code,
writing it in chunks, and testing each chunk separately. You can use `testthat`
to test connections, connection strings, and mock out the data.

Sometimes downloading or uploading data takes forever, depending on where and
how it's accomplished. You want your tests to run quick. So you can test with a
small subset, or make it all up yourself.

If pulling from or pushing to a database, easy enough to have code to set up a local dummy
database in your tests.

Setting up a separate testing environment from your production/analysis
environment (and possibly also a separate development environment) is key, and
I'll go over that in a future post.

<a name="fn1">1</a>: *ERCOT's prices work a bit different from the other
wholesale electricity markets in the US. They have 5-minute LMPs, and 15-minute
SPPs or settlement point prices. These are the prices to which I'm referring.*

<a name="fn2">2</a>: *ERCOT only keeps the past 7 days' worth of data available
on their website. If you want to view these data, they are publicly available,
but you must contact them directly to get copies of them.*

<a name="fn3">3</a>: *This is why all of your data storage and analysis should
be in UTC. There's really no excuse for anything else. I'll rant about this in
a future post*

<a name="fn4">4</a>: *The answer is, of course, FIPS code.*
