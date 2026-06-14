# Chapter 13: Authoring queries

In this chapter we continue our journey, discovering new table functions in DAX. Here, the focus is on
functions that are more useful when preparing queries and calculated tables, rather than in measures. Keep in mind that most of the functions you learn in this chapter can be used in measures too,
although some have limitations that we outline.
For each function, we provide examples of queries using them. The chapter has two goals: learning
new functions and presenting useful patterns that you can implement in your data model.
All the demo fi les in this chapter are provided as a text fi le, containing the query executed with DAX
Studio connected to a common Power BI fi le. The Power BI fi le contains the usual Contoso data model
used through the entire book.
Introducing DAX Studio
DAX Studio is a free tool available at www.daxstudio.org that provides help in authoring queries,
debugging code, and measuring the performance of queries.
DAX Studio is a live project with new features continuously being added to it. Here are a few of the
most relevant features:

> **Note:** Connectivity to Analysis Services, Power BI, or Power Pivot for Excel.

> **Note:** Full text editor to author queries and code.

> **Note:** Automatic formatting of the code through the daxformatter.com service.

> **Note:** Automatic measure defi nition to debug or fi ne-tune performance.

> **Note:** Detailed performance information about your queries.
Though other tools are available to test and write queries in DAX, we strongly encourage the reader
to download, install and learn DAX Studio. If you are unsure, just think that we wrote all the DAX code
in this book using that tool. We work with DAX all day long, and we like to be productive. A complete
documentation of DAX Studio is available at http://daxstudio.org/documentation/.

396 CHAPTER 13 Authoring queries
Understanding EVALUATE
EVALUATE is a DAX statement that is needed to execute a query. EVALUATE followed by any table
expression returns the result of the table expression. Moreover, one or more EVALUATE statements
can be preceded by special defi nitions like local tables, columns, measures, and variables that have the
scope of the entire batch of EVALUATE statements executed together.
For example, the following query returns the red products by using EVALUATE, followed by a simple
CALCULATETABLE function:

## Evaluate


## Calculatetable (

'Product',
'Product'[Color] = "Red"
)
Before diving deeper into the description of more advanced table functions, we must introduce the
syntax and the options available in EVALUATE, which we will use when writing complex queries.
Introducing the EVALUATE syntax
An EVALUATE statement is divided in three parts:

> **Note:** Defi nition section: Introduced by the DEFINE keyword, it includes the defi nition of local entities like tables, columns, variables, and measures. There can be a single defi nition section for the
entire query, even though the query can contain multiple EVALUATE statements.

> **Note:** Query expression: Introduced by the EVALUATE keyword, it contains the table expression to
evaluate and return as the result. There might be multiple query expressions, each introduced
by EVALUATE and each with its own set of result modifi ers.

> **Note:** Result modifi ers: An optional additional section to EVALUATE, which is introduced by the keyword ORDER BY. It includes the sort order of the result and the optional defi nition of which rows
to return, by providing a starting point with START AT.
The fi rst and the third part of the statement are optional. Thus, one can just use EVALUATE followed
by any table expression to produce a query. Nevertheless, by doing so, the developer cannot use many
useful features of EVALUATE. Therefore, time spent learning the whole syntax is time well spent.
Here is an example of a query:

## Define


```dax
VAR MinimumAmount = 2000000
VAR MaximumAmount = 8000000
EVALUATE
FILTER (
ADDCOLUMNS (
SUMMARIZE ( Sales, 'Product'[Category] ),
"CategoryAmount", [Sales Amount]
),
```

CHAPTER 13 Authoring queries 397

## And (

[CategoryAmount] >= MinimumAmount,
[CategoryAmount] <= MaximumAmount
)
)
ORDER BY [CategoryAmount]
The previous query returns the result shown in Figure 13-1.
FIGURE 13-1 The result only includes the category amount included between 2,000,000 and 8,000,000.
The example defi nes two variables storing the upper and lower boundary of the sales amount. The
query then retrieves all the categories whose total sales fall in between the boundaries defi ned by the
variables. Finally, it sorts the result by sales amount. As simple as it is, the syntax is powerful, and in the
next sections, we provide some important considerations about the usage of each part of the EVALUATE syntax.
One important detail is that the defi nition section and the result modifi ers are only available in
conjunction with EVALUATE. Thus, these features are only available when authoring queries. If writing
a query that will later be used as a calculated table, a careful developer should avoid relying on the
DEFINE and ORDER BY sections, only focusing on the query expression. A calculated table is defi ned by
a table expression, not by a DAX query.
Using VAR in DEFINE
In the defi nition section, it is possible to use the VAR keyword to defi ne variables. Each variable is as
simple as a name followed by an expression. Variables introduced in queries do not need the RETURN
part required when variables are used as part of an expression. Indeed, the result is defi ned by the

```dax
EVALUATE section. We distinguish between regular variables (variables used in expressions) and variables defi ned in the DEFINE section by naming the former expression variables, and the latter query
```

variables.
As is the case with expression variables, query variables can contain both values and tables without
restriction. For example, the query shown in the previous section can also be authored with a query
table variable:

## Define


```dax
VAR MinimumAmount = 2000000
VAR MaximumAmount = 8000000
VAR CategoriesSales =
ADDCOLUMNS (
SUMMARIZE ( Sales, 'Product'[Category] ),
"CategoryAmount", [Sales Amount]
```

398 CHAPTER 13 Authoring queries
)

## Evaluate


## Filter (

CategoriesSales,

## And (

[CategoryAmount] >= MinimumAmount,
[CategoryAmount] <= MaximumAmount
)
)
ORDER BY [CategoryAmount]
A query variable has the scope of the entire batch of EVALUATE statements executed together. This
means that after it has been defi ned, the variable can be used anywhere in the following queries. The
one limitation is that a variable can only be referenced after it has been defi ned. In the previous query,
if you defi ne CategoriesSales before MinimumAmount or MaximumAmount, the result is a syntax error:
The expression of CategoriesSales references two variables not yet defi ned. This is useful to prevent
circular dependencies. Besides, the very same limitation exists for expression variables; therefore, query
variables follow the same limitations as expression variables.
If the query contains multiple EVALUATE sections, query variables are available through all of them.
For example, queries generated by Power BI use the DEFINE part to store slicer fi lters in query variables
and then include multiple EVALUATE statements to compute the various parts of the visual.
Variables can also be defi ned in the EVALUATE section; in that case, being expression variables, they
are local to the table expression. The previous query can be equivalently defi ned this way:

## Evaluate


```dax
VAR MinimumAmount = 2000000
VAR MaximumAmount = 8000000
VAR CategoriesSales =
ADDCOLUMNS (
SUMMARIZE ( Sales, 'Product'[Category] ),
"CategoryAmount", [Sales Amount]
)
RETURN
FILTER (
CategoriesSales,
AND (
[CategoryAmount] >= MinimumAmount,
[CategoryAmount] <= MaximumAmount
)
)
ORDER BY [CategoryAmount]
As you can see, the variables are now defi ned as part of the table expression, and the RETURN keyword is needed to defi ne the result of the expression. The scope of the expression variables, in this case,
is the RETURN section.
```

