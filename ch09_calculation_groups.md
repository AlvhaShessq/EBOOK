# Chapter 9: Calculation groups

In 2019, DAX received a major update with the introduction of calculation groups. Calculation groups
are a utility feature inspired from a similar feature available in MDX, known as calculated members. If
you already know what calculated members in MDX are, then learning calculation groups should be
somewhat easier. However, the DAX implementation differs from the MDX implementation. Therefore,
regardless of your previous knowledge, in this chapter you will learn what calculation groups are, what
they were designed for, and how they can help build awesome calculations.
Calculation groups are easy to use; however, designing a model with calculation groups correctly
can be challenging when you create multiple calculation groups or when you use calculation items in
measures. For this reason, we provide best practices to help you avoid any issues. Deviating from these
best practices requires a deep understanding of how calculation groups are designed, if one wants to
obtain a sound model.
Calculation groups are a new feature in DAX, which, as of April 2019, has not been completed and
released. Along this chapter we highlight the parts that may change in the fi nal version of this feature.
Therefore, it is important to visit the web page https://www.sqlbi.com/calculation-groups, where you
will fi nd updated material and examples about calculation groups in DAX.
Introducing calculation groups
Before we provide a description of calculation groups, it is useful to spend some time analyzing the
business requirement that led to the introduction of this feature. Because you just fi nished digesting
the chapter about time intelligence, an example involving time-related calculations fi ts perfectly well.
In our sample model we defi ned calculations to compute the sales amount, the total cost, the margin, and the total quantity sold by using the following DAX code:
Sales Amount := SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
Total Cost := SUMX ( Sales, Sales[Quantity] * Sales[Unit Cost] )
Margin := [Sales Amount] - [Total Cost]
Sales Quantity := SUM ( Sales[Quantity] )
All four measures are useful, and they provide different insights into the business. Moreover, all four
measures are good candidates for time intelligence calculations. A year-to-date over sales quantity can
be as interesting as a year-to-date over sales amount and over margin. The same consideration is true
for many other time intelligence calculations: same period last year, growth in percentage against the
previous year, and many others.

280 CHAPTER 9 Calculation groups
Nevertheless, if one wants to build all the different time intelligence calculations for all the measures,
the number of measures in the data model may grow very quickly. In the real world, managing a data
model with hundreds of measures is intimidating for both users and developers. Finally, consider that all
the different measures for time intelligence calculations are simple variations of a common pattern. For
example, the year-to-date versions of the previous list of four measures would look like the following:
YTD Sales Amount :=

## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
)
YTD Total Cost :=

## Calculate (

[Total Cost],
DATESYTD ( 'Date'[Date] )
)
YTD Margin :=

## Calculate (

[Margin],
DATESYTD ( 'Date'[Date] )
)
YTD Sales Quantity :=

## Calculate (

[Sales Quantity],
DATESYTD ( 'Date'[Date] )
)
All the previous measures only differ in their base measure; they all apply the same DATESYTD fi lter
context to different base measures. It would be great if a developer were given the opportunity to
defi ne a more generic calculation, using a placeholder for the measure:
YTD <Measure> :=

## Calculate (

<Measure>,
DATESYTD ( 'Date'[Date] )
)
The previous code is not a valid DAX syntax, but it provides a very good description of what calculation items are. You can read the previous code as: When you need to apply the YTD calculation to a
measure, call the measure after applying DATESYTD to the Date[Date] column. This is what a calculation item is: A calculation item is a DAX expression containing a special placeholder. The placeholder is
replaced with a measure by the engine just before evaluating the result. In other words, a calculation
item is a variation of an expression that can be applied to any measure.
Moreover, a developer will likely fi nd themselves needing several time intelligence calculations. As
we noted at the beginning of the section, year-to-date, quarter-to-date, and same period last year are
all calculations that somehow belong to the same group of calculations. Therefore, DAX offers calculation items and calculation groups. A calculation group is a set of calculation items that are conveniently
grouped together because they are variations on the same topic.

CHAPTER 9 Calculation groups 281
Let us continue with DAX pseudo-code:
CALCULATION GROUP "Time Intelligence"
CALCULATION ITEM CY := <Measure>

```dax
CALCULATION ITEM PY := CALCULATE ( <Measure>, SAMPEPERIODLASTYEAR ( 'Date'[Date] ) )
CALCULATION ITEM QTD := CALCULATE ( <Measure>, DATESQTD ( 'Date'[Date] ) )
CALCULATION ITEM YTD := CALCULATE ( <Measure>, DATESYTD ( 'Date'[Date] ) )
```

As you can see, we grouped four time-related calculations in a group named Time Intelligence. In
only four lines, the code defi nes dozens of different measures because the calculation items apply their
variation to any measure in the model. Thus, as soon as a developer creates a new measure, the CY, PY,
QTD, and YTD variations will be available at no cost.
There are still several details missing in our understanding of calculation groups, but only one is
required to start taking advantage of them and to defi ne the fi rst calculation group: How does the
user choose one variation? As we said, a calculation item is not a measure; it is a variation of a measure.
Therefore, a user needs a way to put in a report a specifi c measure with one or more variations of the
measure itself. Because users have the habit of selecting columns from tables, calculation groups are
implemented as if they were columns in tables, whereas calculation items are like values of the given
columns. This way, the user can use the calculation group in the columns of a matrix to display different
variations of a measure in the report. For example, the calculation items previously described are applied
to the columns of the matrix in Figure 9-1, showing different variations of the Sales Amount measure.
FIGURE 9-1 The user can use a calculation group as if it were a column of the model, applying it to matrix columns.
Creating calculation groups
The implementation of calculation groups in a Tabular model depends on the user interface of the
editor tool. At the time of writing (April 2019), neither Power BI nor SQL Server Data Tools (SSDT) for
Analysis Services have a user interface for this feature, which is only available at the API level of Tabular
Object Model (TOM). The fi rst tool providing an editor for this feature is Tabular Editor, an open source
tool available for free at https://tabulareditor.github.io/.

