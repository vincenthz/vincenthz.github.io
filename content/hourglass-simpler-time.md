+++
title = "Simple time with Hourglass"
date = 2014-05-05
draft = false
[taxonomies]
tags = ["hourglass","time","simple"]
categories = ["programming"]
+++

Each time, I've used the [time](http://hackage.haskell.org/package/time) API in
Haskell, I'm left with the distinct feeling that the API is not what I want it
to be. After one time too many searching the API to do some basic thing, I've
decided to look at the design space and just try implementing what I want to
use.

<!-- more -->

Before going into this re-design, this is my list of issues with the current API:

* UTCTime is represented as number of day since a date (sometimes in 19th
  century), plus a time difference in seconds from the beginning of the day.
  This is probably the worst representation to settle to as main type as it
  neither a good computer representation nor a good human representation.
* Every time I need to use the time API, i need to look at the documentation.
  With the number of time I used the time API, I feel like I shouldn't need to
  anymore. Sure it got easier, but it's not as trivial at I want it to be.
  The number of functions, and the number of types make it difficult. YMMV.
* Too many calendar modules. I just want the standard western calendar module.
  It's called the gregorian calendar and time make sure you need to remember
  that, as it's part of many function names useful to do things.
* C time format string for parsing and printing. Each time I need to format time,
  does pretty much mean I need to consult the documentation (again), as there's almost
  50 different formatters, that are represented with single letter (that for some of them doesn't have any link to what they represent).
* Need to add the [old-locale](http://hackage.haskell.org/package/old-locale)
  package when doing formatting. Why is this old, if it's still in use and
  doesn't have a replacement ?
* A local time API that get on the way, different types than global time.
  TimeOfDay, ZonedTime, LocalTime. YMMV.

Ironically, old-time seems much closer to what I have in mind with some part of
the time API.  The name seems to imply that this was the time api before it got
changed to what is currently available.

Re-design
---------

So I've got 4 items on this design list:

1) Some better types
2) Use the system API to go faster
3) Unified and open system
4) Better capability for printing and parsing

Better types
------------

I wanted the main time type to be computer friendly, and linked to how existing API return the time:

* On Windows system, it's the number of 100 nanoseconds (1 tick) since 1 January 1601.
* On Unix system, it's simply the number of seconds since 1st January 1970.

It's probably fair to expect other systems to have similar accounting method,
and anyway just those two flavors covers probably 99% of usage. I originally
planned to keep the system referential in the type, but instead it's simpler to
choose one.

Inventing a new one would be fairly pointless, as it would force both system to
do operations. Converting between windows and unix epoch, is really simple and
very cheap (one int64 addition, one int64 multiplication), so Unix has been chosen.

Along with the computer types, proper human types are useful for interacting
with the users. This mean a Date type, a TimeOfDay, and a combined DateTime
describe in pseudo haskell as:

~~~~ {.haskell}
    data Date = Date Year Month Day
    data TimeOfDay = TimeOfDay Hour Minute Seconds
    data DateTime = DateTime Date TimeOfDay
~~~~

Use the System, Luke !
----------------------

Heavy conversion between seconds and date is done by the system. Most
systems got a very efficient way to do that:

* In Unix that means [gmtime](http://pubs.opengroup.org/onlinepubs/009695399/functions/gmtime.html)
* in Windows [FileTimeToSystemTime](http://www.cs.rpi.edu/courses/fall01/os/FileTimeToSystemTime.html)

One side effect is that we have the same working code as the system.  There's
much less need to worry about exactness or bugs in this critical piece.

For futureproofing, a haskell implementation could be used as fall back for
other systems or different compiler target (e.g. haste), if anyone is
interested.

Unified API
-----------

I don't want to have to remember many different functions to interact with many types.
Also time representation should be all equivalent as to which time value they represent.
So that mean it's easy to convert between them with a unified system.

So 2 type classes have been devised:

* one Timeable typeclass to represent type that can be converted to a time
  value.

* one Time typeclass to represent time type that can be created from a time
  value.

With this, hourglass support conversion between time types:

~~~~ {.haskell}
> timeConvert (Elasped 0) :: Date
Date { dateYear = 1970, dateMonth = January, dateDay = 1 }
> timeConvert (Date 1970 January 1) :: Elapsed
Elapsed 0
> timeConvert (DateTime (Date 1970 January 1) (TimeOfDay 0 0 0 0)) :: Date
Date { dateYear = 1970, dateMonth = January, dateDay = 1 }
~~~~

Anyone can add new calendar types or other low level types, and still interact
with them with the built-in functions, provided it implement conversion with
the Elapsed. It allow anyone to define new calendar for example, without
complicating anything.

Better formatting API
---------------------

Formatter have a known enumeration types:

~~~~ {.haskell}
> timePrint [Format_Day,Format_Text '-',Format_Month2] (Date 2011 January 12)
"12-01"
~~~~

But can be overloaded either by string, or some known formats:

~~~~ {.haskell}
> timePrint "DD-MM-YYYY" (Date 2011 January 12)
"12-01-2011"
> timePrint ISO8601_Date (Date 2011 January 12)
"2011-01-12"
~~~~

Someone could also re-add C time format string too with this design,
without changing the API.

Implementation
--------------

The API and values returned has been tested under 32 and 64 bits linux,
freeBSD, and Windows 7.  It's got the same limitations that the system has:

* 32 bit linux or BSD: between year 1902 and 2038. this doesn't apply to the x32 flavor of linux, and the latest openbsd 5.5.
* 64 bit linux or BSD: between year 1 (as BC date before bring all sort of random problems) and few billions of years. this ought to be enough for everyone :-)
* windows is limited to date between 1601 and 9999.

I find the tradeoff acceptable considering that in counterpart we have descent
performance, and all-in-all a working range that is enough.

For a look on performance, as measured by criterion:

* [A quick report, showing trends quite well](http://tab.snarc.org/misc/hourglass-small-criterion.html)
* [the long and heavy to load report](http://tab.snarc.org/misc/hourglass-criterion.html)

The library is small too:

* time      (haskell=1434 (94.5%), C=84 (5.5%)
* hourglass (haskell=884 (98%), C=19 (2%)

And its documentation is available on [hackage](http://hackage.haskell.org/package/hourglass), and the code on [github](https://github.com/vincenthz/hs-hourglass).

Example of use
--------------

~~~ {.haskell}
> t <- timeCurrent
> timeGetDate t
Date {dateYear = 2014, dateMonth = May, dateDay = 4}
> t
1399183466s
> timeGetElapsed t
1399183466s
> timeGetDateTimeOfDay t
DateTime { dtDate = Date {dateYear = 2014, dateMonth = May, dateDay = 4}
         , dtTime = TimeOfDay {todHour = 6, todMin = 4, todSec = 26, todNSec = 0ns}}
> timePrint "YYYY-MM-DD" t
"2014-05-04"
> timePrint "DD Mon YYYY EPOCH TZHM" t
"04 May 2014 1399183466 +0000"
~~~~~

Q&A
---

* Q: Report issue, wishlist ..
* A: [issue-tracker](https://github.com/vincenthz/hs-hourglass/issues)

* Q: Do I have to use this ?
* A: No, you can still use [time](http://hackage.haskell.org/package/time) if you prefer.
