# Chapter 20: Optimizing DAX

This is the last chapter of the book, and it is time to use all the knowledge you have gained so far to
explore the most fascinating DAX topic: optimizing formulas. You have learned how the DAX engines
work, how to read a query plan, and the internals of the formula engine and of the storage engine.
Now all the pieces are in place and you are ready to learn how to use that information to write
faster code.
There is one very important warning before approaching this chapter. Do not expect to learn best
practices or a simple way to write fast code. Simply stated: There is no way in DAX to write code that
is always the fastest. The speed of a DAX formula depends on many factors, the most important of
which unfortunately is not in the DAX code itself: It is data distribution. You have already learned that
VertiPaq compression strongly depends on data distribution. The size of a column (hence, the speed
to scan it) depends on its cardinality: the larger, the slower. Thus, the very same formula might behave
differently when executed on one column or another.
You will learn how to measure the speed of a formula, and we will provide you with several examples
where rewriting the expression differently leads to a faster execution time. Learn all these examples for
what they are—examples that might help you in fi nding new ideas for your code. Do not take them as
golden rules, because they are not.
We are not teaching you rules; we are trying to teach you how to fi nd the best rules in the very
specifi c scenario that is your data model. Be prepared to change them when the data model changes
or when you approach a new scenario. Flexibility is key when optimizing DAX code: fl exibility, a deep
technical knowledge of the engine, and a good amount of creativity, to be prepared to test formulas
and expressions that might be not so intuitive.
Finally, all the information we provide in this book is valid at the time of printing. New versions of
the engine come on the market every month, and the development team is always working on improving the DAX engine. So be prepared to measure different numbers for the examples of the book in
the version of the engine you will be running and be prepared to use different optimization methods
if necessary. If one day you measure your code and reach the educated conclusion that “Marco and
Alberto are wrong; this code runs much faster than their suggested code,” that will be our brightest
day, because we will have been able to teach you all that we know, and you are moving forward in
writing better DAX code than ours.

658 CHAPTER 20 Optimizing DAX
Defi ning optimization strategies
The optimization process for a DAX query, expression, or measure requires a strategy to reproduce a
performance issue, identify the bottleneck, and remove it. Initially, you always observe a slowness in
a complex query, but optimizing a complicated expression including several DAX measures is more
involved than optimizing one measure at a time. For this reason, the approach we suggest is to isolate
the slowest measure or expression fi rst, and optimize it in a simpler query that reproduces the issue
with a shorter query plan.
This is a simple to-do list you should follow every time you want to optimize DAX:
1. Identify a single DAX expression to optimize.
2. Create a query that reproduces the issue.
3. Analyze server timings and query plan information.
4. Identify bottlenecks in the storage engine or formula engine.
5. Implement changes and rerun the test query.
You can see a more complete description of each of these steps in the following sections.
Identifying a single DAX expression to optimize
If you have already found the slowest measure in your model, you probably can skip this section and
move to the following one. However, it is common to get a performance issue in a report that might
generate several queries. Each of these queries might include several measures. The fi rst step is to
identify a single DAX expression to optimize. Doing this, you reduce the reproduction steps to a single
query and possibly to a single measure returned in the result.
A complete refresh of a report in Power BI or Reporting Services or of a Microsoft Excel workbook
typically generates several queries in either DAX or MDX (PivotTables and charts in Excel always generate the latter). When a report generates several queries, you have to identify the slowest query fi rst.
In Chapter 19, “Analyzing DAX query plans,” you saw how DAX Studio can intercept all the queries sent
to the DAX engine and identify the slowest query looking at the largest Duration amount.
If you are using Excel, you can also use a different technique to isolate a query. You can extract
the MDX query it generates by using OLAP PivotTable Extensions, a free Excel add-in available at
https://olappivottableextensions.github.io/.
Once you extract the slowest DAX or MDX query, you have to further restrict your focus and isolate
the DAX expression that is causing the slowness. This way, you will concentrate your efforts on the right
area. You can reduce the measures included in a query by modifying and executing the query interactively in DAX Studio.
For example, consider the following table result in Power BI with four expressions (two distinct
counts and two measures) grouped by product brand, as shown in Figure 20-1.

CHAPTER 20 Optimizing DAX 659
FIGURE 20-1 Simple visualization in Power BI generated by a DAX query with four expressions.
The report generates the following DAX query, captured by using DAX Studio:

## Evaluate


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),

```dax
"DistinctCountProductKey", CALCULATE (
DISTINCTCOUNT ( 'Product'[ProductKey] )
),
"Sales_Amount", 'Sales'[Sales Amount],
"Margin__", 'Sales'[Margin %],
"DistinctCountOrder_Number", CALCULATE (
DISTINCTCOUNT ( 'Sales'[Order Number] )
)
),
[IsGrandTotalRowTotal], 0,
'Product'[Brand], 1
)
```


## Order By

[IsGrandTotalRowTotal] DESC,
'Product'[Brand]
You should reduce the query by trying one calculation at a time, to locate the slowest one. If you can
manipulate the report, you might just include one calculation at a time. By accessing the DAX code, it
is enough to comment or remove three of the four columns calculated in the SUMMARIZECOLUMNS
function (DistinctCountProductKey, Sales_Amount, Margin__, and DistinctCountOrder_Number), fi nding the slowest one before proceeding. In this case, the most expensive calculation is the last one. The
following query takes up 80% of the time required to compute the original query, meaning that the
distinct count over Sales[Order Number] is the most expensive operation in the entire report:

## Evaluate


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),

660 CHAPTER 20 Optimizing DAX

```dax
// "DistinctCountProductKey", CALCULATE (
// DISTINCTCOUNT ( 'Product'[ProductKey] )
// ),
// "Sales_Amount", 'Sales'[Sales Amount],
// "Margin__", 'Sales'[Margin %],
"DistinctCountOrder_Number", CALCULATE (
DISTINCTCOUNT ( 'Sales'[Order Number] )
)
),
[IsGrandTotalRowTotal], 0,
'Product'[Brand], 1
)
```


## Order By

[IsGrandTotalRowTotal] DESC,
'Product'[Brand]
Another example is the following MDX query generated by the pivot table in Excel as seen in
Figure 20-2:

## Select {

[Measures].[Sales Amount],
[Measures].[Total Cost],
[Measures].[Margin],
[Measures].[Margin %]

## } Dimension Properties Parent_Unique_Name, Hierarchy_Unique_Name On Columns,


## Non Empty Hierarchize(


## Drilldownmember(


## { { Drilldownmember(


## { { Drilldownlevel(

{ [Date].[Calendar].[All] },,, include_calc_members )

## } },

{ [Date].[Calendar].[Year].&[CY 2008] },,, include_calc_members )

## } },

{ [Date].[Calendar].[Quarter].&[Q4-2008] },,, include_calc_members
)
)

## Dimension Properties Parent_Unique_Name,Hierarchy_Unique_Name On Rows

FROM [Model]
CELL PROPERTIES VALUE, FORMAT_STRING, LANGUAGE, BACK_COLOR, FORE_COLOR, FONT_FLAGS
FIGURE 20-2 Simple pivot table in Excel that generates an MDX query with four measures.

CHAPTER 20 Optimizing DAX 661
You can reduce the measures either in the pivot table or directly in the MDX code. You can manipulate the MDX code by reducing the list of measures in braces. For example, you reduce the code to only
the Sales Amount measure by modifying the list, as in the following initial part of the query:

## Select

{ [Measures].[Sales Amount] }

## Dimension Properties Parent_Unique_Name, Hierarchy_Unique_Name On Columns,

...
Regardless of the technique you use, once you identify the DAX expression (or measure) that is
responsible for a performance issue, you need a reproduction query to use in DAX Studio.
Creating a reproduction query
The optimization process requires a query that you can execute several times, possibly changing the
defi nition of the measure in order to evaluate different levels of performance.
If you captured a query in DAX or MDX, you already have a good starting point for the reproduction (repro) query. You should try to simplify the query as much as you can, so that it becomes easier to
fi nd the bottleneck. You should only keep a complex query structure when it is fundamental in order to
observe the performance issue.
Creating a reproduction query in DAX
When a measure is constantly slow, you should be able to create a repro query producing a single value
as a result. Using CALCULATE or CALCULATETABLE, you can apply all the fi lters you need. For example,
you can execute the Sales Amount measure for November 2008 using the following code, obtaining
the same result ($96,777,975.30) you see in Figure 20-2 for that month:

## Evaluate

{

## Calculate (

[Sales Amount],
'Date'[Calendar Year] = "CY 2008",
'Date'[Calendar Year Quarter] = "Q4-2008",
'Date'[Calendar Year Month] = "November 2008"
)
}
You can also write the previous query using CALCULATETABLE instead of CALCULATE:

## Evaluate


## Calculatetable (

{ [Sales Amount] },
'Date'[Calendar Year] = "CY 2008",
'Date'[Calendar Year Quarter] = "Q4-2008",
'Date'[Calendar Year Month] = "November 2008"
)
The two approaches produce the same result. You should consider CALCULATETABLE when the
query you use to test the measure is more complex than a simple table constructor.

662 CHAPTER 20 Optimizing DAX
Once you have a repro query for a specifi c measure defi ned in the data model, you should consider
writing the DAX expression of the measure as local in the query, using the MEASURE syntax.
For example, you can transform the previous repro query into the following one:

## Define


```dax
MEASURE Sales[Sales Amount] =
SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
EVALUATE
CALCULATETABLE (
{ [Sales Amount] },
'Date'[Calendar Year] = "CY 2008",
'Date'[Calendar Year Quarter] = "Q4-2008",
'Date'[Calendar Year Month] = "November 2008"
)
```

At this point, you can apply changes to the DAX expression assigned to the measure directly into the
query statement. This way, you do not have to deploy a change to the data model before executing the
query again. You can change the query, clear the cache, and run the query in DAX Studio, immediately
measuring the performance results of the modifi ed expression.
Creating query measures with DAX Studio
DAX Studio can generate the MEASURE syntax for a measure defi ned in the model by using the Defi ne
Measure context menu item. The latter is available by selecting a measure in the Metadata pane, as
shown in Figure 20-3.
FIGURE 20-3 Screenshot of how a user would access the “Defi ne Measure” menu item.
If a measure references other measures, all of them should be included as query measures in order
to consider any possible change to the repro query. The Defi ne Dependent Measures feature includes
the defi nition of all the measures that are referenced by the selected measure, whereas Defi ne and
Expand Measure replaces any measure reference with the corresponding measure expression.
For example, consider the following query that just evaluates the Margin % measure:

## Evaluate

{ [Margin %] }

CHAPTER 20 Optimizing DAX 663
By clicking Defi ne Measure on Margin %, you get the following code, where there are two other
references to Sales Amount and Margin measures:

## Define


```dax
MEASURE Sales[Margin %] =
DIVIDE ( [Margin], [Sales Amount] )
EVALUATE
{ [Margin %] }
```

Instead of repeating the Defi ne Measure action on all the other measures, you can click on Defi ne
Dependent Measures on Margin %, obtaining the defi nition of all the other measures required; this
includes Total Cost, which is used in the Margin defi nition:

## Define


```dax
MEASURE Sales[Margin] = [Sales Amount] - [Total Cost]
MEASURE Sales[Sales Amount] =
SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
MEASURE Sales[Total Cost] =
SUMX ( Sales, Sales[Quantity] * Sales[Unit Cost] )
MEASURE Sales[Margin %] =
DIVIDE ( [Margin], [Sales Amount] )
EVALUATE
{ [Margin %] }
```

You can also obtain a single DAX expression without measure references by clicking Defi ne and
Expand Measure on Margin %:

## Define


```dax
MEASURE Sales[Margin %] =
DIVIDE (
CALCULATE (
CALCULATE ( SUMX ( Sales, Sales[Quantity] * Sales[Net Price] ) )
- CALCULATE ( SUMX ( Sales, Sales[Quantity] * Sales[Unit Cost] ) )
),
CALCULATE ( SUMX ( Sales, Sales[Quantity] * Sales[Net Price] ) )
)
EVALUATE
{ [Margin %] }
```

This latter technique can be useful to quickly evaluate whether a measure includes nested iterators
or not, though it could generate very verbose results.
Creating a reproduction query in MDX
In certain conditions, you have to use an MDX query to reproduce a problem that only happens in MDX
and not in DAX. The same DAX measure, executed in a DAX or in an MDX query, generates different
query plans; it might display a different behavior depending on the language of the query. However in

664 CHAPTER 20 Optimizing DAX
this case too, you can defi ne the DAX measure local to the query. That way, it is more effi cient to edit
and run again. For instance, you can defi ne the Sales Amount measure local to the MDX query using
the WITH MEASURE syntax:

## With


```dax
MEASURE Sales[Sales Amount] = SUMX ( Sales, Sales[Quantity] * Sales[Unit Price] )
SELECT {
[Measures].[Sales Amount],
[Measures].[Total Cost],
[Measures].[Margin],
[Measures].[Margin %]
} DIMENSION PROPERTIES PARENT_UNIQUE_NAME, HIERARCHY_UNIQUE_NAME ON COLUMNS,
```


## Non Empty Hierarchize(


## Drilldownmember(


## { { Drilldownmember(


## { { Drilldownlevel(

{ [Date].[Calendar].[All] },,, include_calc_members )

## } },

{ [Date].[Calendar].[Year].&[CY 2008] },,, include_calc_members )

## } },

{ [Date].[Calendar].[Quarter].&[Q4-2008] },,, include_calc_members
)
)

## Dimension Properties Parent_Unique_Name,Hierarchy_Unique_Name On Rows

FROM [Model]
CELL PROPERTIES VALUE, FORMAT_STRING, LANGUAGE, BACK_COLOR, FORE_COLOR, FONT_FLAGS
As you see, in MDX you must use WITH instead of DEFINE, which is how you can rename the syntax
generated by DAX Studio if you optimize an MDX query. The syntax after MEASURE is always DAX
code, so you will follow the same optimization process for an MDX query. Regardless of the repro
query language (either DAX or MDX), you always have a DAX expression to optimize, which you can
defi ne within a local MEASURE defi nition.
Analyzing server timings and query plan information
Once you have a repro query, you run it and collect information about execution time and query
plan. You saw in Chapter 19 how to read the information provided by DAX Studio or SQL Server
Profi ler. In this section, we recap the steps required to analyze a simple query in DAX Studio.
For example, consider the following DAX query:

## Define


```dax
MEASURE Sales[Sales Amount] =
SUMX ( Sales, Sales[Quantity] * Sales[Unit Price] )
EVALUATE
```


## Addcolumns (

VALUES ( 'Date'[Calendar Year] ),
"Result", [Sales Amount]
)

CHAPTER 20 Optimizing DAX 665
If you execute this query in DAX Studio after clearing the cache and enabling Query Plan and Server
Timings, you obtain a result with one row for each year in the Date table, and the total of Sales Amount
for sales made in that year. The starting point for an analysis is always the Server Timings pane, which
displays information about the entire query, as shown in Figure 20-4.
FIGURE 20-4 Server Timings pane after a simple query execution.
Our query returned the result in 25 ms (Total), and it spent 72 percent of this time in the storage
engine (SE), whereas the formula engine (FE) only used up 7 ms of the total time. This pane does not
provide much information about the formula engine internals, but it is rich in details on storage engine
activity. For example, there were two storage engine queries (SE Queries) that consumed a total of
94 ms of processing time (SE CPU). The CPU time can be larger than Duration thanks to the parallelism
of the storage engine. Indeed, the engine used 94 ms of logical processors working in parallel, so that
the duration time is a fraction of that number. The hardware used in this test had 8 logical processors,
and the parallelism degree of this query (ratio between SE CPU and SE) is 5.2. The parallelism cannot be
higher than the number of logical processors you have.
The storage engine queries are available in the list, and you can see that a single storage engine
operation (the fi rst one) consumes the entire duration and CPU time. By enabling the display of Internal
and Cache subclass events, you can see in Figure 20-5 that the two storage engine queries were actually executed by the storage engine.
FIGURE 20-5 Server Timings pane with internal subclass events visible.

666 CHAPTER 20 Optimizing DAX
If you execute the same query again without clearing the cache, you see the results in Figure 20-6.
Both storage engine queries retrieved the values from the cache (SE cache), and the storage engine
queries resolved in the cache are visible in the Subclass column.
FIGURE 20-6 Server Timings pane with cache subclass events visible, after second execution of the same DAX query.
Usually, we will use the repro query with a cold cache (clearing the cache before the execution), but
in some cases it is important to evaluate whether a given DAX expression can leverage the cache in an
upcoming request or not. For this reason, the Cache visualization in DAX Studio is disabled by default,
and you enable it on demand.
At this point, you can start looking at the query plans. In Figure 20-7 you see the physical and logical
query plans of the query used in the previous example.
The physical query plan is the one you will use more often. In the query of the previous example, there
are two datacaches—one for each storage engine query. Every Cache row in the physical query plan
consumes one of the datacaches available. However, there is no simple way to match the correspondence
between a query plan operation and a datacache. You can infer the datacache by looking at the columns
used in the operations requiring a Cache result (the Spool_Iterator and SpoolLookup rows, in Figure 20-7).
FIGURE 20-7 Query Plan pane showing physical and logical query plans.

CHAPTER 20 Optimizing DAX 667
An important piece of information available in the physical query plan is the column showing the
number of records processed. As you will see, when optimizing bottlenecks in the formula engine, it
might be useful to identify the slowest operation in the formula engine by searching for the line with
the largest number of records. You can sort the rows by clicking the Records column header, as you see
in Figure 20-8. You restore the original sort order by clicking the Line column header.
FIGURE 20-8 Steps in physical query plan sorted by Records column.
Identifying bottlenecks in the storage engine
or formula engine
There are many possible optimizations usually available for any query. The fi rst and most important
step is to identify whether a query spends most of the time in the formula engine or in the storage
engine. A fi rst indication is available in the percentages provided by DAX Studio for FE and SE. Usually,
this is a good starting point, but you also have to identify the distribution of the workload in both the
formula engine and the storage engine. In complex queries, a large amount of time spent in the storage engine might correspond to a large number of small storage engine queries or to a small number
of storage engine queries that concentrate the most of the workload. As you will see, these differences
require different approaches in your optimization strategy.
When you identify the execution bottleneck of a query, you should also prioritize the optimization
areas. For example, there might be different ineffi ciencies in the query plan resulting in a large formula
engine execution time. You should identify the most important ineffi ciency and concentrate on that
fi rst. If you do not follow this approach, you might end up spending time optimizing an expression that
only marginally affects the execution time. Sometimes the more effi cient optimizations are simple but
hidden in counterintuitive context transitions or in other details of the DAX syntax. You should always
measure the execution time before and after each optimization attempt, making sure that you obtain a
real advantage and that you are not just applying some optimization pattern you found on the web or
in this book without any real benefi t.

668 CHAPTER 20 Optimizing DAX
Finally, remember that even if you have an issue in the formula engine, you should always start your
analysis by looking at the storage engine queries. They provide valuable information about the content
and size of the datacaches used by the formula engine. Reading the query plan that describes the operations made by the formula engine is a very complex process. It is easier to consider that the formula
engine will use the content of datacaches and will have to do all the operations required to produce the
result of a DAX query that has not already been produced by the storage engine. This approach is especially effi cient for large and complex DAX queries. Indeed, these might generate thousands of lines in a
query plan, but a relatively small number of datacaches produced by storage engine queries.
Implementing changes and rerunning the test query
Once the bottlenecks have been identifi ed, the next step is to change the DAX expressions and/or the
data model, so that the query plan is more effi cient. Running the test query again, it is possible to verify
that the improvement is effective, starting the search for the next bottleneck and continuing the loop
restarting at the step “Analyzing server timings and query plan information.” This process will continue
until the performance is optimal or there are no further possible improvements that are worth the effort.
Optimizing bottlenecks in DAX expressions
A longer execution time in the storage engine is usually the consequence of one or more of the
following causes (explained in more detail in Chapter 19):

> **Note:** Longer scan time. Even for a simple aggregation, a DAX query must scan one or more columns. The cost for this scan depends on the size of the column, which depends on the number
of unique values and on the data distribution. Different columns in the same table can have
very different execution times.

