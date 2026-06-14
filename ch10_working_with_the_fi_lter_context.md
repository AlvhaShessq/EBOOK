# Chapter 10: Working with the fi lter context

In the previous chapters you learned how to create fi lter contexts to perform advanced calculations.
For example, in the time intelligence chapter, you learned how to mix time intelligence calculations
to provide a comparison of different periods. In Chapter 9, “Calculation groups,” you learned how to
simplify the user experience and the DAX code using calculation groups. In this chapter, you learn
many functions that read the state of the current fi lter context, in order to change the behavior of
your formulas according to existing selections and fi lters. These functions, though powerful, are not
frequently used. Nevertheless, a correct understanding of these functions is needed to create measures
that work well in different reports rather than just in the report where they were used the fi rst time.
A formula might work or not depending on how the fi lter context is set. For example, you might
write a formula that works correctly at the month level but returns an inaccurate result at the year level.
Another example would be the ranking of a customer against all other customers. That formula works
if a single customer is selected in the fi lter context, but it would return an incorrect result if multiple
customers were visible. Therefore, a measure designed to work in any report should inspect the fi lter
context prior to returning a value. If the fi lter context satisfi es the requirements of the formula, then it
can return a meaningful value. Otherwise, if the fi lter context contains fi lters that are not compatible
with the code, then returning a blank is a better choice.
Be mindful that no formula should ever return an incorrect value. It is always better to return no
value than to compute an incorrect value. Users are expected to be able to browse your model without
any previous knowledge of the internals of the code. As a DAX author, you are responsible for making
sure your code works in any situation.
For each function introduced in this chapter, we show several scenarios where it might be useful and
logical to use the function itself. But your scenario is surely different from any of our examples. Thus,
when reading about these functions, try to fi gure out how they could improve the features of your
model.
Besides, in this chapter we also introduce two important concepts: data lineage and the TREATAS
function. Data lineage is an intuitive concept you have been using so far without a complete explanation. In this chapter we go deeper, describing its behavior and several scenarios where it is useful to
consider it.

314 CHAPTER 10 Working with the fi lter context
Using HASONEVALUE and SELECTEDVALUE
As outlined in the introduction, many calculations provide meaningful values based on the current
selection. Nevertheless, the same calculation on a different selection provides incorrect fi gures. As an
example, look at the following formula computing a simple quarter-to-date (QTD) of the sales amount:
QTD Sales :=

## Calculate (

[Sales Amount],
DATESQTD ( 'Date'[Date] )
)
As you can see in Figure 10-1, the code works well for months and quarters, but at the year level
(CY 2007) it produces a result stating that the QTD value of Sales Amount for 2017 is 2,731,424.16.
FIGURE 10-1 QTD Sales reports values at the year level too, but the numbers might confuse some users.
Actually, the value reported by QTD Sales at the year level is the value of the last quarter of the year,
which corresponds to the QTD Sales value of December. One might argue that—at the year level—the
value of QTD does not make sense. To be correct, the value should not appear at the quarter level.
Indeed, a QTD aggregation makes sense at the month level and below, but not starting at the quarter
level and above. In other words, the formula should report the value of QTD at the month level and
blank the value otherwise.

CHAPTER 10 Working with the fi lter context 315
In such a scenario, the function HASONEVALUE becomes useful. For example, to remove the total at
the quarter level and above, it is enough to detect multiple months selected. This is the case at the year
and quarter levels, whereas at the month level there is only one month selected. Thus, protecting the
code with an IF statement provides the desired behavior. The following code does the job:
QTD Sales :=

## If (

HASONEVALUE ( 'Date'[Month] ),

## Calculate (

[Sales Amount],
DATESQTD ( 'Date'[Date] )
)
)
The result of this new formula is visible in Figure 10-2.
FIGURE 10-2 Protecting QTD Sales with HASONEVALUE lets you blank undesired values.
This fi rst example is already an important one. Instead of just leaving the calculation “as is,” we
decided to go one step further and question exactly “when” the calculation produces a value that
makes sense. If it turns out that a certain formula does not produce accurate results in a fi lter
context, it is better to verify whether the fi lter context satisfi es the minimum requirements and
operate accordingly.
In Chapter 7, “Working with iterators and with CALCULATE,” you saw a similar scenario when you
learned about the RANKX function. There, we had to produce a ranking of the current customer
against all other customers, and we used HASONEVALUE to guarantee that such ranking is only
produced when a single customer is selected in the current fi lter context.

316 CHAPTER 10 Working with the fi lter context
Time intelligence is a scenario where HASONEVALUE is frequently used because many aggregations—
like YTD, for example—only make sense when the fi lter context is fi ltering one quarter, one month,
or one specifi c time period. In all other cases, the formula should avoid returning a value and should
return BLANK instead.
Another common scenario where HASONEVALUE is useful is to extract one selected value from
the fi lter context. There used to be many scenarios where this could be useful, but with the advent of
calculation groups, their number is much lower. We will describe a scenario performing some sort of
what-if analysis. In such a case, the developer typically builds a parameter table that lets the user select
one value through a slicer; then, the code uses this parameter to adjust the calculation.
For example, consider evaluating the sales amount by adjusting the values of previous years based
on the infl ation rate. In order to perform the analysis, the report lets the user select a yearly infl ation
rate that should be used from each transaction date up to today. The infl ation rate is a parameter of the
algorithm. A solution to this scenario is to build a table with all the values that a user can select. In our
example, we created a table with all the values from 0% to 20% with a step of 0.5%, obtaining the table
you can partially see in Figure 10-3.
FIGURE 10-3 Infl ation contains all the values between 0% and 20% with a step of 0.5%.
The user selects the desired value with a slicer; then the formula needs to apply the selected infl ation rate for all the years from the transaction date up to the current date. If the user does not perform
a selection or if they select multiple values, then the formula should use a default infl ation rate of 0% to
report the actual sales amount.
The fi nal report looks like the one in Figure 10-4.

CHAPTER 10 Working with the fi lter context 317
FIGURE 10-4 The Infl ation parameter controls the multiplier of previous years.
Note The What-If parameter feature in Power BI generates a table and a slicer using the
same technique described here.
There are several interesting notes about this report:

