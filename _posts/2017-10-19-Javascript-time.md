---
layout: post
title: Javascript date and the mythical year 0
---

Here is the story/rant of when I tried to use time in Javascript.

> There are two hard things in computer science: cache invalidation, naming things, and off-by-one errors.
-- Everyone

I would like to add time to that list!


# What I was trying to do #

I was working on making a minor contribution to the open source
repository [wp2ghost](https://github.com/jonhoo/wp2ghost) which
converts Wordpress XML exports to Ghost JSON import files. I had, as
you might expect, previously written a blog in wordpress and wanted to
export it to [ghost](https://ghost.org/). It so happened that there
was an issue with the date fields for posts in my Wordpress XML expport
so that wp2ghost gave all posts the *"date"* 0000-00-00 00:00:00
instead. Luckily for me there was still a field with data information,
it just was not the one wp2ghost was expecting so it was an issue I
could fix. Said and done, i patched the issue in the software and
posted a Pull Request and explained the issue to the author.

The author liked the changes overall but being a good developer he
wanted to make sure that everything fit into the wider codebase so he
suggested some sensible changes. One of those requrest conserned the
unit tests. Because me being a good developer I had added some to test
the new functionallity. However one of my tests had a magic number
checking for milliseconds to 0000-00-00 00:00:00 or some such and he
asked me to calucluate the value so that the value would not be a
magic number but something understandable.

After some chat about things I asked this:

>I can add testcases when I get a greenlight on the code. Hopefully my second commit is better.
>
>Should I assume that the time is year 0 or should I maybe go for 1970?
-me
>The old code did not produce a sensible outcome in that case, so I think either is fine. 0000 is probably semantically more correct (1970 is just a UNIX timestamp artifact). The current change looks fine :)
-he

So I was just gonna do a quick change to my unit test, this will only
take a few minutes... few minutes it took indeed...

# The Rant #

This lead to the creation of this post below: It can also be seen in
its original form and context
[here](https://github.com/jonhoo/wp2ghost/pull/8#discussion_r84797728). I
took the liberty to fix a few minor things here in the version posted
on the blog.

We have an issue here. First of I just realised that there is no such thing as a month or day 0. So we should use "0000-01-01 00:00:00 UTC" I guess.

As it so happens Date.UTC does [not support](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/UTC) years before 1900, well there is still just Date.  
To use the normal Date you need to first set a year:
`var date = new Date(0,1,1)`
and since years 00-99 are defaulted to 19xx (of course) we instead have to use .setFullYear(0); which seems to be implemented for this very purpose.
`date.setFullYear(0);`

Now typing date into a console directly in chrome gives us:
`Tue Feb 01    0 00:00:00 GMT+0100 (W. Europe Standard Time)`

So somehow its apparently February the first now... So we have: `0000-02-01`.

Though playing around with date to string functions gives me this:

```
date.toLocaleDateString()
>"1-02-01" 
date.toLocaleTimeString()
>"00:12:12"
date.toString()
>"Tue Feb 01    0 00:00:00 GMT+0100 (W. Europe Standard Time)"
```

So Im not sure if its year 0 or 1 right now and if the time is 12 minutes past midnight or not.

It is at this point I find out that there is no year 0 in [ISO 8601](https://en.wikipedia.org/wiki/0_%28year%29#ISO_8601) so I guess year 1 makes sense then and Jesus was not born on his birthday. 

You can however, as a sidenote, write 0000-00-00 and get a date, as I did try a few times. A day or month 0 simply is simply the previous day/month in the last month/year. So by that logic I think year 0000 should default to 1 BC which is also supported by [wikipedia](https://en.wikipedia.org/wiki/0_%28year%29#ISO_8601)

> The "basic" format for year 0 is the four-digit form 0000, which equals the historical year 1 BC.
> Also, for a fun read there is also this on that same just below:
> Several "expanded" formats are possible: −0000 and +0000, as well as five- and six-digit versions. Earlier years are also negative four-, five- or six-digit years, which have an absolute value one less than the equivalent BC year, hence -0001 = 2 BC. Because only ISO 646 (7-bit ASCII) characters are allowed by ISO 8601, the minus sign is represented by a hyphen-minus."

Well knowing that the issue was that there is no year 0 Imma just put in a 1 there.

```
var date = new Date(1,1,1)
date.setFullYear(0);
date
Tue Feb 01    0 00:00:00 GMT+0100 (W. Europe Standard Time)
```

Notice Feb thing right there? Yeah, duno about that..

Ok so lets give date the longformat instead, where you even can specify the timezone. Which really looks like this:
`**Sat** Jan 01 0001 00:00:00 UTC`

But you can actually just skip the day of the week. which is great since you do not have to check that yourself! However I did check that in python and it says its a Monday, so Im not sure if I should care about this or not, Im hoping that I dont have to so I will just leave it at that!

```
>>> import time
>>> import datetime
>>> datetime.date(1,1,1)
>>> datetime.date(1,1,1).ctime()
'**Mon** Jan  1 00:00:00 000**1**'
```

Anyways, we can use that longer format now:

```
var date = new Date("Sat Jan 01 0001 00:00:00 UTC");
date.setFullYear(0);
```

Again, even with the long form it messes up the year. You know like last time where it defaulted to 1901 when you gave it the year 01, but this time its to 2001(!). So it seems that 01 gives you 1901, but 0001 gives you 2001. just a FYI for anyone out there.

Now to get the unix time we use [.getTime()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTime)

```
date.getTime()
-62167219200000
```

which returns you the number of milliseconds, so / 1000 and we should be done finally!

unixtime/universal time of midnight january 1st at year 1 AC should be:
`-62167219200`
Great!

Since I have not come to trust javascripts Date and my understanding of how to use it I really want to make sure this is the correct time. A quick check against http://www.onlineconversion.com/unix_time.htm seems to give the same thing. However looking at the source of the page it uses javascript Date so thats moot for confirmation on the correctness of javascript Date.

I was thinking of using python for checking, but decided against [it](http://stackoverflow.com/a/8778548/1723886).

So leeme just make a quick check manually to see if it is resonable.
`60*60*24*365.25 =31557600`, seconds in a typical year
`(1970 - 1) * 31557600 = 62136914400`, 1969 years in between year 1 and 1970.
`62167219200-62136914400 = 30304800`, seconds we differ
`30304800 / 31557600 =0.96`, these seconds converted back to years. So we seem to have an off-by-one and then some... You know what, I'm fine with that, its late and I need to get to bed. 

What I used in the code was -2211753600000 which I got out from
http://www.epochconverter.com/ when giving it the time 0000-00-00
00:00:00. Looking at it now I can see that it gives a timestamp in
milliseconds and that the "Human time" is 1899 which goes back to the
issue mentioned early on, years 00-99 gets year 19xx instead, a 0 in
day and month just means we go back in time a bit. Makes sense. Though
somehow this site says that it should be the 30th of November, so
while it makes sense that it goes backwards to 1899 its baffling why
it would be November. Also, come to think about it 0000 got translated
to 2001 in an earlier instance but I guess it all depends on how the
site implements its things in the backend.

So while I agree that a magic number is not great I would suggest that we change the date to unixtime 0 instead and use:

```
var date = new Date(1970,01,01)
date.getTime()
> 2674800000
```

Ok, obviously something is a bit off here..

```
date
> Sun Feb 01 1970 00:00:00 GMT+0100 (W. Europe Standard Time)
```

Ok, so why is it Febuary again? As it turns out, when giving time in full javascript format or whatever then the months range from 01-12. **But**, and this is the fun part, when giving the numbers in any other way its range is 0-11. This *of course* makes sense since we are programmers and count from 0, not from 1 like those simpleminded people on the streets!

So lets do that last part again! And I kid you not, I assumed at this point that days would work the same way as months in this format, starting from 0. But NO it is 1-31! So months from 0-11 and days from 1-31. Again this makes sense since we are programmers we count from 0 besides when it would be confusing! **asdasdaslfjhsg!**

```
var date = new Date(1970,00,01)
date.getTime() / 1000 // lets convert to seconds.
> -3600
```

and now we just have my GMT+1 error left. So lets use the longform then.

```
var date = new Date("1970-01-01 00:00:00 UTC") 
date.getTime() / 1000
> 0
```

So, finally! My suggestion is then let us change the default fallback day to unix time 0.  I will replace the other magic number with the corresponding new Date("year-in-longform").getTime / 1000. 

Rant done.