Choosing between using a query variable or an expression variable comes with advantages and
disadvantages. If the variable is needed in further table or column defi nitions, then you need to use
a query variable. On the other hand, if the variable is not required in other defi nitions (or in multiple

CHAPTER 13 Authoring queries 399
EVALUATE sections), then it is better to use an expression variable. Indeed, if the variable is part of the
expression, then it will be much easier to use the expression to compute a calculated table or to embed
it into a measure. Otherwise, there will always be the need to update the syntax of the query to transform it into an expression.
The rule of thumb for choosing between query variables and expression variables is simple. Use
expression variables whenever possible and use query variables when strictly necessary; indeed, query
variables require additional work to re-use the code in a different formula.
Using MEASURE in DEFINE
Another entity that one can defi ne locally to a query is a measure. This is achieved by using the keyword MEASURE. A query measure behaves in all respects like a regular measure, but it exists only for
the lifetime of the query. In the defi nition of the measure it is mandatory to specify the table that hosts
the measure. The following is an example of a query measure:

## Define


```dax
MEASURE Sales[LargeSales] =
CALCULATE (
[Sales Amount],
Sales[Net Price] >= 200
)
EVALUATE
```


## Addcolumns (

VALUES ( 'Product'[Category] ),
"Large Sales", [LargeSales]
)
The result of the query is visible in Figure 13-2.
FIGURE 13-2 The LargeSales query measure is evaluated for every Category in the Large Sales column of the result.
Query measures are useful for two purposes: the fi rst, more obvious, is to write complex expressions that can be called multiple times inside the query. The other reason is that query measures are
extremely useful for debugging and for performance tuning. Indeed, if a query measure has the same
name as a model measure, it gains precedence in the query. In other words, references to the measure
name in the query will use the query measure and not the model measure. However, any other model
measures that reference the redefi ned measure still use the original measure. Therefore, you should

400 CHAPTER 13 Authoring queries
include all the dependent measures as query measures to evaluate the impact of changing a measure
in the model.
Thus, when testing the behavior of a measure, the best strategy is to write a query that uses the
measure, add the local defi nition of the measure, and then perform various tests to either debug or
optimize the code. Once the process is done, the code of the measure can be updated in the model
with the new version. DAX Studio offers a specifi c feature for this purpose: It lets a developer automatically add the DEFINE MEASURE statement to a query to speed up these steps.
Implementing common DAX query patterns
Now that we have described the syntax of EVALUATE, we introduce many functions that are common in
authoring queries. For the most commonly used functions, we also provide sample queries that allow
further elaborating on their use.
Using ROW to test measures
Introduced in the previous chapter, ROW is typically used to obtain the value of a measure or to
perform an investigation on the measure query plan. EVALUATE requires a table as an argument, and
it returns a table as a result. If all you need is the value of a measure, EVALUATE will not accept it as an
argument. It will require a table instead. So, by using ROW, you can transform any value into a table,
like in the following example:

## Evaluate

ROW ( "Result", [Sales Amount] )
The result is visible in Figure 13-3.
FIGURE 13-3 The ROW function returns a table with a single row.
Be mindful that the same behavior can be obtained by using the table constructor syntax:

## Evaluate

{ [Sales Amount] }
Figure 13-4 displays the result of the preceding example.
FIGURE 13-4 The table constructor returns a row with a column named Value.

CHAPTER 13 Authoring queries 401
ROW provides the developer with control over the resulting column’s name, which on the other
hand is generated automatically with the table constructor. ROW allows the developer to generate a
table with more than one column, where they can provide for each column a column name and its corresponding expression. In case one needs to simulate the presence of a slicer, CALCULATETABLE comes
in handy:

## Evaluate


## Calculatetable (


## Row (

"Sales", [Sales Amount],
"Cost", [Total Cost]
),
'Product'[Color] = "Red"
)
The result is visible in Figure 13-5.
FIGURE 13-5 The ROW function can return multiple columns, and values provided are computed in a fi lter context.
Using SUMMARIZE
We introduced and used SUMMARIZE in previous chapters of the book. We mentioned that SUMMARIZE performs two operations: grouping by columns and adding values. Using SUMMARIZE to group
tables is a safe operation, whereas using SUMMARIZE to add new columns might lead to unexpected
results that are hard to debug.
Though adding columns with SUMMARIZE is a bad idea, at this point we introduce two additional
features of SUMMARIZE used in order to add columns. Our intention is to support our reader in understanding code they might run into, written by someone else. However, we reiterate here that using
SUMMARIZE to add columns aggregating values should be avoided.
In case one uses SUMMARIZE to compute values, the option is there to let SUMMARIZE compute
additional rows that represent subtotals. There is a SUMMARIZE modifi er named ROLLUP that changes
the aggregation function of columns requiring for the subtotals to be added to the result. Look at the
following query:

## Evaluate


## Summarize (

Sales,

## Rollup (

'Product'[Category],
'Date'[Calendar Year]
),
"Sales", [Sales Amount]
)

## Order By


402 CHAPTER 13 Authoring queries
'Product'[Category],
'Date'[Calendar Year]
ROLLUP instructs SUMMARIZE to not only compute the value of Sales for each category and year,
but also to add additional rows that contain a blank in the year and that represent the subtotal at the
category level. Because the category is also marked as ROLLUP, one row in the set contains a blank in
both category and year along with the grand total for Sales. This is shown in Figure 13-6.
FIGURE 13-6 The ROLLUP function creates additional total rows in the SUMMARIZE result.
The rows added by ROLLUP contain a blank instead of the value of the column they are summing
up. In case there are blanks in the column, then the output contains two rows with a blank category:
one with the value for the blank category and one with the total by category. To distinguish between
the two, and to make it easier to mark subtotal rows, one can add a new column using the ISSUBTOTAL
function:

## Evaluate


## Summarize (

Sales,

## Rollup (

'Product'[Category],
'Date'[Calendar Year]
),
"Sales", [Sales Amount],
"SubtotalCategory", ISSUBTOTAL ( 'Product'[Category] ),
"SubtotalYear", ISSUBTOTAL ( 'Date'[Calendar Year] )
)

## Order By

'Product'[Category],
'Date'[Calendar Year]
The last two columns of the previous query contain a Boolean value that is set to TRUE when the row
contains a subtotal (on category or on year) and FALSE otherwise, as shown in Figure 13-7.

CHAPTER 13 Authoring queries 403
FIGURE 13-7 The ISSUBTOTAL function returns True whenever a column is a subtotal in the SUMMARIZE result.
By adding these additional columns using ISSUBTOTAL, it is possible to clearly distinguish between
rows containing actual data and rows containing subtotals.
Important SUMMARIZE should not be used to add new columns. Therefore, we mention
the syntax of ROLLUP and ISSUBTOTAL just to be able to read existing code. You should
never use SUMMARIZE this way, but prefer SUMMARIZECOLUMNS instead, or use ADDCOLUMNS and SUMMARIZE when the use of SUMMARIZECOLUMNS is not possible.
Using SUMMARIZECOLUMNS
SUMMARIZECOLUMNS is an extremely powerful query function that is intended to be the “one function fi ts all” to run queries. In a single function, SUMMARIZECOLUMNS contains all the features needed
to execute a query. SUMMARIZECOLUMNS lets you specify:

> **Note:** A set of columns used to perform the group-by, like in SUMMARIZE, with the option of producing subtotals.

> **Note:** A set of new columns to add to the result, like both SUMMARIZE and ADDCOLUMNS.

> **Note:** A set of fi lters to apply to the model prior to performing the group-by, like CALCULATETABLE.
Finally, SUMMARIZECOLUMNS automatically removes from the output any row for which all the
added columns produce a blank value. It does not come as a surprise that Power BI uses SUMMARIZECOLUMNS for nearly all the queries it runs.
The following is a fi rst, simple query using SUMMARIZECOLUMNS:

## Evaluate


## Summarizecolumns (