> **Note:** Large cardinality. A large number of unique values in a column affects the DISTINCTCOUNT
calculation and the fi lter arguments of the CALCULATE and CALCULATETABLE functions. A large
cardinality can also affect the scan time of a column, but it could be an issue by itself regardless
of the column data size.

> **Note:** High frequency of CallbackDataID. A large number of calls made by the storage engine to
the formula engine can affect the overall performance of a query.

> **Note:** Large materialization. If a storage engine query produces a large datacache, its generation
requires time (allocating and writing RAM). Moreover, its consumption (by the formula engine)
is also another potential bottleneck.
In the following sections, you will see several examples of optimization. Starting with the concepts you
learned in previous chapters, you will see a typical problem reproduced in a simpler query and optimized.
Optimizing fi lter conditions
Whenever possible, a fi lter argument of a CALCULATE/CALCULATETABLE function should always fi lter
columns rather than tables. The DAX engine has improved over the years, and several simple table

CHAPTER 20 Optimizing DAX 669
fi lters are relatively well optimized in 2019 or newer engine versions. However, expressing a fi lter
condition by columns rather than by tables is always a best practice.
For example, consider the report in Figure 20-9 that compares the total of Sales Amount with the
sum of the sales transactions larger than $1,000 (Big Sales Amount) for each product brand.
FIGURE 20-9 Sales Amount and Big Sales Amount reported by product brand.
Because the fi lter condition in the Big Sales Amount measure requires two columns, a trivial way to
defi ne the fi lter is by using a fi lter over the Sales table. The following query computes just the Big Sales
Amount measure in the previous report, generating the server timings results visible in Figure 20-10:

## Define


```dax
MEASURE Sales[Big Sales Amount (slow)] =
CALCULATE (
[Sales Amount],
FILTER (
Sales,
Sales[Quantity] * Sales[Net Price] > 1000
)
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),
"Big_Sales_Amount", 'Sales'[Big Sales Amount (slow)]
)
FIGURE 20-10 Server Timings running the query for the Big Sales Amount (slow) measure.

670 CHAPTER 20 Optimizing DAX
Because FILTER is iterating a table, this query is generating a larger datacache than necessary. The
result in Figure 20-9 only displays 11 brands and one additional row for the grand total. Nevertheless,
the query plan estimates that the fi rst two datacaches return 3,937 rows, which is the same number as
reported also in the Query Plan pane visible in Figure 20-11.
FIGURE 20-11 Query Plan pane running the query for Big Sales Amount (slow) measure.
The formula engine receives a much larger datacache than the one required for the query result
because there are two additional columns. Indeed, the xmSQL query at line 2 is the following:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Product'[Brand],
'DaxBook Sales'[Quantity],
'DaxBook Sales'[Net Price],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]

## Where

( COALESCE ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) ) ) > COALESCE ( 1000.000000 ) );
The structure of the xmSQL query at line 4 in Figure 20-10 is similar to the previous one, just without
the SUM aggregation. The presence of a table fi lter in CALCULATE results in this side effect in the query
plan because the semantic of the fi lter includes all the columns of the Sales expanded table (expanded
tables are described in Chapter 14, “Advanced DAX concepts”).
The optimization of the measure only requires a column fi lter. Because the fi lter expression uses
two columns, a row context requires a table with just those two columns to produce a corresponding
and more effi cient fi lter argument to CALCULATE. The following query implements the columns fi lter
adding KEEPFILTERS to keep the same semantic as the previous version, generating the server timings
results visible in Figure 20-12:

CHAPTER 20 Optimizing DAX 671

## Define


```dax
MEASURE Sales[Big Sales Amount (fast)] =
CALCULATE (
[Sales Amount],
KEEPFILTERS (
FILTER (
ALL (
Sales[Quantity],
Sales[Net Price]
),
Sales[Quantity] * Sales[Net Price] > 1000
)
)
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),
"Big_Sales_Amount", 'Sales'[Big Sales Amount (fast)]
)
FIGURE 20-12 Server Timings when running the query for the Big Sales Amount (fast) measure.
The DAX query runs faster, but what is more important is that there is only one datacache for the
rows of the result, excluding the grand total, which still has a separate xmSQL query. The materialization of the datacache at line 2 in Figure 20-12 only returns 14 estimated rows, when there are only 11
in the actual count visible in the Query Plan pane in Figure 20-13.
FIGURE 20-13 Query Plan pane running the query for Big Sales Amount (fast) measure.
The reason for this optimization is that the query plan can create a much more effi cient calculation
in the storage engine without returning additional data to the formula engine because of the semantic
required by a table fi lter. The following is the xmSQL query at line 2 in Figure 20-12:

672 CHAPTER 20 Optimizing DAX

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Product'[Brand],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]

## Where

( COALESCE ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) ) ) > COALESCE ( 1000.000000 ) );
The datacache no longer includes the Quantity and Net Price columns, and its cardinality
corresponds to the cardinality of the DAX result. This is an ideal condition for minimal materialization.
Keeping the fi lter conditions using columns rather than tables is an important effort to achieve
this goal.
The important takeaway of this section is that you should always pay attention to the rows returned
by storage engine queries. When their number is much bigger than the rows included in the result of
a DAX query, there might be some overhead caused by the additional work performed by the storage
engine to materialize datacaches and by the formula engine to consume such datacaches. Table fi lters
are one of the most common reasons for excessive materialization, though they are not always
responsible for bad performance.
Note When you write a DAX fi lter, consider the cardinality of the resulting fi lter. If the cardinality using a table fi lter is identical to a column fi lter and the table fi lter does not expand
to other tables, then the table fi lter can be used safely. For example, there is not usually
much difference between fi ltering a Date table versus the Date[Date] column.
Optimizing context transitions
The storage engine can only compute simple aggregations and simple grouping over columns of the
model. Anything else must be computed by the formula engine. Every time there is an iteration and a
corresponding context transition, the storage engine materializes a datacache at the granularity level
of the iterated table. If the expression computed during the iteration is simple enough to be solved
by the storage engine, the performance is typically good. Otherwise, if the expression is too complex, a large materialization and/or a CallbackDataID might occur as we demonstrate in the following
example. In these scenarios, simplifying the code by reducing the number of context transitions and
by reducing the granularity of the iterated table greatly helps in improving performance. For example,
consider a Cashback measure that multiplies the Sales Amount by the Cashback % attribute assigned
to each Customer based on an algorithm defi ned by the marketing department. The report in Figure
20-14 displays the Cashback amount for each country.

CHAPTER 20 Optimizing DAX 673
FIGURE 20-14 Cashback reported by customer country.
The easiest and most intuitive way to create the Cashback measure is also the slowest, which multiplies the Cashback % by the Sales Amount for each customer, summing the result. The following query
computes the slowest Cashback measure in the previous report, generating the server timings results
visible in Figure 20-15:

## Define


```dax
MEASURE Sales[Cashback (slow)] =
SUMX (
Customer,
[Sales Amount] * Customer[Cashback %]
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Customer'[Country], "IsGrandTotalRowTotal" ),
"Cashback", 'Sales'[Cashback (slow)]
)
FIGURE 20-15 Server Timings running the query for the Cashback (slow) measure reported by country.
The queries at lines 2 and 4 of Figure 20-15 compute the result at the Country level, whereas the
queries at lines 6 and 8 run the same task for the grand total. We will focus exclusively on the fi rst two
storage engine queries. In order to check whether the estimation for the rows materialized is correct,
you can look at the query plan in Figure 20-16. This could be surprising, because it seems that a few
storage engine queries are not used at all.

674 CHAPTER 20 Optimizing DAX
FIGURE 20-16 Query Plan pane running the query for the Cashback (slow) measure reported by country.
The query plan in Figure 20-16 only reports two Cache nodes, which correspond to lines 4 and 8 of
the Server Timings pane in Figure 20-15. This is another example of why looking at the query plan
could be confusing. The formula engine is actually doing some other work, but the execution within
a CallbackDataID is not always reported in the query plan, and this is one of those cases. This is the
xmSQL query at line 4 of Figure 20-15, which returns 29 effective rows instead of the estimated 32:

## With

$Expr0 := ( [CallbackDataID ( SUMX ( Sales, Sales[Quantity]] * Sales[Net Price]] ) )
] ( PFDATAID ( 'DaxBook Customer'[CustomerKey] ) )
* PFCAST ( 'DaxBook Customer'[Cashback %] AS REAL ) )

## Select

'DaxBook Customer'[Country],
SUM ( @$Expr0 )
FROM 'DaxBook Customer';
The DAX code passed to CallbackDataID must be computed for each customer by the formula
engine, which receives the CustomerKey as argument. You can see the additional storage engine queries, but the corresponding query plan is not visible in this case. Therefore, we can only imagine what
the query plan does by looking at the other storage engine query at line 2 of Figure 20-15:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Customer'[CustomerKey],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey];
The result of this xmSQL query only contains two columns: the CustomerKey and the result of the
Sales Amount measure for that customer. Thus, the formula engine uses the result of this query to
provide a result to the CallbackDataID request of the former query.

CHAPTER 20 Optimizing DAX 675
Once again, instead of trying to describe the exact sequence of operations performed by the
engine, it is easier to analyze the result of the storage engine queries, checking whether the materialization is larger than what is required for the query result. In this case the answer is yes: the DAX query
returns only 6 visible countries, whereas a total of 29 countries were computed by the formula engine.
In any case, there is a huge difference with the materialization of 18,872 customers produced by the
latter xmSQL query analyzed. Is it possible to push more workload to the storage engine, aggregating
the data by country instead of by customer? The answer is yes, by reducing the number of context transitions. Consider the original Cashback measure: the expression executed in the row context depends
on a single column of the Customer table (Cashback %):
Sales[Cashback (slow)] :=

## Sumx (

Customer,
[Sales Amount] * Customer[Cashback %]
)
Because the Sales Amount measure can be computed for a group of customers that have the same
Cashback %, the optimal cardinality for the SUMX iterator is defi ned by the unique values of the Cashback % column. The following optimized version just replaces the fi rst argument of SUMX using the
unique values of Cashback % visible in the fi lter context:

## Define


```dax
MEASURE Sales[Cashback (fast)] =
SUMX (
VALUES ( Customer[Cashback %] ),
[Sales Amount] * Customer[Cashback %]
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Customer'[Country], "IsGrandTotalRowTotal" ),
"Cashback", 'Sales'[Cashback (fast)]
)
This way, the materialization is much smaller, as visible in Figure 20-17. However, even though the
number of rows materialized is signifi cantly smaller, the overall execution time is similar if not larger;
remember that a difference of a few milliseconds should not be considered relevant.
FIGURE 20-17 Server Timings running the query for Cashback (fast) reported by country.

676 CHAPTER 20 Optimizing DAX
This time there is a single xmSQL query to compute the amount by country. This is the xmSQL query
at line 2 of Figure 20-17:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Customer'[Country],
'DaxBook Customer'[Cashback %],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey];
The result of this query contains three columns: Country, Cashback %, and the corresponding Sales
Amount value. Thus, the formula engine multiplies Cashback % by Sales Amount for each row, aggregating the rows belonging to the same country. The result presents an estimated count of 288 rows, whereas
there are only 65 rows consumed by the formula engine. This is visible in the query plan in Figure 20-18.
FIGURE 20-18 Query Plan pane running the query for Cashback (fast) reported by country.
Even though it is not evident, this measure is faster than the original measure. Having a smaller
footprint in memory, it performs better in more complex reports. This is immediately visible by using a
slightly different report like the one in Figure 20-19, grouping the Cashback measure by product brand
instead of by customer country.
FIGURE 20-19 Cashback reported by product brand.

CHAPTER 20 Optimizing DAX 677
The following query computes the slowest Cashback measure in the report shown in Figure 20-19,
generating the server timings results visible in Figure 20-20:

## Define


```dax
MEASURE Sales[Cashback (slow)] =
SUMX (
Customer,
[Sales Amount] * Customer[Cashback %]
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( Product[Brand], "IsGrandTotalRowTotal" ),
"Cashback", 'Sales'[Cashback (slow)]
)
FIGURE 20-20 Server Timings running the query for Cashback (slow) reported by brand.
There are a few differences in this query plan, but we focus on the materialization of 192,514 rows
produced by the following xmSQL query at line 2 of Figure 20-20:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Customer'[CustomerKey],
'DaxBook Product'[Brand],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey]
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
The reason for the larger materialization is that now, the inner calculation computes Sales Amount
for each combination of CustomerKey and Brand. The estimated count of 192,514 rows is confi rmed by
the actual count visible in the query plan in Figure 20-21.
FIGURE 20-21 Query Plan pane running the query for the Cashback (slow) measure reported by country.

678 CHAPTER 20 Optimizing DAX
When the test query is using the faster measure, the materialization is much smaller and the query
response time is also much faster. The execution of the following DAX query produces the server
timings results visible in Figure 20-22:

## Define


```dax
MEASURE Sales[Cashback (fast)] =
SUMX (
VALUES ( Customer[Cashback %] ),
[Sales Amount] * Customer[Cashback %]
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( Product[Brand], "IsGrandTotalRowTotal" ),
"Cashback", 'Sales'[Cashback (fast)]
)
FIGURE 20-22 Server Timings running the query for Cashback (fast) reported by brand.
The materialization is three orders of magnitude smaller (126 rows instead of 192,000), and the total
execution time is 9 times faster than the slow version (it was 415 milliseconds and it is 48 milliseconds
with the fast version). Because these differences depend on the cardinality of the report, you should
focus on the formula that minimizes the work in the formula engine by computing most of the aggregations in the storage engine. Reducing the number of context transitions is an important step to
achieve this goal.

Note Excessive materialization generated by unnecessary context transitions is the most
common performance issue in DAX measures. Using table fi lters instead of column fi lters is
the second most common performance issue. Therefore, making sure that your DAX measures do not have these two problems should be your priority in an optimization effort. By
inspecting the server timings, you should be able to quickly see the symptoms by looking at
the materialization size.
Optimizing IF conditions
An IF function is always executed by the formula engine. When there is an IF function within an iteration, there could be a CallbackDataID involved in the execution. Moreover, the engine might evaluate
the arguments of the IF regardless of the result of the condition in the fi rst argument. Even though

CHAPTER 20 Optimizing DAX 679
the result is correct, you might pay the full cost of processing all the possible solutions. As usual, there
could be different behaviors depending on the version of the DAX engine used.
Optimizing IF in measures
Conditional statements in a measure could trigger a dangerous side effect in the query plan, generating the calculation of every conditional branch regardless of whether it is needed or not. In general, it is
a good idea to avoid or at least reduce the number of conditional statements in expressions evaluated
for measures, applying fi lters through the fi lter context whenever possible.
For example, the report in Figure 20-23 displays a Fam. Sales measure that only considers customers with at least one child at home. Because the goal is to display the value for individual customers,
the fi rst implementation (slow) does not work for aggregations of two or more customers (Total row is
blank), whereas the alternative, faster implementation also works at aggregated levels.
FIGURE 20-23 Fam. Sales reported by product brand.
The following query computes the Fam. Sales (slow) measure in a report similar to the one in
Figure 20-1. For each customer, an IF statement checks the number of children at home to fi lter customers classifi ed as a family. The execution of the following DAX query produces the server timings
results visible in Figure 20-22:

## Define


```dax
MEASURE Sales[Fam. Sales (slow)] =
VAR ChildrenAtHome = SELECTEDVALUE ( Customer[Children At Home] )
VAR Result =
IF (
ChildrenAtHome > 0,
[Sales Amount]
)
RETURN Result
EVALUATE
CALCULATETABLE (
SUMMARIZECOLUMNS (
ROLLUPADDISSUBTOTAL (
ROLLUPGROUP (
'Customer'[CustomerKey],
'Customer'[Name]
), "IsGrandTotalRowTotal"
```

680 CHAPTER 20 Optimizing DAX
),
"Fam__Sales__slow_", 'Sales'[Fam. Sales (slow)]
),
'Product Category'[Category] = "Home Appliances",
'Product'[Manufacturer] = "Northwind Traders",
'Product'[Class] = "Regular",

## Datesbetween (

'Date'[Date],

## Date ( 2007, 5, 10 ),


## Date ( 2007, 5, 10 )

)
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Customer'[CustomerKey],
'Customer'[Name]
FIGURE 20-24 Server Timings running the query for Fam. Sales (slow) reported by customer.
The query is not that slow, but we wanted a query result with a small number or rows because
the focus is mainly on the materialization required. We can avoid looking at the query plan, which is
already 62 lines long, because the information provided in the Server Timings pane already highlights
several facts:

> **Note:** Even though the DAX result only has 7 rows, the rows materialized in three xmSQL queries have
more than 18,000 rows, a number close to the number of customers.

> **Note:** The materialization produced by the storage engine query at line 4 in Figure 20-24 includes
information about the number of children at home computed for each customer.

> **Note:** The materialization produced by the storage engine query at line 9 in Figure 20-24 includes the
Sales Amount measure computed for each customer.

> **Note:** The grand total is not computed by any storage engine query, so it is the formula engine that
aggregates the customers to obtain that number.
This is the storage engine query at line 4 in Figure 20-24. It provides the information required by the
formula engine to fi lter customers based on the number of children at home:

## Select

'DaxBook Customer'[CustomerKey],
SUM ( ( PFDATAID ( 'DaxBook Customer'[Children At Home] ) <> 2 ) ),
MIN ( 'DaxBook Customer'[Children At Home] ),

CHAPTER 20 Optimizing DAX 681
MAX ( 'DaxBook Customer'[Children At Home] ),

## Count ( )

FROM 'DaxBook Customer';
This result is used as an argument to the following storage engine query at line 9 in Figure 20-24 in
order to fi lter an estimate of 7,368 customers that have at least one child at home:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Customer'[CustomerKey],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey]
LEFT OUTER JOIN 'DaxBook Date'
ON 'DaxBook Sales'[OrderDateKey]='DaxBook Date'[DateKey]
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]
LEFT OUTER JOIN 'DaxBook Product Subcategory'
ON 'DaxBook Product'[ProductSubcategoryKey]
='DaxBook Product Subcategory'[ProductSubcategoryKey]
LEFT OUTER JOIN 'DaxBook Product Category'
ON 'DaxBook Product Subcategory'[ProductCategoryKey]
='DaxBook Product Category'[ProductCategoryKey]

## Where

'DaxBook Customer'[CustomerKey]

## In ( 2241, 13407, 5544, 7787, 11090, 7368, 17055, 16636, 1329, 12914..

[7368 total values, not all displayed] )
VAND 'DaxBook Date'[Date] = 39212.000000
VAND 'DaxBook Product'[Manufacturer] = 'Northwind Traders'
VAND 'DaxBook Product'[Class] = 'Regular'
VAND 'DaxBook Product Category'[Category] = 'Home Appliances';
The estimated number of rows in this result is wrong, because there are only 7 rows received in the
previous storage engine query. This is visible in the query plan; however, it might not be trivial to fi nd
the corresponding xmSQL query for each Cache node in the query plan shown in Figure 20-25.
FIGURE 20-25 Server Timings running the query for the Fam. Sales (slow) measure reported by customer.
The previous storage engine query receives a fi lter over the CustomerKey column. The formula
engine requires a materialization of such a list of values in CustomerKey in order to provide the corresponding fi lter in a storage engine query. However, the materialization of a large number of customers in the formula engine is likely to be the bigger cost for this query. The size of this materialization
depends on the number of customers. Therefore, a model with hundreds of thousands or millions of
customers would make the performance issue evident. In this case you should look at the size of the

682 CHAPTER 20 Optimizing DAX
materialization rather than just the execution time. The latter is still relatively quick. Understanding
whether the materialization is effi cient is important to create a formula that scales up well with a growing number of rows in the model.
The IF statement in the measure can only be evaluated by the formula engine. This requires either
materialization like in this example, or CallbackDataID calls, which we describe later. A better approach
is to apply a fi lter to the fi lter context using CALCULATE. This removes the need to evaluate an IF condition for every cell of the query result.
When the test query is using the faster measure, the materialization is much smaller and the query
response time is also much shorter. The execution of the following DAX query produces the server
timings results visible in Figure 20-26:

## Define