> **Note:** A user can select the infl ation rate to apply through the top-left slicer.

> **Note:** The report shows the year used to perform the adjustment, reporting the year of the last sale in
the data model in the top-right label.

> **Note:** Infl ation Adjusted Sales multiplies the sales amount of the given year by a factor that depends
on the user-selected infl ation.

> **Note:** At the grand total level, the calculation needs to apply a different multiplier to each year.
The code for the reporting year label is the simplest calculation in the report; it only needs to
retrieve the year of the maximum order date from the Sales table:
Reporting year := "Reporting year: " & YEAR ( MAX ( Sales[Order Date] ) )

318 CHAPTER 10 Working with the fi lter context
Similarly, one could retrieve the selected user infl ation by using MIN or MAX, because when the
user fi lters one value with the slicer, both MIN and MAX return the same value—that is, the only
value selected. Nevertheless, a user might make an invalid selection by fi ltering multiple values or
by applying no fi lter at all. In that case, the formula needs to behave correctly and still provide a
default value.
Thus, a better option is to check with HASONEVALUE whether the user has actively fi ltered a single
value with the slicer, and have the code behave accordingly to the HASONEVALUE result:
User Selected Inflation :=

## If (

HASONEVALUE ( 'Inflation Rate'[Inflation] ),
VALUES ( 'Inflation Rate'[Inflation] ),
0
)
Because this pattern is very common, DAX also offers an additional choice. The SELECTEDVALUE
function provides the behavior of the previous code in a single function call:
User Selected Inflation := SELECTEDVALUE ( 'Inflation Rate'[Inflation], 0 )
SELECTEDVALUE has two arguments. The second argument is the default returned in case there is
more than one element selected in the column passed as the fi rst argument.
Once the User Selected Infl ation measure is in the model, one needs to compute the multiplier for
the selected year. If the last year in the model is considered the year to use for the adjustment, then the
multiplier needs to iterate over all the years between the last year and the selected year performing the
multiplication of 1+Infl ation for each year:
Inflation Multiplier :=

```dax
VAR ReportingYear =
YEAR ( CALCULATE ( MAX ( Sales[Order Date] ), ALL ( Sales ) ) )
VAR CurrentYear =
SELECTEDVALUE ( 'Date'[Calendar Year Number] )
VAR Inflation = [User Selected Inflation]
VAR Years =
FILTER (
ALL ( 'Date'[Calendar Year Number] ),
AND (
'Date'[Calendar Year Number] >= CurrentYear,
'Date'[Calendar Year Number] < ReportingYear
)
)
VAR Multiplier =
MAX ( PRODUCTX ( Years, 1 + Inflation ), 1 )
RETURN
Multiplier
```

CHAPTER 10 Working with the fi lter context 319
The last step is to use the multiplier on a year-by-year basis. Here is the code of Infl ation Adjusted
Sales:
Inflation Adjusted Sales :=

## Sumx (

VALUES ( 'Date'[Calendar Year] ),
[Sales Amount] * [Inflation Multiplier]
)
Introducing ISFILTERED and ISCROSSFILTERED
Sometimes the goal is not to gather a single value from the fi lter context; instead, the goal is to check
whether a column or a table has an active fi lter on it. The reason one might want to check for the presence of a fi lter is usually to verify that all the values of a column are currently visible. In the presence of
a fi lter, some values might be hidden and the number—at that point—might be inaccurate.
A column might be fi ltered because there is a fi lter applied to it or because some other column is
being fi ltered, and therefore there is an indirect fi lter on the column. We can elaborate on this with a
simple example:
RedColors :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red"
)
During the evaluation of Sales Amount, the outer CALCULATE applies a fi lter on the Product[Color]
column. Consequently, Product[Color] is fi ltered. There is a specifi c function in DAX that checks whether
a column is fi ltered or not: ISFILTERED. ISFILTERED returns TRUE or FALSE, depending on whether the
column passed as an argument has a direct fi lter on it or not. When ISFILTERED receives a table as an
argument, it returns TRUE if any columns of the table are being fi ltered directly; otherwise, it returns

## False.

Although the fi lter is on Product[Color], all the columns of the Product table are indirectly
fi ltered. For example, the Brand column only shows the brands that have at least one red product.
Any brand with no red products will not be visible because of the fi lter on the color column. Apart
from Product[Color], all the other columns of the Product table have no direct fi lter. Nevertheless, their
visible values are limited. Indeed, all the columns of Product are cross-fi ltered. A column is cross-fi ltered
if there is a fi lter that may reduce its set of visible values, either a direct or an indirect fi lter. The function
to use to check whether a column is cross-fi ltered or not is ISCROSSFILTERED.
It is important to note that if a column is fi ltered, it is also cross-fi ltered. The opposite does not hold
true: A column can be cross-fi ltered even though it is not fi ltered. Moreover, ISCROSSFILTERED works

320 CHAPTER 10 Working with the fi lter context
either with a column or with a table. Indeed, whenever any column of a table is cross-fi ltered, all the
remaining columns of the table are cross-fi ltered too. Therefore, ISCROSSFILTERED should be used with
a table rather than with a column. You might still fi nd ISCROSSFILTERED used with a column because—
originally—ISCROSSFILTERED used to only work with columns. Only later was ISCROSSFILTERED
introduced for tables. Thus, some old code might still only use ISCROSSFILTERED with a column.
Because fi lters work over the entire data model, a fi lter on the Product table also affects the related
tables. Thus, the fi lter on Product[Color] applies its effect to the Sales table too. Therefore, any column
in the Sales table is cross-fi ltered by the fi lter on Product[Color].
To demonstrate the behavior of these functions, we used a slightly different model than the usual
model used in the rest of the book. We removed some tables, and we upgraded the relationship
between Sales and Product using bidirectional cross-fi ltering. You can see the resulting model in
Figure 10-5.
FIGURE 10-5 In this model the relationship between Sales and Product is bidirectional.
In this model, we authored this set of measures:

