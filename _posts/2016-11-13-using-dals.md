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

### 1. Use the data as it comes

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

The thing is, you don't want your analysis to stop working if your data source
changes formats.

### 2. Multiple 'Get' or 'Push' Options



### 3. DAL tests