282 CHAPTER 9 Calculation groups
In Tabular Editor, the Model / New Calculation Group menu item creates a new calculation group,
which appears as a table in the model with a special icon. This is shown in Figure 9-2 where the
calculation group has been renamed Time Intelligence.
FIGURE 9-2 Tabular Editor displays the Time Intelligence calculation group as a special table.
A calculation group is a special table with a single column, named Attribute by default in Tabular
Editor. In our sample model we renamed this column Time calc; then we added three items (YTD, QTD,
and SPLY for same period last year) by using the New Calculation Item context menu item available by
right-clicking on the Time calc column. Each calculation item has a DAX expression, as shown in Figure 9-3.
FIGURE 9-3 Every calculation item has a DAX expression that can be modifi ed in Tabular Editor.

CHAPTER 9 Calculation groups 283
The SELECTEDMEASURE function is the DAX implementation of the <Measure> placeholder we
used in the previous DAX pseudo-code. The DAX code for each calculation item is described in the
following code. The comment preceding each DAX expression identifi es the corresponding calculation
item:
Note It is best practice to always expose the business logic through measures in the
model. When the model includes calculation groups, the Power BI client does not allow
developers to aggregate columns because calculation groups can only be applied to measures, and they do not produce any effect on aggregation functions; they only operate on
measures.
--
-- Calculation Item: YTD
--

## Calculate (


## Selectedmeasure (),

DATESYTD ( 'Date'[Date] )
)
--
-- Calculation Item: QTD
--

## Calculate (


## Selectedmeasure (),

DATESQTD ( 'Date'[Date] )
)
--
-- Calculation Item: SPLY
--

## Calculate (


## Selectedmeasure (),

SAMEPERIODLASTYEAR ( 'Date'[Date] )
)
With this defi nition, the user will see a new table named Time Intelligence, with a column named
Time calc containing three values: YTD, QTD, and SPLY. The user can create a slicer on that column, or
use it on the rows and columns of visuals, as if it were a real column in the model. For example, when
the user selects YTD, the engine applies the YTD calculation item to whatever measure is in the report.
In Figure 9-4 you can see a matrix containing the Sales Amount measure. Because the slicer selects the
YTD variation of the measure, the numbers shown are year-to-date values.

284 CHAPTER 9 Calculation groups
FIGURE 9-4 When the user selects YTD, the values in the matrix represent the YTD variation of the Sales
Amount measure.
If on the same report the user selects SPLY, the result will be very different, as you can appreciate in
Figure 9-5.
FIGURE 9-5 Selecting SPLY changes the results of the Sales Amount measure because it now uses a different
variation. The values are the original Sales Amount values, shifted back one year.
If the user does not select one value or if the user selects multiple values together, then the engine
does not apply any variation to the original measure. You can see this in Figure 9-6.

CHAPTER 9 Calculation groups 285
FIGURE 9-6 When no calculation item is selected, the report shows the original measure.
Note The behavior of calculation groups with no selection or with multiple items selected
may change in the future. As of April 2019, when multiple calculation items are selected, the
behavior is the same as if there were no selection on a calculation group. Nevertheless, this
condition might return different results in future versions—for example, raising an error in
case of a multiple selection.
Calculation groups can go further than that. At the beginning of this section we introduced four different measures: Sales Amount, Total Cost, Margin, and Sales Quantity. It would be extremely nice if the
user could use a slicer in order to select the metric to show and not only the time intelligence calculation to apply. We would like to present a generic report that slices any of the four metrics by month
and year, letting the user choose the desired metric. In other words, we want to obtain the report in
Figure 9-7.
FIGURE 9-7 The report shows the YTD time intelligence calculation applied to Margin, but the user can choose any
other combination through the slicers.

286 CHAPTER 9 Calculation groups
In the example shown in Figure 9-7, the user is browsing the margin amount using a year-to-date
variation. Nevertheless, the user can choose any combination of the slicers linked to the two calculation
groups, Metric and Time calc.
In order to obtain this report, we created an additional calculation group named Metric, which
includes the Sales Amount, Total Cost, Margin, and Sales Quantity calculation items. The expression for each calculation item just evaluates the corresponding measure, as shown in Figure 9-8 for the
Sales Amount calculation item.
FIGURE 9-8 The Metric calculation group contains four calculation items; each one simply evaluates a corresponding measure.
When there are multiple calculation groups in the same data model, it is important to defi ne in
which order they should be applied by the DAX engine. The Precedence property of the calculation
group defi nes the order of application: the fi rst calculation group applied is the one with the larger
value. In order to obtain the desired result, we increased the Precedence property of the Time Intelligence calculation group to 10, as shown in Figure 9-9. As a consequence, the engine applies the Time
Intelligence calculation group before the Metric calculation group, which keeps the Precedence property at the default value of zero. We discuss the precedence of calculation groups in more detail later in
this chapter.

CHAPTER 9 Calculation groups 287
FIGURE 9-9 The Precedence property defi nes the order in which each calculation group is applied to a measure.
The following DAX code includes the defi nition of each calculation item in the Metric
calculation group:
--
-- Calculation Item: Margin
--
[Margin]
--
-- Calculation Item: Sales Amount
--
[Sales Amount]
--
-- Calculation Item: Sales Quantity
--
[Sales Quantity]
--
-- Calculation Item: Total Cost
--
[Total Cost]
These calculation items are not modifi ers of the original measure. Instead, they completely replace
the original measure with a new one. To obtain this behavior, we omitted a reference to SELECTEDMEASURE in the expression. SELECTEDMEASURE is used very often in calculation items, but it is not
mandatory.

288 CHAPTER 9 Calculation groups
This last example is useful to introduce the fi rst of the many complexities that we will need to
address with calculation groups. If the user selects Quantity, then the report shows the quantity, but it
still uses the same format strings (with two decimals) as the other measures. Because the Quantity measure is an integer, it would be useful to remove the decimal places or, in general, to adopt a different
format string. We discussed earlier the fact that the presence of multiple calculation groups in a calculation requires the defi nition of a precedence order, as was the case in the previous example. These are
the fi rst of several details to consider in order to create useful calculation groups.
Note If you are using Analysis Services, be mindful that adding a calculation group to a
model is an operation that requires a refresh of the table corresponding to the calculation
group in order to make the calculation items visible to the client. This may prove to be counterintuitive because deploying measures does not require such an update—measures are visible to the clients just after the deployment. However, because calculation groups and items
are presented to the client in tables and columns, after the deployment it is necessary to run
a refresh operation to populate the internal structures of the tables and columns. In Power BI
this operation will likely be handled automatically by the user interface—though this is pure
speculation because calculation groups are not present in Power BI at the time of printing.
Understanding calculation groups
In the previous sections we focused on the use of calculation groups and how to implement them with
Tabular Editor. In this section, we describe in more detail the properties and behavior of calculation
groups and calculation items.
There are two entities: calculation groups and calculation items. A calculation group is a collection of
calculation items, grouped together based on a user-defi ned criterion. For both calculation groups and
calculation items, there are properties that the developer must set correctly. We introduce these entities
and their properties here, providing more examples and details in the remaining part of this chapter.
A calculation group is a simple entity, defi ned by