```dax
MEASURE Sales[Fam. Sales (fast)] =
CALCULATE (
[Sales Amount],
KEEPFILTERS ( Customer[Children At Home] > 0 )
)
EVALUATE
CALCULATETABLE (
SUMMARIZECOLUMNS (
ROLLUPADDISSUBTOTAL (
ROLLUPGROUP (
'Customer'[CustomerKey],
'Customer'[Name]
), "IsGrandTotalRowTotal"
),
"Fam__Sales__fast_", 'Sales'[Fam. Sales (fast)]
),
'Product Category'[Category] = "Home Appliances",
'Product'[Manufacturer] = "Northwind Traders",
'Product'[Class] = "Regular",
DATESBETWEEN (
'Date'[Date],
DATE ( 2007, 5, 10 ),
DATE ( 2007, 5, 10 )
)
)
```


## Order By

[IsGrandTotalRowTotal] DESC,
'Customer'[CustomerKey],
'Customer'[Name]
FIGURE 20-26 Server Timings running the query for Fam. Sales (fast) reported by customer.

CHAPTER 20 Optimizing DAX 683
Even though there are still four storage engine queries, the query at line 4 in Figure 20-24 is no
longer used. The query at line 4 in Figure 20-26 corresponds to the query at line 9 in Figure 20-24. It
includes the fi lter over the number of children, highlighted in the last two lines of the following
xmSQL query:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Customer'[CustomerKey],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey]
LEFT OUTER JOIN 'DaxBook Date'
ON 'DaxBook Sales'[OrderDateKey]='DaxBook Date'[DateKey]
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]
LEFT OUTER JOIN 'DaxBook Product Subcategory'
ON 'DaxBook Product'[ProductSubcategoryKey]
='DaxBook Product Subcategory'[ProductSubcategoryKey]
LEFT OUTER JOIN 'DaxBook Product Category'
ON 'DaxBook Product Subcategory'[ProductCategoryKey]
='DaxBook Product Category'[ProductCategoryKey]

## Where

'DaxBook Date'[Date] = 39212.000000
VAND 'DaxBook Product'[Manufacturer] = 'Northwind Traders'
VAND 'DaxBook Product'[Class] = 'Regular'
VAND 'DaxBook Product Category'[Category] = 'Home Appliances'
VAND ( PFCASTCOALESCE ( 'DaxBook Customer'[Children At Home] AS INT )

## > Coalesce ( 0 ) );

This different query plan has pros and cons. The advantage is that the formula engine bears a lower
workload, not having to transfer the fi lter of customers back and forth between storage engine queries.
The price to pay for this is that the execution of the fi lters is applied at the storage engine level, which
results in an increased cost moving from a former 32 ms of SE CPU time to the current 94 ms of SE
CPU time.
Another side effect of the new query plan is the additional storage engine query at line 8 in Figure
20-26; this query computes the aggregation at the grand total without having to perform such aggregation in the formula engine, as was the case in the slower measure. The code is similar to the previous
xmSQL query, without the aggregation by CustomerKey.
As a rule of thumb, replacing a conditional statement with a fi lter argument in CALCULATE is usually
a good idea, prioritizing a smaller materialization rather than looking at the execution time for small
queries. This way, the expression is usually more scalable with larger data models. However, you should
always evaluate the performance in specifi c conditions, analyzing the metrics provided by DAX Studio
using different implementations; you might otherwise choose an implementation that, in a particular
scenario, turns out to be slower and not faster.

684 CHAPTER 20 Optimizing DAX
Choosing between IF and DIVIDE
A very common use of the IF statement is to make sure that an expression is only evaluated with valid
arguments. For example, an IF function can validate the denominator of a division to avoid a division
by zero. For this specifi c condition, the DIVIDE function provides a faster alternative. It is interesting to
consider why the code is faster by analyzing the different executions with DAX Studio.
The report in Figure 20-27 displays an Average Price measure by customer and brand.
FIGURE 20-27 Average Price reported by product brand and customer.
The following query computes the Average Price (slow) measure in the report shown in Figure 20-27.
For each combination of product brand and customer, it divides the sales amount by the sum of
quantity—only if the latter is not equal to zero. The execution of this DAX query produces the server
timings results visible in Figure 20-28:

## Define


```dax
MEASURE Sales[Average Price (slow)] =
VAR Quantity = SUM ( Sales[Quantity] )
VAR SalesAmount = [Sales Amount]
VAR Result =
IF (
Quantity <> 0,
SalesAmount / Quantity
)
RETURN Result
EVALUATE
```


## Topn (

502,

## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Customer'[CustomerKey],
'Product'[Brand]
), "IsGrandTotalRowTotal"
),
"Average_Price__slow_", 'Sales'[Average Price (slow)]
),
[IsGrandTotalRowTotal], 0,

CHAPTER 20 Optimizing DAX 685
'Customer'[CustomerKey], 1,
'Product'[Brand], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Customer'[CustomerKey],
'Product'[Brand]
FIGURE 20-28 Server Timings running the query for Average Price (slow) reported by product brand and customer.
Though the result of the query is limited to 500 rows, the materialization of the datacaches returned
by the storage engine queries is much larger. The following xmSQL query is executed at line 2 in
Figure 20-28, and returns one row for each combination of customer and brand:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Customer'[CustomerKey],
'DaxBook Product'[Brand],
SUM ( @$Expr0 ),
SUM ( 'DaxBook Sales'[Quantity] )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey]
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
The query does not have any fi lter; therefore, the formula engine evaluates every row returned
by this datacache, sorting the result and choosing the fi rst 500 rows to return. This is certainly the
most expensive part of the storage engine execution, which consumes 90% of the query duration time. The other three storage engine queries return the list of product brands (line 4), the list
of customers (line 6), and the value of sales amount and quantity at the grand total level (line 8).
However, these queries are less important in the optimization process. What matters is the formula
engine cost required to execute the IF condition on more than 190,000 rows. The query plan resulting from the slow version of the measure has more than 80 lines (not reported here), and it consumes every datacache multiple times. This is a side effect of having different execution branches in
an IF statement.

686 CHAPTER 20 Optimizing DAX
The optimization of the Average Price measure is based on replacing the IF function with DIVIDE.
The execution of the following DAX query produces the server timings results visible in Figure 20-29:

## Define


```dax
MEASURE Sales[Average Price (fast)] =
VAR Quantity = SUM ( Sales[Quantity] )
VAR SalesAmount = [Sales Amount]
VAR Result =
DIVIDE (
SalesAmount,
Quantity
)
RETURN Result
EVALUATE
```


## Topn (

502,

## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Customer'[CustomerKey],
'Product'[Brand]
), "IsGrandTotalRowTotal"
),
"Average_Price__fast_", 'Sales'[Average Price (fast)]
),
[IsGrandTotalRowTotal], 0,
'Customer'[CustomerKey], 1,
'Product'[Brand], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Customer'[CustomerKey],
'Product'[Brand]
FIGURE 20-29 Server Timings running the query for Average Price (fast) reported by product brand and customer.
The query now runs in 413 milliseconds, saving more than 80% of the execution time. At fi rst sight,
there being only two storage engine queries instead of four might seem like a good reason for the
improved performance. However, this is not really the case. Overall, the SE CPU time did not change
signifi cantly, and the larger materialization is still there. The optimization is obtained by a shorter and
more effi cient query plan, which has only 36 lines instead of more than 80 generated by the slower

CHAPTER 20 Optimizing DAX 687
query. In other words, DIVIDE reduces the size and complexity of the query plan, saving time in the
formula engine execution by almost one order of magnitude.
Optimizing IF in iterators
Using the IF statement within a large iterator might create expensive callbacks to the formula engine.
For example, consider a Discounted Sales measure that applies a 10% discount to every transaction
that has a quantity greater than or equal to 3. The report in Figure 20-30 displays the Discounted Sales
amount for each product brand.
FIGURE 20-30 Discounted Sales reported by product brand.
The following query computes the slower Discounted Sales measure in the previous report,
generating the server timings results visible in Figure 20-31:

## Define


```dax
MEASURE Sales[Discounted Sales (slow)] =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price] * IF (
Sales[Quantity] >= 3,
.9,
1
)
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),
"Sales_Amount", 'Sales'[Sales Amount],
"Discounted_Sales__slow_", 'Sales'[Discounted Sales (slow)]
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Product'[Brand]

688 CHAPTER 20 Optimizing DAX
FIGURE 20-31 Server Timings running the query for Discounted Sales (slow) reported by product brand.
The IF statement executed in the SUMX iterator produces two storage engine queries with a
CallbackDataID call. The following is the xmSQL query at line 2 of Figure 20-31:

## With

$Expr0 := ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )
* [CallbackDataID ( IF ( Sales[Quantity]] >= 3, .9, 1 ) ) ]
( PFDATAID ( 'DaxBook Sales'[Quantity] ) ) ) ,
$Expr1 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) )

## Select

'DaxBook Product'[Brand],
SUM ( @$Expr0 ),
SUM ( @$Expr1 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
The presence of a CallbackDataID comes with two consequences: a slower execution time compared
to the storage engine performance and the unavailability of the storage engine cache. The datacache
must be computed every time and cannot be retrieved from the cache in subsequent requests. The
second issue could be more important than the fi rst one, as is the case for this example.
The CallbackDataID can be removed by rewriting the measure in a different way, summing the value
of two CALCULATE statements with different fi lters. For example, the Discounted Sales measure can
be rewritten using two CALCULATE functions, one for each percentage, fi ltering the transactions that
share the same multiplicator. The following DAX query implements a version of Discounted Sales that
does not rely on any CallbackDataID. The code is longer and requires KEEPFILTERS to provide the same
semantic as in the original measure, producing the server timings results visible in Figure 20-32:

## Define


```dax
MEASURE Sales[Discounted Sales (scalable)] =
CALCULATE (
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
) * .9,
KEEPFILTERS ( Sales[Quantity] >= 3 )
) + CALCULATE (
```

CHAPTER 20 Optimizing DAX 689

## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
),

```dax
KEEPFILTERS ( NOT ( Sales[Quantity] >= 3 ) )
)
EVALUATE
```


## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),
"Sales_Amount", 'Sales'[Sales Amount],
"Discounted_Sales__slow_", 'Sales'[Discounted Sales (scalable)]
)
FIGURE 20-32 Server Timings running the query for Discounted Sales (scalable) by product brand for the fi rst time.
Actually, in this simple query the result is not faster at all. The query required 159 milliseconds
instead of the 142 milliseconds of the “slow” version. However, we called this measure “scalable.”
Indeed, the important advantage is that a second execution of the last query with a warm cache produces the results visible in Figure 20-33, whereas multiple executions of the query for the “slow” version
always produce a result similar to the one shown in Figure 20-31.
FIGURE 20-33 Server Timings running the query for Discounted Sales (scalable) by product brand a second time.
The Server Timings in Figure 20-33 show that there is no SE CPU cost after the fi rst execution of the
query. This is important when a model is published on a server and many users open the same reports:
Users experience a faster response time, and the memory and CPU workload on the server side is
reduced. This optimization is particularly relevant in environments with a fi xed reserved capacity, such
as Power BI Premium and Power BI Report Server.
The rule of thumb is to carefully consider the IF function in the expression of an iterator with a large
cardinality because of the possible presence of CallbackDataID in the storage engine queries. The next
section includes a deeper discussion on the impact of CallbackDataID, which might be required by
many other DAX functions used in iterators.

690 CHAPTER 20 Optimizing DAX
Note The SWITCH function in DAX is similar to a series of nested IF functions and can be
optimized in a similar way.
Reducing the impact of CallbackDataID
In Chapter 19, you saw that the CallbackDataID function in a storage engine query can have a huge
performance impact. This is because it slows down the storage engine execution, and it disables the
use of the storage engine cache for the datacache produced. Identifying the CallbackDataID is important because this is often the reason behind a bottleneck in the storage engine, especially for models
that only have a few million rows in their largest table (scan time should typically be in the order of
magnitude of 10–100 milliseconds).
For example, consider the following query where the Rounded Sales measure computes its result
rounding Unit Price to the nearest integer. The report in Figure 20-34 displays the Rounded Sales
amount for each product brand.
FIGURE 20-34 Rounded Sales reported by product brand.
The simpler implementation of Rounded Sales applies the ROUND function to every row of the
Sales table. This results in a CallbackDataID call, which slows down the execution, thus lowering performance. The following query computes the slowest Rounded Sales measure in the previous report,
generating the server timings results visible in Figure 20-35:

## Define


```dax
MEASURE Sales[Rounded Sales (slow)] =
SUMX (
Sales,
Sales[Quantity] * ROUND ( Sales[Net Price], 0 )
)
EVALUATE
```


## Topn (

502,

CHAPTER 20 Optimizing DAX 691

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),
"Rounded_Sales", 'Sales'[Rounded Sales (slow)]
),
[IsGrandTotalRowTotal], 0,
'Product'[Brand], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Product'[Brand]
FIGURE 20-35 Server Timings running the query for Rounded Sales (slow).
The two storage engine queries at lines 2 and 4 compute the value for each brand and for the grand
total, respectively. This is the xmSQL query at line 2 of Figure 20-35:

## With

$Expr0 := ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* [CallbackDataID ( ROUND ( Sales[Net Price]], 0 ) ) ]
( PFDATAID ( 'DaxBook Sales'[Net Price] ) ) )

## Select

'DaxBook Product'[Brand],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
The Sales table contains more than 12 million rows, and each storage engine query computes an
equivalent amount of CallbackDataID calls to execute the ROUND function. Indeed, the formula
engine executes the ROUND operation to remove the decimal part of the Unit Price value. Based on the
Server Timings report, we can estimate that the formula engine executes around 7,000 ROUND functions per millisecond. It is important to keep these numbers in mind, so that you can evaluate whether
or not the cardinality of an iterator generating CallbackDataID calls would benefi t from some amount
of optimization. If the table contained 12,000 rows instead of 12 million rows, the priority would be to
optimize something else. However, optimizing the measure in the current model requires reducing the
number of CallbackDataID calls.
We aim to reduce the number of CallbackDataID calls by refactoring the measure. By looking at the
information provided by VertiPaq Analyzer, we know that the Sales table has more than 12 million rows,
whereas the Net Price column in the Sales table has less than 2,500 unique values. Accordingly, the
formula can compute the same result by multiplying the rounded value of each unique Unit Price value
by the sum of Quantity for all the Sales transaction with the same Unit Price.

692 CHAPTER 20 Optimizing DAX
Note You should always use the statistics of your data model during DAX optimization.
A quick way to obtain these numbers for a data model is by using VertiPaq Analyzer
(http://www.sqlbi.com/tools/vertipaq-analyzer/).
The following optimized version of Rounded Sales materializes up to 2,500 rows computing the sum
of Quantity iterating the unique values of Unit Price:

## Define


```dax
MEASURE Sales[Rounded Sales (fast)] =
SUMX (
VALUES ( Sales[Net Price] ),
CALCULATE ( SUM ( Sales[Quantity] ) ) * ROUND ( Sales[Net Price], 0 )
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Brand], "IsGrandTotalRowTotal" ),
"Rounded_Sales", 'Sales'[Rounded Sales (fast)]
),
[IsGrandTotalRowTotal], 0,
'Product'[Brand], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Product'[Brand]
This way, the formula engine executes the ROUND function using the result of the datacache returning the sum of Quantity for each Net Price. Despite a larger materialization compared to the slow version, the time required to obtain the solution is reduced by almost one order of magnitude. Moreover,
the results provided by the storage engine queries can be reused in following executions because the
storage engine cache will store the result of xmSQL queries that do not have any CallbackDataID calls.
FIGURE 20-36 Server Timings running the query for Rounded Sales (fast).
The following is the xmSQL query at line 2 of Figure 20-36. This query returns the Net Price and the
sum of the Quantity for each brand and does not have any CallbackDataID calls:

CHAPTER 20 Optimizing DAX 693

## Select

'DaxBook Product'[Brand],
'DaxBook Sales'[Net Price],
SUM ( 'DaxBook Sales'[Quantity] )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
In this latter version, the rounding is executed by the formula engine and not by the storage engine
through the CallbackDataID. Be mindful that a very large number of unique values in Net Price would
require a bigger materialization, up to the point where the previous version could be faster with a different data distribution. If Net Price had millions of unique values, a benchmark comparison between
the two solutions would be required in order to determine the optimal solution. Moreover, the result
could be different depending on the hardware. Rather than assuming that one technique is better
than another, you should always evaluate the performance using a real database and not just a sample
before making a decision.
Finally, remember that most of the scalar DAX functions that do not aggregate data require a CallbackDataID if executed in an iterator. For example, DATE, VALUE, most of the type conversions, IFERROR, DIVIDE, and all the rounding, mathematical, and date/time functions are only implemented in the
formula engine. Most of the time, their presence in an iterator generates a CallbackDataID call. However, you always have to check the xmSQL query to verify whether a CallbackDataID is present or not.
Optimizing nested iterators
Nested iterators in DAX cannot be merged into a single storage engine query. Only the innermost
iterator can be executed using a storage engine query, whereas the outer iterators typically require
either a larger materialization or additional storage engine queries.
For example, consider another Cashback measure named “Cashback Sim.” that simulates a cashback
for each customer using the current price of each product multiplied by the historical quantity and the
cashback percentage of each customer. The report in Figure 20-37 displays the Cashback Sim. amount
for each country.
FIGURE 20-37 Cashback Sim. reported by customer country.
The fi rst and slowest implementation iterates the Customer and Product tables in order to retrieve
the cashback percentage of the customer and the current price of the product, respectively. The innermost iterators retrieve the quantity sold for each combination of customer and product, multiplying it

694 CHAPTER 20 Optimizing DAX
by Unit Price and Cashback %. The following query computes the slowest Cashback Sim. measure in the
previous report, generating the server timings results visible in Figure 20-38:

## Define


```dax
MEASURE Sales[Cashback Sim. (slow)] =
SUMX (
Customer,
SUMX (
'Product',
SUMX (
RELATEDTABLE ( Sales ),
Sales[Quantity] * 'Product'[Unit Price] * Customer[Cashback %]
)
)
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Customer'[Country], "IsGrandTotalRowTotal" ),
"Cashback Sim. (slow)", 'Sales'[Cashback Sim. (slow)]
),
[IsGrandTotalRowTotal], 0,
'Customer'[Country], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Customer'[Country]
FIGURE 20-38 Server Timings running the query for the Cashback Sim. (slow) measure reported by country.
The execution cost is split between the storage engine and the formula engine. The former pays a
big price to produce a large materialization, whereas the latter spends time consuming that large set
of materialized data. The storage engine queries at lines 2 and 10 of Figure 20-38 are identical and
materialize the following columns for the entire Sales table: CustomerKey, ProductKey, Quantity, and
RowNumber:

## Select

'DaxBook Customer'[CustomerKey],
'DaxBook Product'[ProductKey],
'DaxBook Sales'[RowNumber],
'DaxBook Sales'[Quantity]

CHAPTER 20 Optimizing DAX 695
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey]
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
The RowNumber is a special column inaccessible to DAX that is used to uniquely identify a row in
a table. These four columns are used in the formula engine to compute the formula in the innermost
iterator, which considers the sales for each combination of Customer and Product. The query at line
2 creates the datacache that is also returned at line 10, hitting the cache. The presence of this second
storage engine query is caused by the need to compute the grand total in SUMMARIZECOLUMNS.
Without the two levels of granularity in the result, half the query plan and half the storage engine queries would not be necessary.
The DAX measure iterates two tables (Customer and Product) producing all the possible combinations. For each combination of customer and product, the innermost SUMX function iterates only the
corresponding rows in Sales. The formula also considers the combinations of Customer and Product
that do not have any rows in the Sales table, potentially wasting precious CPU time. The query plan
shows that there are 2,517 products and 18,869 customers; these are the same numbers estimated for
the storage engine queries at lines 4 and 6 in Figure 20-38, respectively. Therefore, the formula engine
performs 1,326,280 aggregations of the rows materialized by the Sales table, as shown in the excerpt of
the query plan in Figure 20-39. The Records column shows the number of rows iterated by consumed
datacaches returned by storage engine queries (see the Cache nodes at lines 28, 33, and 36) or computed by other formula engine operations (see the CrossApply node at line 23).
FIGURE 20-39 Query Plan pane running the query for the Cashback Sim. (slow) measure reported by country.
Although the DAX code iterates the tables, the xmSQL code only retrieves the columns of the tables
uniquely representing one row of each table. This reduces the number of columns materialized, even
though the cardinality of the tables iterated is larger than necessary. At this point, there are two important considerations:

> **Note:** The cardinality of the iterators is larger than required. Thanks to the context transition, it is
possible to reduce the cardinality of the outer iterators; that way, the query context considers

696 CHAPTER 20 Optimizing DAX
all the rows in Sales for a given combination of Unit Price and Cashback %, instead of each combination of product and customer.

> **Note:** Removing nested iterators would produce a better query plan, also removing expensive
materialization.
The fi rst consideration should suggest applying the technique previously described to optimize the
context transitions. Indeed, the RELATEDTABLE function is like a CALCULATETABLE without fi lter arguments that only performs a context transition. The fi rst variation to the DAX measure is a “medium” version
that iterates the Cashback % and Unit Price columns, instead of iterating by Customer and Product. The
semantic of the query is still the same because the innermost expression only depends on these columns:

## Define


```dax
MEASURE Sales[Cashback Sim. (medium)] =
SUMX (
VALUES ( Customer[Cashback %] ),
SUMX (
VALUES ( 'Product'[Unit Price] ),
SUMX (
RELATEDTABLE ( Sales ),
Sales[Quantity] * 'Product'[Unit Price] * Customer[Cashback %]
)
)
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Customer'[Country], "IsGrandTotalRowTotal" ),
"Cashback Sim. (medium)", 'Sales'[Cashback Sim. (medium)]
),
[IsGrandTotalRowTotal], 0,
'Customer'[Country], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Customer'[Country]
Figure 20-40 shows that the execution of the “medium” version is orders of magnitude faster than
the “slow” version, thanks to a smaller granularity and a simpler dependency between tables iterated
and columns referenced.
FIGURE 20-40 Server Timings running the query for the Cashback Sim. (medium) measure reported by country.