```dax
Filter Gender := ISFILTERED ( Customer[Gender] )
Cross Filter Customer := ISCROSSFILTERED ( Customer )
Cross Filter Sales := ISCROSSFILTERED ( Sales )
Cross Filter Product := ISCROSSFILTERED ( 'Product' )
Cross Filter Store := ISCROSSFILTERED ( Store )
```

CHAPTER 10 Working with the fi lter context 321
Finally, we projected all the measures in a matrix with Customer[Continent] and Customer[Gender]
on the rows. You can see the result in Figure 10-6.
FIGURE 10-6 The matrix shows the behavior of ISFILTERED and ISCROSSFILTERED.
Here are a few considerations about the results:

> **Note:** Customer[Gender] is only fi ltered on the rows where there is an active fi lter on
Customer[Gender]. At the subtotal level—where the fi lter is only on Customer[Continent]—the
column is not fi ltered.

> **Note:** The entire Customer table is cross-fi ltered when there is a fi lter on either Customer[Continent] or
Customer[Gender].

> **Note:** The same applies to the Sales table. The presence of a fi lter on any column of the Customer
table applies a cross-fi lter on the Sales table because Sales is on the many-side of a many-toone relationship with Customer.

> **Note:** Store is not cross-fi ltered because the fi lter on Sales does not propagate to Customer. Indeed,
the relationship between Sales and Store is unidirectional, so the fi lter does not propagate from
Sales to Store.

> **Note:** Because the relationship between Sales and Product is bidirectional, then the fi lter on Sales
propagates to Product. Therefore, Product is cross-fi ltered by any fi lter in other tables of this
data model.
ISFILTERED and ISCROSSFILTERED are not frequently used in DAX expressions. They are used when
performing advanced optimization by checking the set of fi lters on a column—to make the code follow
different paths depending on the fi lters. Another common scenario is when working with hierarchies,
as we will show in Chapter 11, “Handling hierarchies.”

322 CHAPTER 10 Working with the fi lter context
Beware that one cannot rely on the presence of a fi lter to determine whether all the values of a
column are visible. In fact, a column can be both fi ltered and cross-fi ltered but still show all the values.
A simple measure demonstrates this:
Test :=

## Calculate (


```dax
ISFILTERED ( Customer[City] ),
Customer[City] <> "DAX"
)
```

There is no city named DAX in the Customer table. Thus, the fi lter does not have any effect on the
Customer table because it shows all the rows. Therefore, Customer[City] shows all the possible values of
the column, even though a fi lter is active on the same column and the Test measure returns TRUE.
To check whether all the possible values are visible in a column or in a table, the best option is to
count the rows under different contexts. In this case, there are some important details to learn, which
we discuss in the following sections.
Understanding differences between VALUES and FILTERS
FILTERS is a function like VALUES, with one important difference. VALUES returns the values visible in
the fi lter context; FILTERS returns the values that are currently being fi ltered by the fi lter context.
Although the two descriptions look the same, they are not. Indeed, one might fi lter four product
colors with a slicer, say Black, Brown, Azure, and Blue. Imagine that because of other fi lters in the fi lter
context, only two of them are visible in the data if the other two are not used in any product. In that
scenario, VALUES returns two colors, whereas FILTER returns all the fi ltered four. An example is useful to
clarify this concept.
For this example, we use an Excel fi le connected to a Power BI model. The reason is that—at the time
of writing—FILTERS does not work as expected when used by SUMMARIZECOLUMNS, which is the
function used by Power BI to query the model. Thus, the example would not work in Power BI.
Note Microsoft is aware of the issue of using FILTERS in Power BI, and it is possible that this
problem will be solved in the future. However, for illustrating the concept in this book, we
had to use Excel as a client because Excel does not leverage SUMMARIZECOLUMNS.
In Chapter 7 we demonstrated how to use CONCATENATEX to show a label in a report indicating
the colors selected through a slicer. There, we ended up with a complex formula useful to demonstrate the usage of iterators and variables. Here, we recall the simpler version of that code for your
convenience:

CHAPTER 10 Working with the fi lter context 323
Selected Colors :=
"Showing " &

## Concatenatex (

VALUES ( 'Product'[Color] ),
'Product'[Color],

## ", ",

'Product'[Color],
ASC
) & " colors."
Consider a report with two slicers: one slicer fi lters only one category, and the other slicer fi lters
several colors, as shown in Figure 10-7.
FIGURE 10-7 Although there are four colors selected, the Selected Colors measure only shows
two of them.
Though there are four colors selected in the slicer, the Selected Colors measure only returns two of
them. The reason is that VALUES returns the values of a column under the current fi lter context. There
are no TV and Video products which are either blue or azure. Thus, even though the fi lter context is
fi ltering four colors, VALUES only returns two of them.
If the measure is changed to use FILTERS instead of VALUES, then FILTERS returns the fi ltered values,
regardless of whether there is any product in the current fi lter context representing those values:
Selected Colors :=
"Showing " &

## Concatenatex (


```dax
FILTERS ( 'Product'[Color] ),
'Product'[Color],
", ",
'Product'[Color],
ASC
) & " colors."
```

With this new version of Selected Colors, now the report shows all four colors as the selected ones, as
you can see in Figure 10-8.

324 CHAPTER 10 Working with the fi lter context
FIGURE 10-8 Using FILTERS, the Selected Colors measure now returns all four selected colors.
Similar to HASONEVALUE, DAX also offers a function to check whether a column only has one active
fi lter: HASONEFILTER. Its use and syntax are similar to that of HASONEVALUE. The only difference is
that HASONEFILTER might return TRUE when there is a single fi lter active, and at the same time
HASONEVALUE returns FALSE because the value, although fi ltered, is not part of the visible values.
Understanding the difference between ALLEXCEPT and

## All/Values

In a previous section we introduced ISFILTERED and ISCROSSFILTERED to check for the presence of a
fi lter. The presence of a fi lter is not enough to verify that all the values of a column—or of a table—are
visible. A better option is to count the number of rows in the current fi lter context and check it against
the count of all the rows without any fi lter.
As an example, look at Figure 10-9. The Filtered Gender measure checks for ISFILTERED on the
Customer[Gender] column, whereas NumOfCustomers simply counts the number of rows in the
Customer table:
NumOfCustomers := COUNTROWS ( Customer )
FIGURE 10-9 Even though Customer[Gender] is fi ltered, all the customers are visible.

