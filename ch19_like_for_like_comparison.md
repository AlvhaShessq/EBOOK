# Chapter 19: Like-for-like comparison

> **Sample files:** https://sql.bi/dax-225


The like-for-like sales comparison is an adjusted metric that
compares two time periods, restricting the comparison to products or
stores with the same characteristics. In this example, we use the
like-for-like technique to compare the sales of Contoso stores that
had sales in all the time periods considered. The stores are
continuously updated: new stores are opened, other stores are
closed or renovated. The like-for-like comparison only evaluates
those stores that were open in all the periods considered. This way,
the report does not show a store that seems to be underperforming
simply because it was closed during the period analyzed.
As is the case with many other patterns, like-for-like can be
computed statically or dynamically. The choice is both in terms of
performance and in terms of business requirements. The variations
of the “Same store sales” measure described in the following
paragraphs are examples of like-for-like sales comparisons.

Introduction
If you analyze sales figures without considering whether stores were
open or closed within the time period you are analyzing, looking at
the following report might mislead you into thinking that there were
issues in 2009 because of the dramatic drop in sales.


*📊 FIGURE 19-1 Sales in 2009 dropped significantly.*
*In 2009 many stores were closed. Therefore, the numbers reflect a*

substantial drop in sales due to the lower number of open stores, as
you can see in the following report that shows which stores were
open in different years. A blank cell means that the store was closed
in that particular year.


*📊 FIGURE 19-2 Not all the stores are open every year.*
*In the “same store sales” measure, you must compute the sales*

amount just for the stores that were open during the entire time
period (2007-2009), namely three stores.


*📊 FIGURE 19-3 Only three stores were open during the entire three-year period.*
*The measure must compute the correct value even when sliced by*

different attributes, as shown in Figure 19-4.


*📊 FIGURE 19-4 The measure totals the same numbers also when sliced by other attributes.*
*Same store sales with snapshot*

The best method to solve the same store sales scenario is to use a
snapshot table to manage store statuses. Later in this pattern we
also demonstrate how to compute same store sales in a dynamic
way without a snapshot table. Nevertheless, the snapshot table is
the best option for both performance and manageability.
The snapshot table must contain all the stores and years, with an
additional column indicating the status.


*📊 FIGURE 19-5 The snapshot table StoreStatus indicates the status of each store in different*
*years.*


The StoreStatus snapshot table can be created with the following
calculated table:


> *Calculated table*

```dax
The StoreStatus snapshot table has a granularity by store and year.
```

Therefore, it has a regular strong relationship with the Store table and
a weak Many-Many-Relationship (MMR) with the Date table. If weak
relationships are not available in your tool - like in Power Pivot - then
you must transfer the filter from Date to Store in DAX using TREATAS
or INTERSECT.


*📊 FIGURE 19-6 The data model requires a weak MMR relationship between Date and*
*StoreStatus.*


The Same Store Sales measure checks the stores whose status is always “Open” during the
entire selected period. If a store is “Closed” at any point, then SELECTEDVALUE returns
either blank or “Closed”, filtering out that store:


> *Measure in the Receipts table*

```dax
The formula requires the snapshot table to contain the rows for all
the years and stores. If you store in the snapshot table only the
years when a store was open, then the code no longer works.
```


Same store sales without snapshot
In case you do not have the option of building a snapshot table,
same store sales can be computed in a more dynamic way using
only DAX code.
If the snapshot table is not available, then you must compute the
number of years of the report dynamically, and then filter all the
stores that have sales in all the years. In other words, if the report is
showing three years, then only the stores that have sales in all three
years should survive the filter. If a store does not have sales in any
one of the selected years, then that store will not be considered for
the calculation:


> *Measure in the Receipts table*

From a computation perspective, this formula is much more
expensive than the one using the snapshot. Besides, the entire logic
to determine whether a store is open or closed lies inside the
formula. In our experience, such business logic is better handled
outside of DAX, possibly stored in the data source. Therefore, if you
do not have that information available in the data source we suggest
the implementation using the snapshot - even for smaller data
models.

The Same Store Sales Dynamic measure shows three stores that
were open in Canada for the entire time period (2007-2009).


*📊 FIGURE 19-7 Only three stores were open in Canada during the entire three-year period.*