CHAPTER 20 Optimizing DAX 697
The two storage engine queries provide a result for each of the cardinalities of the result. The following is the storage query at line 2, whereas the similar query at line 4 does not include the Country
column and is used for the grand total:

## With

$Expr0 := ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Product'[Unit Price] AS REAL ) )
* PFCAST ( 'DaxBook Customer'[Cashback] AS REAL ) )

## Select

'DaxBook Customer'[Country],
'DaxBook Customer'[Cashback],
'DaxBook Product'[Unit Price],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey]
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
The “medium” version of the Cashback Sim. measure still contains the same number of nested iterators, potentially considering all the possible combinations between the values of the Unit Price and
Cashback % columns. In this simple measure, the query plan is able to establish the dependencies on
the Sales table, reducing the calculation to the existing combinations. However, there is an alternative
DAX syntax to explicitly instruct the engine to only consider the existing combinations. Instead of using
nested iterators, a single iterator over the result of a SUMMARIZE enforces a query plan that does not
compute calculations over non-existing combinations. The following version named “improved” could
produce a more effi cient query plan in complex scenarios, even though in this example it generates the
same result and query plan:

```dax
MEASURE Sales[Cashback Sim. (improved)] =
SUMX (
SUMMARIZE (
Sales,
'Product'[Unit Price],
Customer[Cashback %]
),
CALCULATE ( SUM ( Sales[Quantity] ) )
* 'Product'[Unit Price] * Customer[Cashback %]
)
```

The “medium” and “improved” versions of the Cashback Sim. measure can easily be adapted to
use existing measures in the innermost calculations. Indeed, the “improved” version uses a CALCULATE function to compute the sum of Sales[Quantity] for a given combination of Unit Price and
Cashback %, just like a measure reference would. You should consider this approach to write effi cient
code that is easier to maintain. However, a more effi cient version is possible by removing any nested
iterators.

698 CHAPTER 20 Optimizing DAX
Note A measure defi nition often includes aggregation functions such as SUM. With the
exception of DISTINCTCOUNT, simple aggregation functions are just a shorter syntax for
an iterator. For example, SUM internally invokes SUMX. Hence, a measure reference in an
iterator often implies the execution of another nested iterator with a context transition in
the middle. When this is required by the nature of the calculation, this is a necessary computational cost. When the nested iterators are additive like the two nested SUMX/SUM of the
Cashback Sim. (improved) measure, then a consolidation of the calculation may be considered to optimize the performance; however, this could affect the readability and reusability
of the measure.
The following “fast” version of the Cashback Sim. measure optimizes the performance, at the cost of
reducing the ability to reuse the business logic of existing measures:

## Define


```dax
MEASURE Sales[Cashback Sim. (fast)] =
SUMX (
Sales,
Sales[Quantity]
* RELATED ( 'Product'[Unit Price] )
* RELATED ( Customer[Cashback %] )
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Customer'[Country], "IsGrandTotalRowTotal" ),
"Cashback Sim. (fast)", 'Sales'[Cashback Sim. (fast)]
),
[IsGrandTotalRowTotal], 0,
'Customer'[Country], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Customer'[Country]
Figure 20-41 shows the server timings information of the “fast” version, which saves more than 50%
of the execution time compared to the “medium” and “improved” versions.
FIGURE 20-41 Server Timings running the query for the Cashback Sim. (fast) measure reported by country.

CHAPTER 20 Optimizing DAX 699
The measure with a single iterator without context transitions generates the following simple
storage engine query, reported at line 2 of Figure 20-41:

## With

$Expr0 := ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Product'[Unit Price] AS REAL ) )
* PFCAST ( 'DaxBook Customer'[Cashback] AS REAL ) )

## Select

'DaxBook Customer'[Country],
SUM ( @$Expr0 )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Customer'
ON 'DaxBook Sales'[CustomerKey]='DaxBook Customer'[CustomerKey]
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey];
Using the RELATED function does not require any CallbackDataID. Indeed, the only consequence
of RELATED is that it enforces a join in the storage engine to enable the access to the related column,
which typically has a smaller performance impact compared to a CallbackDataID. However, the “fast”
version of the measure is not suggested unless it is critical to obtain the last additional performance
improvement and to keep the materialization at a minimal level.
Avoiding table fi lters for DISTINCTCOUNT
We already mentioned that fi lter arguments in CALCULATE/CALCULATETABLE functions should be
applied to columns instead of tables. The goal of this example on the same topic is to show you an
additional query plan pattern that you might fi nd in server timings. A side effect of a table fi lter is that
it requires a large materialization to the storage engine, to enable the formula engine to compute the
result. However, for non-additive expressions, the query plan might generate one storage engine query
for each element included in the granularity of the result. The DISTINCTCOUNT aggregation is a simple
and common example of a non-additive expression.
For example, consider the report in Figure 20-42 that shows the number of customers that made
purchases over $1,000 (Customers 1k) for each product name.
FIGURE 20-42 Customers with purchase amounts over $1,000 for each product.

700 CHAPTER 20 Optimizing DAX
The fi lter condition in the Customers 1k measure requires two columns. The less effi cient way to
implement such a condition is by using a fi lter over the Sales table. The following query computes
the Customers 1k measure in the previous report, generating the server timings results visible in
Figure 20-43:

## Define


```dax
MEASURE Sales[Customers 1k (slow)] =
CALCULATE (
DISTINCTCOUNT ( Sales[CustomerKey] ),
FILTER (
Sales,
Sales[Quantity] * Sales[Net Price] > 1000
)
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Product Name], "IsGrandTotalRowTotal" ),
"Customers_1k__slow_", 'Sales'[Customers 1k (slow)]
),
[IsGrandTotalRowTotal], 0,
'Product'[Product Name], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Product'[Product Name]
FIGURE 20-43 Server Timings running the query for the Customers 1k (slow) measure.
This query generates a large number of storage engine queries—one query for each product
included in the result. Because each storage engine query requires 100 to 200 milliseconds, there are a
total of several minutes of CPU cost, and the latency is below one minute just because of the parallelism
of the storage engine.
The fi rst xmSQL query at line 2 of Figure 20-43 returns the list of product names, including Quantity and Net Price for the sales transactions of that product. Indeed, even though there are only 1,091
products used at least once in the Sales table in transactions with an amount greater than $1,000, the

CHAPTER 20 Optimizing DAX 701
granularity of the datacache is larger because it also includes additional details other than the product
name, returning more rows for the same product:

## Select

'DaxBook Product'[Product Name],
'DaxBook Sales'[Quantity],
'DaxBook Sales'[Net Price]
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]

## Where

( COALESCE ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) ) )

## > Coalesce ( 1000.000000 )

);
There are 1,091 xmSQL queries that are very similar to the one at line 6 of Figure 20-43 and return
a single value obtained with a distinct count aggregation. In this case, the fi lter condition has all the
combinations of Quantity and Net Price that return a value greater than 1,000 for the Adventure Works
52″ LCD HDTV X790W Silver product:

## Select

DCOUNT ( 'DaxBook Sales'[CustomerKey] )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]

## Where

( COALESCE ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) ) )

## > Coalesce ( 1000.000000 )

)

## Vand (

'DaxBook Product'[Product Name],
'DaxBook Sales'[Quantity],
'DaxBook Sales'[Net Price] )

## In {

( 'Adventure Works 52" LCD HDTV X790W Silver', 2, 1592.200000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 4, 1432.980000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 1, 1273.760000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 3, 1480.746000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 4, 1512.590000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 3, 1592.200000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 3, 1353.370000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 4, 1273.760000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 1, 1480.746000 ) ,
( 'Adventure Works 52" LCD HDTV X790W Silver', 1, 1592.200000 )
..[24 total tuples, not all displayed]};

702 CHAPTER 20 Optimizing DAX
Indeed, the following xmSQL query at line 10 of Figure 20-43 only differs from the latter in the fi nal
fi lter condition, which includes valid combinations of Quantity and Net Price for the Contoso Washer &
Dryer 21in E210 Blue product:

## Select

DCOUNT ( 'DaxBook Sales'[CustomerKey] )
FROM 'DaxBook Sales'
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]

## Where

( COALESCE ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) ) )

## > Coalesce ( 1000.000000 )

)

## Vand (

'DaxBook Product'[Product Name],
'DaxBook Sales'[Quantity],
'DaxBook Sales'[Net Price] )

## In {

( 'Contoso Washer & Dryer 21in E210 Blue', 2, 1519.050000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 2, 1279.200000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 2, 1359.150000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 4, 1487.070000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 3, 1439.100000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 3, 1519.050000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 3, 1359.150000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 2, 1599.000000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 1, 1439.100000 ) ,
( 'Contoso Washer & Dryer 21in E210 Blue', 3, 1279.200000 )
..[24 total tuples, not all displayed]};
The presence of multiple similar storage engine queries is also visible in the Query Plan pane shown
in Figure 20-44. Each row starting at line 15 corresponds to a single datacache with just one column
produced by one of the storage engine queries described before.
FIGURE 20-44 Query Plan pane running the query for Customers 1k (slow).
The presence of the table fi lter applied to the fi lter context forces a query plan that is not effi cient.
In this case, a table fi lter produces multiple storage engine queries instead of a single large materialization. However, the optimization required is always the same: Column fi lters are better than table fi lters

CHAPTER 20 Optimizing DAX 703
in CALCULATE and CALCULATETABLE. The optimized version of the Customer 1k measure applies a fi lter
over the two columns Quantity and Net Price, using KEEPFILTERS in order to use the fi lter semantic of
the original measure. The following query produces the Server Timings results visible in Figure 20-45:

## Define


```dax
MEASURE Sales[Customers 1k (fast)] =
CALCULATE (
DISTINCTCOUNT ( Sales[CustomerKey] ),
KEEPFILTERS (
FILTER (
ALL (
Sales[Quantity],
Sales[Net Price]
),
Sales[Quantity] * Sales[Net Price] > 1000
)
)
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (

ROLLUPADDISSUBTOTAL ( 'Product'[Product Name], "IsGrandTotalRowTotal" ),
"Customers_1k__fast_", 'Sales'[Customers 1k (fast)]
),
[IsGrandTotalRowTotal], 0,
'Product'[Product Name], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Product'[Product Name]
FIGURE 20-45 Server Timings running the query for Customers 1k (fast).
The column fi lter in CALCULATE simplifi es the query plan, which now only requires two storage
engine queries—one for each granularity level of the result (one product versus total of all products).
The following is the xmSQL query at line 4 in Figure 20-45:

## Select

'DaxBook Product'[Product Name],
DCOUNT ( 'DaxBook Sales'[CustomerKey] )
FROM 'DaxBook Sales'

704 CHAPTER 20 Optimizing DAX
LEFT OUTER JOIN 'DaxBook Product'
ON 'DaxBook Sales'[ProductKey]='DaxBook Product'[ProductKey]

## Where

( COALESCE ( ( CAST ( PFCAST ( 'DaxBook Sales'[Quantity] AS INT ) AS REAL )
* PFCAST ( 'DaxBook Sales'[Net Price] AS REAL ) ) )

## > Coalesce ( 1000.000000 )

);
The datacache obtained corresponds to the result of the DAX query. The formula engine does not
have to do any further processing. This is an optimal condition for the performance of this query. The
lesson here is that the number of storage engine queries can also matter. A large number of storage
engine queries might be the result of a bad query plan. Non-additive measures combined with table
fi lters or bidirectional fi lters could be one of the reasons for this behavior, impacting performance in a
negative way.
Avoiding multiple evaluations by using variables
When a DAX expression evaluates the same subexpression multiple times, it is usually a good idea to
store the result of the subexpression in a variable, referencing the variable name in following parts of the
original DAX expression. The use of variables is a best practice which improves code readability and can
provide a better and more effi cient query plan—with just some exceptions described later in this section.
For example, the report in Figure 20-46 shows a Sales YOY % measure computing the percentage
difference between the value of Sales Amount displayed in the row of the report and the corresponding value in the previous year.
FIGURE 20-46 Difference in sales year over year reported by year and month.
The Sales YOY % measure uses other measures internally. In order to be able to modify each part of
the calculation, it is useful to include all the underlying measures using the Defi ne Dependent Measure
feature in DAX Studio. The following query computes the original Sales YOY % (slow) measure in the
previous report, generating the server timings results visible in Figure 20-47:

## Define


```dax
MEASURE Sales[Sales PY] =
CALCULATE (
```

CHAPTER 20 Optimizing DAX 705
[Sales Amount],
SAMEPERIODLASTYEAR ( 'Date'[Date] )
)

```dax
MEASURE Sales[Sales YOY (slow)] =
IF (
NOT ISBLANK ( [Sales Amount] ) && NOT ISBLANK ( [Sales PY] ),
[Sales Amount] - [Sales PY]
)
MEASURE Sales[Sales Amount] =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
MEASURE Sales[Sales YOY % (slow)] =
DIVIDE (
[Sales YOY (slow)],
[Sales PY]
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Date'[Calendar Year Month],
'Date'[Calendar Year Month Number]
), "IsGrandTotalRowTotal"
),
"Sales_YOY____slow_", 'Sales'[Sales YOY % (slow)]
),
[IsGrandTotalRowTotal], 0,
'Date'[Calendar Year Month Number], 1,
'Date'[Calendar Year Month], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Date'[Calendar Year Month Number],
'Date'[Calendar Year Month]
FIGURE 20-47 Server Timings running the query for the Sales YOY % (slow) measure.
The description of the query plan includes 1,819 rows, not reported here. Moreover, there are four
storage engine queries retrieved by the storage engine cache (SE Cache), even though we executed a
clear cache command before running the query. This indicates that different parts of the query plan

706 CHAPTER 20 Optimizing DAX
generate different requests for the same storage engine query. Although the cache improves the
performance of the storage engine request, the presence of such redundancy in the query plan is an
indicator that there is room for further improvements.
When a query plan is so complex and there are many storage engine queries, it is a good idea to
review the DAX code and reduce redundant evaluations by using variables. Indeed, redundant evaluations could be responsible for these duplicated requests. In general, the DAX engine should be able
to locate similar subexpressions executed within the same fi lter context, and reuse their results without
multiple evaluations. However, the presence of logical conditions such as IF and SWITCH creating
different branches of execution can easily stop this internal optimization.
For example, consider the Sales YOY (slow) measure implementation: the Sales Amount and Sales
PY measures are executed in different branches of the evaluation. The fi rst argument of the IF function
must always be evaluated, whereas the second argument should only be evaluated whenever the fi rst
argument evaluates to TRUE. A DAX expression that is present in both the fi rst and the second argument might be evaluated twice in the query plan, which might not consider the result obtained for the
fi rst argument as something that can be reused when evaluating the second argument. The technical
reasons why this happens and when it turns out to be preferable are outside the scope of this book.
The following excerpt of the previous query highlights the measure references that might be
evaluated twice because they are in both the fi rst and the second argument:

```dax
MEASURE Sales[Sales YOY (slow)] =
IF (
NOT ISBLANK ( [Sales Amount] ) && NOT ISBLANK ( [Sales PY] ),
[Sales Amount] - [Sales PY]
)
```

By storing the values returned by the two measures Sales Amount and Sales PY in two variables, it
is possible to instruct the DAX engine to enforce a single evaluation of the two measures before the IF
condition, reusing the result in both the fi rst and the second argument. The following excerpt of the
Sales YOY (fast) measure shows how to implement this technique in the DAX code:

```dax
MEASURE Sales[Sales YOY (fast)] =
VAR SalesPY = [Sales PY]
VAR SalesAmount = [Sales Amount]
RETURN
IF (
NOT ISBLANK ( SalesAmount ) && NOT ISBLANK ( SalesPY ),
SalesAmount - SalesPY
)
The following query includes a full implementation of the Sales YOY (fast) % measure, which internally relies on Sales YOY (fast) instead of Sales YOY (slow). The execution of the query produces the
```

server timings results visible in Figure 20-48:

## Define


```dax
MEASURE Sales[Sales PY] =
CALCULATE (
```

CHAPTER 20 Optimizing DAX 707
[Sales Amount],
SAMEPERIODLASTYEAR ( 'Date'[Date] )
)

```dax
MEASURE Sales[Sales YOY (fast)] =
VAR SalesPY = [Sales PY]
VAR SalesAmount = [Sales Amount]
RETURN
IF (
NOT ISBLANK ( SalesAmount ) && NOT ISBLANK ( SalesPY ),
SalesAmount - SalesPY
)
MEASURE Sales[Sales Amount] =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
MEASURE Sales[Sales YOY % (fast)] =
DIVIDE (
[Sales YOY (fast)],
[Sales PY]
)
EVALUATE
```


## Topn (

502,

## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Date'[Calendar Year Month],
'Date'[Calendar Year Month Number]
), "IsGrandTotalRowTotal"
),
"Sales_YOY____fast_", 'Sales'[Sales YOY % (fast)]
),
[IsGrandTotalRowTotal], 0,
'Date'[Calendar Year Month Number], 1,
'Date'[Calendar Year Month], 1
)

## Order By

[IsGrandTotalRowTotal] DESC,
'Date'[Calendar Year Month Number],
'Date'[Calendar Year Month]
FIGURE 20-48 Server Timings running the query for Sales YOY % (fast).

708 CHAPTER 20 Optimizing DAX
The description of the query plan includes 488 rows (not reported here), reducing the complexity
of the query plan by 73%; the previous query plan was 1,819 rows long. The new query plan reduces
the cost for the storage engine in terms of both execution time and number of queries, and it also
reduces the execution time in the formula engine. Overall, the optimized measure reduces the execution time by about 50%, but the optimization could be even bigger in more complex models and
expressions. If the same optimization were applied to nested measures, the improvement might be
exponential.
However, pay attention to possible side effects of assigning variables before conditional statements.
Only the subexpressions used in the fi rst argument can be assigned to variables defi ned before an IF
or SWITCH statement; otherwise, the effect could be the opposite, enforcing the evaluation of expressions that would otherwise be ignored. You should follow these guidelines:

> **Note:** When the same DAX expression is evaluated multiple times within the same fi lter context,
assign it to a variable and reference the variable instead of the DAX expression.

> **Note:** When a DAX expression is evaluated within the branches of an IF or SWITCH, whenever necessary assign the expression to a variable within the conditional branch.

> **Note:** Do not assign a variable outside an IF or SWITCH statement if the variable is only used within
the conditional branch.

> **Note:** The fi rst argument of IF and SWITCH can use variables defi ned before IF and SWITCH without it
affecting performance.
More examples about these guidelines are included in this article: https://www.sqlbi.com/articles/
optimizing-if-and-switch-expressions-using-variables/