'Product'[Category],
'Date'[Calendar Year],
"Amount", [Sales Amount]
)

## Order By

'Product'[Category],
'Date'[Calendar Year]

404 CHAPTER 13 Authoring queries
The previous query groups data by category and year, computing the sales amount in a fi lter context
containing the given category and year for every row of the result. The result is visible in Figure 13-8.
FIGURE 13-8 The result contains the category, year, and the amount of the given category and year.
Years with no sales (like 2005) do not appear in the result. The reason is that, for that specifi c row of
the result, the new Amount column returned a blank, so SUMMARIZECOLUMNS removed the row from
the result. If the developer needs to ignore this behavior for certain columns, they can use the IGNORE
modifi er like in the following variation of the same query:

## Evaluate


## Summarizecolumns (

'Product'[Category],
'Date'[Calendar Year],
"Amount", IGNORE ( [Sales Amount] )
)

## Order By

'Product'[Category],
'Date'[Calendar Year]
As a result, SUMMARIZECOLUMNS ignores the fact that Sales Amount returns a blank; the result
also contains sales for Audio in 2005 and 2006, as you can see in Figure 13-9.
FIGURE 13-9 Using IGNORE, combinations producing blank results in a measure are still returned.

CHAPTER 13 Authoring queries 405
In case multiple columns are added by SUMMARIZECOLUMNS, it is possible to choose which one
to tag with IGNORE and which one to use for blank checks. The common practice is that of removing
blanks anyway, to avoid empty results.
SUMMARIZECOLUMNS offers the option of computing subtotals too, using both ROLLUPADDSUBTOTAL and ROLLUPGROUP. In the previous query, if you need the yearly subtotal, you should mark the
Date[Calendar Year] column with ROLLUPADDISSUBTOTAL, also specifying the name of a column that
indicates whether a given row is a subtotal or not:

## Evaluate


## Summarizecolumns (

'Product'[Category],

## Rollupaddissubtotal (

'Date'[Calendar Year],
"YearTotal"
),
"Amount", [Sales Amount]
)

## Order By

'Product'[Category],
'Date'[Calendar Year]
The result now contains additional rows representing the subtotal at the year level, with an additional column named YearTotal containing TRUE only for the subtotal rows. You see this in Figure 13-10
where the subtotal rows are highlighted.
FIGURE 13-10 ROLLUPADDISSUBTOTAL creates a Boolean column indicating the presence of a subtotal, and new
rows with the subtotal amounts.
When summarizing by multiple columns, you can mark several columns with ROLLUPADDISSUBTOTAL. This produces several total groups. For example, the following query produces both the subtotal
of a category for all years and a subtotal of a year over all categories:

406 CHAPTER 13 Authoring queries

## Evaluate


## Summarizecolumns (


## Rollupaddissubtotal (

'Product'[Category],
"CategoryTotal"
),

## Rollupaddissubtotal (

'Date'[Calendar Year],
"YearTotal"
),
"Amount", [Sales Amount]
)

## Order By

'Product'[Category],
'Date'[Calendar Year]
The subtotal of a year over all categories and an example of a subtotal of a category for all years are
highlighted in that order, in Figure 13-11.
FIGURE 13-11 ROLLUPADDISSUBTOTAL can group multiple columns.
If you need subtotals for a group of columns instead of just one column, then the modifi er ROLLUPGROUP becomes useful. The following query produces only one subtotal for both category and year,
adding only one extra row to the result:

## Evaluate


## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Product'[Category],
'Date'[Calendar Year]
),
"CategoryYearTotal"
),
"Amount", [Sales Amount]
)

## Order By


CHAPTER 13 Authoring queries 407
'Product'[Category],
'Date'[Calendar Year]
You can see the result with only one total row in Figure 13-12.
FIGURE 13-12 ROLLUPADDISSUBTOTAL creates both new rows and one new column with the subtotals.
The last feature of SUMMARIZECOLUMNS is the ability to fi lter the result, like CALCULATETABLE
does. One can specify one or more fi lters by using tables as additional arguments. For example, the following query only retrieves the sales of customers with a high school education; the result is similar to
Figure 13-13, but with smaller amounts:

## Evaluate


## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Product'[Category],
'Date'[Calendar Year]
),
"CategoryYearTotal"
),

## Filter (

ALL ( Customer[Education] ),
Customer[Education] = "High School"
),
"Amount", [Sales Amount]
)
Please note that with SUMMARIZECOLUMNS, the compact syntax of fi lter arguments using predicates in CALCULATE and CALCULATETABLE is not available. Thus, the following query generates a
syntax error:

## Evaluate


## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Product'[Category],

408 CHAPTER 13 Authoring queries
'Date'[Calendar Year]
),
"CategoryYearTotal"
),
Customer[Education] = "High School", -- This syntax is not available
"Amount", [Sales Amount]
)
The reason is that the fi lter arguments of SUMMARIZECOLUMNS need to be tables, and there are
no shortcuts in this case. An easy and compact way of expressing a fi lter with SUMMARIZECOLUMNS is
to use TREATAS:

## Evaluate


## Summarizecolumns (


## Rollupaddissubtotal (


## Rollupgroup (

'Product'[Category],
'Date'[Calendar Year]
),
"CategoryYearTotal"
),
TREATAS ( { "High School" }, Customer[Education] ),
"Amount", [Sales Amount]
)
SUMMARIZECOLUMNS is extremely powerful, but it comes with a strong limitation: It cannot be
called if the external fi lter context has performed a context transition. For this reason, SUMMARIZECOLUMNS is useful when authoring queries; however, it is not available as a replacement for ADDCOLUMNS and SUMMARIZE in measures because it will not work in most reports. Indeed, a measure is
often used in a visual like a matrix or a chart, which internally executes the measure in a row context for
each value displayed in the report.
As a further example of SUMMARIZECOLUMNS limitations in a row context, consider the following
query that returns the total sales of all products using an ineffi cient but still valid approach:

## Evaluate

{

## Sumx (

VALUES ( 'Product'[Category] ),

## Calculate (


## Sumx (


## Addcolumns (

VALUES ( 'Product'[Subcategory] ),
"SubcategoryTotal", [Sales Amount]
),
[SubcategoryTotal]
)
)
)
}

CHAPTER 13 Authoring queries 409
If you replace the innermost ADDCOLUMNS with SUMMARIZECOLUMNS, then the query fails
because SUMMARIZECOLUMNS is being called in a context where CALCULATE forced context transition. Therefore, the following query is not valid:

## Evaluate

{

## Sumx (

VALUES ( 'Product'[Category] ),

## Calculate (


## Sumx (


## Summarizecolumns (

'Product'[Subcategory],
"SubcategoryTotal", [Sales Amount]
),
[SubcategoryTotal]
)
)
)
}
In general, SUMMARIZECOLUMNS is not suitable in measures because the measure will be called
inside a much more complex query generated by the client tool. That query is likely to contain context
transitions, making SUMMARIZECOLUMNS fail.
Using TOPN
TOPN is a function that sorts a table and then returns a subset of the fi rst rows only. It is useful whenever one needs to reduce the number of rows of a set. For example, when Power BI shows the result of
a table, it does not retrieve the full result from the database. Instead, it only retrieves the fi rst few rows
that are needed to produce the page on the screen. The remaining part of the result is retrieved only
on demand, when the user scrolls down the visual. Another scenario where TOPN is useful is to retrieve
top performers, like top products, top customers, and so on.
The top three products based on sales can be computed with the following query, which evaluates
the Sales Amount measure for each row of the Product table:

## Evaluate


## Topn (

3,
'Product',
[Sales Amount]
)
The resulting table contains all the columns of the source table. When a table is used in a query,
one is seldom interested in all the columns, so the input table of TOPN should reduce the columns
to merely the ones needed. The following variation produces fewer columns than are available in the
entire Product table. This is shown in Figure 13-13:

## Evaluate


```dax
VAR ProductsBrands =
```

410 CHAPTER 13 Authoring queries

## Summarize (

Sales,
'Product'[Product Name],
'Product'[Brand]
)

```dax
VAR Result =
TOPN (
3,
ProductsBrands,
[Sales Amount]
)
RETURN Result
ORDER BY 'Product'[Product Name]
```

FIGURE 13-13 TOPN fi lters the rows of a table expression based on the value of the Sales Amount measure.
It is likely that one also needs the value of Sales Amount in the result, in order to correctly sort the
resulting three rows. In such a case, the best option is to precompute the value inside the parameter
of SUMMARIZE and then reference it in TOPN. Thus, the most frequently used pattern of TOPN is the
following:

## Evaluate


```dax
VAR ProductsBrands =
SUMMARIZE (
Sales,
'Product'[Product Name],
'Product'[Brand]
)
VAR ProductsBrandsSales =
ADDCOLUMNS (
ProductsBrands,
"Product Sales", [Sales Amount]
)
VAR Result =
TOPN (
3,
ProductsBrandsSales,
[Product Sales]
)
RETURN Result
ORDER BY [Product Sales] DESC
```

You can see the result of this query in Figure 13-14.

CHAPTER 13 Authoring queries 411
FIGURE 13-14 TOPN returns the top N rows of a table sorted by an expression.
The table can be sorted ascending or descending order to apply the top fi lter. By default, it is sorted
in descending order so that it returns the rows with the largest values fi rst. The third, optional parameter can change the sort order. The values can be 0 or FALSE for the default descending order, or 1 or
TRUE for the ascending order.
Important Do not confuse the sort order of TOPN with the sort order of the result of the
query; the latter is managed by the ORDER BY condition of the EVALUATE statement. The third
parameter of TOPN only affects how to sort the table generated internally by TOPN itself.
In the presence of ties, TOPN is not guaranteed to return the exact number of rows requested. Instead,
it returns all the rows with the same value. For example, in the following query we request the top four
brands, and we introduced a modifi ed calculation that uses MROUND to fi ctitiously introduce ties:

## Evaluate


```dax
VAR SalesByBrand =
ADDCOLUMNS (
VALUES ( 'Product'[Brand] ),
"Product Sales", MROUND ( [Sales Amount], 1000000 )
)
VAR Result =
TOPN (
4,
SalesByBrand,
[Product Sales]
)
RETURN Result
ORDER BY [Product Sales] DESC
```

The result contains fi ve rows, not just four, because both Litware and Proseware produce a result of
3,000,000. Finding ties and not knowing how to differentiate between the two, TOPN returns both, as
you can see in Figure 13-15.
FIGURE 13-15 In the presence of ties, TOPN might return more values than requested.

412 CHAPTER 13 Authoring queries
A common technique to avoid this problem is to add extra columns to the expression of TOPN.
Indeed, in the third parameter, multiple columns can be used to sort the result of TOPN. For example,
to retrieve the top four brands and to choose the fi rst brand in alphabetical order in case of a tie, you
can use additional sort orders:

## Evaluate


```dax
VAR SalesByBrand =
ADDCOLUMNS (
VALUES ( 'Product'[Brand] ),
"Product Sales", MROUND ( [Sales Amount], 1000000 )
)
VAR Result =
TOPN (
4,
SalesByBrand,
[Product Sales], 0,
'Product'[Brand], 1
)
RETURN Result
ORDER BY [Product Sales] DESC
```

The result shown in Figure 13-16 removes Proseware because alphabetically it comes after Litware.
Please note that in the query, we used a descending order for the sales and an ascending order for the
brand.
FIGURE 13-16 Using additional sort orders, one can remove ties in the table.
Be mindful that adding columns to the sort order does not guarantee that only the right number of
rows will be returned. TOPN can always return multiple rows in the presence of ties. Adding columns to
the sort order only mitigates the problem by reducing the number of ties. If one needs a guarantee to
retrieve an exact number of rows, then a column with unique values should be added to the sort order,
removing any possible ties.
Consider a more complex example where TOPN is mixed with set functions and variables. The
requirement is a report showing the sales of the top 10 products plus an additional “Others” row showing the sales of all other products combined. A possible implementation is the following:

## Evaluate


```dax
VAR NumOfTopProducts = 10
VAR ProdsWithSales =
ADDCOLUMNS (
```

CHAPTER 13 Authoring queries 413
VALUES ( 'Product'[Product Name] ),
"Product Sales", [Sales Amount]
)

```dax
VAR TopNProducts =
TOPN (
NumOfTopProducts,
ProdsWithSales,
[Product Sales]
)
VAR RemainingProducts =
EXCEPT ( ProdsWithSales, TopNProducts )
VAR OtherRow =
ROW (
"Product Name", "Others",
"Product Sales", SUMX (
RemainingProducts,
[Product Sales]
)
)
VAR Result =
UNION ( TopNProducts, OtherRow )
RETURN Result
ORDER BY [Product Sales] DESC
```

The ProdsWithSales variable computes a table with products and sales. Then TopNProducts only
computes the top 10 products. The RemainingProducts variable uses EXCEPT to compute the products that are not in the top 10. Once the code has split the products into two sets (TopNProducts and
RemainingProducts), it builds a single-row table containing the string “Others”; it also aggregates all
the products in the RemainingProducts variable, summing all the remaining products. The result is then
the UNION of the top 10 products with the additional row, computed in the formula. The result is visible in Figure 13-17.
FIGURE 13-17 The additional row containing Others is created by the query.

414 CHAPTER 13 Authoring queries
Although correct, this result is not perfect yet. Indeed, the Others row appears at the beginning of
the report, but it could actually appear in any position depending on its value. One might want to sort
the rows in such a way that the Others row is always at the end of the report, while the top products are
sorted by their sales, with the top performer being fi rst.
The result can be achieved by introducing a sort column that moves the Others row to the end by
using a ranking based on Product Sales for the top rows:

## Evaluate


```dax
VAR NumOfTopProducts = 10
VAR ProdsWithSales =
ADDCOLUMNS (
VALUES ( 'Product'[Product Name] ),
"Product Sales", [Sales Amount]
)
VAR TopNProducts =
TOPN (
NumOfTopProducts,
ProdsWithSales,
[Product Sales]
)
VAR RemainingProducts =
EXCEPT ( ProdsWithSales, TopNProducts )
VAR RankedTopProducts =
ADDCOLUMNS(
TopNProducts,
"SortColumn", RANKX ( TopNProducts, [Product Sales] )
)
VAR OtherRow =
ROW (
"Product Name", "Others",
"Product Sales", SUMX (
RemainingProducts,
[Product Sales]
),
"SortColumn", NumOfTopProducts + 1
)
VAR Result =
UNION ( RankedTopProducts, OtherRow )
RETURN
Result
ORDER BY [SortColumn]
```

The result visible in Figure 13-18 is now sorted better.

