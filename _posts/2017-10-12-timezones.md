---
title: Time Zones
date: 2017-10-12T22:00:00Z
layout: post
path: "/timezones/"
tags:
  - R
  - time
---

## The Problems
Time series data has a number of huge, related problems. Time zones and daylight savings time are the biggest, but there are so many other specific problems with this class of data. I simply can't enumerate the number of times these issues have caused problems for me. For example, say you encounter a years' worth of hourly data. The first things you should ask are:

5. What's the format of the time stamps?
1. What time zone are the time stamps in?
2. Do the data include daylight savings time?
3. If the data include daylight savings time, how are they accounting for it?
4. Are the data period-beginning or period-ending? Or something else?

And of course, a slew of other issues, including accounting for missing data, stuck data, etc.

So how do you deal with these issues? Always the first step is asking these questions. After that, it depends on who is originating those data; are they your data, are they from elsewhere. But here are some things that you can look out for and some strategies for dealing with the data that you have.

## 1. Dealing with Time Formatting
Far more often than not, time series data are going to be tabular. As in, you'll be interested in tracking the same variables over time<sup>[1](#fn1)</sup>. Starting from that place is actually quite helpful when you consider the mess it could otherwise be.

You're probably looking at somewhere between 1 and 6 (or possibly more) columns denoting what time each record is. This is going to be because different systems are going to format time differently, and they'll try to make their timestamps unambiguous<sup>[2](#fn2)[3](#fn3)</sup>. Things I have seen:

* Any combination of date/time
* Separated dates and times
* A column for each of year, month, day, minute, second, maybe fractions of seconds
* A possible extra column to denote a repeated hour<sup>[4](#fn4)</sup>
* Possibly one column for a timestamp given in a local timezone, and a second column in a second time zone (usually UTC)
* Maybe a year, and an hour number between 1 and 8760 (or 8784 for leap years), and maybe a minute and second column, but maybe minutes and seconds are just as a fraction of the hour

And so many other things. When all is said and done though, almost completely without exception, you'll want to have your timestamps saved as a single variable. This is just good practice, even if you could cut out a step or two by not doing it. Trust me, thinking this way is just going to get you into trouble.

If you're using R, the built-in `as.POSIXct()` or `as.POSIXlt()` could be a bit trying, even with a nicely formatted date:

```R
> X <- "2017-10-29 18:30"
> as.POSIXct(X, tz = "UTC")
[1] "2017-10-27 18:30:00 UTC"
> X <- "10/27/17 18:30"
> as.POSIXct(X, tz = "UTC")
Error in as.POSIXlt.character(x, tz, ...) :
  character string is not in a standard unambiguous format
```

Oh before I go any further, just specify what time zone it is you're talking about when creating your variables. Just do it. Otherwise, you have no idea what's going to happen - as it goes by what time zone your computer is in (usually). If you're at all like me, I mock things out on my laptop, then later on, I might move it all to the cloud if I need the extra compute power. Suddenly, when you move your code to a different computer, you don't know what it will give as its default time zone. So just be explicit. If you're certain you want to do things this way (don't!), you can always check what time zone you're in with `Sys.timezone()`

If you're a python user, the built-in `datetime` class will probably do you just fine. If you're like me and do most of your analytics in R, I highly recommend looking into the `lubridate` package. It doesn't add much functionality not there in base R, but it can help you cut a few steps.

If you are the source of these data you're analyzing, make things easy for yourself, and everybody else (including me!). Store everything in one column, and adhere to [ISO-8601](https://en.wikipedia.org/wiki/ISO_8601).

## 2 & 3. Dealing with Time Zones and Daylight Savings Time

The above example I gave, I knew I wanted those times to be in UTC. Actually more often than not, I want times to be in UTC. Now that you mention it, I really wish we could just abolish time zones altogether. I mean, what are they good for anyway? Most people generally wake up when the sun comes up, or a bit before or a bit after, go to school or work throughout most of the daylight hours, come home after that, and sleep when it's dark out. Who cares what the clock says? It's only a number. Let's make all of those numbers everywhere in the world, why don't we? to be coordinated? It would be so much easier!

But I digress.

The world is a messed-up place, and you, as the data analyst or scientist, have to deal with it. Truly sorry.

So how to deal with time zones? Good data ought to be marked. Somewhere, somehow. It they're publicly available data, it should be documented somewhere with your data source. I've been known to pick up the phone and call when not immediately obvious.

Things get messy though when the answer is something like "US Eastern Time Zone". That means one of three different things:

1. They mean Eastern Prevailing Time which has a repeated hour in the fall and a missing hour in the spring
2. They mean Eastern Standard Time, which does not exhibit those characteristics, but will disagree with people's clocks for two-thirds of the year.
3. They mean Eastern Daylight Time, which also does not exhibit those characteristics, but will disagree with *everybody* and *everything*.

Hopefully they don't mean anything other than these. Your data may not say. The trick is, look for the repeated hour and the missing hour (which in 2017 was Nov 5, at 2 o'clock AM). If both are there (that is, a repeated hour, and the missing hour is *not* there), it means your data are in prevailing time. If one is there, probably the missing hour, it means your data source accidentally deleted an hour, but it's still prevailing time. Otherwise, it's probably standard time.