Implementing alternative conditional statements
In the last example we used a simple IF statement to show a possible optimization using
variables. While using variables is a best practice, it is worth mentioning that there are
alternative ways to express the same conditional logic in DAX. For example, whenever an
IF function returns a numeric value and the expression of the second argument does not
raise an execution error when the condition of the fi rst argument is TRUE, it is possible to
convert this code:
IF ( <condition>, <expression> )
Into:
<expression> * <condition>
For example, the Sales YOY (fast) measure can be implemented using this expression:

```dax
MEASURE Sales[Sales YOY (fast)] =
( [Sales Amount] - [Sales PY] )
* ( NOT ISBLANK ( [Sales Amount] ) && NOT ISBLANK ( [Sales PY] ) )
```

CHAPTER 20 Optimizing DAX 709
The result produces only 208 rows in the query plan, despite a very similar query
duration. Nevertheless, in more complex models the reduction of the query plan might
have more visible benefi ts. However, different versions of the engine will tend to produce
different results. Consider this alternative coding style one of the options available in case
you need to further optimize your code. Do not apply such techniques without checking
the effects on performance and query plans, verifying whether they improve performance and whether they are worth reducing the readability of your code.
Conclusions
The lesson in this last chapter (to be honest, in the entire book) is that you must consider all the factors
that affect a query plan in order to fi nd the real bottleneck. Looking at the percentages of FE and SE
shown in server timings is a good starting point, but you should always investigate the reason behind
the numbers. Tools like DAX Studio and VertiPaq Analyzer provide you with the ability to measure the
effects of a bad query plan, but these are only clues and pieces of evidence pointing to the reasons for
a slow query.
Welcome to the DAX world!


Index
Numbers
1:1 relationships (data models), 2
A
active relationships
ambiguity, 514–515
CALCULATETABLE function, 451–453
expanded tables and, 450–453
USERELATIONSHIP function, 450–451
ADDCOLUMNS function, 223–224, 366–369,
371–372
ADDCOLUMNS iterators, 196–199
ADDMISSINGITEMS function
authoring queries, 419–420, 432–433
auto-exists feature (queries), 432–433
aggregation functions, xmSQL queries, 625–627
aggregations, 568–571
in data models, 587–588, 647–648

## Se, 548

VertiPaq aggregations, managing,
604–607
aggregators, 42, 43, 44, 45–46
AVERAGE function, 43–44
AVERAGEX function, 44
COUNT function, 46
COUNTA function, 46
COUNTBLANK function, 46
COUNTROWS function, 46
DISTINCTCOUNT function, 46
DISTINCTCOUNTNOBLANK function, 46
MAX function, 43
MIN function, 43
SUM function, 42–43, 44–45
SUMX function, 45
ALL function, 464–465
ALLEXCEPT function versus, 326–328
CALCULATE function and, 125–132, 164, 169–172
calculated physical relationships, circular
dependencies, 478
columns and, 64–65
computing percentages, 125–132
context transitions, avoiding, 328–330
evaluation contexts, 100–101
fi lter contexts, 324–326, 327–330
measures and, 63–64
nonworking days between two dates, computing,
523–525
percentages, computing, 63–64
syntax of, 63
top categories/subcategories example,
66–67
VALUES function and, 67, 327–328
ALL* functions, 462–464
ALLCROSSFILTERED function, 464, 465
ALLEXCEPT function, 65–66, 464, 465
ALL function versus, 326–328
computing percentages, 135
fi lter contexts, 326–328
VALUES function versus, 326–328
ALLNOBLANKROW function, 464, 465, 478
ALLSELECTED function, 74–75, 76, 455–457,
464, 465
CALCULATE function and, 171–172
computing percentages, 75–76
iterated rows, returning, 460–462
shadow fi lter contexts, 459–462
alternate/primary keys column (tables),
599, 600
ambiguity in relationships, 512–513
active relationships, 514–515
non-active relationships, 515–517
Analysis Services 2012/2014 and CallbackDataID
function, 644
annual totals (moving), computing, 243–244
arbitrarily shaped fi lters, 336
best practices, 343
building, 338–343

arbitrarily shaped fi lters
column fi lters versus, 336
defi ned, 337–338
simple fi lters versus, 337
uses of, 343
arithmetic operators, 23
error-handling
division by zero, 32–33
empty/missing values, 33–35
xmSQL queries, 627
arrows (cross fi lter direction), 3
attributes, data model optimization
disabling attribute hierarchies, 604
optimizing drill-through attributes, 604
authoring queries, 395
ADDMISSINGITEMS function, 419–420,
432–433
auto-exists feature, 428–434
DAX Studio, 395
DEFINE sections
MEASURE keyword in, 399
VAR keyword in, 397–399
EVALUATE statements
ADDMISSINGITEMS function, 419–420,
432–433
example of, 396
expression variables and, 398
GENERATE function, 414–417
GENERATEALL function, 417
GROUPBY function, 420–423
ISONORAFTER function, 417–419
NATURALINNERJOIN function, 423–425
NATURALLEFTOUTERJOIN function, 423–425
query variables and, 398
ROW function, 400–401
SAMPLE function, 427–428
SUBSTITUTEWITHINDEX function, 425–427
SUMMARIZE function, 401–403, 433–434
SUMMARIZECOLUMNS function, 403–409,
429–434
syntax of, 396–399
TOPN function, 409–414
TOPNSKIP function, 420
expression variables, 397–399
GENERATE function, 414–417
GENERATEALL function, 417
GROUPBY function, 420–423
ISONORAFTER function, 417–419
MEASURE in DEFINE sections, 399
measures
query measures, 399
testing, 399–401
NATURALINNERJOIN function, 423–425
NATURALLEFTOUTERJOIN function, 423–425
query variables, 397–399
ROW function, testing measures, 400–401
SAMPLE function, 427–428
shadow fi lter contexts, 457–462
SUBSTITUTEWITHINDEX function, 425–427
SUMMARIZE function, 401–403, 433–434
SUMMARIZECOLUMNS function, 403–409,
429–434
TOPN function, 409–414
TOPNSKIP function, 420
VAR in DEFINE sections, 397–399
Auto Date/Time (Power BI), 218–219
auto-exists feature (queries), 428–434
automatic date columns (Power Pivot for Excel), 219
AVERAGE function, 43–44, 199
AVERAGEA function, returning averages, 199
averages (means)
computing averages, AVERAGEX function,
199–201
moving averages, 201–202
returning averages
AVERAGE function, 199
AVERAGEA function, 199
AVERAGEX function, 44
computing averages, 199–201
fi lter contexts, 111–112
AVERAGEX iterators, 188
B
batch events (xmSQL queries), 630–632
bidirectional cross-fi lter direction (physical
relationships), 490, 491–493, 507
bidirectional fi ltering (relationships), 3–4
bidirectional relationships, 106, 109
Binary data type, 23
BLANK function, 36
blank rows, invalid relationships, 68–71
Boolean calculated columns, data model optimization,
597–598
Boolean conditions, CALCULATE function, 119–120,
123–124
Boolean data type, 22

calculation items
Boolean logic, 23
bottlenecks, DAX optimization, 667–668
identifying SE/FE bottlenecks, 667–668
optimizing bottlenecks, 668
bridge tables, MMR (Many-Many Relationships),
494–499
budget/sales information (calculations), showing
together, 527–530
C
CALCULATE function, 115
ALL function, 125–132, 164, 169–172
ALLSELECTED function, 171–172
Boolean conditions, 119–120, 123–124
calculated physical relationships, circular
dependencies, 478–480
calculation items, applying to expressions,
291–299
circular dependencies, 161–164
computing percentages, 124, 135
ALL function, 125–132
ALLEXCEPT function, 135
VALUES function, 133–134
context transitions, 148, 151–154
calculated columns, 154–157
measures, 157–160
CROSSFILTER function, 168
evaluation contexts, 79
evaluation order, 144–148
fi lter arguments, 118–119, 122, 123, 445–447
fi lter contexts, 148–151
fi ltering
multiple columns, 140–143
a single column, 138–140
KEEPFILTERS function, 135–138, 139–143, 164,
168–169
evaluation order, 146–148
fi ltering multiple columns, 142–143
moving averages, 201–202
numbering sequences of events (calculations),
537–538
overwriting fi lters, 120–122, 136
Precedence calculation group, 299–304
range-based relationships (calculated physical
relationships), 474–476
RELATED function and, 443–444
row contexts, 148–151
rules for, 172–173
semantics of, 122–123
syntax of, 118, 119–120
table fi lters, 382–384, 445–447
time intelligence calculations, 228–232
transferring fi lters, 482–483, 484–485
UNION function and, 376–378
USERELATIONSHIP function, 164–168
calculated columns, 25–26
Boolean calculated columns, data model
optimization, 597–598
context transitions, 154–157
data model optimization, 595–599
DISTINCT function, 68
expressions, 29
measures, 42
choosing between calculated columns and
measures, 29–30
differences between calculated columns and
measures, 29
using measures in calculated columns, 30
processing, 599
RELATED function, 443–444
SUM function, evaluation contexts, 88–89
table functions, 59
VALUES function, 68
calculated physical relationships, 471
circular dependencies, 476–480
multiple-column relationships, 471–473
range-based relationships, 474–476
calculated tables, 59
creating, 390–391
DISTINCT function, 68
SELECTCOLUMNS function, 390–391
VALUES function, 68
CALCULATETABLE function, 115, 363
active relationships, 451–453
FILTER function versus, 363–365
time intelligence functions, 259, 260–261
calculation granularity and iterators, 211–214
calculation groups, 279–281
calculation items and, 288
creating, 281–288
defi ned, 288
Name calculation group, 288
Precedence calculation group, 288, 299–304
calculation items
applying to expressions, 291
CALCULATE function, 291–299

calculation items
DATESYTD function, 293–296
YTD calculations, 294
best practices, 311
calculation groups and, 288
Expression calculation item, 289
format strings, 289–291
including/excluding measures from calculation
items, 304–306
Name calculation item, 288
Ordinal values, 289
properties of, 288–289
sideways recursion, 306–311
YOY calculation item, 289–290
YOY% calculation item, 289–290
calculations
budget/sales information (calculations), showing
together, 527–530
nonworking days between two dates, computing,
523–525
precomputing values (calculations), computing work
days between two dates, 525–527
sales
computing previous year sales up to last day sales
(calculations), 539–544
computing same-store sales, 530–536
showing budget/sales information together,
527–530
syntax of, 17–18
work days between two dates, computing,
519–523
nonworking days, 523–525
precomputing values (calculations), 525–527
CALENDAR function, building date tables, 222
CALENDARAUTO function, building date tables,
222–224
calendars (custom), time intelligence calculations,
DATESYTD function, 276–277
weeks, 272–275
CallbackDataID function
Analysis Services 2012/2014 and, 644
DAX optimization, 690–693
parallelism and, 641
VertiPaq and, 640–644
capturing DAX queries, 609–611
cardinality
columns (tables)
data model optimization, 591–592
optimizing high-cardinality columns, 603
iterators, 188–190
relationships (data models), 489–490, 586–587,
590–591
Cardinality column (VertiPaq Analyzer), 581, 583
categories/subcategories example, ALL function and,
66–67
cells (Excel), 5
chains (relationships), 3
circular dependencies
CALCULATE function and, 161–164
calculated physical relationships, 476–480
code documentation, variables, 183–184
code maintenance/readability, FILTER function, 62–63
column fi lters
arbitrarily shaped fi lters versus, 336
defi ned, 336
columnar databases, 550–553
columns (tables), 5–7
ADDCOLUMNS function, 223–224, 366–369,
371–372
ADDCOLUMNS iterators, 196–199
ALL function and, 64–65
ALLEXCEPT function and, 65–66
automatic date columns (Power Pivot for Excel), 219
Boolean calculated columns, data model
optimization, 597–598
calculated columns, 25–26, 42, 443–444
Boolean calculated columns, 597–598
choosing between calculated columns and
measures, 29–30
context transitions, 154–157
data model optimization, 595–599
differences between calculated columns and
measures, 29
DISTINCT function, 68
expressions, 29
processing, 599
SUM function, 88–89
table functions, 59
using measures in calculated columns, 30
VALUES function, 68
cardinality
data model optimization, 591–592
optimizing high-cardinality columns, 603
Date column, data model optimization, 592–595
defi ned, 2
descriptive attributes column (tables), 600,
601–602
fi ltering

CROSSFILTER function
CALCULATE function, 138–140
multiple columns, 140–143
a single column, 138–140
table fi lters versus, 444–447
measures, evaluation contexts, 89–90
multiple columns
DISTINCT function and, 71
VALUES function and, 71
primary/alternate keys column (tables),
599, 600
qualitative attributes column (tables),
599, 600
quantitative attributes column (tables), 599,
600–601
referencing, 17–18
relationships, 3
row contexts, 87
SELECTCOLUMNS function, 390–391, 393–394
SELECTCOLUMNS iterators, 196, 197–199
split optimization, 602–603
storage optimization, 602
column split optimization, 602–603
high-cardinality columns, 603
storing, 601–602
SUBSTITUTEWITHINDEX function, 425–427
SUMMARIZE function and, 401
SUMMARIZECOLUMNS function, 403–409,
429–434
technical attributes column (tables), 600, 602
Time column, data model optimization, 592–595
VertiPaq Analyzer, 580–583
Columns # column (VertiPaq Analyzer), 582
Columns Hierarchies Size column (VertiPaq
Analyzer), 582
Columns Total Size column (VertiPaq Analyzer), 581
COMBINEVALUES function, multiple-column
relationships (calculated physical relationships),
472–473
comments
at the end of expressions, 18
expressions, comment placement in expressions, 18
multi-line comments, 18
single-line comments, 18
comparison operators, 23
composite data models, 646–647
DirectQuery mode, 488
VertiPaq mode, 488
compression (VertiPaq), 553–554
hash encoding, 555–556
re-encoding, 559

## Rle, 556–559

value encoding, 554–555
CONCATENATEX function
iterators and, 194–196
tables as scalar values, 74
conditional statements, 24–25, 708–709
conditions

## Dax, 11


## Sql, 11

CONTAINS function
tables and, 387–388
transferring fi lters, 481–482, 484–485
CONTAINSROW function and tables, 387–388
context transitions, 148
ALL function and, 328–330
CALCULATE function and, 151–154
calculated columns, 154–157
DAX optimization, 672–678
expanded tables, 454–455
iterators, leveraging context transitions, 190–194
measures, 157–160
time intelligence functions, 260
conversion functions, 51
CURRENCY function, 51
DATE function, 51, 52
DATEVALUE function, 51
FORMAT function, 51
INT function, 51
TIME function, 51, 52
VALUE function, 51
conversions, error-handling, 31–32
cores (number of), VertiPaq hardware selection,
574, 576
COUNT function, 46
COUNTA function, 46
COUNTBLANK function, 46
COUNTROWS function, 46
fi lter contexts and relationships, 109
nested row contexts on the same table, 92–95
tables as scalar values, 73
CPU model, VertiPaq hardware selection, 574–575
cross-fi lter directions (physical relationships), 3, 490
bidirectional cross-fi lter direction, 490, 491–493, 507
single cross-fi lter direction, 490
cross-fi ltering, data model optimization, 590
cross-island relationships, 489
CROSSFILTER function
bidirectional relationships, 109
CALCULATE function and, 168

CROSSJOIN function and tables
CROSSJOIN function and tables, 372–374, 383–384
Currency data type, 21
CURRENCY function, 51
custom calendars, time intelligence calculations, 272
DATESYTD function, 276–277
weeks, 272–275
customers (new), computing (tables), 380–381,
386–387
D
Daily AVG
calculation group precedence, 299–303
calculation items, including/excluding measures,
304–306
data lineage, 332–336, 465–468
data models
aggregations, 647–648
composite data models, 646–647
DirectQuery mode, 488
VertiPaq mode, 488
defi ned, 1–2
optimizing with VertiPaq, 579
aggregations, 587–588, 604–607
calculated columns, 595–599
choosing columns for storage, 599–602
column cardinality, 591–592
cross-fi ltering, 590
Date column, 592–595
denormalizing data, 584–591
disabling attribute hierarchies, 604
gathering data model information,
579–584
optimizing column storage, 602–603
optimizing drill-through attributes, 604
relationship cardinality, 586–587, 590–591
Time column, 592–595
relationships, 2
1:1 relationships, 2
active relationships, 450–453
bidirectional fi ltering, 3–4
cardinality, 586–587, 590–591
chains, 3
columns, 3
cross fi lter direction, 3
DAX and SQL, 9
directions of, 3–4
many-sided relationships, 2, 3
one-sided relationships, 2, 3
Relationship reports (VertiPaq Analyzer), 584
unidirectional fi ltering, 4
weak relationships, 2
single data models
DirectQuery mode, 488
VertiPaq mode, 488
tables, defi ned, 2
weak relationships, 439
data refreshes, SSAS (SQL Server Analysis Services),
549–550
Data Size column (VertiPaq Analyzer), 581
data types, 19
Binary data type, 23
Boolean data type, 22
Currency data type, 21
DateTime data type, 21–22
Decimal data type, 21
Integer data type, 21
operators, 23
arithmetic operators, 23
comparison operators, 23
logical operators, 23
overloading, 19–20
parenthesis operators, 23
text concatenation operators, 23
string/number conversions, 19–21
strings, 22
Variant data type, 22
Database Size % column (VertiPaq Analyzer), 582
databases (columnar), 550–553
datacaches

## Fe, 547


## Se, 547

VertiPaq, 549, 635–637
DATATABLE function, creating static tables,
392–393
Date column, data model optimization, 592–595
DATE function, 51, 52
date table templates (Power Pivot for Excel), 220
date tables
building, 220–221
ADDCOLUMNS function, 223–224
CALENDAR function, 222
CALENDARAUTO function, 222–224
date templates, 224
duplicating, 227
loading from other data sources, 221

DAX (Data Analysis eXpressions)
Mark as Date Table, 232–233
multiple dates, managing, 224
multiple date tables, 226–228
multiple relationships to date tables,
224–226
naming, 221
date templates, 224
date/time-related calculations, 217
Auto Date/Time (Power BI), 218–219
automatic date columns (Power Pivot for
Excel), 219
basic calculations, 228–232
basic functions, 233–235
CALCULATE function, 228–232
CALCULATETABLE function, 259, 260–261
context transitions, 260
custom calendars, 272
DATESYTD function, 276–277
weeks, 272–275
date tables
ADDCOLUMNS function, 223–224
building, 220–224
CALENDAR function, 222
CALENDARAUTO function, 222–224
date table templates (Power Pivot for
Excel), 220
date templates, 224
duplicating, 227
loading from other data sources, 221
managing multiple dates, 224–228
Mark as Date Table, 232–233
multiple date tables, 226–228
multiple relationships to date tables, 224–226
naming, 221
DATEADD function, 237–238, 262–269
DATESINPERIOD function, 243–244
DATESMTD function, 259, 276–277
DATESQTD function, 259, 276–277
DATESYTD function, 259, 260, 261–262, 276–277
differences over previous periods, computing,
241–243
drillthrough operations, 271
FILTER function, 228–232
FIRSTDATE function, 269, 270
FIRSTNONBLANK function, 256–257, 270–271
LASTDATE function, 248–249, 254, 255, 269–270
LASTNONBLANK function, 250–254, 255, 270–271
mixing functions, 239–241
moving annual totals, computing, 243–244
MTD calculations, 235–236, 259–262, 276–277
nested functions, call order of, 245–246
NEXTDAY function, 245–246
nonworking days between two dates, computing,
523–525
opening/closing balances, 254–258
PARALLELPERIOD function, 238–239
periods to date, 259–262
PREVIOUSMONTH function, 239
QTD calculations, 235–236, 259–262, 276–277
SAMEPERIODLASTYEAR function, 237, 245–246
semi-additive calculations, 246–248
STARTOFQUARTER function, 256–257
time periods, computing from prior periods,
237–239
work days between two dates, computing,
519–523
nonworking days, 523–525
precomputing values (calculations),
525–527
YTD calculations, 235–236, 259–262, 276–277
DATEADD function, time intelligence calculations,
237–238, 262–269
DATESINPERIOD function, computing moving annual
totals, 243–244
DATESMTD function, time intelligence calculations,
259, 276–277
DATESQTD function, time intelligence calculations,
259, 276–277
DATESYTD function
calculation items, applying to expressions,
293–296
time intelligence calculations, 259, 260, 261–262,
276–277
DateTime data type, 21–22
DATEVALUE function, 51
DAX (Data Analysis eXpressions), 1
conditions, 11
data models
defi ned, 1–2
relationships, 2–4
tables, 2
date templates, 224
DAX and, cells and tables, 5–7
Excel and
functional languages, 7
theories, 8–9
expressions

