# Chapter 1: Time-related calculations — Introduction

> **Sample files:** https://sql.bi/dax-200


This chapter introduces the four time-related calculations patterns
presented in the next chapters. The goal here is to help you choose
the right pattern based on your specific needs. Indeed, when it
comes to time-related calculations, the choice of the pattern is hard.
First, what is a time-related calculation? A time-related calculation
refers to any calculation that involves time. Examples include the
set of period-to-date calculations, like year-to-date, quarter-to-date,
or month-to-date. These calculations accumulate values from the
beginning of a time period – year, quarter, month – and they return
the aggregation of the measure from the start of the period to the
date shown in the report. The definition of a time period changes
depending on whether you work with the Gregorian calendar or a
fiscal calendar. In Figure 1-1, you can see an example of period-to-
date calculations, where YTD stands for year-to-date, and QTD for
quarter-to-date.


*📊 FIGURE 1-1 Examples of period-to-date calculations.*
*Included in these patterns are also comparisons of a parameter*

over a certain period of time, with a different period of time. For
example, you can compare the sales of the current month against
the sales of the same month in the previous year. Another example
of time-related calculations is the moving average over a time
period, like a rolling average over 12 months which smoothes out
line charts and removes the effect of seasonality from calculations.
The four time-related patterns implement the same set of
calculations.
What makes the patterns so different from one another, is the
definition of what a calendar is. You can already appreciate the
different definitions of a year-to-date calculation by looking at Figure
1-1. Depending on whether you are working with the Gregorian or
the fiscal calendar, the numbers are different. When talking about a

calendar, things can easily become very complicated because of the
definition of the calendar.
For example, you might have a week-based calendar following an
ISO standard or your own definition. In a week-based calendar
every month starts the same day of the week, and the same goes
for the year. Therefore, a year in a week-based calendar might start
in the Gregorian year before, or end in the next one. Moreover,
some calendars split a year into 13 periods instead of 12 months,
for accounting purposes. The calendar requirements are the main
driver for the choice of the time-related pattern.
The four time-related patterns are presented in order of increasing
complexity:

1. Standard time-related calculations
2. Month-related calculations
3. Week-related calculations
4. Custom time-related calculations

The Standard time-related calculations pattern is implemented
using regular DAX time intelligence functions. It works based on the
assumption that your calendar is a regular Gregorian calendar and
that your fiscal calendar starts at the beginning of a Gregorian
quarter. For example, DAX time intelligence functions work fine if
your fiscal calendar starts on July 1 (start of the third quarter of a
Gregorian calendar). Yet, they might provide unexpected results if
your fiscal calendar starts on March 1 – both because March does
not start a Gregorian quarter, and because of a historical bug in
handling leap years with fiscal calendars. Despite these limitations,
the pattern is easy to use and implement because it relies on
standard DAX functions and works with a regular date table, with

few requirements.
The next three patterns do not use DAX time intelligence
calculations. Rather, they are written using basic DAX functions –
which leaves much more flexibility in the definition of what a
calendar is in terms of quarters, months, and weeks. These patterns
require you to build a Date table whose columns are required for the
DAX measures to identify the fractions of the year. For example,
you need one column containing the year, one for the quarter, one
for the month, plus additional columns to simplify the calculations.
Moreover, many details need to be considered when detecting and
filtering periods. Many calculations that look easy to humans prove
to be very complicated for a computer. When you compare one
quarter against the previous one, you need to select a different
number of days for the two quarters: the January-March quarter is
shorter than the April-June quarter. The same goes for the months:
January is longer than February, but if you want to make a
comparison month-over-month, you need two date selections of
different lengths.
If standard time intelligence functions do not meet your needs,
then you need to implement one of the other three patterns. All of
them require the creation of your own Date table.
The Month-related calculations pattern is the easiest one. It
implements all the calculations assuming that you are not interested
in the daily details. For example, if you need calculations and
reports that compare one month against another, then the pattern is
a good fit. The pattern does not support sub-month selections. If
you want to compare three days in a quarter against the same three
days in the previous quarter, then you exceed the potential of the
pattern: it just does not work. Despite strong limitations in its
analytical power (limited to monthly granularity) the month-related

pattern is extremely fast and simple to implement. Moreover, it can
handle scenarios where you have more than 12 months seamlessly.
It comes with the flexibility of a custom-made pattern, and it is
simpler than the standard time-related pattern. If you are ok with its
limitation about the month granularity, this should be the pattern of
choice.
In the Week-related calculations pattern, the week is the foundation
of the calendar. The ISO 8601 is one of the standards that provide a
definition of a week date system – even though many countries
adopt different national standards to identify years, quarters, and
weeks. One year has 52 or 53 weeks, one quarter has 13 weeks,
and each quarter is subdivided into 5+4+4 weeks, 4+5+4 weeks, or
4+4+5 weeks. When there are 53 weeks in a year, there are 14
weeks in one of the quarters. Because a week is not necessarily
entirely included in a month, the group of weeks within a quarter
should be called a “period” even though it is often referred to as a
month. For this reason, we refer to the month names as “periods” in
the following description.
Because weeks are the main entity, there is no correspondence
between a year in a Gregorian calendar and a year in a week-based
calendar. A week-based calendar always starts on the same
weekday, like Monday or Sunday. Therefore, only occasionally does
this day happen to fall on January 1. For a weekly year, it is totally
fine to start on December 29 of the previous year, or on January 3
of the current year. Despite being somewhat unusual, weekly
calendars come with some great characteristics: every “month” in a
quarter includes the same number of weekdays. Comparing one
quarter against another means comparing the same number of days
and the same distribution of weekdays.
Week-based calendars require a dedicated Date table with several

columns to drive the DAX calculations. Moreover, there are no pre-
existing DAX functions available to compute calculations over such
calendars. Therefore, week-based calculations are implemented
with custom DAX code. The complexity is higher than the month-
related pattern because the week-related pattern lets you filter any
time period, down to the day level. If you have a calendar based on
weeks, the week-based calculations pattern is what you have to
implement.
The Custom time-related calculations pattern is the most flexible
(and complex) one. This last pattern provides the same calculations
as the standard time-related pattern. The relevant difference is that
the entire pattern is written using basic DAX functions: we do not
use any DAX time intelligence functions. Consequently, the pattern
is extremely flexible because you can freely change the behavior of
the calculations. With greater flexibility comes greater complexity.
The DAX code of the last pattern is not trivial. It requires much
attention to small details. Use it only if none of the other patterns
satisfies your business requirements, and you really need the
complete flexibility it provides.
Finally, which pattern should you choose?

If your requirements are satisfied by a regular Gregorian
calendar, the Standard time-related calculations pattern is the
obvious choice.
If the month granularity is enough for your reporting needs –
which is often the case, more often than expected – then the
Month-related calculations pattern is the optimal choice: fast
and simple.
If you work with a calendar based on weeks, then you need
the Week-related calculations pattern.

If none of the above is enough and you really need total
flexibility, be prepared for a long and fascinating trip into the
intricacies of filter contexts and dive straight into the Custom
time-related calculations pattern.

Remember: with a Business Intelligence project, simpler is better.
Choose the most straightforward pattern that satisfies your needs.
Needless to say, if you are curious about the differences between
the various implementations, it might be useful to have a quick read
through all four chapters before making your choice.