CHAPTER 13 Authoring queries 415
FIGURE 13-18 The SortColumn index is how a developer can sort the results as desired.
Using GENERATE and GENERATEALL
GENERATE is a powerful function that implements the OUTER APPLY logic from the SQL language.
GENERATE takes two arguments: a table and an expression. It iterates the table, evaluates the expression in the row context of the iteration, and then joins the row of the iteration with the rows returned
by the table expression. Its behavior is like a regular join, but instead of joining with a table, it joins with
an expression evaluated for each row. It is an extremely versatile function.
To demonstrate its behavior, we extend the previous TOPN example. Instead of computing the top
products of all time, the requirement is to compute the top three products by year. We can split this
problem into two steps: fi rst, computing the top three products, and then repeating this calculation for
every year. One possible solution for the top three products is the following:

## Evaluate


```dax
VAR ProductsSold =
SUMMARIZE (
Sales,
'Product'[Product Name]
)
VAR ProductsSales =
ADDCOLUMNS (
ProductsSold,
"Product Sales", [Sales Amount]
)
VAR Top3Products =
TOPN (
3,
ProductsSales,
[Product Sales]
)
RETURN
Top3Products
ORDER BY [Product Sales] DESC
```

416 CHAPTER 13 Authoring queries
The result shown in Figure 13-19 contains just three products.
FIGURE 13-19 TOPN returns the top three products of all time.
If the previous query is evaluated in a fi lter context that fi lters the year, the result is different: It
returns the top three products of the given year. Here is where GENERATE comes in handy: We use
GENERATE to iterate the years, and for each year we compute the TOPN expression. During each iteration, TOPN returns the top three products of the selected year. Finally, GENERATE joins the years with
the result of the expression at each iteration. This is the complete query:

## Evaluate


## Generate (

VALUES ( 'Date'[Calendar Year] ),

## Calculatetable (


```dax
VAR ProductsSold =
SUMMARIZE ( Sales, 'Product'[Product Name] )
VAR ProductsSales =
ADDCOLUMNS ( ProductsSold, "Product Sales", [Sales Amount] )
VAR Top3Products =
TOPN ( 3, ProductsSales, [Product Sales] )
RETURN Top3Products
)
)
```


## Order By

'Date'[Calendar Year],
[Product Sales] DESC
The result of the query is visible in Figure 13-20.
FIGURE 13-20 GENERATE joins the years with the top three products by year.

CHAPTER 13 Authoring queries 417
If one needs to compute the top products by category, the only thing that needs to be updated
in the formula is the table iterated by GENERATE. The following produces the top three products by
category:

## Evaluate


## Generate (

VALUES ( 'Product'[Category] ),

## Calculatetable (


```dax
VAR ProductsSold =
SUMMARIZE ( Sales, 'Product'[Product Name] )
VAR ProductsSales =
ADDCOLUMNS ( ProductsSold, "Product Sales", [Sales Amount] )
VAR Top3Products =
TOPN ( 3, ProductsSales, [Product Sales] )
RETURN Top3Products
)
)
```


## Order By

'Product'[Category],
[Product Sales] DESC
As shown in Figure 13-21, the result now contains three products for each category.
FIGURE 13-21 Iterating over the categories, the result shows the top three products by category.
If the expression provided as the second argument of GENERATE produces an empty table, then
GENERATE skips the row from the result. If one needs to also retrieve rows of the fi rst table producing
an empty result, then GENERATEALL is needed. For example, there are no sales in 2005, so there are no
top three products in 2005; GENERATE does not return any row for 2005. The following query leverages GENERATEALL and returns 2005 and 2006:

## Evaluate


## Generateall (

VALUES ( 'Date'[Calendar Year] ),

## Calculatetable (


```dax
VAR ProductsSold =
SUMMARIZE ( Sales, 'Product'[Product Name] )
VAR ProductsSales =
ADDCOLUMNS ( ProductsSold, "Product Sales", [Sales Amount] )
```

418 CHAPTER 13 Authoring queries

```dax
VAR Top3Products =
TOPN ( 3, ProductsSales, [Product Sales] )
RETURN Top3Products
)
)
```


## Order By

'Date'[Calendar Year],
[Product Sales] DESC
The result of this query is visible in Figure 13-22.
FIGURE 13-22 GENERATEALL returns years for which there are no sales, whereas GENERATE did not.
Using ISONORAFTER
ISONORAFTER is a utility function. It is heavily used by Power BI and reporting tools to provide pagination, and it is seldom used by developers in queries and measures. When a user browses a report
in Power BI, the engine only retrieves the rows needed for the current page from the data model. To
obtain this, it always uses a TOPN function.
If a user is browsing a products table, they might reach a certain point during the scanning. For
example, in Figure 13-23 the last row shown is Stereo Bluetooth Headphones New Gen, and the arrow
shows the relative position in the list.

CHAPTER 13 Authoring queries 419
FIGURE 13-23 The user is browsing the Product table and has reached a certain point in the list.
When the user scrolls down, they might reach the bottom of the rows retrieved previously; at this
point, Power BI needs to retrieve the next rows. The query that retrieves the next rows will still be a
TOPN because Power BI always retrieves a subset of the whole data. Moreover, it needs to be the next
TOPN. This is where ISONORAFTER comes in. This is the full query executed by Power BI when scrolling
down, and its result is shown in Figure 13-24:

## Evaluate


## Topn (

501,

## Filter (


## Keepfilters (


## Summarizecolumns (

'Product'[Category],
'Product'[Color],
'Product'[Product Name],
"Sales_Amount", 'Sales'[Sales Amount]
)
),

## Isonorafter (

'Product'[Category], "Audio", ASC,
'Product'[Color], "Yellow", ASC,
'Product'[Product Name],
"WWI Stereo Bluetooth Headphones New Generation M370 Yellow", ASC
)
),
'Product'[Category], 1,
'Product'[Color], 1,
'Product'[Product Name], 1
)

## Order By

'Product'[Category],
'Product'[Color],
'Product'[Product Name]

420 CHAPTER 13 Authoring queries
FIGURE 13-24 This is the next set of rows starting from the last row in the previous fi gure.
The code executes a TOPN 501 of a FILTER. FILTER is used to remove previously retrieved rows, and
in order to obtain the scope, it leverages ISONORAFTER. That same condition of ISONORAFTER could
have been expressed with standard Boolean logic. Indeed, the whole preceding ISONORAFTER expression could be written this way:
'Product'[Category] > "Audio"
|| ( 'Product'[Category] = "Audio" && 'Product'[Color] > "Yellow" )
|| ( 'Product'[Category] = "Audio"
&& 'Product'[Color] = "Yellow"
&& 'Product'[Product Name]
>= "WWI Stereo Bluetooth Headphones New Generation M370 Yellow"
)
The advantage of using ISONORAFTER is twofold: The code is easier to write, and the query plan is
potentially better.
Using ADDMISSINGITEMS
ADDMISSINGITEMS is another function frequently used by Power BI and seldom used in authoring
data models. Its purpose is to add rows that might have been skipped by SUMMARIZECOLUMNS.
For example, the following query uses SUMMARIZECOLUMNS grouping by year; its result is visible in
Figure 13-25.

## Evaluate


## Summarizecolumns (

'Date'[Calendar Year],
"Amt", [Sales Amount]
)
ORDER BY 'Date'[Calendar Year]
FIGURE 13-25 SUMMARIZECOLUMNS does not include years without sales where Amt column would be blank.

CHAPTER 13 Authoring queries 421
Years with no sales are not returned by SUMMARIZECOLUMNS. To retrieve the rows removed by
SUMMARIZECOLUMNS, one option is to use ADDMISSINGITEMS:

## Evaluate


## Addmissingitems (

'Date'[Calendar Year],