DAX (Data Analysis eXpressions)
identifying a single DAX expression for
optimization, 658–661
optimizing bottlenecks, 668
as functional language, 10
functions, 6–7
iterators, 8

## Mdx, 12

hierarchies, 13–14
leaf-level calculations, 14
multidimensional versus tabular space, 12
as programming language, 12–13
as querying language, 12–13
queries, 613
optimizing, 657
bottlenecks, 668
CallbackDataID function, 690–693
change implementation, 668
conditional statements, 708–709
context transitions, 672–678
creating reproduction queries, 661–664
DISTINCTCOUNT function, 699–704
to-do list, 658
fi lter conditions, 668–672
identifying a single DAX expression for
optimization, 658–661
identifying SE/FE bottlenecks, 667–668
IF conditions, 678–690
multiple evaluations, avoiding with variables,
704–708
nested iterators, 693–699
query plans, 664–667
rerunning test queries, 668
server timings, 664–667
variables, 704–708
Power BI and, 14–15
as programming language, 10–11
queries
capturing, 609–611
creating reproduction queries, 661–662
DISTINCTCOUNT function, 634–635
executing, 546
query plans, 612–613
collecting, 613–614
DAX Studio, 617–620
logical query plans, 612, 614
physical query plans, 612–613, 614–616
SQL Server Profi ler, 620–623
as querying language, 10–11
SQL and, 9
subqueries, 11
DAX engines
DirectQuery, 546, 548, 549

## Fe, 546, 547

datacaches, 547
operators of, 547
single-threaded implementation, 547

## Se, 546

aggregations, 548
datacaches, 547
DirectQuery, 548, 549
operators of, 547
parallel implementations, 548
VertiPaq, 547–549, 550–577
Tabular model and, 545–546
VertiPaq, 546, 547–548, 550. See also data models,
optimizing with VertiPaq
aggregations, 571–573
columnar databases, 550–553
compression, 553–562
datacaches, 549

## Dmv, 563–565

hardware selection, 573–577
hash encoding, 555–556
hierarchies, 561–562
materialization, 568–571
multithreaded implementations, 548
partitioning, 562–563
processing tables, 550
re-encoding, 559
relationships (data models), 561–562,
565–568

## Rle, 556–559

scan operations, 549
segmentation, 562–563
sort orders, 560–561
value encoding, 554–555
DAX Studio, 395
capturing DAX queries, 609–611
Power BI and, 609–611
query measures, creating, 662–663
query plans, capturing profi ling information,
617–620
VertiPaq caches, 639–640
DAXFormatter.com, 41
Decimal data type, 21
DEFINE MEASURE clauses in EVALUATE statements, 59

evaluation contexts

```dax
DEFINE sections (authoring queries)
MEASURE keyword in, 399
VAR keyword in, 397–399
```

denormalizing data and data model optimization,
584–591
descriptive attributes column (tables), 600, 601–602
DETAILROWS function, reusing table expressions,
388–389
dictionary encoding. See hash encoding
Dictionary Size column (VertiPaq Analyzer), 581
DirectQuery, 488–489, 546, 548, 549, 617
calculated columns, 25–26
composite data models, 488
End events (SQL Server Profi ler), 621

## Se, 549

composite data models, 646–647
reading, 645–646
single data models, 488
Disk I/O performance, VertiPaq hardware selection,
574, 576–577
DISTINCT function, 71
blank rows and invalid relataionships, 68, 70–71
calculated columns, 68
calculated physical relationships
circular dependencies, 477–478
range-based relationships, 476
fi lter contexts, 111–112
multiple columns, 71
UNION function and, 375–378
VALUES function versus, 68
DISTINCTCOUNT function, 46
DAX optimization, 699–704
same-store sales (calculations), computing,
535–536
table fi lters, avoiding, 699–704
VertiPaq SE queries, 634–635
DISTINCTCOUNTNOBLANK function, 46
DIVIDE function, DAX optimization, 684–687
division by zero, arithmetic operators, 32–33
DMV (Dynamic Management Views) and SSAS,
563–565
documenting code, variables, 183–184
drill-through attributes, optimizing, 604
drillthrough operations, time intelligence calculations,
duplicating, date tables, 227
duration of an order example, 26
dynamic segmentation, virtual relationships and,
485–488
E
EARLIER function, evaluation contexts, 97–98
editing text, formatting DAX code, 42
empty/missing values, error-handling, 33–35
Encoding column (VertiPaq Analyzer), 582, 583
error-handling
BLANK function, 36
Excel, empty/missing values, 35
expressions, 31
arithmetic operator errors, 32–35
conversion errors, 31–32
generating errors, 38–39
IF function, 36, 37
IFERROR function, 35–36, 37–38
ISBLANK function, 36
ISERROR function, 36, 38
SQRT function, 36
variables, 37
EVALUATE statements
ADDMISSINGITEMS function, 419–420,
432–433
DEFINE MEASURE clauses, 59
example of, 396
expression variables and, 398
GENERATE function, 414–417
GENERATEALL function, 417
GROUPBY function, 420–423
ISONORAFTER function, 417–419
NATURALINNERJOIN function, 423–425
NATURALLEFTOUTERJOIN function, 423–425
ORDER BY clauses, 60
query variables and, 398
ROW function, 400–401
SAMPLE function, 427–428
SUBSTITUTEWITHINDEX function, 425–427
SUMMARIZE function, 401–403, 433–434
SUMMARIZECOLUMNS function, 403–409,
429–434
syntax of, 59–60, 396–399
TOPN function, 409–414
TOPNSKIP function, 420
evaluation contexts, 79
ALL function, 100–101
AVERAGEX function, fi lter contexts, 111–112
CALCULATE function, 79
columns in measures, 89–90
COUNTROWS function, fi lter contexts and
relationships, 107–108
defi ned, 80

evaluation contexts
DISTINCT function, fi lter contexts, 111–112
EARLIER function, 97–98
fi lter contexts, 80, 109–110
AVERAGEX function, 111–112
CALCULATE function, 118–119
CALCULATE function and, 148–151
creating, 115–119
DISTINCT function, 111–112
examples of, 80–85
fi lter arguments, 118–119
relationships and, 106–109
row contexts versus, 85
SUMMARIZE function, 112
FILTER function, 92–93, 94–95, 98–101
multiple tables, working with, 101–102
fi lter contexts and relationships, 106–109
row contexts and relationships, 102–105
RELATED function
fi lter contexts and relationships, 109
nested row contexts on different tables, 92
row contexts and relationships, 103–105
RELATEDTABLE function
fi lter contexts and relationships, 109
nested row contexts on different tables,
91–92
row contexts and relationships, 103–105
relationships and, 101–102
fi lter contexts, 106–109
row contexts, 102–105
row contexts, 80
CALCULATE function and, 148–151
column references, 87
examples of, 86–87
fi lter contexts versus, 85
iterators and, 90–91
nested row contexts on different tables,
91–92
nested row contexts on the same table, 92–97
relationships and, 102–105
SUM function, in calculated columns, 88–89
SUMMARIZE function, fi lter contexts, 112
evaluations (multiple), avoiding with variables,
704–708
events (calculations), numbering sequences of, 536–539
Excel
calculations, 8
cells, 5
columns, 5–7
DAX and
cells and tables, 5–7
functional languages, 7
theories, 8–9
error-handling, empty/missing values, 35
formulas, 6
functions, 6–7
Power Pivot for Excel
automatic date columns, 219
date table templates, 220
EXCEPT function, tables and, 379–381
expanded tables
active relationships, 450–453
column fi lters versus table fi lters, 444–447
context transitions, 454–455
fi lter contexts, 439–441
fi ltering, 444–447
active relationships and, 450–453
differences between table fi lters and expanded
tables, 453–454
RELATED function, 441–444
relationships, 437–441
table fi lters
column fi lters versus, 444–447
in measures, 447–450
Expression calculation item, 289
Expression Trees, 612
expressions
calculated columns, 29
calculation items, applying to expressions, 291
CALCULATE function, 291–299
DATESYTD function, 293–296
YTD calculations, 294
comments, placement in expressions, 18
DAX optimization, 658–661, 668
error-handling, 31
arithmetic operator errors, 32–35
conversion errors, 31–32
formatting, 39–40, 42
MDX
DAX and, 12–13, 14
queries, 546, 604, 613, 663–664
query measures, 399
scalar expressions, 57–58
table expressions
EVALUATE statements, 59–60
reusing, 388–389
variables, 30–31, 397–399

fi ltering
F
FE (Formula Engines), 546, 547
bottlenecks, identifying, 667–668
datacaches, 547
operators of, 547
query plans, reading, 652–653, 654–655
single-threaded implementation, 547, 642
fi lter arguments
CALCULATE function, 118–119, 122, 123,
445–447
defi ned, 120
multiple column references, 140
SUMMARIZECOLUMNS function, 406–409
fi lter contexts, 80, 109–110, 313, 343–344
ALL function, 324–326, 327–330
ALLEXCEPT function, 326–328
arbitrarily shaped fi lters, 336
best practices, 343
building, 338–343
column fi lters versus, 336
defi ned, 337–338
simple fi lters versus, 337
uses of, 343
AVERAGEX function, 111–112
CALCULATE function, 148–151
fi lter arguments, 118–119
overwriting fi lters, 120–122
column fi lters
arbitrarily shaped fi lters versus, 336
defi ned, 336
creating, 115–119
data lineage, 332–336
DISTINCT function, 111–112
examples of, 80–85
expanded tables, 439–441
FILTERS function, 322–324
HASONVALUE function, 314–318
ISCROSSFILTERED function, 319–322
ISEMPTY function, 330–332
ISFILTERED function, 319, 320–322
nesting in variables, 184–185
relationships and, 106–109
row contexts versus, 85
SELECTEDVALUE function, 318–319
simple fi lters
arbitrarily shaped fi lters versus, 337
defi ned, 337
SUMMARIZE function, 112
TREATAS function, 334–336
VALUES function, 322–324, 327–328
FILTER function, 57–58
CALCULATETABLE function versus, 363–365
code maintenance/readability, 62–63
evaluation contexts, 98–101
as iterator, 60–61
nested row contexts on the same table,
92–93, 94–95
nesting, 61–62
range-based relationships (calculated physical
relationships), 474–476
syntax of, 60
time intelligence calculations, 228–232
transferring fi lters, 481–482, 484–485
fi lter operations, xmSQL queries, 628–630
fi ltering
ALLCROSSFILTERED function, 464, 465
columns (tables) versus table fi lters, 444–447
DAX optimization, fi lter conditions, 668–672
expanded tables
differences between table fi lters and expanded
tables, 453–454
table fi lters and active relationships,
450–453
FILTER function
range-based relationships (calculated physical
relationships), 474–476
transferring fi lters, 484–485
KEEPFILTERS function, 461–462, 482–483, 484
relationships
bidirectional fi ltering, 3–4
unidirectional fi ltering, 4
shadow fi lter contexts, 457–462
tables, 381
CALCULATE function and, 445–447
column fi lters versus, 444–447
differences between table fi lters and expanded
tables, 453–454
DISTINCTCOUNT function, 699–704
in measures, 447–450
OR conditions, 381–384
table fi lters and active relationships,
450–453
transferring fi lters, 480–481
CALCULATE function, 482

fi ltering
CONTAINS function, 481–482
FILTER function, 481–482, 484–485
INTERSECT function, 483–484
TREATAS function, 482–483, 484
FILTERS function
fi lter contexts, 322–324
VALUES function versus, 322–324
FIRSTDATE function, time intelligence calculations,
269, 270
FIRSTNONBLANK function, time intelligence calculations,
256–257, 270–271
FORMAT function, 51
format strings
calculation items and, 289–291
defi ned, 291
SELECTEDMEASUREFORMATSTRING function,
formatting DAX code, 39, 41–42
DAXFormatter.com, 41
editing text, 42
expressions, 39–40, 42
formulas, 42
help, 42
variables, 40–41
formulas
Excel, 6
formatting, 42
IN function, tables and, 387–388
functions
ADDCOLUMNS function, 223–224, 366–369,
371–372
ADDMISSINGITEMS function
authoring queries, 419–420, 432–433
auto-exists feature (queries), 432–433
aggregation functions, xmSQL queries,
625–627
aggregators, 42, 44, 45–46
AVERAGE function, 43–44
AVERAGEX function, 44
COUNT function, 46
COUNTA function, 46
COUNTBLANK function, 46
COUNTROWS function, 46
DISTINCTCOUNT function, 46
DISTINCTCOUNTNOBLANK function, 46
MAX function, 43
MIN function, 43
SUM function, 42–43, 44–45
SUMX function, 45
ALL function, 464–465
ALLEXCEPT function versus, 326–328
CALCULATE function and, 164, 169–172
calculated physical relationships and circular
dependencies, 478
computing nonworking days between two dates,
523–525
computing percentages, 125–132
context transitions, 328–330
evaluation contexts, 100–101
fi lter contexts, 324–326, 327–330
VALUES function and, 327–328
ALL* functions, 462–464
ALLCROSSFILTERED function, 464, 465
ALLEXCEPT function, 464, 465
ALL function versus, 326–328
computing percentages, 135
fi lter contexts, 326–328
VALUES function versus, 326–328
ALLNOBLANKROW function, 464, 465, 478
ALLSELECTED function, 455–457, 464, 465
CALCULATE function and, 171–172
returning iterated rows, 460–462
shadow fi lter contexts, 459–462
AVERAGE function, returning averages,
AVERAGEA function, returning averages, 199
AVERAGEX function
computing averages, 199–201
fi lter contexts, 111–112
Boolean conditions, 123–124
CALCULATE function, 115
ALL function, 125–132, 164, 169–172
ALLSELECTED function, 171–172
Boolean conditions, 119–120
calculated physical relationships and circular
dependencies, 478–480
calculation items, applying to expressions,
291–299
circular dependencies, 161–164
computing percentages, 124–135
context transitions, 148, 151–160
CROSSFILTER function, 168
evaluation contexts, 79
evaluation order, 144–148

functions
fi lter arguments, 118–119, 122, 123,
445–447
fi lter contexts, 148–151
fi ltering a single column, 138–140
fi ltering multiple columns, 140–143
KEEPFILTERS function, 135–138,
139–143, 164, 168–169
KEEPFILTERS function and, 146–148
moving averages, 201–202
numbering sequences of events (calculations),
537–538
overwriting fi lters, 120–122
Precedence calculation group, 299–304
range-based relationships (calculated physical
relationships), 474–476
RELATED function and, 443–444
row contexts, 148–151
rules for, 172–173
semantics of, 122–123
syntax of, 118, 119–120
table fi lters, 445–447
tables as fi lters, 382–384
time intelligence calculations, 228–232
transferring fi lters, 482–483, 484–485
UNION function and, 376–378
USERELATIONSHIP function, 164–168
CALCULATETABLE function, 115, 363
active relationships, 451–453
FILTER function versus, 363–365
time intelligence functions, 259, 260–261
CALENDAR function, date tables, 222
CALENDARAUTO function, date tables, 222–224
CallbackDataID function
Analysis Services 2012/2014 and, 644
DAX optimization, 690–693
parallelism and, 641
VertiPaq and, 640–644
COMBINEVALUES function, multiple-column
relationships (calculated physical relationships),
472–473
CONCATENATEX function
iterators and, 194–196
tables as scalar values, 74
CONTAINS function
tables and, 387–388
transferring fi lters, 481–482, 484–485
CONTAINSROW function, tables and, 387–388
conversion functions, 51
COUNTROWS function
fi lter contexts and relationships, 107–108
nested row contexts on the same table,
92–95
tables as scalar values, 73
CROSSFILTER function
bidirectional relationships, 109
CALCULATE function and, 168
CROSSJOIN function, tables and, 372–374,
383–384
CURRENCY function, 51
DATATABLE function, creating static tables,
392–393
DATE function, 51, 52
DATEADD function, time intelligence calculations,
237–238, 262–269
DATESINPERIOD function, moving annual totals,
243–244
DATESMTD function, time intelligence calculations,
259, 276–277
DATESQTD function, time intelligence calculations,
259, 276–277
DATESYTD function
calculation items, applying to expressions,
293–296
time intelligence calculations, 259, 260, 261–262,
276–277
DATEVALUE function, 51
DETAILROWS function, reusing table expressions,
388–389
DISTINCT function
calculated physical relationships and circular
dependencies, 477–478
fi lter contexts, 111–112
range-based relationships (calculated physical
relationships), 476
UNION function and, 375–378
DISTINCTCOUNT function
avoiding table fi lters, 699–704
computing same-store sales, 535–536
DAX optimization, 699–704
DIVIDE function, DAX optimization, 684–687
EARLIER function, evaluation contexts, 97–98
Excel, 6–7
EXCEPT function, tables and, 379–381
FILTER function
CALCULATETABLE function versus,
363–365
evaluation contexts, 98–101

functions
nested row contexts on the same table, 92–93,
94–95
range-based relationships (calculated physical
relationships), 474–476
time intelligence calculations, 228–232
transferring fi lters, 481–482, 484–485
FILTERS function
fi lter contexts, 322–324
VALUES function versus, 322–324
FIRSTDATE function, time intelligence calculations,
269, 270
FIRSTNONBLANK function, time intelligence
calculations, 256–257, 270–271
FORMAT function, 51
IN function, tables and, 387–388
GENERATE function, authoring queries, 414–417
GENERATEALL function, authoring queries, 417
GENERATESERIES function, tables and, 393–394
GROUPBY function
authoring queries, 420–423
SUMMARIZE function and, 420–423
HASONEVALUE function
fi lter contexts, 314–318
tables as scalar values, 73
information functions, 48–49
INT function, 51
INTERSECT function
tables and, 378–379
transferring fi lters, 483–484
ISCROSSFILTERED function, fi lter contexts,
319–322
ISEMPTY function, fi lter contexts, 330–332
ISFILTERED function
fi lter contexts, 319, 320–322
time intelligence calculations, 268–269
ISNUMBER function, 48–49
ISONORAFTER function
authoring queries, 417–419
TOPN function and, 417–419
ISSELECTEDMEASURE function, including/excluding
measures from calculation items, 304–306
ISSUBTOTAL function and SUMMARIZE function,
402–403
KEEPFILTERS function, 461–462
CALCULATE function and, 135–138, 142–143,
146–148, 164, 168–169
evaluation order, 146–148
transferring fi lters, 482–483, 484
LASTDATE function, time intelligence calculations,
248–249, 254, 255, 269–270
LASTNONBLANK function, 250–254, 255, 270–271
logical functions
IF function, 46–47
IFERROR function, 47
SWITCH function, 47–48
LOOKUPVALUE function, 444, 473
mathematical functions, 49
NATURALINNERJOIN function, authoring queries,
423–425
NATURALLEFTOUTERJOIN function, authoring
queries, 423–425
nested functions, call order of time intelligence
functions, 245–246
NEXTDAY function, call order of nested time
intelligence functions, 245–246
PARALLELPERIOD function, time intelligence
calculations, 238–239
PREVIOUSMONTH function, time intelligence
calculations, 239
RANK.EQ function, 210
RANKX function, numbering sequences of events
(calculations), 538–539
RELATED function
CALCULATE function and, 443–444
calculated columns, 443–444
context transitions in expanded tables, 455
expanded tables, 441–444
fi lter contexts and relationships, 109
nested row contexts on different tables, 92
row contexts and relationships, 103–105
table fi lters and expanded tables, 454
RELATEDTABLE function
fi lter contexts and relationships, 109
nested row contexts on different tables,
91–92
row contexts and relationships, 103–105
relational functions, 53–54
ROLLUP function, 401–402, 403
ROW function
creating static tables, 391–392
testing measures, 400–401
SAMEPERIODLASTYEAR function
call order of nested time intelligence functions,
245–246
computing previous year sales up to last day sales
(calculations), 540–544
time intelligence calculations, 237