CHAPTER 10 Working with the fi lter context 325
You can observe that whenever a customer is a company, as expected the gender is always a blank
value. In the second row of the matrix, the fi lter on Gender is active; indeed, Filtered Gender returns
TRUE. At the same time, the fi lter does not really fi lter anything because there is only one possible
Gender value, and it is visible.
The presence of a fi lter does not imply that the table is actually fi ltered. It only states that a fi lter
is active. To check whether all the customers are visible or not, it is better to rely on a simple count.
Checking that the number of customers with and without the fi lter on Gender is the same helps identify
that, although active, the fi lter is not effective.
When performing such calculations, one should pay attention to the details of the fi lter context and
of the behavior of CALCULATE. There are two possible ways of checking the same condition:

> **Note:** Counting the customers of ALL genders.

> **Note:** Counting the customers with the same customer type (Company or Person).
Even though in the report of Figure 10-9 the two calculations return the same value, if one changes
the columns used in the matrix, they compute different results. Besides, both calculations have pros
and cons; these are worth learning because they might be useful in several scenarios. We start from the
fi rst and easiest one:
All Gender :=

## Calculate (

[NumOfCustomers],
ALL ( Customer[Gender] )
)
ALL removes the fi lter on the Gender column, leaving all the remaining fi lters in place. As a result, it
computes the number of customers in the current fi lter, regardless of the gender. In Figure 10-10 you
can see the result, along with the All customers visible measure that compares the two counts.
FIGURE 10-10 On the second row, All customers visible returns True, even though Gender is fi ltered.
All Gender is a measure that works well. However, it has the disadvantage of hardcoding in the
measure the fact that it only removes the fi lter from Gender. For example, using the same measure on
a matrix that slices by Continent, the result is not the desired one. You see this in Figure 10-11 where the
All customers visible measure is always TRUE.

326 CHAPTER 10 Working with the fi lter context
FIGURE 10-11 Filtering by continent, All customers visible returns incorrect results.
Beware that the measure is not incorrect. It computes the value correctly, but it only works if the
report is slicing by gender. To obtain a measure that is independent from the Gender, then the path
to follow is the other one: removing all the fi lters from the Customer table except for the Customer
Type column.
Removing all the fi lters but one looks like a simple operation. However, it hides a trap that you should
be aware of. As a matter of fact, the fi rst function that comes to a student’s mind is ALLEXCEPT. Unfortunately, ALLEXCEPT might return unexpected results in this scenario. Consider the following formula:
AllExcept Type :=

## Calculate (

[NumOfCustomers],
ALLEXCEPT ( Customer, Customer[Customer Type] )
)
ALLEXCEPT removes all the existing fi lters from the Customer table except for the Customer Type
column. When used in the previous report, it computes a correct result, as shown in Figure 10-12.
FIGURE 10-12 ALLEXCEPT removes the dependency from the gender; it works with any column.
The measure does not only work with the Continent. By replacing the continent with the gender in
the report, it still produces a correct result, as shown in Figure 10-13.

CHAPTER 10 Working with the fi lter context 327
FIGURE 10-13 ALLEXCEPT works with the gender too.
Despite the report being accurate, there is a hidden trap in the formula. The ALL* functions, when
used as fi lter arguments in CALCULATE, act as CALCULATE modifi ers. This is explained in Chapter 5,
“Understanding CALCULATE and CALCULATETABLE.” These modifi ers do not return a table that is used
as a fi lter. Instead, they only remove fi lters from the fi lter context.
Focus your attention on the row with a blank gender. There are 385 customers in this group: All of
them are companies. If one removes the Customer Type column from the report, the only remaining
column in the fi lter context is the gender. When the gender shows the blank row, we know that only
companies are visible in the fi lter context. Nevertheless, Figure 10-14 is surprising because it shows the
same value for all the rows in the report; this value is the total number of customers.
FIGURE 10-14 ALLEXCEPT produces unexpected values if the customer type is not part of the report.
Here is a caveat: ALLEXCEPT removed all the fi lters from the Customer table apart from the fi lter on
the customer type. However, there is no fi lter on the customer type to retain. Indeed, the only fi lter in
the fi lter context is the fi lter on the Gender, which ALLEXCEPT removes.
Customer Type is cross-fi ltered, but it is not fi ltered. As a result, ALLEXCEPT has no fi lter to retain and
its net effect is the same as an ALL on the customer table. The correct way of expressing this condition
is by using a pair of ALL and VALUES instead of ALLEXCEPT. Look at the following formula:
All Values Type :=

## Calculate (

[NumOfCustomers],
ALL ( Customer ),
VALUES ( Customer[Customer Type] )
)

328 CHAPTER 10 Working with the fi lter context
Though similar to the previous defi nition, its semantics are different. ALL removes any fi lter from the
Customer table. VALUES evaluates the values of Customer[Customer Type] in the current fi lter context.
There is no fi lter on the customer type, but customer type is cross-fi ltered. Therefore, VALUES only
returns the values visible in the current fi lter context, regardless of which column is generating the fi lter
that is cross-fi ltering the customer type. You can see the result in Figure 10-15.
FIGURE 10-15 Using ALL and VALUES together produces the desired result.
The important lesson here is that there is a big difference between using ALLEXCEPT and using ALL
and VALUES together as fi lter arguments in CALCULATE. The reason is that the semantic of an ALL*
function is always that of removing fi lters. ALL* functions never add fi lters to the context; they can only
remove them.
The difference between the two behaviors, adding fi lters or removing fi lters, is not relevant in many
scenarios. Nevertheless, there are situations where this difference has a strong impact, like in the
previous example.
This, along with many other examples in this book, shows that DAX requires you to be very precise
in the defi nition of your code. Using a function like ALLEXCEPT without thinking carefully about all the
implications might result in your code producing unexpected values. DAX hides a lot of its complexity by providing intuitive behaviors in most situations. Nevertheless, the complexity, although hidden,
is still there. One should understand the behaviors of fi lter contexts and CALCULATE well in order to
master DAX.
Using ALL to avoid context transition
By now, at this point in the book our readers have a solid understanding of context transition. It is an
extremely powerful feature, and we have leveraged it many times to compute useful values. Nevertheless, sometimes it is useful to avoid it or at least to mitigate its effects. To avoid the effects of the
context transition, the ALL* functions are the tools to use.
It is important to remember that when CALCULATE performs its operations, it executes each step
in a precise order: The fi lter arguments are evaluated fi rst, then the context transition happens if there
are row contexts, then the CALCULATE modifi ers are applied, and fi nally CALCULATE applies the
result of the fi lter arguments to the fi lter context. You can leverage this order of execution noting that