## Summarizecolumns (

'Date'[Calendar Year],
"Amt", [Sales Amount]
),
'Date'[Calendar Year]
)
ORDER BY 'Date'[Calendar Year]
The result of this query is visible in Figure 13-26, where we highlighted the rows returned by SUMMARIZECOLUMNS. The rows with a blank in the Amt column were added by ADDMISSINGITEMS.
FIGURE 13-26 ADDMISSINGITEMS added the rows with a blank value for Amt.
ADDMISSINGITEMS accepts several modifi ers and parameters to better control the result for subtotals and other fi lters.
Using TOPNSKIP
The TOPNSKIP function is used extensively by Power BI to send just a few rows of a large raw dataset
to the Data View of Power BI. Other tools, such as Power Pivot and SQL Server Data Tools, use other
techniques to quickly browse and fi lter the raw data of a table. The reason for using them is to quickly
browse over a large table without having to wait for the materialization of the entire set of rows.
Both TOPNSKIP and other techniques are described in the article at http://www.sqlbi.com/articles/
querying-raw-data-to-tabular/.
Using GROUPBY
GROUPBY is a function used to group a table by one or more columns, aggregating other data similarly
to what is possible using ADDCOLUMNS and SUMMARIZE. The main difference between SUMMARIZE
and GROUPBY is that GROUPBY can group columns whose data lineage does not correspond to
columns in the data model, whereas SUMMARIZE can only use columns defi ned in the data model.

422 CHAPTER 13 Authoring queries
In addition, columns added by GROUPBY need to use an iterator that aggregates data such as SUMX,
AVERAGEX, or other “X” aggregation functions.
For example, consider the requirement to group sales by year and month and compute the sales
amount. This is a possible solution using GROUPBY; the query result is visible in Figure 13-27:

## Evaluate


## Groupby (

Sales,
'Date'[Calendar Year],
'Date'[Month],
'Date'[Month Number],
"Amt", AVERAGEX (

## Currentgroup (),

Sales[Quantity] * Sales[Net Price]
)
)

## Order By

'Date'[Calendar Year],
'Date'[Month Number]
FIGURE 13-27 GROUPBY in this example aggregates the average of the line amount by year and month.
Performance-wise, GROUPBY can be slow in handling larger datasets—tens of thousands of rows or
more. Indeed, GROUPBY performs the grouping after having materialized the table; it is thus not the
suggested option to scan larger datasets. Besides, most queries can be expressed more easily by using
the ADDCOLUMNS and SUMMARIZE pair. Indeed, the previous query is better written as:

## Evaluate


## Addcolumns (


## Summarize (

Sales,
'Date'[Calendar Year],
'Date'[Month],
'Date'[Month Number],
),
"Amt", AVERAGEX (
RELATEDTABLE ( Sales ),
Sales[Quantity] * Sales[Net Price]
)
)

## Order By


CHAPTER 13 Authoring queries 423
'Date'[Calendar Year],
'Date'[Month Number]
Note In the previous query, it is worthwhile to note that the result of SUMMARIZE is
a table containing columns from the Date table. Therefore, when AVERAGEX later iterates over the result of RELATEDTABLE, the table returned by RELATEDTABLE is the table of
the year and month currently iterated by ADDCOLUMNS over the result of SUMMARIZE.
Remember that data lineage is kept; therefore, the result of SUMMARIZE is a table along
with its data lineage.
One advantage of GROUPBY is its option to group by columns added to the query by ADDCOLUMNS
or SUMMARIZE. The following is an example where SUMMARIZE would not be an alternative:

## Evaluate


```dax
VAR AvgCustomerSales =
AVERAGEX (
Customer,
[Sales Amount]
)
VAR ClassifiedCustomers =
ADDCOLUMNS (
VALUES ( Customer[Customer Code] ),
"Customer Category", IF (
[Sales Amount] >= AvgCustomerSales,
"Above Average",
"Below Average"
)
)
VAR GroupedResult =
GROUPBY (
ClassifiedCustomers,
[Customer Category],
"Number of Customers", SUMX (
CURRENTGROUP (),
1
)
)
RETURN GroupedResult
ORDER BY [Customer Category]
```

You can see the result in Figure 13-28.
FIGURE 13-28 GROUPBY can group columns computed during the query.

424 CHAPTER 13 Authoring queries
The previous formula shows both the advantages and the disadvantages of GROUPBY at the same
time. Indeed, the code fi rst creates a new column in the customer table that checks if the customer
sales are above or below the average sales. It then groups by this temporary column, and it returns the
number of customers.
Grouping by a temporary column is a useful feature; however, to compute the number of customers,
the code needs to use a SUMX over a CURRENTGROUP using a constant expression of 1. The reason is
that columns added by GROUPBY need to be iterations over CURRENTGROUP. A simple function like
COUNTROWS ( CURRENTGROUP () ) would not work here.
There are only a few scenarios where GROUPBY is useful. In general, GROUPBY can be used when
there is the need to group by a column added in the query, but be mindful that the column used to
group by should have a small cardinality. Otherwise, you might face performance and memory consumption issues.
Using NATURALINNERJOIN and NATURALLEFTOUTERJOIN
DAX uses model relationships automatically whenever a developer runs a query. Still, it might be useful
to join two tables that have no relationships. For example, one might defi ne a variable containing a
table and then join a calculated table with that variable.
Consider the requirement to compute the average sales per category and to then build a report
showing the categories below, around, and above the average. This column is easy to compute with a
simple SWITCH function. However, if the results need to be sorted in a particular way, then it is necessary to compute both the category description and the sort order (as a new column) at the same time,
using a similar piece of code.
Another approach would be to compute only one of the two values and then use a temporary table
with a temporary relationship to retrieve the description. This is exactly what the following query does:

## Evaluate


```dax
VAR AvgSales =
AVERAGEX (
VALUES ( 'Product'[Brand] ),
[Sales Amount]
)
VAR LowerBoundary = AvgSales * 0.8
VAR UpperBoundary = AvgSales * 1.2
VAR Categories =
DATATABLE (
"Cat Sort", INTEGER,
"Category", STRING,
{
{ 0, "Below Average" },
{ 1, "Around Average" },
{ 2, "Above Average" }
}
)
VAR BrandsClassified =
```

CHAPTER 13 Authoring queries 425

## Addcolumns (

VALUES ( 'Product'[Brand] ),
"Sales Amt", [Sales Amount],
"Cat Sort", SWITCH (

## True (),

[Sales Amount] <= LowerBoundary, 0,
[Sales Amount] >= UpperBoundary, 2,
1
)
)

```dax
VAR JoinedResult =
NATURALINNERJOIN (
Categories,
BrandsClassified
)
RETURN JoinedResult
```


## Order By

[Cat Sort],
'Product'[Brand]
It is useful to look at the result of the query shown in Figure 13-29 before commenting on it.
FIGURE 13-29 The Cat Sort column must be used as the “sort by column” argument on Category.
The query fi rst builds a table containing the brands, the sales amounts, and a column with values
between 0 and 2. The value will be used as a key in the Categories variable to retrieve the category
description. This fi nal join between the temporary table and the variable is performed by NATURALINNERJOIN, which joins the two tables based on the Cat Sort column.
NATURALINNERJOIN performs the join between two tables based on columns that have the same
name in both tables. NATURALLEFTOUTERJOIN performs the same operation, but instead of an inner
join, it uses a left outer join. By using a left outer join, NATURALLEFTOUTERJOIN keeps rows in the fi rst
table even if there are no matches in the second table.
In case the two tables are physically defi ned in the data model, they can only be joined using a
relationship. This can be useful to obtain the result of the join between two tables—similarly to what is