> **Note:** The calculation group Name. This is the name of the table that represents the calculation group
on the client side.

> **Note:** The calculation group Precedence. When there are multiple active calculation groups, a number that defi nes the precedence used to apply each calculation group to a measure reference.

> **Note:** The calculation group attribute Name. This is the name of the column that includes the calculation items, displayed to the client as unique items available in the column.
A calculation item is a much more sophisticated entity, and here is the list of its properties:

> **Note:** The calculation item Name. This becomes one value of the calculation group column. Indeed, a
calculation item is like one row in the calculation group table.

CHAPTER 9 Calculation groups 289

> **Note:** The calculation item Expression. A DAX expression that might contain special functions like
SELECTEDMEASURE. This is the expression that defi nes how to apply the calculation item.

> **Note:** The sort order of the calculation item is defi ned by the Ordinal value. This property defi nes
how the different calculation items are sorted when presented to the user. It is very similar to
the sort-by-column feature of the data model. This feature is not available as of April 2019 but
should be implemented before calculation groups are released.

> **Note:** Format String. If not specifi ed, a calculation item inherits the format string of its base measure.
Nevertheless, if the modifi er changes the calculation, then it is possible to override the measure
format string with the format of the calculation item.
The Format String property is important in order to obtain a consistent behavior of the measures in
the model according to the calculation item being applied to them. For example, consider the following calculation group containing two calculation items for time intelligence: year-over-year (YOY) is
the difference between a selected period and the same period in the previous year; year-over-year
percentage (YOY%) is the percentage of YOY over the amount in the same period in the previous year:
--
-- Calculation Item: YOY
--

```dax
VAR CurrYear =
SELECTEDMEASURE ()
VAR PrevYear =
CALCULATE (
SELECTEDMEASURE (),
SAMEPERIODLASTYEAR ( 'Date'[Date] )
)
VAR Result =
CurrYear - PrevYear
RETURN Result
```

--
-- Calculation Item: YOY%
--

```dax
VAR CurrYear =
SELECTEDMEASURE ()
VAR PrevYear =
CALCULATE (
SELECTEDMEASURE (),
SAMEPERIODLASTYEAR ( 'Date'[Date] )
)
VAR Result =
DIVIDE (
CurrYear - PrevYear,
PrevYear
)
RETURN Result
```

290 CHAPTER 9 Calculation groups
The result produced by these two calculation items in a report is correct, but if the Format String
property does not override the default format string, then YOY% is displayed as a decimal number
instead of as a percentage, as shown in Figure 9-10.
FIGURE 9-10 The two calculation items YOY and YOY% share the same format as the Sales Amount measure.
The example shown in Figure 9-10 displays the YOY evaluation of the Sales Amount measure using
the same format string as the original Sales Amount measure. This is the correct behavior to display
a difference. However, the YOY% calculation item displays the same amount as a percentage of the
value of the previous year. The number shown is correct, but for January one would expect to see −12%
instead of −0.12. In this case the expected format string should be a percentage, regardless of the
format of the original measure. To obtain the desired behavior, set the Format String property of the
YOY% calculation item to percentage, overriding the behavior of the underlying measure. You can see
the result in Figure 9-11. If the Format String property is not assigned to a calculation item, the existing
format string is used.
FIGURE 9-11 The YOY% calculation item overrides the format of the Sales Amount measure displaying the value as
a percentage.

CHAPTER 9 Calculation groups 291
The format string can be defi ned using a fi xed format string or—in more complex scenarios—
by using a DAX expression that returns the format string. When one is writing a DAX expression, it
becomes possible to refer to the format string of the current measure using the SELECTEDMEASUREFORMATSTRING function, which returns the format string currently defi ned for the measure. For
example, if the model contains a measure that returns the currently selected currency and you want to
include the currency symbol as part of the format string, you can use this code to append the currency
symbol to the current format string:

```dax
SELECTEDMEASUREFORMATSTRING () & " " & [Selected Currency]
```

Customizing the format string of a calculation item is useful to preserve user experience consistency
when browsing the model. However, a careful developer should consider that the format string operates on any measure used with the calculation item. When there are multiple calculation groups in a
report, the result produced by these properties also depends on the calculation group precedence, as
explained in a later section of this chapter.
Understanding calculation item application
So far, the description we gave of how a calculation item works has never been extremely precise. The
reason is mainly educational: we wanted to introduce the concept of calculation items, without diving
too deep into details that might be distracting. Indeed, we said that calculation items can be applied
by the user using, for example, a slicer. A calculation item is applied by replacing measure references
invoked when there is a calculation item active in the fi lter context. In this scenario, the calculation item
rewrites the measure reference by applying the expression defi ned in the calculation item itself.
For example, consider the following calculation item:
--
-- Calculation Item: YTD
--

## Calculate (


## Selectedmeasure (),

DATESYTD ( 'Date'[Date] )
)
In order to apply the calculation item in an expression, you need to fi lter the calculation group. You
can create this fi lter using CALCULATE, like in the following example; this is the same technique used by
the client tool when using slicers and visuals:

## Calculate (

[Sales Amount],
'Time Intelligence'[Time calc] = "YTD"
)
There is nothing magical about calculation groups: They are tables, and as such they can be fi ltered
by CALCULATE like any other table. When CALCULATE applies a fi lter to a calculation item, DAX uses
the defi nition of the calculation item to rewrite the expression before evaluating it.

292 CHAPTER 9 Calculation groups
Therefore, based on the defi nition of the calculation item, the previous code is interpreted as
follows:

## Calculate (


## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
)
)
Note Inside the inner CALCULATE, one can check with ISFILTERED whether the calculation
item is fi ltered or not. In the example, we removed the outer fi lter on the calculation item for
the sake of simplicity, to show that the calculation item has already been applied. Nevertheless, a calculation item retains its fi lters, and further sub-expressions might still perform the
replacement of measures.
Despite being very intuitive in simple examples, this behavior hides some level of complexity. The
application of a calculation item replaces a measure reference with the expression of the calculation
item. Focus your attention on this last sentence: A measure reference is replaced. Without a measure
reference, a calculation item does not apply any modifi cation. For example, the following code is not
affected by any calculation item because it does not contain any measure reference:

## Calculate (

SUMX ( Sales, Sales[Quantity] * Sales[Net Price] ),
'Time Intelligence'[Time calc] = "YTD"
)
In this example, the calculation item does not perform any transformation because the code inside
CALCULATE does not use any measure. The following code is the one executed after the application of
the calculation item:

## Calculate (

SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
)
If the expression inside CALCULATE contains multiple measure references, all of them are replaced
with the calculation item defi nition. For example, the expression in the following Cost Ratio YTD measure contains two measure references, Total Cost and Sales Amount:

## Cr Ytd :=


## Calculate (


## Divide (

[Total Cost],
[Sales Amount]
),
'Time Intelligence'[Time calc] = "YTD"
)

CHAPTER 9 Calculation groups 293
To obtain the actual code executed, replace the measure references with the expansion of the
calculation item defi nition, as in the following CR YTD Actual Code measure:
CR YTD Actual Code :=

## Calculate (


## Divide (


## Calculate (

[Total Cost],
DATESYTD ( 'Date'[Date] )
),

## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
)
)
)
In this example, the code generated produces the same result as the next version in the CR YTD
Simplifi ed measure, which is more intuitive:
CR YTD Simplified :=

## Calculate (


## Calculate (


## Divide (

[Total Cost],
[Sales Amount]
),
DATESYTD ( 'Date'[Date] )
)
)
These three measures return the same result, as shown in Figure 9-12.
FIGURE 9-12 The CR YTD, CR YTD Actual Code, and CR YTD Simplifi ed measures produce the same result.

294 CHAPTER 9 Calculation groups
Nevertheless, you must be very careful because the CR YTD Simplifi ed measure does not correspond
to the actual code generated by the calculation item, which is the code in CR YTD Actual Code. In this
very special case, the two versions are equivalent. However, in more complex scenarios the difference
is signifi cant, and such a large difference can lead to unintended results that are extremely hard to follow and understand. Let us analyze a couple of examples. In the fi rst example the Sales YTD 2008 2009
measure has two nested CALCULATE functions: the outer CALCULATE sets a fi lter on the year 2008,
whereas the inner CALCULATE sets a fi lter on the year 2009:
Sales YTD 2008 2009 :=

## Calculate (


## Calculate (

[Sales Amount],
'Date'[Calendar Year] = "CY 2009"
),
'Time Intelligence'[Time calc] = "YTD",
'Date'[Calendar Year] = "CY 2008"
)
The outer CALCULATE fi lters the calculation item to the YTD value. Nevertheless, the application of
the calculation item does not change the expression because the expression does not directly contain
any measure. CALCULATE fi lters the calculation item, but its application does not lead to any modifi cations to the code.
Pay attention to the fact that the Sales Amount measure is within the scope of the inner CALCULATE.
The application of a calculation item modifi es the measures in the current scope of the fi lter context; it
does not affect nested fi lter context scopes. Those are handled by their own CALCULATE—or equivalent code, such as CALCULATETABLE or context transitions—which may or may not retain the same
fi lter on the calculation item.
When the inner CALCULATE applies its fi lter context, it does not change the fi lter status of the calculation item. Therefore, the engine fi nds that the calculation item is still fi ltered, and it remains fi ltered if
no other CALCULATE changes it. Same as if it were a regular column. The inner CALCULATE contains a
measure reference, and DAX performs the application of the calculation item. The resulting code corresponds to the defi nition of the Sales YTD 2008 2009 Actual Code measure:
Sales YTD 2008 2009 Actual Code :=

## Calculate (


## Calculate (


## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
),
'Date'[Calendar Year] = "CY 2009"
),
'Date'[Calendar Year] = "CY 2008"
)
The result of these two measures is visible in Figure 9-13. The selection made by the slicer on the left
applies to the matrix in the middle of the fi gure, which includes the Sales YTD 2008 2009 and Sales YTD

CHAPTER 9 Calculation groups 295
2008 2009 Actual Code measures. However, the selection of the year CY 2008 is overridden by CY 2009.
This can be verifi ed by looking at the matrix on the right-hand side, which shows the Sales Amount
measure transformed with the YTD calculation item for the CY 2008 and CY 2009 years. The numbers
in the center matrix correspond to the CY 2009 column of the matrix on the right.
FIGURE 9-13 The Sales YTD 2008 2009 and Sales YTD 2008 2009 Actual Code measures produce the same result.
The DATESYTD function is applied when the fi lter context is fi ltering the year 2009, not 2008.
Despite the calculation item being fi ltered along with the fi lter for the year 2008, its actual application
took place in a different fi lter context, namely the inner fi lter context. The behavior is counterintuitive
to say the least. The more complex the expression used inside CALCULATE, the harder it becomes to
understand how the application works.
The behavior of calculation items leads to one very important best practice: You need to use
calculation items to modify an expression if and only if this expression is a single measure. The previous example was only useful to introduce the rule; let us now analyze the best practice with a more
complex expression. The next expression computes the number of working days only for the months
where there are sales:

## Sumx (

VALUES ( 'Date'[Calendar Year month] ),

## If (

[Sales Amount] > 0, -- Measure reference
[# Working Days]    -- Measure reference
)
)
This calculation is useful to compute Sales Amount per working day considering only the months
with sales. The following example uses this calculation in a more complex expression:

## Divide (

[Sales Amount],  -- Measure reference

## Sumx (

VALUES ( 'Date'[Calendar Year month] ),

## If (


296 CHAPTER 9 Calculation groups
[Sales Amount] > 0, -- Measure reference
[# Working Days]    -- Measure reference
)
)
)
If this expression is executed within an outer CALCULATE that changes the calculation to a YTD, the
result is the following new formula that produces an unexpected result:
Sales WD YTD 2008 :=

## Calculate (


## Divide (

[Sales Amount],  -- Measure reference

## Sumx (

VALUES ( 'Date'[Calendar Year month] ),

## If (