Dealing with these particularities in R, when setting your time zone using `lubridate` or `as.POSIXct()` or whatever, this is going to depend on what system you're using. Do a call for your `OlsenNames()` to see what time zones you have available on your computer. The ones that are actually real time zones, like "America/New York" should be available on any modern computer. Ones like "Eastern Standard Time" (year round) may or may not be. But in those situations, applying the proper offset and just using "UTC" is usually a good way to go.

If these data are your data, your best option is to set all your time stamps to UTC (it's on every computer), and while you're at it, title the column "utc". Just to avoid ambiguity!

## 4. Data accounting for Daylight Savings Time

As above, if not explicitly stated, you can generally figure out if your data accounts for daylight savings time with a bit of digging. Some of the weirder ways of calling this out I've run into included having a separate column for "repeated hour", which is "no" for 8759 hours out of the year. It works, but it's also a bit of a waste of data storage space.

Once I came across a time series that just repeated the hour without saying which one came first and which came second. If you ask me, there isn't even really a point in having time stamps if you're going to do that. I mean, the point of timestamping data is so you know what time it is. If you're not saying exactly what time it is why even bother?

But essentially, this is just a factor when encountering a new data set you'll just have to figure out. And as usual, if you are the originator of these data, just do everybody a favor, call your timestamp column "utc", and keep all your data in UTC. You completely avoid needing to deal with daylight savings time, and you and anybody else will have no problem converting it into any kind of local time, using software built into any programming language.

## 5. Period-ending or Period-beginning? Instantaneous or Time-Averaged?

The essential problem is that, given a time series, generally of equally-spaced time intervals, does your data point represent the value at the beginning of the period or the end of the period? Or is it an instantaneous read?

Instantaneous reads are easier to deal with, to be sure. That could be something like a temperature reading. But a lot of data is, in fact, time averaged over some period. When associated with a time stamp, you always should be wary of what the time stamp represents - the period the data start, the period the data end, or something else?

This is an issue that really and truly, there aren't very good ways around. Data ought to be documented, and if you're the originator of the data, you should say this somewhere in your documentation. Otherwise it's like providing GIS data and not documenting what spatial datum it's in. You just wouldn't. So why not document your time series data with the same rigor?

If you're not the origin of the data, there are sometimes some clues to determine if the data are period beginning or ending. The first clue is if your data is period ending, you're likely to have time stamps for times that don't actually exist<sup>[5](#fn5)</sup>. You can also sometimes use the first and last data points as reference.

One creative thing I've seen a few times, is to pick the midpoint in time-averaged data. For hourly data, this would look like all the data points occur on the half-hour. It's nice since it removes any ambiguity, and if you want to look at the data as period-ending or period-beginning, it's as simple as adding or subtracting a bit of time.

## Conclusion

In essence, timestamps are a real pain. Daylight Savings Time is a pain. Documentation is too frequently poor. Sometimes, you just have to live with it. If you're the originator of the data you can:

* Have good documentation available
* When possible, just use UTC to avoid time zones and daylight savings time
* If doing that, call your column something like "utc".
* Be clear about period-ending vs period-beginning. You might even call your column "utc.pe", "utc.pb", or something like that.
* Write your state representatives expressing your frustration.


<a name="fn1">1:</a> Only very rarely have I seen non-tabular time series data. If that confuses you, yes, well, it confused me too. The thing is that if you're dealing with key-value pairs where the keys aren't consistent, well, you can convert that into a table with a lot of blank values. If they're hierarchical data, well, you'll need to be kind of creative, and will depend on how you're analyzing them.
<a name="fn2">2:</a> Usually.
<a name="fn3">3:</a> Unambiguous perhaps, but not necessarily straightforward.
<a name="fn4">4:</a> If you're not familiar with this, repeated hours happen in the Fall, early November, when we all change our clocks back an hour. It is by far, the biggest most annoying thing you will have to do in analyzing time series data. If you were previously unaware of this problem, I'm really sorry to be the one to have to tell you about it.
<a name="fn5">5:</a> An aside: I generally prefer period-beginning time stamps. Yes, that puts me in a minority category. But hear me out. The biggest reason to use period-ending time stamps is because you don't know what the time-averaged data actually say until the period is over. Okay, that's fair. But when analyzing, you run into the opposite problem, in a big way. If your data are hourly, you'll have time stamps that say things like "2017-10-05 24:00", which is of course, not a real time. Date/Time parsers will have a problem with it. Okay, so do you reformat that timestamp to say "2017-10-06 00:00"? Well, that's misleading. If you go to aggregate your data by day, month, or year, it will be wrong. No point of that data actually occurred on October 6th, it was all on the 5th. So why write it on the 6th? It makes no sense. So either you'll end up with times that aren't times, or something that's wrong. And to argue with anybody who thinks hour-ending is a good idea because it's more convenient based on how the data are generated, well, that's just lazy. It's not hard to subtract one period's worth of time from each time stamp when you're done if you need to.