CHAPTER 10 Working with the fi lter context 329
CALCULATE modifi ers—among which we count the ALL* functions—are applied after the context transition. Because of this, a fi lter modifi er has the option of overriding the effect of the context transition.
For example, consider the following piece of code:

## Sumx (

Sales,

## Calculate (

…,
ALL ( Sales )
)
)
CALCULATE runs in the row context generated by SUMX, which is iterating over Sales. As such,
it should perform a context transition. Because CALCULATE is invoked with a modifi er—that is
ALL ( Sales )—the DAX engine knows that any fi lter on the Sales table should be removed.
When we described the behavior of CALCULATE, we said that CALCULATE fi rst performs context
transition (that is, it fi lters all the columns in Sales) and then uses ALL to remove those fi lters. Nevertheless, the DAX optimizer is smarter than that. Because it knows that ALL removes any fi lter from Sales, it
also knows that it would be totally useless to apply a fi lter and then remove it straight after. Thus, the
net effect is that in this case CALCULATE does not perform any context transition, even though it
removes all the existing row contexts.
This behavior is important in many scenarios. It becomes particularly useful in calculated columns.
In a calculated column there is always a row context. Therefore, whenever the code in a calculated
column invokes a measure, it is always executed in a fi lter context only for the current row.
For example, imagine computing the percentage of sales of the current product against all products
in a calculated column. Inside a calculated column one can easily compute the value of sales of the
current product by just invoking the Sales Amount measure. The context transition makes sure that the
value returned only represents the sales of the current product. Nevertheless, the denominator should
compute the sales of all the products, but the context transition is a problem. So, one can avoid the
context transition by using ALL, like in the following code:
'Product'[GlobalPct] =

```dax
VAR SalesProduct = [Sales Amount]
VAR SalesAllProducts =
CALCULATE (
[Sales Amount],
ALL ( 'Product' )
)
VAR Result =
DIVIDE ( SalesProduct, SalesAllProducts )
RETURN
Result
```

330 CHAPTER 10 Working with the fi lter context
Remember: the reason why ALL is removing the effect of the context transition is because ALL—
being a CALCULATE modifi er—is executed after the context transition. For this reason, ALL can override the effects of the context transition.
Similarly, the percentage against all the products in the same category is a slight variation of the
previous code:
'Product'[CategoryPct] =

```dax
VAR SalesProduct = [Sales Amount]
VAR SalesCategory =
CALCULATE (
[Sales Amount],
ALLEXCEPT ( 'Product', 'Product'[Category] )
)
VAR Result
DIVIDE ( SalesProduct, SalesCategory )
RETURN
Result
```

You can look at the result of these two calculated columns in Figure 10-16.
FIGURE 10-16 GlocalPct and CategoryPct use ALL and ALLEXCEPT to avoid the effect of the context transition.
Using ISEMPTY
ISEMPTY is a function used to test whether a table is empty, meaning that it has no values visible in the
current fi lter context. Without ISEMPTY, the following expression would test that a table expression
returns zero rows:
COUNTROWS ( VALUES ( 'Product'[Color] ) ) = 0

CHAPTER 10 Working with the fi lter context 331
Using ISEMPTY makes the code easier:
ISEMPTY ( VALUES ( 'Product'[Color] ) )
From a performance point of view, using ISEMPTY is always a better choice because it informs the
engine exactly what to check. COUNTROWS requires DAX to count the number of rows in the table,
whereas ISEMPTY is more effi cient and usually does not require a complete scan of the visible values of
the target table.
For example, imagine computing the number of customers who never bought certain products.
A solution to this requirement is the following measure, NonBuyingCustomers:
NonBuyingCustomers :=

```dax
VAR SelectedCustomers =
CALCULATETABLE (
DISTINCT ( Sales[CustomerKey] ),
ALLSELECTED ()
)
VAR CustomersWithoutSales =
FILTER (
SelectedCustomers,
ISEMPTY ( RELATEDTABLE ( Sales ) )
)
VAR Result =
COUNTROWS ( CustomersWithoutSales )
RETURN
Result
```

You can see in Figure 10-17 a report showing the number of customers and the number of
nonbuying customers side-by-side.
FIGURE 10-17 NonBuyingCustomers counts the customers who never bought any of the selected products.

332 CHAPTER 10 Working with the fi lter context
ISEMPTY is a simple function. Here we use it as an example to point the reader’s attention to one
detail. The previous code saved a list of customer keys in a variable, and later it iterated this list with
FILTER to check whether the RELATEDTABLE result was empty or not.
If the content of the table in the SelectedCustomer variable were the list of customer keys, how could
DAX know that those values have a relationship with Sales? A customer key, as a value, is not different
from a product quantity. A number is a number. The difference is in the meaning of the number. As a
customer key, 120 represents the customer with key 120, whereas as a quantity, it indicates the number
of products sold.
Thus, a list of numbers has no clear meaning as a fi lter, unless one knows where these numbers
come from. DAX maintains the knowledge about the source of column values through data lineage,
which we explain in the next section.
Introducing data lineage and TREATAS
As we anticipated in the previous section, “Using ISEMPTY,” a list of values is meaningless unless one
knows what those values represent. For example, imagine a table of strings containing “Red” and
“Blue” like the following anonymous table:
{ "Red", "Blue" }
As humans, we know these are colors. More likely, at this point in the book all our readers know
that we are referencing product colors. But to DAX, this only represents a table containing two strings.
Therefore, the following measure always produces the grand total of sales because the table containing
two values cannot fi lter anything:
Test :=

## Calculate (