granularity
SAMPLE function, authoring queries,
427–428
SELECTCOLUMNS function, 390–391, 393–394
SELECTEDMEASURE function, including/excluding
measures from calculation items, 304–306
SELECTEDMEASUREFORMATSTRING function, 291
SELECTEDVALUE function
calculated physical relationships and circular
dependencies, 479–480
computing same-store sales, 533–534
context transitions in expanded tables,
454–455
fi lter contexts, 318–319
tables as scalar values, 73–74
STARTOFQUARTER function, time intelligence
calculations, 256–257
SUBSTITUTEWITHINDEX function, authoring queries,
425–427
SUM function in calculated columns, 88–89
SUMMARIZE function
authoring queries, 401–403, 433–434
auto-exists feature (queries), 433–434
columns (tables) and, 401
fi lter contexts, 112
GROUPBY function and, 420–423
ISSUBTOTAL function and, 402–403
ROLLUP function and, 401–402, 403
table fi lters and expanded tables, 453–454
tables and, 369–372, 373–374, 383–384
transferring fi lters, 484–485
SUMMARIZECOLUMNS function
authoring queries, 403–409, 429–434
auto-exists feature (queries), 429–434
fi lter arguments, 406–409
IGNORE modifi er, 403–404
ROLLUPADDISSUBTOTAL modifi er, 404–406
ROLLUPGROUP modifi er, 406
TREATAS function and, 407–408
table functions, 57
ALL function, 63–65, 66–67
ALLEXCEPT function, 65–66
ALLSELECTED function, 74–76
calculated columns and, 59
calculated tables, 59
DISTINCT function, 68, 70–71
FILTER function, 57–58, 60–63
measures and, 59
nesting, 58–59
RELATEDTABLE function, 58–59
VALUES function, 67–74
text functions, 50–51
TIME function, 51, 52
time intelligence functions (nested), call order of,
245–246
TOPN function
authoring queries, 409–414
ISONORAFTER function and, 417–419
sort order, 410
TOPNSKIP function, authoring queries, 420
TREATAS function, 378
data lineage, 467–468
fi lter contexts and data lineage, 334–336
SUMMARIZECOLUMNS function and, 407–408
transferring fi lters, 482–483, 484
UNION function and, 377–378
trigonometric functions, 50
UNION function
CALCULATE function and, 376–378
DISTINCT function and, 375–378
tables and, 374–378
TREATAS function and, 377–378
USERELATIONSHIP function
active relationships, 450–451
CALCULATE function and, 164–168
non-active relationships and ambiguity, 516–517
VALUE function, 51
VALUES function
ALL function and, 327–328
ALLEXCEPT function versus, 326–328
calculated physical relationships and circular
dependencies, 477–480
computing percentages, 133–134
fi lter contexts, 322–324, 327–328
FILTERS function versus, 322–324
range-based relationships (calculated physical
relationships), 474–476
G
GENERATE function, authoring queries, 414–417
GENERATEALL function, authoring queries, 417
GENERATESERIES function, tables and, 393–394
generating errors (error-handling), 38–39
granularity
calculations and iterators, 211–214
relationships (data models), 507–512

GROUPBY function
GROUPBY function
authoring queries, 420–423
SUMMARIZE function and, 420–423
H
hash encoding (VertiPaq compression),
555–556
HASONEVALUE function
fi lter contexts, 314–318
tables as scalar values, 73
help, formatting DAX code, 42
hierarchies, 345, 362
attribute hierarchies (data model optimization),
disabling, 604
Columns Hierarchies Size column (VertiPaq
Analyzer), 582

## Dax, 13–14


## Mdx, 13–14

P/C (Parent/Child) hierarchies, 350–361, 362
percentages, computing, 345
IF conditions, 349
PercOnCategory measures, 348
PercOnParent measures, 346–349
ratio to parent calculations, 345
SSAS and, 561–562
Use Hierarchies Size column (VertiPaq Analyzer),
I
IF conditions
computing percentages over hierarchies, 349
DAX optimization, 678–679
DIVIDE function and, 684–687
iterators, 687–690
in measures, 679–683
IF function, 36, 37, 46–47
IFERROR function, 35–36, 37–38, 47
IGNORE modifi er, SUMMARIZECOLUMNS function,
403–404
information functions, 48–49
INT function, 51
Integer data type, 21
INTERSECT function
tables and, 378–379
transferring fi lters, 483–484
intra-island relationships, 489
invalid relationships, blank rows and, 68–71
ISBLANK function, 36
ISCROSSFILTERED function, fi lter contexts,
319–322
ISEMPTY function, fi lter contexts, 330–332
ISERROR function, 36, 38
ISFILTERED function
fi lter contexts, 319, 320–322
time intelligence calculations, 268–269
ISNUMBER function, 48–49
ISONORAFTER function
authoring queries, 417–419
TOPN function and, 417–419
ISSELECTEDMEASURE function, including/excluding
measures from calculation items, 304–306
ISSUBTOTAL function, 402–403
iterators, 8, 43, 44, 209–215
ADDCOLUMNS iterators, 196–199
averages (means)
computing with AVERAGEX function,
199–201
moving averages, 201–202
returning with AVERAGE function, 199
returning with AVERAGEA function, 199
AVERAGEX iterators, 188
behavior of, 91
calculation granularity, 211–214
cardinality, 188–190
CONCATENATEX function and, 194–196
context transitions, leveraging, 190–194
DAX optimization
IF conditions, 687–690
nested iterators, 693–699
FILTER function as, 60–61
nested iterators
DAX optimization, 693–699
leveraging context transitions, 190–194
parameters of, 187–188
RANK.EQ function, 210
RANKX iterators, 188, 202–210
ROW CONTEXT iterators, 187–188
row contexts and, 90–91
SELECTCOLUMNS iterators, 196, 197–199
SUMX iterators, 187–188
tables, returning, 196–199
J
join operators, xmSQL queries, 628–630

MIN function
K
KEEPFILTERS function, 461–462
CALCULATE function and, 135–138, 139–143, 164,
168–169
evaluation order, 146–148
fi ltering multiple columns, 142–143
transferring fi lters, 482–483, 484
L
last day sales (calculations), computing previous year
sales up to, 539–544
LASTDATE function, time intelligence calculations,
248–249, 254, 255, 269–270
LASTNONBLANK function, time intelligence calculations,
250–254, 255, 270–271
lazy evaluations, variables, 181–183
leaf-level calculations

## Dax, 14


## Mdx, 14

leap year bug, 22
list of values. See fi lter arguments
logical functions
IF function, 46–47
IFERROR function, 47
SWITCH function, 47–48
logical operators, 23
logical query plans, 612, 614, 650–651
LOOKUPVALUE function, 444, 473
M

```dax
maintenance (code), FILTER function, 62–63
many-sided relationships (data models), 2, 3
```

many-to-many relationships. See MMR
Mark as Date Table, 232–233
materialization (queries), 568–571
mathematical functions, 49
MAX function, 43
MDX (Multidimensional Expressions)
DAX and, 12
hierarchies, 13–14
leaf-level calculations, 14
multidimensional versus tabular space, 12
as programming language, 12–13
as querying language, 12–13
queries, 546
attribute hierarchies (data model optimization),
disabling, 604
DAX and, 613
executing, 546
reproduction queries, creating, 663–664
means (averages)
computing averages, AVERAGEX function,
199–201
moving averages, 201–202
returning averages
AVERAGE function, 199
AVERAGEA function, 199

```dax
MEASURE keyword, DEFINE sections (authoring
queries), 399
```

measures, 26–28
ALL function and, 63–64
calculated columns, 42
choosing between calculated columns and
measures, 29–30
differences between calculated columns and
measures, 29
using measures in calculated columns, 30
calculation items, including/excluding measures from,
304–306
columns in, evaluation contexts, 89–90
context transitions, 157–160
DEFINE MEASURE clauses in EVALUATE
statements, 59
defi ning in tables, 29
expressions, 29
IF conditions, DAX optimization, 679–683
ISSELECTEDMEASURE function, including/excluding
measures from calculation items, 304–306
PercOnCategory measures, computing percentages
over hierarchies, 348
PercOnParent measures, computing percentages
over hierarchies, 346–349
query measures, 399, 662–663
SELECTEDMEASURE function, including/excluding
measures from calculation items, 304–306
table fi lters in, 447–450
table functions, 59
testing, 399–401
VALUES function and, 67–68
memory size, VertiPaq hardware selection,
574, 576
memory speed, VertiPaq hardware selection, 574,
575–576
MIN function, 43

MMR (Many-Many Relationships)
MMR (Many-Many Relationships), 489, 490, 494, 507
bridge tables, 494–499
common dimensionality, 500–504
weak relationships, 504–506
moving annual totals, computing, 243–244
moving averages, CALCULATE function, 201–202
MTD (Month-to-Date) calculations, time intelligence
calculations, 235–236, 259–262, 276–277
multi-line comments, 18
multiple columns
DISTINCT function and, 71
multiple-column relationships (calculated physical
relationships), 471–473
VALUES function and, 71
MultipleItemSales variable, 58
N
Name calculation group, 288
Name calculation item, 288
naming variables, 182
narrowing table computations, 384–386
NATURALINNERJOIN function, authoring queries,
423–425
NATURALLEFTOUTERJOIN function, authoring queries,
424–425
nested functions, call order of time intelligence
functions, 245–246
nested iterators
DAX optimization, 693–699
leveraging context transitions, 190–194
nesting
fi lter contexts, in variables, 184–185
FILTER functions, 61–62
multiple rows, in variables, 184
row contexts
on different tables, 91–92
on the same table, 92–97
table functions, 58–59
VAR/RETURN statements, 179–180
new customers, computing (tables), 380–381,
386–387
NEXTDAY function, call order of nested time intelligence
functions, 245–246
non-active relationships, ambiguity, 515–517
nonworking days between two dates, computing,
523–525
numbering sequences of events (calculations), 536–539
numbers, conversions, 19–21
O
one-sided relationships (data models), 2, 3
one-to-many relationships. See SMR
one-to-one relationships. See SSR
opening/closing balances (time intelligence
calculations), 254–258
operators, 23
arithmetic operators, 23
division by zero, 32–33
empty/missing values, 33–35
error-handling, 32–35
comparison operators, 23
logical operators, 23
overloading, 19–20
parenthesis operators, 23
text concatenation operators, 23
optimizing
columns
high-cardinality columns, 603
split optimization, 602–603
storage optimization, 602–603
data models with VertiPac, 579
aggregations, 587–588
cross-fi ltering, 590
denormalizing data, 584–591
gathering data model information,
579–584
relationship cardinality, 586–587

## Dax, 657

bottlenecks, 668
CallbackDataID function, 690–693
change implementation, 668
conditional statements, 708–709
context transitions, 672–678
DISTINCTCOUNT function, 699–704
expressions, identifying a single DAX expression
for optimization, 658–661
fi lter conditions, 668–672
IF conditions, 678–683, 684–690
multiple evaluations, avoiding with variables,
704–708
nested iterators, 693–699
query plans, 664–667
reproduction queries, creating,
661–664
SE/FE bottlenecks, identifying, 667–668
server timings, 664–667

queries
test queries, rerunning, 668
to-do list, 658
variables, 704–708
OR conditions, tables as fi lters, 381–384
ORDER BY clauses in EVALUATE statements, 60
orders (example), computing duration of, 26
Ordinal values, calculated items, 289
overwriting fi lters, CALCULATE function, 120–122, 136
P
P/C (Parent/Child) hierarchies, 350–361, 362
paging, VertiPaq hardware selection, 576–577
parallelism
CallbackDataID function, 641
VertiPaq SE queries, 641
PARALLELPERIOD function, time intelligence calculations,
238–239
parenthesis operators, 23
partitioning and SSAS, 562–563
Partitions # column (VertiPaq Analyzer), 582
percentages, computing, 135
ALL function, 63–64
ALLSELECTED function, 75–76
CALCULATE function, 124
ALL function, 125–132
ALLEXCEPT function, 135
VALUES function, 133–134
hierarchies, 345
IF conditions, 349
PercOnCategory measures, 348
PercOnParent measures, 346–349
ratio to parent calculations, 345
PercOnCategory measures, computing percentages
over hierarchies, 348
PercOnParent measures, computing percentages over
hierarchies, 346, 348–349
PercOnSubcategory measures, computing percentages
over hierarchies, 346–348
physical query plans, 612–613, 614–616, 651–652
physical relationships
calculated physical relationships, 471–473
circular dependencies, 476–480
range-based relationships, 474–476
cardinality, 489–490
choosing, 506–507
cross-fi lter directions, 490
bidirectional cross-fi lter direction, 490,
491–493, 507
single cross-fi lter direction, 490
cross-island relationships, 489
intra-island relationships, 489

## Mmr, 489, 490, 494, 507

bridge tables, 494–499
common dimensionality, 500–504
weak relationships, 504–506

## Smr, 489, 490, 493, 507


## Ssr, 489, 490, 493–494

strong relationships, 488
virtual relationships versus, 506–507
weak relationships, 488, 489, 504–506
Power BI
Auto Date/Time, 218–219
DAX and, 14–15
DAX Studio and, 609–611
fi lter contexts, 84–85
Power BI reports and DAX queries, 609–610
Power Pivot for Excel
automatic date columns, 219
date table templates, 220
Precedence calculation group, 288, 299–304
precomputing values (calculations), computing work days
between two dates, 525–527
previous year sales up to last day sales (calculations),
computing, 539–544
PREVIOUSMONTH function, time intelligence calculations, 239
Primary/Alternate Keys column (tables), 599
primary/alternate keys column (tables), 600
processing tables, 550
PYTD (Previous Year-To-Date) calculations, calculation
items and sideways recursion, 307–308
Q
QTD (Quarter-to-Date) calculations, time intelligence
calculations, 235–236, 259–262, 276–277
qualitative attributes column (tables), 599, 600
quantitative attributes column (tables), 599, 600–601
queries
DAX queries
capturing, 609–611
DISTINCTCOUNT function, 634–635
executing, 546
DAX query plans, 612–613

queries
DirectQuery, 546, 548, 549, 617
DirectQuery SE queries
composite data models, 646–647
reading, 645–646
Expression Trees, 612

## Fe, 546, 547

datacaches, 547
operators of, 547
single-threaded implementation, 547
materialization, 568–571
MDX queries, 546
DAX and, 613
disabling attribute hierarchies (data model
optimization), 604
executing, 546
query measures, creating with DAX Studio,
662–663
reproduction queries, creating
creating query measures with DAX Studio,
662–663
in DAX, 661–662
in MDX, 663–664

## Se, 546, 616–617

aggregations, 548
datacaches, 547
DirectQuery, 548
operators of, 547
parallel implementations, 548
VertiPaq, 547–549, 550–577
test queries, rerunning (DAX optimization), 668
VertiPaq, 546, 547–548, 550. See also data models,
optimizing with VertiPaq
aggregations, 571–573
columnar databases, 550–553
compression, 553–562
datacaches, 549

## Dmv, 563–565

hardware selection, 573–577
hash encoding, 555–556
hierarchies, 561–562
materialization, 568–571
multithreaded implementations, 548
partitioning, 562–563
processing tables, 550
re-encoding, 559
relationships (data models), 561–562,
565–568

## Rle, 556–559

scan operations, 549
segmentation, 562–563
sort orders, 560–561
value encoding, 554–555
VertiPaq SE queries, 624
composite data models, 646–647
datacaches and parallelism, 635–637
DISTINCTCOUNT function, 634–635
scan time, 632–634
xmSQL queries and, 624–632
xmSQL queries, 624
aggregation functions, 625–627
arithmetical operations, 627
batch events, 630–632
fi lter operations, 628–630
join operators, 630
queries, authoring, 395
ADDMISSINGITEMS function, 419–420,
432–433
auto-exists feature, 428–434
DAX Studio, 395
DEFINE sections
MEASURE keyword in, 399
VAR keyword in, 397–399
EVALUATE statements
ADDMISSINGITEMS function, 419–420,
432–433
example of, 396
expression variables and, 398
GENERATE function, 414–417
GENERATEALL function, 417
GROUPBY function, 420–423
ISONORAFTER function, 417–419
NATURALINNERJOIN function, 423–425
NATURALLEFTOUTERJOIN function, 423–425
query variables and, 398
ROW function, 400–401
SAMPLE function, 427–428
SUBSTITUTEWITHINDEX function, 425–427
SUMMARIZE function, 401–403, 433–434
SUMMARIZECOLUMNS function, 403–409,
429–434
syntax of, 396–399
TOPN function, 409–414
TOPNSKIP function, 420
expression variables, 397–399
GENERATE function, 414–417

relationships (data models)
GENERATEALL function, 417
GROUPBY function, 420–423
ISONORAFTER function, 417–419
MEASURE in DEFINE sections, 399
measures
query measures, 399
testing, 399–401
NATURALINNERJOIN function, 423–425
NATURALLEFTOUTERJOIN function, 423–425
query variables, 397–399
ROW function, testing measures, 400–401
SAMPLE function, 427–428
shadow fi lter contexts, 457–462
SUBSTITUTEWITHINDEX function, 425–427
SUMMARIZE function, 401–403, 433–434
SUMMARIZECOLUMNS function, 403–409,
429–434
TOPN function, 409–414
TOPNSKIP function, 420
VAR in DEFINE sections, 397–399
Query End events (SQL Server Profi ler), 621
query plans
capturing queries
DAX Studio, 617–620
SQL Server Profi ler, 620–623
collecting, 613–614
DAX optimization, 664–667
logical query plans, 612, -614, 650–651
physical query plans, 612–613, 614–616, 651–652
reading, 649–655
query variables, 397–399
R
range-based relationships (calculated physical
relationships), 474–476
RANK.EQ function, 210
RANKX function, numbering sequences of events
(calculations), 538–539
RANKX iterators, 188, 202–210
ratio to parent calculations, computing percentages over
hierarchies, 345

```dax
readability (code), FILTER function, 62–63
recursion (sideways), calculation items, 306–311
```

re-encoding
SSAS and, 559
VertiPaq, 559
referencing columns in tables, 17–18
refreshing data, SSAS (SQL Server Analysis Services),
549–550
RELATED function
CALCULATE function and, 443–444
calculated columns, 443–444
context transitions in expanded tables, 455
expanded tables, 441–444
fi lter contexts, relationships and, 109
nested row contexts on different tables, 92
row contexts and relationships, 103–105
table fi lters and expanded tables, 454
RELATEDTABLE function, 58–59
fi lter contexts, relationships and, 109
nested row contexts on different tables, 91–92
row contexts and relationships, 103–105
relational functions, 53–54
relationships (data models), 2
1:1 relationships, 2
active relationships
ambiguity, 514–515
CALCULATETABLE function, 451–453
expanded tables and, 450–453
USERELATIONSHIP function, 450–451
ambiguity, 512–513
active relationships, 514–515
non-active relationships, 515–517
bidirectional fi ltering, 3–4
bidirectional relationships, 106, 109
calculated physical relationships, 471
circular dependencies, 476–480
multiple-column relationships, 471–473
range-based relationships, 474–476
cardinality, 489–490, 586–587, 590–591
chains, 3
columns, 3
cross-fi lter directions, 3, 490
bidirectional cross-fi lter direction, 490,
491–493, 507
single cross-fi lter direction, 490
cross-island relationships, 489
DAX and SQL, 9
directions of, 3–4
evaluation contexts and, 101–102
fi lter contexts, 106–109
row contexts, 102–105
expanded tables, 437–441

relationships (data models)
granularity, 507–512
intra-island relationships, 489
invalid relationships and blank rows, 68–71
many-sided relationships, 2, 3

## Mmr, 489, 490, 494, 507

bridge tables, 494–499
common dimensionality, 500–504
weak relationships, 504–506
non-active relationships, ambiguity, 515–517
one-sided relationships, 2, 3
performance, 507
physical relationships
calculated physical relationships,
471–480
cardinality, 489–490
choosing, 506–507
cross-fi lter directions, 490–493
cross-island relationships, 489
intra-island relationships, 489

## Mmr, 489, 490, 494–506, 507


## Smr, 489, 490, 493, 507


## Ssr, 489, 490, 493–494

strong relationships, 488
virtual relationships versus, 506–507
weak relationships, 488, 489, 504–506
Relationship reports (VertiPaq Analyzer), 584
Relationship Size column (VertiPaq Analyzer), 582
relationships, expanded tables, 437–441
shallow relationships in batch events (xmSQL queries),
630–632

