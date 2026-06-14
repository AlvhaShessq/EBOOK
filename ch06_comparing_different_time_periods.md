# Chapter 6: Comparing different time periods

> **Sample files:** https://sql.bi/dax-205


This pattern is a useful technique to compare the value of a measure
in different time periods. For example, we can compare the sales of
the last month against a user-defined period. The two time periods
might have a different number of days, like comparing one month
against a full year. When the durations of both time periods are
different, we should adjust the values to make a fair comparison.


Pattern description
The user selects two different time periods (current, comparison)
through slicers. The report in Figure 6-1 shows the sales in the
current period and in a comparison period. The sales of the
comparison period must be adjusted using the number of days in
each period as the allocation factor.


*📊 FIGURE 6-1 The report shows sales in different periods, alongside the adjusted comparison*
*value.*


In order to enable the choice of two different time periods, the
model must contain two date tables: one to select the current period,
one to select the comparison period. As shown in Figure 6-2, the
additional Comparison Date table is linked to the original Date table
with an inactive relationship: This simplifies the handling of
relationships with other fact tables.


*📊 FIGURE 6-2 The Comparison Date table is linked to the Date table through an inactive*
*relationship.*


When a measure evaluates an expression filtered by the
Comparison Date table, the measure expression activates the
relationship between Comparison Date and Date; it also performs a
REMOVEFILTERS on the Date table in order to use - in Sales - the
filter from Comparison Date. Using this model, any existing measure
can compute the value in the current or comparison period with a
simple change in the active relationship.
The following is the definition of the Comparison Sales Amount
measure:


> *Measure in the Sales table*

```dax
In order to adjust the value of Comparison Sales Amount, we need an
```

allocation method. In the example we use the number of days in the
two periods as the allocation factor; the business logic may dictate
that only working days should be used for the adjustment. In other
words, a different adjustment logic is possible and depends on the
business requirements.
In this example of adjustment logic, if the comparison period has
more days than the current time period, we reduce the Comparison
Sales Amount result according to the ratio between the number of
days in the two periods:


> *Measure in the Sales table*