[Sales Amount],
{ "Red", "Blue" }
)
Note The previous measure does not raise any error. The fi lter argument is applied to an
anonymous table, without any effect on physical tables of the data model.
In Figure 10-18 you can see that the result is the same as Sales Amount because CALCULATE does not
apply any further fi ltering.

CHAPTER 10 Working with the fi lter context 333
FIGURE 10-18 Filtering with an anonymous table does not produce any fi lter.
For a value to fi lter the model, DAX needs to know the data lineage of the value itself. A value that
represents a column in the data model holds the data lineage of that column. On the other hand,
a value that is not linked to any column in the data model is an anonymous value. In the previous
example, the Test measure used an anonymous table to fi lter the model and, as such, it did not fi lter
any column of the data model.
The following is a correct way of applying a fi lter. Be mindful that we use the full syntax of the CALCULATE fi lter argument for educational purposes; a predicate to fi lter Product[Color] would be enough:
Test :=

## Calculate (

[Sales Amount],

## Filter (

ALL ( 'Product'[Color] ),
'Product'[Color] IN { "Red", "Blue" }
)
)
Data lineage fl ows this way: ALL returns a table that contains all product colors. The result contains
the values from the original column, so DAX knows the meaning of each value. FILTER scans the table
containing all the colors and checks whether each color is included in the anonymous table containing
Red and Blue. As a result, FILTER returns a table containing the values of Product[Color], so CALCULATE
knows that the fi lter is applied to the Product[Color] column.

334 CHAPTER 10 Working with the fi lter context
One can imagine data lineage as a special tag added to each column, identifying its position in the
data model.
You typically do not have to worry about data lineage because DAX handles the complexity of data
lineage by itself in a natural and intuitive way. For example, when a table value is assigned to a variable, the table contains data lineage information that is maintained through the whole DAX evaluation
process using that variable.
The reason why it is important to learn data lineage is because one has the option of either maintaining or changing data lineage at will. In some scenarios it is important to keep the data lineage,
whereas in other scenarios one might want to change the lineage of a column.
The function that can change the lineage of a column is TREATAS. TREATAS accepts a table as its fi rst
argument and then a set of column references. TREATAS updates the data lineage of the table tagging each column with the appropriate target column. For example, the previous Test measure can be
rewritten this way:
Test :=

## Calculate (

[Sales Amount],
TREATAS ( { "Red", "Blue" }, 'Product'[Color] )
)
TREATAS returns a table containing values tagged with the Product[Color] column. As such, this new
version of the Test measure only fi lters the red and blue colors, as shown in Figure 10-19.
FIGURE 10-19 TREATAS updates the lineage of the anonymous table, so that fi ltering now works as expected.

CHAPTER 10 Working with the fi lter context 335
The rules for data lineage are simple. A simple column reference maintains its data lineage, whereas
an expression is always anonymous. Indeed, an expression generates a reference to an anonymous
column. For example, the following expression returns a table with two columns that have the same
content. The difference between the two columns is that the fi rst one retains the data lineage information, whereas the second one does not because it is a new column:

## Addcolumns (

VALUES ( 'Product'[Color] ),
"Color without lineage", 'Product'[Color] & ""
)
TREATAS is useful to update the data lineage of one or more columns in a table expression. The
example shown so far was only for educational purposes. Now we show a better example related to
time intelligence calculations. In Chapter 8, “Time intelligence calculations,” we showed the following
formula to compute the LASTNONBLANK date for semi-additive calculations:
LastBalanceIndividualCustomer :=

## Sumx (

VALUES ( Balances[Name] ),

## Calculate (

SUM ( Balances[Balance] ),

## Lastnonblank (

'Date'[Date],
COUNTROWS ( RELATEDTABLE ( Balances ) )
)
)
)
This code works, but it suffers from a major drawback: It contains two iterations, and the optimizer is
likely to use a suboptimal execution plan for the measure. It would be better to create a table containing the customer name and the date of the last balance, and then use that table as a fi lter argument in
CALCULATE to fi lter the last date available for each customer. It turns out that this is possible by using

## Treatas:

LastBalanceIndividualCustomer Optimized :=

```dax
VAR LastCustomerDate =
ADDCOLUMNS (
VALUES ( Balances[Name] ),
"LastDate", CALCULATE (
MAX ( Balances[Date] ),
DATESBETWEEN ( 'Date'[Date], BLANK(), MAX ( Balances[Date] ) )
)
)
VAR FilterCustomerDate =
TREATAS (
LastCustomerDate,
Balances[Name],
'Date'[Date]
)
```

336 CHAPTER 10 Working with the fi lter context

```dax
VAR SumLastBalance =
CALCULATE (
SUM ( Balances[Balance] ),
FilterCustomerDate
)
RETURN
SumLastBalance
```

The measure performs the following operations:

> **Note:** LastCustomerDate contains the last date for which there is data for each customer. The result is
a table that contains two columns: the fi rst is the Balances[Name] column, whereas the second
is an anonymous column because it is the result of an expression.

> **Note:** FilterCustomerDate has the same content as LastCustomerDate. By using TREATAS, both columns are now tagged with the desired data lineage. The fi rst column targets Balances[Name],
whereas the second column targets Date[Date].

> **Note:** The last step is to use FilterCustomerDate as a fi lter argument of CALCULATE. Because the table
is now correctly tagged with the data lineage, CALCULATE fi lters the model in such a way that
only one date is selected for every customer. This date is the last date with data in the Balances
table for the given customer.
Most of the time, TREATAS is applied to change the data lineage of a table with a single column.
The previous example shows a more complex scenario where the data lineage is modifi ed on a table
containing two columns. The data lineage of a table resulting from a DAX expression can include columns of different tables. When this table is applied to the fi lter context, it often generates an arbitrarily
shaped fi lter, discussed in the next section.
Understanding arbitrarily shaped fi lters
Filters in the fi lter context can have two different shapes: simple fi lters and arbitrarily shaped fi lters. All
the fi lters we have used so far are simple fi lters. In this section, we describe arbitrarily shaped fi lters, and
we briefl y discuss the implications of using them in your code. Arbitrarily shaped fi lters can be created
by using a PivotTable in Excel or by writing DAX code in a measure, whereas the Power BI user interface
currently requires a custom visual to create arbitrarily shaped fi lters. This section describes what these
fi lters are and how to manage them in DAX.
We can start by describing the difference between a simple fi lter and an arbitrarily shaped fi lter in
the fi lter context.