## Smr, 489, 490, 493, 507

SSAS and, 561–562

## Ssr, 489, 490, 493–494

strong relationships, 488
transferring fi lters, 480–481
CALCULATE function, 482
CONTAINS function, 481–482
FILTER function, 481–482, 484–485
INTERSECT function, 483–484
TREATAS function, 482–483, 484
unidirectional fi ltering, 4
USERELATIONSHIP function, non-active relationships
and ambiguity, 516–517
VertiPaq and, 565–568
virtual relationships, 480, 507
dynamic segmentation, 485–488
physical relationships versus, 506–507
transferring fi lters, 480–485
weak relationships, 2, 439, 488, 489, 504–506
reproduction queries, creating
in DAX, 661–662
in MDX, 663–664
query measures, creating with DAX Studio,
662–663
reusing table expressions, 388–389
RLE (Run Length Encoding), VertiPaq, 556–559
ROLLUP function, 401–402, 403
ROLLUPADDISSUBTOTAL modifi er, SUMMARIZECOLUMNS function, 404–406
ROLLUPGROUP modifi er, SUMMARIZECOLUMNS function, 406
ROW CONTEXT iterators, 187–188
row contexts, 80
CALCULATE function and, 148–151
column references, 87
examples of, 86–87
fi lter contexts versus, 85
iterators and, 90–91
nested row contexts
on different tables, 91–92
on the same table, 92–97
relationships and, 102–105
ROW function
static tables, creating, 391–392
testing measures, 400–401
rows (tables)
ALLNOBLANKROW function, 464, 465
blank rows, invalid relationships, 68–71
CONTAINSROW function, 387–388
DETAILROWS function, 388–389
nesting in variables, 184
SAMPLE function, 427–428
TOPN function, 409–414
Rows column (VertiPaq Analyzer), 581, 583
S
sales
budget/sales information (calculations), showing
together, 527–530
previous year sales up to last day sales (calculations),
computing, 539–544
same-store sales (calculations), computing, 530–536
same-store sales (calculations), computing, 530–536
SAMEPERIODLASTYEAR function

SQL Server Profi ler
computing previous year sales up to last day sales
(calculations), 540–544
nested time intelligence functions, call order of,
245–246
time intelligence calculations, 237
SAMPLE function, authoring queries, 427–428
scalar expressions, 57–58
scalar values
storing in variables, 176, 181
tables as, 71–74
SE (Storage Engines), 546
aggregations, 548
bottlenecks, identifying, 667–668
datacaches, 547
DirectQuery, 548, 549
operators of, 547
parallel implementations, 548
queries, 616–617
SE queries, copy VertiPaq SE queries entries
VertiPaq, 547–548, 550. See also data models,
optimizing with VertiPaq
aggregations, 571–573
columnar databases, 550–553
compression, 553–562
datacaches, 549

## Dmv, 563–565

hardware selection, 573–577
hash encoding, 555–556
hierarchies, 561–562
materialization, 568–571
multithreaded implementations, 548
partitioning, 562–563
processing tables, 550
re-encoding, 559
relationships (data models), 561–562,
565–568

## Rle, 556–559

scan operations, 549
segmentation, 562–563
sort orders, 560–561
value encoding, 554–555
VertiPaq SE queries, 624–632
segmentation
dynamic segmentation and virtual relationships,
485–488
SSAS and, 562–563
Segments # column (VertiPaq Analyzer), 582
SELECTCOLUMNS function, 390–391, 393–394
SELECTCOLUMNS iterators, 196, 197–199
SELECTEDMEASURE function, including/excluding
measures from calculation items, 304–306
SELECTEDMEASUREFORMATSTRING function, 291
SELECTEDVALUE function
calculated physical relationships, circular
dependencies, 479–480
context transitions in expanded tables,
454–455
fi lter contexts, 318–319
same-store sales (calculations), computing,
533–534
tables as scalar values, 73–74
semi-additive calculations, time intelligence calculations,
246–248
sequences of events (calculations), numbering, 536–539
server timings, DAX optimization, 664–667
shadow fi lter contexts, 457–462
shallow relationships in batch events (xmSQL queries),
630–632
sideways recursion, calculation items, 306–311
simple fi lters
arbitrarily shaped fi lters versus, 337
defi ned, 337
single cross-fi lter direction (physical relationships), 490
single data models
DirectQuery mode, 488
VertiPaq mode, 488
single-line comments, 18
SMR (Single-Many Relationships), 489, 490,
493, 507
sort order, determining, ORDER BY clauses, 60
sort orders
SSAS and, 560–561
VertiPaq, 560–561
SQL (Structured Query Language)
conditions, 11
DAX and, 9
as declarative language, 10
error-handling, empty/missing values, 35
subqueries, 11
SQL Server Profi ler
DirectQuery End events, 621
Query End events, 621
query plans, capturing profi ling information,
620–623
VertiPaq SE Query Cache Match events, 621
VertiPaq SE Query End events, 621

SQRT function
SQRT function, 36
SSAS (SQL Server Analysis Services)
data refreshes, 549–550

## Dmv, 563–565

hierarchies, 561–562
partitioning, 562–563
processing tables, 550
re-encoding, 559
relationships (data models), 561–562
segmentation, 562–563
sort orders, 560–561
SSR (Single-Single Relationships), 489, 490, 493–494
star schemas, denormalizing data and data model optimization, 586
STARTOFQUARTER function, time intelligence calculations, 256–257
static tables, creating
DATATABLE function, 392–393
ROW function, 391–392
storing
blockz, in variables, 176, 181
columns (tables), 601–602
partial results of calculations, in variables, 176–177
scalar values, in variables, 176, 181
tables, in variables, 58
string conversions, 19–21
strong relationships, 488
subcategories/categories example, ALL function and,
66–67
subqueries

## Dax, 11


## Sql, 11

SUBSTITUTEWITHINDEX function, authoring queries,
425–427
SUM function, 42–43, 44–45, 88–89
SUMMARIZE function
authoring queries, 401–403, 433–434
auto-exists feature (queries), 433–434
columns (tables) and, 401
fi lter contexts, 112
GROUPBY function and, 420–423
ISSUBTOTAL function and, 402–403
ROLLUP function and, 401–402, 403
table fi lters and expanded tables, 453–454
tables and, 369–372, 373–374, 383–384
transferring fi lters, 484–485
SUMMARIZECOLUMNS function
authoring queries, 403–409, 429–434
auto-exists feature (queries), 429–434
fi lter arguments, 406–409
IGNORE modifi er, 403–404
ROLLUPADDISSUBTOTAL modifi er, 404–406
ROLLUPGROUP modifi er, 406
TREATAS function and, 407–408
SUMX function, 45
SUMX iterators, 187–188
SWITCH function, 47–48
T
table constructors, 24
table expressions, EVALUATE statements,
59–60
table fi lters, DISTINCTCOUNT function,
699–704
table functions, 57
ALL function
columns and, 64–65
computing percentages, 63–64
measures and, 63–64
syntax of, 63
top categories/subcategories example,
66–67
VALUES function versus, 67
ALLEXCEPT function, 65–66
ALLSELECTED function, 74–76
calculated columns and, 59
calculated tables, 59
DISTINCT function, 71
blank rows and invalid relationships, 68, 70–71
calculated columns, 68
multiple columns, 71
VALUES function versus, 68
FILTER function, 57–58
code maintenance/readability, 62–63
as iterator, 60–61
nesting, 61–62
syntax of, 60
measures and, 59
nesting, 58–59
RELATEDTABLE function, 58–59
VALUES function, 71
ALL function versus, 67
blank rows and invalid relationships, 68–71

tables
calculated columns, 68
calculated tables, 68
DISTINCT function versus, 68
measures and, 67–68
multiple columns, 71
tables as scalar values, 71–74
Table Size % column (VertiPaq Analyzer), 582
Table Size column (VertiPaq Analyzer), 581
table variables, 181–182
tables, 363
ADDCOLUMNS function, 366–369, 371–372
blank rows, invalid relationships, 68–71
bridge tables, MMR, 494–499
CALCULATE function, tables as fi lters, 382–384
calculated columns, 25–26, 42
choosing between calculated columns and
measures, 29–30
differences between calculated columns and
measures, 29
expressions, 29
using measures in calculated columns, 30
calculated tables, 59
creating, 390–391
DISTINCT function, 68
SELECTCOLUMNS function, 390–391
VALUES function, 68
CALCULATETABLE function, 363–365
columns
ADDCOLUMNS function, 366–369, 371–372
Boolean calculated columns, 597–598
calculated columns and data model optimization,
595–599
calculated columns, RELATED function, 443–444
cardinality, 603
cardinality and data model optimization, 591–592
Date column, 592–595
defi ned, 2
descriptive attributes column (tables), 600,
601–602
fi ltering, 444–447
optimizing high-cardinality columns, 603
Primary/Alternate Keys column (tables), 599
primary/alternate keys column (tables), 600
qualitative attributes column (tables), 599, 600
quantitative attributes column (tables), 599,
600–601
referencing, 17–18
relationships, 3
SELECTCOLUMNS function, 390–391,
393–394
storage optimization, 602–603
storing, 601–602
SUBSTITUTEWITHINDEX function, 425–427
SUMMARIZE function and, 401
SUMMARIZECOLUMNS function, 403–409,
429–434
technical attributes column (tables), 600, 602
Time column, 592–595
VertiPaq Analyzer, 580–583
computing new customers, 380–381, 386–387
CONTAINS function, 387–388
CONTAINSROW function, 387–388
CROSSJOIN function, 372–374, 383–384
date tables
ADDCOLUMNS function, 223–224
building, 220–224
CALENDAR function, 222
CALENDARAUTO function, 222–224
date table templates (Power Pivot for Excel),
date templates, 224
duplicating, 227
loading from other data sources, 221
managing multiple dates, 224–228
Mark as Date Table, 232–233
multiple date tables, 226–228
multiple relationships to date tables,
224–226
naming, 221
defi ned, 2
DETAILROWS function, 388–389
EXCEPT function, 379–381
expanded tables
active relationships, 450–453
column fi lters versus table fi lters, 444–447
context transitions, 454–455
differences between table fi lters and expanded
tables, 453–454
fi lter contexts, 439–441
fi ltering, 444–447, 450–453
RELATED function, 441–444
relationships, 437–441
table fi lters in measures, 447–450
table fi lters versus column fi lters, 444–447

tables
expressions, reusing, 388–389
FILTER function versus CALCULATETABLE function,
363–365
fi ltering
CALCULATE function and, 445–447
column fi lters versus, 444–447
in measures, 447–450
as fi lters, 381–384
GENERATESERIES function, 393–394
IN function, 387–388
INTERSECT function, 378–379
iterators, returning tables with, 196–199
measures, defi ning in tables, 29
narrowing computations, 384–386
NATURALINNERJOIN function, 423–425
NATURALLEFTOUTERJOIN function, 423–425
processing, 550
records, 2
reusing expressions, 388–389
rows
ALLNOBLANKROW function, 464, 465
CONTAINSROW function, 387–388
DETAILROWS function, 388–389
SAMPLE function, 427–428
TOPN function, 409–414
as scalar values, 71–74
SELECTCOLUMNS function, 390–391, 393–394
static tables
creating with DATATABLE function, 392–393
creating with ROW function, 391–392
storing in variables, 176, 181
SUMMARIZE function, 369–372, 373–374, 383–384
temporary tables in batch events (xmSQL queries),
630–632
TOPN function, 409–414
UNION function, 374–378
variables, storing tables in, 58
Tabular model
calculation groups, creating, 281–288
DAX engines and, 545–546
DAX queries, executing, 546
DirectQuery, 546
MDX queries, executing, 546
VertiPaq, 546
technical attributes column (tables), 600, 602
templates
date table templates (Power Pivot for Excel), 220
date templates, 224
temporary tables in batch events (xmSQL queries),
630–632
test queries, rerunning (DAX optimization), 668
text
concatenation operators, 23
editing, formatting DAX code, 42
text functions, 50–51
Time column, data model optimization, 592–595
TIME function, 51, 52
time intelligence calculations, 217
Auto Date/Time (Power BI), 218–219
automatic date columns (Power Pivot for Excel),
basic calculations, 228–232
basic functions, 233–235
CALCULATE function, 228–232
CALCULATETABLE function, 259, 260–261
context transitions, 260
custom calendars, 272
DATESYTD function, 276–277
weeks, 272–275
date tables
ADDCOLUMNS function, 223–224
building, 220–224
CALENDAR function, 222
CALENDARAUTO function, 222–224
date table templates (Power Pivot for Excel),
date templates, 224
duplicating, 227
loading from other data sources, 221
managing multiple dates, 224–228
Mark as Date Table, 232–233
multiple date tables, 226–228
multiple relationships to date tables,
224–226
naming, 221
DATEADD function, 237–238, 262–269
DATESINPERIOD function, 243–244
DATESMTD function, 259, 276–277
DATESQTD function, 259, 276–277
DATESYTD function, 259, 260, 261–262, 276–277
differences over previous periods, computing,
241–243
drillthrough operations, 271
FILTER function, 228–232
FIRSTDATE function, 269, 270
FIRSTNONBLANK function, 256–257,
270–271

variables
LASTDATE function, 248–249, 254, 255,
269–270
LASTNONBLANK function, 250–254, 255,
270–271
mixing functions, 239–241
moving annual totals, computing, 243–244
MTD calculations, 235–236, 259–262, 276–277
nested functions, call order of, 245–246
NEXTDAY function, 245–246
opening/closing balances, 254–258
PARALLELPERIOD function, 238–239
periods to date, 259–262
PREVIOUSMONTH function, 239
QTD calculations, 235–236, 259–262,
276–277
SAMEPERIODLASTYEAR function, 237, 245–246
semi-additive calculations, 246–248
STARTOFQUARTER function, 256–257
time periods, computing from prior periods,
237–239
YTD calculations, 235–236, 259–262, 276–277
time periods, computing from prior periods,
237–239
top categories/subcategories example, ALL function and,
66–67
TOPN function
authoring queries, 409–414
ISONORAFTER function and, 417–419
sort order, 410
TOPNSKIP function, authoring queries, 420
transferring fi lters, 480–481
CALCULATE function, 482
CONTAINS function, 481–482
FILTER function, 481–482, 484–485
INTERSECT function, 483–484
TREATAS function, 482–483, 484
TREATAS function, 378
data lineage, 467–468
fi lter contexts and data lineage, 334–336
SUMMARIZECOLUMNS function and,
407–408
transferring fi lters, 482–483, 484
UNION function and, 377–378
trigonometric functions, 50
U
unary operators, P/C (Parent/Child) hierarchies, 362
unidirectional fi ltering (relationships), 4
UNION function
CALCULATE function and, 376–378
DISTINCT function and, 375–378
tables and, 374–378
TREATAS function and, 377–378
Use Hierarchies Size column (VertiPaq Analyzer),
USERELATIONSHIP function
active relationships, 450–451
CALCULATE function and, 164–168
non-active relationships and ambiguity,
516–517
V
value encoding (VertiPaq compression),
554–555
VALUE function, 51
values, list of. See fi lter arguments
VALUES function, 71
ALL function and, 327–328
ALL function versus, 67
ALLEXCEPT function versus, 326–328
blank rows and invalid relataionships, 68–71
calculated columns, 68
calculated physical relationships
circular dependencies, 477–480
range-based relationships, 474–476
calculated tables, 68
computing percentages, 133–134
DISTINCT function versus, 68
fi lter contexts, 322–324, 327–328
FILTERS function versus, 322–324
measures and, 67–68
multiple columns, 71
tables as scalar values, 71–74

```dax
VAR keyword, DEFINE sections (authoring queries),
```

397–399
variables, 30–31, 175
as a constant, 177–178
defi ning, 176, 178–180
documenting code, 183–184
error-handling, 37
expression variables, 397–399
formatting, 40–41
lazy evaluations, 181–183
multiple evaluations, avoiding with variables,
704–708

variables
MultipleItemSales variable, 58
names, 182
nesting
fi lter contexts, 184–185
multiple rows, 184
query variables, 397–399
scalar values, 58
scope of, 178–180
storing
partial results of calculations, 176–177
scalar values, 176, 181
tables, 176, 181
table variables, 181–182
tables, storing, 58
VAR syntax, 175–177
VAR/RETURN blocks, 175–177, 180
VAR/RETURN statements, nesting, 179–180
Variant data type, 22
VertiPaq, 546, 547–548, 550
aggregations, 571–573, 604–607
caches, 637–640
CallbackDataID function, 640–644
columnar databases, 550–553
compression, 553–554
hash encoding, 555–556
re-encoding, 559

## Rle, 556–559

value encoding, 554–555
data model optimization, 579
aggregations, 587–588, 604–607
calculated columns, 595–599
choosing columns for storage, 599–602
column cardinality, 591–592
cross-fi ltering, 590
Date column, 592–595
denormalizing data, 584–591
disabling attribute hierarchies, 604
gathering data model information, 579–584
optimizing column storage, 602–603
optimizing drill-through attributes, 604
relationship cardinality, 586–587, 590–591
Time column, 592–595
datacaches, 549

## Dmv, 563–565

hardware selection, 573
best practices, 577
CPU model, 574–575
Disk I/O performance, 574, 576–577
memory size, 574, 576
memory speed, 574, 575–576
number of cores, 574, 576
as an option, 573–574
paging, 576–577
setting priorities, 574–576
hierarchies, 561–562
materialization, 568–571
multithreaded implementations, 548
partitioning, 562–563
processing tables, 550
relationships (data models), 561–562, 565–568
row-level security, 639
scan operations, 549
segmentation, 562–563
sort orders, 560–561
VertiPaq Analyzer
columns (tables), 580–583
gathering data model information, 579–584
VertiPaq Analyzer, Relationship reports, 584
VertiPaq mode, 488–489
composite data models, 488
single data models, 488
VertiPaq SE queries, 624
composite data models, 646–647
datacaches, parallelism and, 635–637
DISTINCTCOUNT function, 634–635
scan time, 632–634
xmSQL queries and, 624
aggregation functions, 625–627
arithmetical operations, 627
batch events, 630–632
fi lter operations, 628–630
join operators, 630
VertiPaq SE Query Cache Match events (SQL Server
Profi ler), 621
VertiPaq SE Query End events (SQL Server Profi ler), 621
virtual relationships, 480, 507
dynamic segmentation, 485–488
physical relationships versus, 506–507
transferring fi lters, 480–481
CALCULATE function, 482
CONTAINS function, 481–482
FILTER function, 481–482, 484–485
INTERSECT function, 483–484
TREATAS function, 482–483, 484

YTD (Year-to-Date) calculations
W
weak relationships, 2, 439, 488, 489, 504–506
weeks (custom calendars), time intelligence calculations,
272–275
work days between two dates, computing, 519–523
nonworking days, 523–525
precomputing values (calculations), 525–527
X
xmSQL
CallbackDataID function
parallelism and, 641
VertiPaq and, 640–644
VertiPaq queries, 548
xmSQL queries, 624
aggregation functions, 625–627
arithmetic operations, 627
batch events, 630–632
fi lter operations, 628–630
join operators, 630
Y
YOY (Year-Over-Year) calculation item, 289–290
YOY% (Year-Over-Year Percentage) calculation item,
289–290
YTD (Year-to-Date) calculations
calculation group precedence, 299–303
calculation items
applying to expressions, 294
sideways recursion, 307
time intelligence calculations, 235–236, 259–262,
276–277


Marco Russo and Alberto Ferrari are the founders of sqlbi.com,
where they regularly publish articles about Microsoft Power BI,
Power Pivot, DAX, and SQL Server Analysis Services. They have
worked with DAX since the fi rst beta version of Power Pivot
in 2009 and, during these years, sqlbi.com became one of the
major sources for DAX articles and tutorials. Their courses, both
in-person and online, are the major source of learning for many
DAX enthusiasts.
They both provide consultancy and mentoring on business
intelligence (BI) using Microsoft technologies. They have
written several books and papers about Power BI, DAX, and
Analysis Services. They constantly help the community of DAX
users providing content for the websites daxpatterns.com,
daxformatter.com, and dax.guide.
Marco and Alberto are also regular speakers at major
international conferences, including Microsoft Ignite, PASS
Summit, and SQLBits. Contact Marco at marco.russo@sqlbi.com,
and contact Alberto at alberto.ferrari@sqlbi.com