426 CHAPTER 13 Authoring queries
possible in a SQL query. Both NATURALINNERJOIN and NATURALLEFTOUTERJOIN use the relationship
between the tables if it exists. Otherwise, they need the same data lineage to perform the join.
For example, this query returns all the rows in Sales that have corresponding rows in Product, only
including all the columns of the two tables once:

## Evaluate

NATURALINNERJOIN ( Sales, Product )
The following query returns all the rows in Product, also showing the products that have no Sales:

## Evaluate

NATURALLEFTOUTERJOIN ( Product, Sales )
In both cases, the column that defi nes the relationship is only present once in the result, which
includes all the other columns of the two tables.
However, one important limitation of these join functions is that they do not match two columns
of the data model with different data lineage and no relationship. In practice, two tables of the data
model that have one or more columns with the same name and no relationship cannot be joined
together. As a workaround, one can use TREATAS to change the data lineage of a column so that the
join becomes possible. The article at https://www.sqlbi.com/articles/from-sql-to-dax-joining-tables/
describes this limitation and a possible workaround in detail.
NATURALINNERJOIN or NATURALLEFTOUTERJOIN are useful in a limited number of cases; in DAX,
they are not as frequent as the equivalent join function in the SQL language.
Important NATURALINNERJOIN and NATURALLEFTOUTERJOIN are useful to join the
result of temporary tables, where the data lineage of certain columns does not point to
physical columns of the data model. In order to join tables in the model that do not have
a proper relationship, it is necessary to use TREATAS to change the data lineage of the
columns to use in the join operation.
Using SUBSTITUTEWITHINDEX
The SUBSTITUTEWITHINDEX function can replace the columns in a row set corresponding to the
column headers of a matrix, with indexes representing their positions. SUBSTITUTEWITHINDEX is not a
function a developer would use in a regular query because its behavior is quite intricate. One possible
usage might be when creating a dynamic user interface for querying DAX. Indeed, Power BI internally
uses SUBSTITUTEWITHINDEX for matrix charts.
For example, consider the Power BI matrix in Figure 13-30.

CHAPTER 13 Authoring queries 427
FIGURE 13-30 A matrix in Power BI is populated using a query with SUBSTITUTEWITHINDEX.
The result of a DAX query is always a table. Each cell of the matrix in the report corresponds to a
single row of the table returned by the DAX query. In order to correctly display the data in the report,
Power BI uses SUBSTITUTEWITHINDEX to translate the column names of the matrix (CY 2007, CY 2008,
and CY 2009) into sequential numbers, making it easier to populate the matrix when reading the result.
The following is a simplifi ed version of the DAX request generated for the previous matrix:

## Define


```dax
VAR SalesYearCategory =
SUMMARIZECOLUMNS (
'Product'[Category],
'Date'[Calendar Year],
"Sales_Amount", [Sales Amount]
)
VAR MatrixRows =
SUMMARIZE (
SalesYearCategory,
'Product'[Category]
)
VAR MatrixColumns =
SUMMARIZE (
SalesYearCategory,
'Date'[Calendar Year]
)
VAR SalesYearCategoryIndexed =
SUBSTITUTEWITHINDEX (
SalesYearCategory,
"ColumnIndex", MatrixColumns,
'Date'[Calendar Year], ASC
)
```

-- First result: matrix column headers

## Evaluate

MatrixColumns
ORDER BY 'Date'[Calendar Year]
-- Second result: matrix rows and content

## Evaluate


## Naturalleftouterjoin (


428 CHAPTER 13 Authoring queries
MatrixRows,
SalesYearCategoryIndexed
)

## Order By

'Product'[Category],
[ColumnIndex]
The request contains two EVALUATE statements. The fi rst EVALUATE returns the content of the
column headers, as shown in Figure 13-31.
FIGURE 13-31 Result of the column headers of a matrix in Power BI.
The second EVALUATE returns the remaining content of the matrix, providing one row for each cell
of the matrix content. Every row in the result has the columns required to populate the row header of
the matrix followed by the numbers to display, and one column containing the column index computed
by using the SUBSTITUTEWITHINDEX function. This is shown in Figure 13-32.
FIGURE 13-32 Result of the rows’ content of a matrix in Power BI generated using SUBSTITUTEWITHINDEX.
SUBSTITUTEWITHINDEX is mainly used to build visuals like the matrix in Power BI.
Using SAMPLE
SAMPLE returns a sample of rows from a table. Its arguments are the number of rows to be returned,
the table name, and a sort order. SAMPLE returns the fi rst and the last rows of the table, plus additional
rows up to exactly the number of rows requested. SAMPLE picks evenly distributed rows from the
source table.

CHAPTER 13 Authoring queries 429
For example, the following query returns exactly 10 products after having sorted the input table by
Product Name:

## Evaluate


## Sample (

10,

## Addcolumns (

VALUES ( 'Product'[Product Name] ),
"Sales", [Sales Amount]
),
'Product'[Product Name]
)
ORDER BY 'Product'[Product Name]
The result of the previous query is visible in Figure 13-33.
FIGURE 13-33 SAMPLE returns a subset of a table by choosing evenly distributed rows.
SAMPLE is useful for a DAX client tool to generate values for the axis of a chart. Another scenario is
an analysis where the user needs a sample of a table to perform a statistical calculation.
Understanding the auto-exists behavior in DAX queries
Many DAX functions use a behavior known as auto-exists. Auto-exists is a mechanism used when
a function joins two tables. It is important when authoring queries because, although it is usually
intuitive, it might produce unexpected results.
Consider the following expression:

## Evaluate


## Summarizecolumns (

'Product'[Category],

430 CHAPTER 13 Authoring queries
'Product'[Subcategory]
)

## Order By

'Product'[Category],
'Product'[Subcategory]
The result can be either the full cross-join of categories and subcategories, or only the existing
combinations of categories and subcategories. Indeed, each category contains just a subset of subcategories. Thus, the list of existing combinations is smaller than the full cross-join.
The most intuitive answer would be that SUMMARIZECOLUMNS only returns the existing
combination. This is exactly what happens because of the auto-exists feature. The result in Figure 13-34 shows no more than three subcategories for the Audio category, and not a list of all the
subcategories.
FIGURE 13-34 SUMMARIZECOLUMNS only returns the existing combinations of values.
Auto-exists kicks in whenever the query groups by columns coming from the same table. When the
auto-exists logic is used, existing combinations of values are generated exclusively. This reduces the
number of rows to evaluate, generating better query plans. On the other hand, if one uses columns
coming from different tables, then the result is different. If the columns used in SUMMARIZECOLUMNS
are from different tables, then the result is the full cross-join of the two tables. This is made visible by
the following query whose result is shown in Figure 13-35:

## Evaluate


## Summarizecolumns (

'Product'[Category],
'Date'[Calendar Year]
)

## Order By

'Product'[Category],
'Date'[Calendar Year]
Though the two tables are linked to the Sales table through relationships and there are years without transactions, the auto-exists logic is not used when the columns do not come from the same table.

CHAPTER 13 Authoring queries 431
FIGURE 13-35 Columns coming from different tables generate the full cross-join.
Be mindful that SUMMARIZECOLUMNS removes the columns if all the additional columns computing aggregation expressions are blank. Thus, if the previous query also includes the Sales Amount measure, SUMMARIZECOLUMNS removes the years and categories without sales, as shown in Figure 13-36:

## Define