> **Note:** A column fi lter is a list of values for one column only. A list of three colors, like red, blue, and
green, is a column fi lter. For example, the following CALCULATE generates a column fi lter in the
fi lter context that only affects the Product[Color] column:

## Calculate (

[Sales Amount],
'Product'[Color] IN { "Red", "Blue", "Green" }
)

CHAPTER 10 Working with the fi lter context 337

> **Note:** A simple fi lter is a fi lter over one or more columns that corresponds to a set of simple column
fi lters. Almost all the fi lters used in this book so far are column fi lters. Column fi lters are created
quite simply, by using multiple fi lter arguments in CALCULATE:

## Calculate (

[Sales Amount],
'Product'[Color] IN { "Red", "Blue" },
'Date'[Calendar Year Number] IN { 2007, 2008, 2009 }
)
The previous code could be written using a simple fi lter with two columns:

## Calculate (

[Sales Amount],

## Treatas (

{
( "Red", 2007 ),
( "Red", 2008 ),
( "Red", 2009 ),
( "Blue", 2007 ),
( "Blue", 2008 ),
( "Blue", 2009 )
},
'Product'[Color],
'Date'[Calendar Year Number]
)
)
Because a simple fi lter contains all the possible combinations of two columns, it is simpler to
express it using two column fi lters.

> **Note:** An arbitrarily shaped fi lter is any fi lter that cannot be expressed as a simple fi lter. For example,
look at the following expression:

## Calculate (

[Sales Amount],

## Treatas (

{
( "CY 2007", "December" ),
( "CY 2008", "January" )
},
'Date'[Calendar Year],
'Date'[Month]
)
)
The fi lter on year and month is not a column fi lter because it involves two columns. Moreover,
the fi lter does not include all the combinations of the two columns existing in the data model.
In fact, one cannot fi lter year and month separately. Indeed, there are two year and two month
references, and there are four existing combinations in the Date table for the values provided,
whereas the fi lter only includes two of these combinations. In other words, using two column
fi lters, the resulting fi lter context would also include January 2007 and December 2008, which
are not included in the fi lter described by the previous code. Therefore, this is an arbitrarily
shaped fi lter.

338 CHAPTER 10 Working with the fi lter context
An arbitrarily shaped fi lter is not just a fi lter with multiple columns. Certainly, a fi lter with multiple
columns could be an arbitrarily shaped fi lter, but one can build a fi lter with multiple columns retaining the shape of a simple fi lter. The following example is a simple fi lter, despite it involving multiple
columns:

## Calculate (

[Sales Amount],

## Treatas (

{
( "CY 2007", "December" ),
( "CY 2008", "December" )
},
'Date'[Calendar Year],
'Date'[Month]
)
)
The previous expression can be rewritten as the combination of two column fi lters this way:

## Calculate (

[Sales Amount],
'Date'[Calendar Year] IN { "CY 2007", "CY 2008" },
'Date'[Month] = "December"
)
Although they seem complex to author, arbitrarily shaped fi lters can easily be defi ned through the
user interface of Excel and Power BI. At the time of writing, Power BI can only generate an arbitrarily
shaped fi lter by using the Hierarchy Slicer custom visual, which defi nes fi lters based on a hierarchy
with multiple columns. For example, in Figure 10-20 you can see the Hierarchy Slicer fi ltering different
months in 2007 and 2008.
FIGURE 10-20 Filtering a hierarchy makes it possible to build an arbitrarily shaped fi lter.

CHAPTER 10 Working with the fi lter context 339
In Microsoft Excel you fi nd a native feature to build arbitrarily shaped sets out of hierarchies, as
shown in Figure 10-21.
FIGURE 10-21 Microsoft Excel builds arbitrarily shaped fi lters using the native hierarchy fi lter.
Arbitrarily shaped fi lters are complex to use in DAX because of the way CALCULATE might change
them in the fi lter context. In fact, when CALCULATE applies a fi lter on a column, it removes previous
fi lters on that column only, replacing any previous fi lter with the new fi lter. The result is that typically,
the original shape of the arbitrarily shaped fi lter is lost. This behavior leads to formulas that produce
inaccurate results and that are hard to debug. Thus, to demonstrate the problem, we will increase the
complexity of the code step-by-step, until the problem arises.
Imagine you defi ne a simple measure that overwrites the year, forcing it to be 2007:
Sales Amount 2007 :=

## Calculate (

[Sales Amount],
'Date'[Calendar Year] = "CY 2007"
)
CALCULATE overwrites the fi lter on the year but it does not change the fi lter on the month. When it
is used in a report, the results of the measure might seem unusual, as shown in Figure 10-22.

340 CHAPTER 10 Working with the fi lter context
FIGURE 10-22 The year 2007 replaces the previous fi lter on the year.
When 2007 is selected, the results of the two measures are the same. However, when the year selected
is 2008, it gets replaced with 2007, whereas months are left untouched. The result is that the value shown
for January 2008 is the sales amount of January 2007. The same happens for February and March. The
main thing to note is that the original fi lter did not contain the fi rst three months of 2007, and by replacing the fi lter on the year, our formula shows their value. As anticipated, so far there is nothing special.
Things suddenly become much more intricate if one wants to compute the average monthly sales.
A possible solution to this calculation is to iterate over the months and average the partial results using

## Averagex:

Monthly Avg :=

## Averagex (

VALUES ( 'Date'[Month] ),
[Sales Amount]
)
You can see the result in Figure 10-23. This time, the grand total is surprisingly large.
FIGURE 10-23 The grand total is defi nitely not the average of months; it is too large.

CHAPTER 10 Working with the fi lter context 341
Understanding the problem is much harder than fi xing it. Focus your attention on the cell that computes the wrong value—the grand total of Monthly Avg. The fi lter context of the Total row of the report
is the following:

## Treatas (