[Sales Amount] > 0, -- Measure reference
[# Working Days]    -- Measure reference
)
)
),
'Time Intelligence'[Time calc] = "YTD",
'Date'[Calendar Year] = "CY 2008"
)
Intuitively, one would expect the previous expression to compute the Sales Amount measure per
working days considering all the months before the current one. In other words, one would expect this
code to be executed:
Sales WD YTD 2008 Expected Code :=

## Calculate (


## Calculate (


## Divide (

[Sales Amount],  -- Measure reference

## Sumx (

VALUES ( 'Date'[Calendar Year month] ),

## If (

[Sales Amount] > 0, -- Measure reference
[# Working Days]    -- Measure reference
)
)
) ,
DATESYTD ( 'Date'[Date] )
),
'Date'[Calendar Year] = "CY 2008"
)
Nevertheless, you might have noticed that we have highlighted the three measure references with a
few comments. This was not by chance. The application of a calculation item happens on the measure

CHAPTER 9 Calculation groups 297
references, not on the entire expression. Therefore, the code executed by replacing the measure references with the calculation items active in the fi lter context is very different:
Sales WD YTD 2008 Actual Code :=

## Calculate (


## Divide (


## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
),

## Sumx (

VALUES ( 'Date'[Calendar Year month] ),

## If (


## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )

## ) > 0,


## Calculate (

[# Working Days],
DATESYTD ( 'Date'[Date] )
)
)
)
),
'Date'[Calendar Year] = "CY 2008"
)
This latter version of the code produces an abnormal value for the number of working days because
it sums the year-to-date of the number of working days for all the months visible in the current context. The chances of producing an inaccurate result are extremely high. When an individual month
is selected, the result (by pure luck) is the right one, whereas at the quarter and at the year levels it is
hilariously wrong. This is shown in Figure 9-14.
FIGURE 9-14 Different versions of the Sales WD calculation computed for the all the quarters of 2008.
The Sales WD YTD 2008 Expected Code measure returns the correct number for every quarter,
whereas the Sales WD YTD 2008 and Sales WD YTD 2008 Actual Code measures return a smaller value.
Indeed, the number of working days in the denominator of the ratio is computed as the sum of the
year-to-date number of working days for each month in the period.

298 CHAPTER 9 Calculation groups
You can easily avoid this complexity by obeying the best practice: Use CALCULATE with calculation
items only to invoke an individual measure. When one authors the Sales WD YTD 2008 Fixed measure
that includes the full expression and uses the Sales WD YTD 2008 Fixed measure in a single CALCULATE
function, the code is very different and easier to use:
--
-- Measure Sales WD
--
Sales WD :=

## Divide (

[Sales Amount],

## Sumx (

VALUES ( 'Date'[Calendar Year month] ),

## If (

[Sales Amount] > 0,
[# Working Days]
)
)
)
--
-- Measure Sales WD YTD 2008 Fixed
-- New version of the Sales WD YTD 2008 measure that applies the YTD calculation item
--
Sales WD YTD 2008 Fixed :=

## Calculate (

[Sales WD], -- Measure reference
'Time Intelligence'[Time calc] = "YTD",
'Date'[Calendar Year] = "CY 2008"
)
In this case, the code generated by the application of the calculation item is much more intuitive:
Sales WD YTD 2008 Fixed Actual Code :=

## Calculate (


## Calculate (

[Sales WD],
DATESYTD ( 'Date'[Date] )
),
'Date'[Calendar Year] = "CY 2008"
)
In this latter example the fi lter provided by DATESYTD surrounds the entire expression, leading to
the code that one intuitively expects from the application of the calculation item. The result of the Sales
WD YTD 2008 Fixed and Sales WD YTD 2008 Fixed Actual Code measures is visible in Figure 9-14.
For very simple calculations containing simple expressions, it is possible to deviate from this best
practice. However, when doing so, the developer must always think twice before creating any measure, because as soon as the complexity of the expression is no longer trivial, the chances of producing
wrong calculations become very high.

CHAPTER 9 Calculation groups 299
When using client tools like Power BI, you never have to worry about these details. Indeed, these
tools make sure that calculation items get applied the right way because they always invoke single
measures as part of the query they execute. Nevertheless, as a DAX developer, you will end up using
calculation items as fi lters in CALCULATE. When you do that, pay attention to the expression used in
CALCULATE. If you want to stay on the safe side, use calculation items in CALCULATE to modify a single
measure. Never apply calculation items to an expression.
Finally, we suggest you learn calculation items by rewriting the expression manually, applying the
calculation item, and writing down the complete code that will be executed. It is a mental exercise that
proves very useful in understanding exactly what is happening inside the engine.
Understanding calculation group precedence
In the previous section we described how to use CALCULATE to apply a calculation item to a measure.
It is possible to apply multiple calculation items to the same measure. Even though each calculation
group can only have one active calculation item, the presence of multiple calculation groups can
activate multiple calculation items at the same time. This happens when a user uses multiple slicers
over different calculation groups, or when a CALCULATE function fi lters calculation items in different
calculation groups. For example, at the beginning of this chapter we defi ned two calculation groups:
one to defi ne the base measure and the other to defi ne the time intelligence calculation to apply to the
base measure.
If there are multiple calculation items active in the current fi lter context, it is important to defi ne
which calculation item is applied fi rst, by defi ning a set of precedence rules. DAX enforces this by making
it mandatory to set the Precedence property in a calculation group, in models that have more than one
calculation group. This section describes how to correctly set the Precedence property of a calculation
group through examples where the defi nition of the precedence changes the result of the calculations.
To prepare the demonstration, we created two different calculation groups, each one containing
only one calculation item:
-------------------------------------------------------
-- Calculation Group: 'Time Intelligence'[Time calc]
-------------------------------------------------------
--
-- Calculation Item: YTD
--

## Calculate (


## Selectedmeasure (),

DATESYTD ( 'Date'[Date] )
)

300 CHAPTER 9 Calculation groups
-------------------------------------------------------
-- Calculation Group: 'Averages'[Averages]
-------------------------------------------------------
--
-- Calculation Item: Daily AVG
--

## Divide (


## Selectedmeasure (),

COUNTROWS ( 'Date' )
)
YTD is a regular year-to-date calculation, whereas Daily AVG computes the daily average by
dividing the selected measure by the number of days in the fi lter context. Both calculation items work
just fi ne, as shown in Figure 9-15, where we use two measures to invoke the two calculation items
individually:

## Ytd :=


## Calculate (

[Sales Amount],
'Time Aggregation'[Aggregation] = "YTD"
)
Daily AVG :=

## Calculate (

[Sales Amount],
'Averages'[Averages] = "Daily AVG"
)
FIGURE 9-15 Both Daily AVG and YTD calculation items work just fi ne when invoked individually in separate
measures.
The scenario suddenly becomes more complex when both calculation items are used at the same
time. Look at the following Daily YTD AVG measure defi nition:

CHAPTER 9 Calculation groups 301
Daily YTD AVG :=

## Calculate (

[Sales Amount],
'Time Intelligence'[Time calc] = "YTD",
'Averages'[Averages] = "Daily AVG"
)
The measure invokes both calculation items at the same time, but this raises the issue of precedence.
Should the engine apply YTD fi rst and Daily AVG later, or the other way around? In other words, which
of these two expressions should be evaluated?
--
-- YTD is applied first, and then DIVIDE
--

## Divide (


## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
),
COUNTROWS ( 'Date' )
)
--
-- DIVIDE is applied first, and then YTD
--

## Calculate (


## Divide (

[Sales Amount],
COUNTROWS ( 'Date' )
),
DATESYTD ( 'Date'[Date] )
)
It is likely that the second expression is the correct one. Nevertheless, without further information,
DAX cannot choose between the two. Therefore, the developer must defi ne the correct order of
application of the calculation groups.
The order of application depends on the Precedence property in the two calculation groups: The
calculation group with the highest value is applied fi rst; then the other calculation groups are applied
according to their Precedence value in a descending order. Figure 9-16 shows the wrong result produced with the following settings:

> **Note:** Time Intelligence calculation group—Precedence: 0

> **Note:** Averages calculation group—Precedence: 10

302 CHAPTER 9 Calculation groups
FIGURE 9-16 The Daily YTD AVG measure does not produce an accurate result.
The value of the Daily YTD AVG is clearly wrong in all the months displayed but January. Let us analyze what happened in more depth. Averages has a precedence of 10; therefore, it is applied fi rst. The
application of the Daily AVG calculation item leads to this expression corresponding to the Daily YTD
AVG measure reference:

## Calculate (


## Divide (

[Sales Amount],
COUNTROWS ( 'Date' )
),
'Time Intelligence'[Time calc] = "YTD"
)
At this point, DAX activates the YTD calculation item from the Time Intelligence calculation group.
The application of YTD rewrites the only measure reference in the formula, which is Sales Amount.
Therefore, the fi nal code corresponding to the Daily YTD AVG measure becomes the following:

## Divide (


## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
),
COUNTROWS ( 'Date' )
)
Consequently, the number shown is obtained by dividing the Sales Amount measure computed
using the YTD calculation item, by the number of days in the displayed month. For example, the value
shown in December is obtained by dividing 9,353,814,87 (YTD of Sales Amount) by 31 (the number of
days in December). The number should be much lower because the YTD variation should be applied to
both the numerator and the denominator of the DIVIDE function used in the Daily AVG
calculation item.

CHAPTER 9 Calculation groups 303
To solve the issue, the YTD calculation item must be applied before Daily AVG. This way, the
transformation of the fi lter context for the Date column occurs before the evaluation of COUNTROWS
over the Date table. In order to obtain this, we modify the Precedence property of the Time Intelligence
calculation group to 20, obtaining the following settings:

> **Note:** Time Intelligence calculation group—Precedence: 20

> **Note:** Averages calculation group—Precedence: 10
Using these settings, the Daily YTD AVG measure returns the correct values, as shown in Figure 9-17.
FIGURE 9-17 The Daily YTD AVG measure produces the right result.
This time, the two application steps are the following: DAX fi rst applies the YTD calculation from the
Time Intelligence calculation group, changing the expression to the following:

## Calculate (


## Calculate (

[Sales Amount],
DATESYTD ( 'Date'[Date] )
),
'Averages'[Averages] = "Daily AVG"
)
Then, DAX applies the Daily AVG calculation item from the Averages calculation group, replacing
the measure reference with the DIVIDE function and obtaining the following expression:

## Calculate (


## Divide (

[Sales Amount],
COUNTROWS ( 'Date' )
),
DATESYTD ( 'Date'[Date] )
)

304 CHAPTER 9 Calculation groups
The value displayed in December now considers 365 days in the denominator of DIVIDE, thus
obtaining the correct number. Before moving further, please consider that, in this example, we followed the best practice of using calculation items with a single measure. Indeed, the fi rst call comes
from the visual of Power BI. However, one of the two calculation items rewrote the Sales Amount measure in such a way that the problem arose. In this scenario, following the best practices is not enough.
It is mandatory that a developer understand and defi ne the precedence of application of calculation
groups very well.
All calculation items in a calculation group share the same precedence. It is impossible to defi ne different precedence values for different calculation items within the same group.
The Precedence property is an integer value assigned to a calculation group. A higher value means
a higher precedence of application; the calculation group with the higher precedence is applied fi rst.
In other words, DAX applies the calculation groups according to their Precedence value sorted in a
descending order. The absolute value assigned to Precedence does not mean anything. What matters
is how it compares with the Precedence of other calculation groups. There cannot be two calculation
groups in a model with the same Precedence.
Because assigning different Precedence values to multiple calculation groups is mandatory, you
must pay attention making this choice when you design a model. Choosing the right Precedence
upfront is important because changing the Precedence of a calculation group might affect the existing reports of a model already deployed in production. When you have multiple calculation groups
in a model, you should always spend time verifying that the results of the calculations are the results
expected with any combination of calculation items. The chances of making mistakes in the defi nition
of the precedence values is quite high without proper testing and validation.
Including and excluding measures from calculation items
There are scenarios where a calculation item implements a variation that does not make sense on all
the measures. By default, a calculation item applies its effects on all the measures. Nevertheless, the
developer might want to restrict which measures are affected by a calculation item.
One can write conditions in DAX that analyze the current measure evaluated in the model by using
either ISSELECTEDMEASURE or SELECTEDMEASURENAME. For example, consider the requirement of
restricting the measures affected by the Daily AVG calculation item so that a measure computing a
percentage is not transformed into a daily average. The ISSELECTEDMEASURE function returns True
if the measure evaluated by SELECTEDMEASURE is included in the list of measures specifi ed in the
arguments:
-------------------------------------------------------
-- Calculation Group: 'Averages'[Averages]
-------------------------------------------------------
--
-- Calculation Item: Daily AVG
--