```dax
MEASURE Sales[Sales Amount] =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
EVALUATE
```


## Summarizecolumns (

'Product'[Category],
'Date'[Calendar Year],
"Sales", [Sales Amount]
)

## Order By

'Product'[Category],
'Date'[Calendar Year]
FIGURE 13-36 The presence of an aggregation expression removes the rows with a blank result.

432 CHAPTER 13 Authoring queries
The behavior of the previous query does not correspond to an auto-exists logic because it is based
on the result of an expression that includes an aggregation. Constant expressions are ignored on this
basis. For example, the presence of a 0 instead of a blank generates a list with all the years and categories. The result of the following query is visible in Figure 13-37:

## Define


```dax
MEASURE Sales[Sales Amount] =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
EVALUATE
```


## Summarizecolumns (

'Product'[Category],
'Date'[Calendar Year],
"Sales", [Sales Amount] + 0 -- Returns 0 instead of blank
)

## Order By

'Product'[Category],
'Date'[Calendar Year]
FIGURE 13-37 An aggregation expression resulting in 0 instead of blank maintains the rows in the SUMMARIZECOLUMNS results.
However, the same approach does not produce additional combinations for columns coming from
the same table. The auto-exists behavior is always applied to columns of the same table. The following
query solely generates existing combinations of Category and Subcategory values, despite the measure
expression returning 0 instead of blank:

CHAPTER 13 Authoring queries 433

## Define


```dax
MEASURE Sales[Sales Amount] =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
EVALUATE
```


## Summarizecolumns (

'Product'[Category],
'Product'[Subcategory],
"Sales", [Sales Amount] + 0
)

## Order By

'Product'[Category],
'Product'[Subcategory]
The result is visible in Figure 13-38.
FIGURE 13-38 SUMMARIZECOLUMNS applies the auto-exists to columns from the same table even when aggregation expressions return 0.
It is important to consider the auto-exists logic when using ADDMISSINGITEMS. Indeed, ADDMISSINGITEMS only adds rows that are removed because of blank results in SUMMARIZECOLUMNS.
ADDMISSINGITEMS does not add rows removed by auto-exists for columns of the same table. The
following query thus returns the same result as the one shown in Figure 13-38:

## Define


```dax
MEASURE Sales[Sales Amount] =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
EVALUATE
```


## Addmissingitems (

'Product'[Category],
'Product'[Subcategory],

## Summarizecolumns (

'Product'[Category],
'Product'[Subcategory],
"Sales", [Sales Amount] + 0
),

434 CHAPTER 13 Authoring queries
'Product'[Category],
'Product'[Subcategory]
)

## Order By

'Product'[Category],
'Product'[Subcategory]
Auto-exists is an important aspect to consider when using SUMMARIZECOLUMNS. On the other
hand, the behavior of SUMMARIZE is different. SUMMARIZE always requires a table to use as a bridge
between the columns, acting as an auto-exists between different tables. For example, the following
SUMMARIZE produces just the combinations of category and year where there are corresponding rows
in the Sales table, as shown by the result in Figure 13-39:

## Evaluate


## Summarize (

Sales,
'Product'[Category],
'Date'[Calendar Year]
)
FIGURE 13-39 SUMMARIZE only returns combinations between categories and year where there are matching
rows in Sales.
The reason why nonexisting combinations are not returned is because SUMMARIZE uses the Sales
table as the starting point to perform the grouping. Thus, any value in category or year not referenced
in Sales is not part of the result. Even though the result is identical, SUMMARIZE and SUMMARIZECOLUMNS achieve the same result through different techniques.
Be mindful that the user experience might be different when using a specifi c client tool. Indeed, if a
user puts the category and the year in a Power BI report without including any measure, the result only
shows the existing combinations in the Sales table. The reason is not that auto-exists is in place. The
reason is that Power BI adds its own business rules to the auto-exists logic of DAX. A simple report with
just Year and Category in a table produces a complex query like the following one:

CHAPTER 13 Authoring queries 435

## Evaluate


## Topn (

501,

## Selectcolumns (


## Keepfilters (


## Filter (


## Keepfilters (


## Summarizecolumns (

'Date'[Calendar Year],
'Product'[Category],

```dax
"CountRowsSales", CALCULATE ( COUNTROWS ( 'Sales' ) )
)
),
OR (
NOT ( ISBLANK ( 'Date'[Calendar Year] ) ),
NOT ( ISBLANK ( 'Product'[Category] ) )
)
)
),
"'Date'[Calendar Year]", 'Date'[Calendar Year],
"'Product'[Category]", 'Product'[Category]
),
'Date'[Calendar Year], 1,
'Product'[Category], 1
)
```

The highlighted row shows that Power BI adds a hidden calculation that computes the number of
rows in Sales. Because SUMMARIZECOLUMNS removes all the rows where the aggregation expression
is blank, this results in a behavior similar to the auto-exists obtained by combining columns of the same
table.
Power BI only adds this calculation if there are no measures in the report, including a table that has
a many-to-one relationship with all the tables used in SUMMARIZECOLUMNS. As soon as one adds a
calculation by using a measure, Power BI stops this behavior and checks for the measure value instead
of the number of rows in Sales.
Overall, the behavior of SUMMARIZECOLUMNS and SUMMARIZE is intuitive most of the time.
However, in complex scenarios like many-to-many relationships, the results might be surprising. In this
short section we only introduced auto-exists. A more detailed explanation of how these functions work
in complex scenarios is available in the article “Understanding DAX Auto-Exist,” available at https://
www.sqlbi.com/articles/understanding-dax-auto-exist/. The article also shows how this behavior might
produce reports with unexpected—or just counterintuitive—results.
Conclusions
This chapter presented several functions that are useful to author queries. Always remember that
any of these functions (apart from SUMMARIZECOLUMNS and ADDMISSINGITEMS) can be used in
measures too. Some experience is needed to learn how to mix these functions together to build more
complex queries.

436 CHAPTER 13 Authoring queries
Here is the list of the most relevant topics covered in the chapter:

> **Note:** Some functions are more useful in queries. Others are so technical and specialized that their
purpose is more to serve client tools generating queries, rather than data modelers writing DAX
expressions manually. Regardless, it is important to read about all of them; at some point, it
could become necessary to read somebody else’s code, so basic knowledge of all the functions
is important.

> **Note:** EVALUATE introduces a query. Using EVALUATE, you can defi ne variables and measures that
only exist for the duration of the query.

> **Note:** EVALUATE cannot be used to create calculated tables. A calculated table comes from an expression. Thus, when creating a query for a calculated table, you cannot create local measures or
columns.

> **Note:** SUMMARIZE is useful to perform grouping, and it is usually side-by-side with ADDCOLUMNS.

> **Note:** SUMMARIZECOLUMNS is one-function-fi ts-all. It is useful and powerful to generate complex
queries, and it is used extensively by Power BI. However, SUMMARIZECOLUMNS cannot be
used in a fi lter context that contains a context transition. This usually prevents the use of
SUMMARIZECOLUMNS in measures.

> **Note:** TOPN is extremely useful to retrieve the top (or the bottom) performers out of a category.

> **Note:** GENERATE implements the OUTER APPLY logic of SQL. It becomes handy whenever you need
to produce a table with a fi rst set of columns that act as a fi lter and a second set of columns that
depends on the values of the fi rst set.

> **Note:** Many other functions are mostly useful for query generators.
Finally, remember that all the table functions described in previous chapters can be used to author
queries. The options available to produce queries are not limited to the functions demonstrated in this
chapter.