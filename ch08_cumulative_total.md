# Chapter 8: Cumulative total

> **Sample files:** https://sql.bi/dax-207


The cumulative total pattern allows you to perform calculations such
as running totals. You can use it to implement warehouse stock and
balance sheet calculations using the original transactions instead of
using snapshots of data over time.
For example, in order to create an Inventory table that shows the
stock of each product for every month, you can make that calculation
by using the original warehouse movements table, without
processing and consolidating data in advance.
The most frequent case of running total is the sum of all the
transactions made before a given date. But that same calculation
can be used in any scenario where you accumulate values over any
sortable column. This is shown in one of the examples of this
pattern.


### Basic scenario


We want to create a measure that sums all the sales values up to a
certain date. The result should look like what we show in Figure 8-1.


*📊 FIGURE 8-1 The running total accumulates values from the beginning of time up to the*
*current date.*


The formula must compute the value of Sales Amount for all the
dates which are less than or equal to the last one visible in the
current filter context. The code also performs an additional check to
avoid showing values for future dates – that is, when the minimum
visible date is greater than the last date with sales:


> *Measure in the Sales table*

It is important that the Date table is marked as a date table for the
formula to work. If not, it is necessary to add REMOVEFILTERS over
Date as a further CALCULATE modifier, when applying the filter in
the computation of the Result variable:

Either way, the formula of Sales Amount RT applies a filter to the Date
table which removes all the previously existing filters on Date.
Therefore, if you need to keep existing filters on some columns of
the Date table, you must apply these filters again. For example, in
order to compute the running total while keeping the filter on the day
of the week, the code would be the following:


> *Measure in the Sales table*

*📊 Figure 8-2 shows the two measures RT Weekdays and Sales Amount*
*RT running totals behaving differently, with and without the additional*

filter on the days of the week.


*📊 FIGURE 8-2 The RT Weekdays measure accurately accumulates values from the beginning*
*of time taking into account just the selected days; Sales Amount RT ignores the selection*

made in the Day of Week slicer.


Cumulative total on columns that can
be sorted
Most commonly, the cumulative total pattern tends to be based on
the date. That said, that pattern can be adapted to any column that
can be sorted. The option for a column to be sorted is important
because the code includes a “less than or equal to” condition to work
properly.
As an example, we classify customers based on sales volumes,
according to the table in Figure 8-3.


*📊 FIGURE 8-3 The configuration table controls how to cluster customers based on sales.*
*We want to produce a report that shows the sales amount of each*

class along with the running total of sales by customer class, as you
can see in Figure 8-4.


*📊 FIGURE 8-4 The running total computes the sales amount including “previous” classes of*
*customers.*


The code requires us to pay special attention to the Sort by
Column. Indeed, because the column shown in the report is
Customer[Customer Class] and ordering is achieved by
Customer[Customer Class Number], the calculation must override the
filters on both columns even though the entire calculation is only
based on the class number:


> *Measure in the Sales table*

```dax
The ALLSELECTED function used in order to evaluate the
```

ClassesToSum variable only takes into account the classes visible in
the visual for the running total calculation. In case Sort by Column is
not being used, the ALLSELECTED can include the single column to
filter.