CHAPTER 9 Calculation groups 305

## If (


## Isselectedmeasure (

[Sales Amount],
[Gross Amount],
[Discount Amount],
[Sales Quantity],
[Total Cost],
[Margin]
),

## Divide (


## Selectedmeasure (),

COUNTROWS ( 'Date' )
)
)
As you can see, the code specifi es the measure on which to compute the daily average, returning blank
when the Daily AVG calculation item is applied to any other measure. Now if the requirement is just to
exclude specifi c measures, including any other measure by default, the code can be written this way:
-------------------------------------------------------
-- Calculation Group: 'Averages'[Averages]
-------------------------------------------------------
--
-- Calculation Item: Daily AVG
--

## If (


```dax
NOT ISSELECTEDMEASURE ( [Margin %] ),
DIVIDE (
SELECTEDMEASURE (),
COUNTROWS ( 'Date' )
)
)
```

In both cases the Daily AVG calculation item excludes the calculation for the Margin % measure, as
shown in Figure 9-18.
FIGURE 9-18 The Daily AVG calculation item is not applied to Margin %.

306 CHAPTER 9 Calculation groups
Another function that can be used to analyze the selected measure in a calculation item expression
is SELECTEDMEASURENAME, which returns a string instead of a Boolean value. This function may be
used instead of ISSELECTEDMEASURE, as in the following example:
-------------------------------------------------------
-- Calculation Group: 'Averages'[Averages]
-------------------------------------------------------
--
-- Calculation Item: Daily AVG
--

## If (


```dax
NOT ( SELECTEDMEASURENAME () = "Margin %" ),
DIVIDE (
SELECTEDMEASURE (),
COUNTROWS ( 'Date' )
)
)
The result would be the same, but the ISSELECTEDMEASURE solution is preferable for several
```

reasons:

> **Note:** If the measure name is misspelled using a comparison with SELECTEDMEASURENAME, the DAX
code simply return False without raising an error.

> **Note:** If the measure name is misspelled using ISSELECTEDMEASURE, the expression fails with the
error Invalid input arguments for ISSELECTEDMEASURE.

> **Note:** If a measure is renamed in the model, all the expressions using ISSELECTEDMEASURE are
automatically renamed in the model editor (formula fi xup), whereas the strings compared to
SELECTEDMEASURENAME must be updated manually.
The SELECTEDMEASURENAME function should be considered when the business logic of a calculation item must apply a transformation based on an external confi guration. For example, the function
might be useful when there is a table with a list of measures that should enable a behavior in a calculation item so that the model has an external confi guration that can be modifi ed without requiring an
update of the DAX code.
Understanding sideways recursion
DAX calculation items do not provide full recursion. However, there is a limited form of recursion available, which is called sideways recursion. We describe this complex topic through examples. Let us start
by understanding what recursion is and why it is important to discuss it. Recursion might occur when a
calculation item refers to itself, leading to an infi nite loop in the application of calculation items. Let us
elaborate on this.

CHAPTER 9 Calculation groups 307
Consider a Time Intelligence calculation group with two calculation items defi ned as follows:
-------------------------------------------------------
-- Calculation Group: 'Time Intelligence'[Time calc]
-------------------------------------------------------
--
-- Calculation Item: YTD
--

## Calculate (


## Selectedmeasure (),

DATESYTD ( 'Date'[Date] )
)
--
-- Calculation Item: SPLY
--

## Calculate (


## Selectedmeasure (),

SAMEPERIODLASTYEAR ( 'Date'[Date] )
)
The requirement is to add a third calculation item that computes the year-to-date in the previous
year (PYTD). As you learned in Chapter 8, “Time intelligence calculations,” this can be obtained by mixing two time intelligence functions: DATESYTD and SAMEPERIODLASTYEAR. The following calculation
item solves the scenario:
--
-- Calculation Item: PYTD
--

## Calculate (


## Selectedmeasure (),

DATESYTD ( SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
)
Given the simplicity of the calculation, this solution is already optimal. Nevertheless, as a mind challenge we can try to author the same code in a different way. Indeed, there already is a YTD calculation
item that computes the year-to-date in place; therefore, one could think of using the calculation item
instead of mixing time intelligence calculations within the same formula. Look at the following defi nition of the same PYTD calculation item:
--
-- Calculation Item: PYTD
--

## Calculate (


## Selectedmeasure (),

SAMEPERIODLASTYEAR ( 'Date'[Date] ),
'Time Intelligence'[Time calc] = "YTD"
)

308 CHAPTER 9 Calculation groups
The calculation item achieves the same result as the previous defi nition, but using a different technique. SAMEPERIODLASTYEAR moves the fi lter context back to the previous year, while the year-todate calculation is obtained by applying an existing calculation item in the Time calc calculation group:
YTD. As previously noted, in this example the code is less readable and needlessly more complex. That
said, you can easily imagine that in a more complex scenario the ability to invoke previously defi ned
calculation items might come very handy—to avoid repeating the same code multiple times in your
measures.
This is a powerful mechanism to defi ne complex calculations. It comes with some level of complexity
that needs to be well understood: recursion. As you have seen in the PYTD calculation item, it is possible to defi ne a calculation item based on another calculation item from the same calculation group. In
other words, inside a calculation group certain items can be defi ned in terms of other items of the same
calculation group. If the feature were available without any restriction, this would lead to extremely
complex situations where calculation item A depends on B, which depends on C, which in turn can
depend on A. The following fi ctitious example demonstrates the issue:
-------------------------------------------------------
-- Calculation Group: Infinite[Loop]
-------------------------------------------------------
--
-- Calculation Item: Loop A
--

## Calculate (


## Selectedmeasure (),

Infinite[Loop] = "Loop B"
)
--
-- Calculation Item: Loop B
--

## Calculate (


## Selectedmeasure (),

Infinite[Loop] = "Loop A"
)
If used in an expression like in the following example, DAX would not be able to apply the
calculation items, because A requires the application of B, which in turn requires A, and so on:

## Calculate (

