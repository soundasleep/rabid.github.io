---
layout: post
title:  "Datetime timezones in Python"
subtitle: "I have opinions about them"
date:   2014-08-22 12:00:58
author: brenda
---



A quirk of pytz, lead to learning something interesting about New Zealand.


    In [2]: import pytz; pytz.timezone("Pacific/Auckland")
    Out[2]: <DstTzInfo 'Pacific/Auckland' NZMT+11:30:00 STD>
    
    In [3]: import datetime; datetime.datetime(2011, 06, 12, 12, 54, 23, 00, pytz.timezone('Pacific/Auckland')).strftime('%Y-%m-%d %H:%M:%S %Z%z')
    Out[3]: '2011-06-12 12:54:23 NZMT+1130'

What's that? Our timetime is Auckland, with an offset of 11:30??

<!--break-->


That's because New Zealand hasn't always been in UTC+12  (or +13 in summer). We used to synchronise our clocks to the sun, 12pm noon being when the sun was over head -- but the advent of the telegraph lead to a country wide standardisation of time -- the winner was Christchurch, it being near the centre of NZ in longitude. Christchurch is 11 1/2 hours ahead of GMT, thus we standardised on GMT+11:30.

<a href="http://www.teara.govt.nz/en/timekeeping/page-2">There's lots of intesting details over at Te Ara</a>

    In [4]: datetime.datetime(2011, 06, 12, 12, 54, 23, 00, pytz.timezone('NZ')).strftime('%Y-%m-%d %H:%M:%S %Z%z')
    Out[4]: '2011-06-12 12:54:23 NZMT+1130'

likewise for any other country, it uses the oldest timezone info it
can find (aka the first one)
Finland ditched HMT in the 1920s

    In [11]: pytz.timezone("Europe/Helsinki")
    Out[11]: <DstTzInfo 'Europe/Helsinki' HMT+1:40:00 STD>


For Python, it's not what timezone NZ is, but what timezone NZ was using on that datetime. If i feed it 2011, it'll correct pick UTC+12.

So i have to call it like this:

    In [51]: pytz.timezone('Pacific/Auckland').localize(datetime(2011, 06, 03, 12, 54, 23, 00))
    Out[51]: datetime.datetime(2011, 6, 03, 12, 54, 23, tzinfo=<DstTzInfo 'Pacific/Auckland' NZST+12:00:00 STD>)

or something in summer

    In [52]: pytz.timezone('Pacific/Auckland').localize(datetime(2011, 02, 06, 12, 54, 23, 00))
    Out[52]: datetime.datetime(2011, 2, 6, 12, 54, 23, tzinfo=<DstTzInfo 'Pacific/Auckland' NZDT+13:00:00 DST>)

or something pre Tiriti O Waitangi:

    In [53]: pytz.timezone('Pacific/Auckland').localize(datetime(1829, 02, 06, 12, 54, 23, 00))
    Out[53]: datetime.datetime(1829, 2, 6, 12, 54, 23, tzinfo=<DstTzInfo 'Pacific/Auckland' NZMT+11:30:00 STD>)