{
( "CY 2007", "September" ),
( "CY 2007", "October" ),
( "CY 2007", "November" ),
( "CY 2007", "December" ),
( "CY 2008", "January" ),
( "CY 2008", "February" ),
( "CY 2008", "March" )
},
'Date'[Calendar Year],
'Date'[Month]
)
In order to follow the execution of the DAX code, we expand the full calculation in that cell by
defi ning the corresponding fi lter context in a CALCULATE statement that evaluates the Monthly Avg
measure. Moreover, we expand the code of Monthly Avg to build a single formula that simulates the
execution:

## Calculate (


## Averagex (

VALUES ( 'Date'[Month] ),

## Calculate (


## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
)
),

## Treatas (

{
( "CY 2007", "September" ),
( "CY 2007", "October" ),
( "CY 2007", "November" ),
( "CY 2007", "December" ),
( "CY 2008", "January" ),
( "CY 2008", "February" ),
( "CY 2008", "March" )
},
'Date'[Calendar Year],
'Date'[Month]
)
)
The key to fi xing the problem is to understand what happens when the highlighted CALCULATE is
executed. That CALCULATE is executed in a row context that is iterating over the Date[Month] column.
Consequently, context transition takes place and the current value of the month is added to the fi lter

342 CHAPTER 10 Working with the fi lter context
context. On a given month, say January, CALCULATE adds January to the fi lter context, replacing the
current fi lter on the month but leaving all the other fi lters untouched.
When AVERAGEX is iterating January, the resulting fi lter context is January in both 2007 and 2008;
this is because the original fi lter context fi lters two years for the year column. Therefore, on each iteration DAX computes the sales amount of one month in two distinct years. This is the reason why the
value is much higher than any monthly sales.
The original shape of the arbitrarily shaped fi lter is lost because CALCULATE overrides one of the
columns involved in the arbitrarily shaped fi lter. The net result is that the calculation produces an
incorrect result.
Fixing the problem is much easier than expected. Indeed, it is enough to iterate over a column that
is guaranteed to have a unique value on every month. If, instead of iterating over the month name,
which is not unique over different years, the formula iterates over the Calendar Year Month column,
then the code produces the correct result:
Monthly Avg :=

## Averagex (

VALUES ( 'Date'[Calendar Year Month] ),
[Sales Amount]
)
Using this version of Monthly Avg, on each iteration the context transition overrides the fi lter on
Calendar Year Month, which represents both year and month values in the same column. As a result, it
is guaranteed to always return the sales of an individual month, producing the correct outcome shown
in Figure 10-24.
FIGURE 10-24 Iterating over a unique column makes the code compute the correct result.
If a unique column for the cardinality of the iterator is not available, another viable solution is to
use KEEPFILTERS. The following alternative version of the code works correctly, because instead of

CHAPTER 10 Working with the fi lter context 343
replacing the previous fi lter, it adds the month fi lter to the previously existing arbitrarily shaped set;
this maintains the format of the original fi lter:
Monthly Avg KeepFilters :=

## Averagex (


```dax
KEEPFILTERS ( VALUES ( 'Date'[Month] ) ),
[Sales Amount]
)
```

As anticipated, arbitrarily shaped sets are not commonly observed in real-world reports. Nevertheless, users have multiple and legitimate ways of generating them. In order to guarantee that a measure
works correctly even in the presence of arbitrarily shaped sets, it is important to follow some best
practices:

> **Note:** When iterating over a column, make sure that the column has unique values at the granularity
where the calculation is being performed. For example, if a Date table has more than 12 months,
a YearMonth column should be used for monthly calculations.

> **Note:** If the previous best practice cannot be applied, then protect the code using KEEPFILTERS to
guarantee that the arbitrarily shaped fi lter is maintained in the fi lter context. Be mindful that
KEEPFILTERS might change the semantics of the calculations. Indeed, it is important to
double-check that KEEPFILTERS does not introduce errors in the measure.
Following these simple rules, your code will be safe even in the presence of arbitrarily shaped fi lters.
Conclusions
In this chapter we described several functions that are useful to inspect the content of the fi lter context
and/or to modify the behavior of a measure depending on the context. We also introduced important
techniques to manipulate the fi lter context with the increased knowledge of the possible states of the
fi lter context. Here is a recap of the important concepts you learned in this chapter:

> **Note:** A column can be either fi ltered or cross-fi ltered. It is fi ltered if there is a direct fi lter; it is crossfi ltered if the fi lter is coming from a direct fi lter on another column or table. You can verify
whether a column is fi ltered or not by using ISFILTERED and ISCROSSFILTERED.

> **Note:** HASONEVALUE checks whether a column only has one value visible in the fi lter context. This is
useful before retrieving that value using VALUES. The SELECTEDVALUE function simplifi es the
HASONEVALUE/VALUES pattern.

> **Note:** Using ALLEXCEPT is not the same as using the pair ALL and VALUES. In the presence of crossfi ltering, ALL/VALUES is safer because it also considers cross-fi ltering as part of its evaluation.

> **Note:** ALL and all the ALL* functions are useful to avoid the effect of context transition. Indeed, using
ALL in a calculated column, or in general in a row context, informs DAX that the context transition is not needed.

344 CHAPTER 10 Working with the fi lter context

> **Note:** Each column in a table is tagged with data lineage. Data lineage lets DAX apply fi lters and relationships. Data lineage is maintained whenever one references a column, whereas it is lost when
using expressions.

> **Note:** Data lineage can be assigned to one or more columns by using TREATAS.

> **Note:** Not all fi lters are simple fi lters. A user can build more complex fi lters either through the user
interface or by code. The most complex kind of fi lter is the arbitrarily shaped fi lter, which could
be complex to use because of its interaction with the CALCULATE function and the context
transition.
You will likely not remember all the concepts and functions described in this chapter immediately
after having read them. Regardless, it is crucial that you be exposed to these concepts in your learning
of DAX. You will for sure run into one of the issues described here as you gain DAX experience.
At that point, it will be useful to come back to this chapter and refresh your memory about the specifi c
problem you are dealing with.
In the next chapter, we use many of the functions described here to apply calculations over hierarchies. As you will learn, working with hierarchies is mainly a matter of understanding the shape of the
current fi lter context.