[Sales Amount],
Infinite[Loop] = "Loop A"
)
Some programming languages allow similar circular dependencies to be used in the defi nition of
expressions—typically in functions—leading to recursive defi nitions. A recursive function defi nition is a
defi nition where the function is defi ned in terms of itself. Recursion is extremely powerful, but it is also
extremely complex for developers writing code and for the optimizer looking for the best execution
path.

CHAPTER 9 Calculation groups 309
For these reasons, DAX does not allow the defi nition of recursive calculation items. In DAX, a developer can reference another calculation item of the same calculation group, but without referencing
the same calculation item twice. In other words, it is possible to use CALCULATE to invoke a calculation
item, but the calculation item invoked cannot directly or indirectly invoke the original calculation item.
This feature is called sideways recursion. Its goal is not to implement full recursion; Instead, it aims at
reusing complex calculation items without providing the full power (and complexity) of recursion.
Note If you are familiar with the MDX language, you should be aware that MDX supports
both sideways recursion and full recursion. These capabilities are part of the reasons MDX
is more complex a language than DAX. Moreover, full recursion oftentimes leads to bad
performance. For these reasons, DAX does not support full recursion by design.
Be mindful that recursion might also occur because a measure sets a fi lter on a calculation item,
not only between calculation items. For example, consider the following defi nitions of measures (Sales
Amount, MA, MB) and calculation items (A and B):
--
-- Measures definition
--
Sales Amount := SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )

```dax
MA := CALCULATE ( [Sales Amount], Infinite[Loop] = "A" )
MB := CALCULATE ( [Sales Amount], Infinite[Loop] = "B" )
```

-------------------------------------------------------
-- Calculation Group: Infinite[Loop]
-------------------------------------------------------
--
-- Calculation Item: A
--

## [Mb]

--
-- Calculation Item: B
--

## [Ma]

The calculation items do not reference each other. Instead, they reference a measure that, in turn,
references the calculation items, generating an infi nite loop. We can see this happening by following
the calculation item application step by step. Consider the following expression:

## Calculate (

[Sales Amount],
Infinite[Loop] = "A"
)

310 CHAPTER 9 Calculation groups
The application of calculation item A produces the following result:

## Calculate (


## Calculate ( [Mb] )

)
However, the MB measure internally references both Sales Amount and calculation item B; it
corresponds to the following code:

## Calculate (


## Calculate (


## Calculate (

[Sales Amount],
Infinite[Loop] = "B"
)
)
)
At this point, the application of calculation item B produces the following result:

## Calculate (


## Calculate (


## Calculate (


## Calculate ( [Ma] )

)
)
)
Again, the MA measure internally references Sales Amount and calculation item A, and corresponds
to the following code:

## Calculate (


## Calculate (


## Calculate (


## Calculate (


## Calculate (

[Sales Amount],
Infinite[Loop] = "A"
)
)
)
)
)
Now we are back to the initial expression and we potentially enter into an infi nite loop of calculation
items applied to the expression—although the calculation items do not reference each other. Instead,
they reference a measure that, in turn, references the calculation items. The engine is smart enough to
detect that, in this case, an infi nite loop is present. Therefore, DAX throws an error.

CHAPTER 9 Calculation groups 311
Sideways recursion can lead to very complex expressions that are hard to read and likely to produce
unexpected results. Most of the complexity of calculation items with sideways recursion is seen when
there are measures that internally apply calculation items with CALCULATE—all the while users change
the calculation item through the user interface of the tool, like using a slicer in Power BI.
Our suggestion is to limit the use of sideways recursion in your code as much as you can, though
this might mean repeating the same code in multiple places. Only in hidden calculation groups can you
safely rely on sideways recursion, so that they can be managed by code but not by users. Keep in mind
that Power BI users can defi ne their own measures in a report, and, unaware of a complex topic like
recursion, they might generate errors without properly understanding the reason.
Using the best practices
As we said in the introduction, there are only two best practices to follow to avoid encountering issues
with calculation items:

> **Note:** Use calculation items to modify the behavior of expressions consisting of one measure only.
Never use calculation items to change the behavior of more complex expressions.
--
-- This is a BEST PRACTICE
--
SalesPerWd :=

## Calculate (

[Sales Amount], -- Single measure. This is good
'Time Intelligence'[Time calc] = "YTD"
)
--
-- This is BAD PRACTICE – do not do this!
--
SalesPerWd :=

## Calculate (

SUMX ( Customer, [Sales Amount] ), -- Complex expression, it is not a single
'Time Intelligence'[Time calc] = "YTD" -- measure reference
)

> **Note:** Avoid using sideways recursion in any calculation group that remains public and available to
users. You can safely use sideways recursion in hidden calculation groups. Still, if you use sideways recursion, pay attention not to introduce full recursion, which would produce an error as a
result.
Conclusions
Calculation groups are an extremely powerful tool to simplify the building of complex models. By
letting the developer defi ne variations of measures, calculation groups provide a very compact way of
generating hundreds of measures without duplicating code. Moreover, users love calculation groups
because they have the option of creating their own combination of calculations.

312 CHAPTER 9 Calculation groups
As a DAX developer, you should understand their power and their limitations. These are the lessons
included in this chapter:

> **Note:** Calculation groups are sets of calculation items.

> **Note:** Calculation items are variations of a measure. By using the SELECTEDMEASURE function, calculation items have the option of changing the way a calculation goes.

> **Note:** A calculation item can override the expression and the format string of the current measure.

> **Note:** If multiple calculation groups are being used in a model, the developer must defi ne the order of
application of the calculation items to disambiguate their behavior.

> **Note:** Calculation items are applied to measure references, not to expressions. Using a calculation
item to change the behavior of an expression not consisting of a single measure reference is
likely to produce unexpected results. Therefore, it is a best practice to only apply calculation
items to expressions made up of a single measure reference.

> **Note:** A developer can use sideways recursion in the defi nition of a calculation item, but this suddenly
increases the complexity of the whole expression. The developer should limit the use of sideways recursion to hidden calculation groups and avoid sideways recursion in calculation groups
that are visible to users.

> **Note:** Following the best practices is the easiest way to avoid the complexity involved in calculation
groups.
Finally, keep in mind that calculation groups are a very recent addition to the DAX language. This is
a very powerful feature, and we just started discovering the many uses of calculation groups. We will
update the web page mentioned in the introduction of this chapter with references to new articles and
blog posts where you can continue to learn about calculation groups.