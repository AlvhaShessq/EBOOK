# Chapter 11: Handling hierarchies

Hierarchies are oftentimes present in data models to make it easier for the user to slice and dice using
predefi ned exploration paths. Nevertheless, DAX does not have any built-in function providing a calculation over hierarchies. Computing a simple calculation like the ratio to parent requires complex DAX
code, and the support for calculations over hierarchies proves to be a challenge in general.
However, it is worth learning the DAX code required to handle hierarchies because calculations over
hierarchies are very common. In this chapter, we show how to create basic calculations over hierarchies
and how to use DAX to transform a parent/child hierarchy into a regular hierarchy.
Computing percentages over hierarchies
A common requirement when dealing with hierarchies is to create a measure that behaves differently
depending on the level of the item selected. An example is the ratio to parent calculation. Ratio to
parent displays for each level the percentage of that level against its parent.
For instance, consider a hierarchy made of product category, subcategory, and product name.
A ratio to parent calculation shows the percentage of a category against the grand total, of a
subcategory against its category, and of a product against its subcategory. Thus, depending on the
level of the hierarchy, it shows a different calculation.
An example of this report is visible in Figure 11-1.
In Excel, one might create this calculation by using the PivotTable feature Show Values As, so that
the computation is performed by Excel. However, if you want to use the calculation regardless of specifi c features of the client, then it is better to create a new measure that performs the computation so
that the value is computed in the data model. Moreover, learning the technique comes handy in many
similar scenarios.
Unfortunately, computing the ratio to parent in DAX is not so easy. Here is the fi rst big DAX limitation we face: There is no way of building a generic ratio to parent measure that works on any arbitrary
combination of columns in a report. The reason is that inside DAX, there is no way of knowing how the
report was created or how the hierarchy was used in the client tool. DAX has no knowledge of the way
a user builds a report. It receives a DAX query; the query does not contain information about what is on
the rows, what is on the columns, or what slicers were used to build the report.

346 CHAPTER 11 Handling hierarchies
FIGURE 11-1 The PercOnParent measure is useful to better understand the values in a table.
Though a generic formula cannot be created, it is still possible to create a measure that computes
the correct percentages when used properly. Because there are three levels in the hierarchy (category,
subcategory, and product), we start with three different measures that compute three different percentages, one for each level:
PercOnSubcategory :=

## Divide (

[Sales Amount],

## Calculate (

[Sales Amount],
ALLSELECTED ( Product[Product Name] )
)
)
PercOnCategory :=

## Divide (

[Sales Amount],

## Calculate (

[Sales Amount],
ALLSELECTED ( Product[Subcategory] )
)
)
PercOnTotal :=

## Divide (

[Sales Amount],

## Calculate (

[Sales Amount],

CHAPTER 11 Handling hierarchies 347
ALLSELECTED ( Product[Category] )
)
)
These three measures compute the percentages needed. Figure 11-2 shows the results in a report.
FIGURE 11-2 The three measures work well only at the level where they have meaning.
You can see that the measures only show the correct values where they are relevant. Otherwise, they
return 100%, which is useless. Moreover, there are three different measures, but the goal is to only have
one measure showing different percentages at different levels. This is the next step.
We start by clearing the 100% out of the PercOnSubcategory measure. We want to avoid performing the calculation if the hierarchy is not showing the Product Name column on the rows. This means
checking if the Product Name is currently being fi ltered by the query that produces the matrix. There
is a specifi c function for this purpose: ISINSCOPE. ISINSCOPE returns TRUE if the column passed as the
argument is fi ltered and it is part of the columns used to perform the grouping. Thus, the formula can
be updated to this new expression:
PercOnSubcategory :=

## If (

ISINSCOPE ( Product[Product Name] ),

## Divide (

[Sales Amount],

## Calculate (

[Sales Amount],
ALLSELECTED ( Product[Product Name] )
)
)
)
Figure 11-3 shows the report using this new formula.

348 CHAPTER 11 Handling hierarchies
FIGURE 11-3 Using ISINSCOPE, we remove the useless 100% values from the PercOnSubcategory column.
The same technique can be used to remove the 100% from other measures. Be careful that in
PercOnCategory, we must check that Subcategory is in scope and Product Name is not. This is because
when the report is slicing by Product Name using the hierarchy, it is also slicing by Subcategory—
displaying a product rather than a subcategory. In order to avoid duplicating code to check these
conditions, a better option is to write a single measure that executes a different operation depending
on the level of the hierarchy visible—based on the ISINSCOPE condition tested from the bottom to the
top of the hierarchy levels. Here is the code for the PercOnParent measure:
PercOnParent :=

```dax
VAR CurrentSales = [Sales Amount]
VAR SubcategorySales =
CALCULATE (
[Sales Amount],
ALLSELECTED ( Product[Product Name] )
)
VAR CategorySales =
CALCULATE (
[Sales Amount],
ALLSELECTED ( Product[Subcategory] )
)
VAR TotalSales =
CALCULATE (
[Sales Amount],
ALLSELECTED ( Product[Category] )
)
VAR RatioToParent =
IF (
ISINSCOPE ( Product[Product Name] ),
DIVIDE ( CurrentSales, SubcategorySales ),
IF (
ISINSCOPE ( Product[Subcategory] ),
DIVIDE ( CurrentSales, CategorySales ),
IF (
```

CHAPTER 11 Handling hierarchies 349
ISINSCOPE ( Product[Category] ),
DIVIDE ( CurrentSales, TotalSales )
)
)
)
RETURN RatioToParent
Using the PercOnParent measure, the result is as expected, as you can see in Figure 11-4.
FIGURE 11-4 The PercOnParent measure merges the three columns computed before into a single column.
The three measures created previously are no longer useful. A single measure computes everything needed, putting the right value into a single column by detecting the level the hierarchy is being
browsed at.
Note The order of the IF conditions is important. We want to start by testing the innermost level of the hierarchy and then proceed one step at a time to check the outer levels.
Otherwise, if we reverse the order of the conditions, the results will be incorrect. It is important to remember that when the subcategory is fi ltered through the hierarchy, the category
is fi ltered too.
The PercOnParent measure written in DAX only works if the user puts the correct hierarchy on the
rows. For example, if the user replaces the category hierarchy with the color, the numbers reported are
hard to understand. Indeed, the measure always works considering the product hierarchy regardless of
whether it is used or not in the report.

350 CHAPTER 11 Handling hierarchies
Handling parent/child hierarchies
The native data model used by DAX does not support true parent/child hierarchies, such as the ones
found in a Multidimensional database in Analysis Services. However, several DAX functions are available to fl atten parent/child hierarchies into regular, column-based hierarchies. This is good enough for
most scenarios, though it means making an educated guess at design time about what the maximum
depth of the hierarchy will be. In this section, you learn how to use DAX functions to create a parent/
child hierarchy, often abbreviated as P/C.
You can see a classical P/C hierarchy in Figure 11-5.
Annabel
Michael Catherine Harry
Bill
Brad
Chris Vincent
Julie
FIGURE 11-5 The chart shows a graphical representation of a P/C hierarchy.
P/C hierarchies present certain unique qualities:

> **Note:** The number of levels is not always the same throughout the hierarchy. For example, the path
from Annabel to Michael has a depth of two levels, whereas in the same hierarchy the path from
Bill to Chris has a depth of three levels.

> **Note:** The hierarchy is normally represented in a single table, storing a link to the parent for each row.
The canonical representation of P/C hierarchies is visible in Figure 11-6.
FIGURE 11-6 A table containing a P/C hierarchy.
It is easy to see that the ParentKey is the key of the parent of each node. For example, for Catherine it
shows 6, which is the key of her parent, Annabel. The issue with this data model is that this time, the relationship is self-referenced; that is, the two tables involved in the relationship are really the same table.

CHAPTER 11 Handling hierarchies 351
A tabular data model does not support self-referencing relationships. Consequently, the data model
itself has to be modifi ed; the parent/child hierarchy needs to be turned into a regular hierarchy, based
on one column for each level of the hierarchy.
Before we delve into the details of P/C hierarchies handling, it is worth noting one last point. Look at
the table in Figure 11-7 containing the values we want to aggregate using the hierarchy.
FIGURE 11-7 This table contains the data for the P/C hierarchy.
The rows in the fact table contain references to both leaf-level and middle nodes in the hierarchy.
For example, the highlighted row references Annabel. Not only does Annabel have a value by herself,
she also has three children nodes. Therefore, when summarizing all her data, the formula needs to
aggregate both her numbers and her children’s values.
Figure 11-8 displays the result we want to achieve.
FIGURE 11-8 This report shows the result of browsing a P/C with a matrix visual.

352 CHAPTER 11 Handling hierarchies
There are many steps to cover before reaching the fi nal goal. Once the tables have been loaded
in the data model, the fi rst step is to create a calculated column that contains the path to reach each
node, respectively. In fact, because we cannot use standard relationships, we will need to use a set of
special functions available in DAX and designed for P/C hierarchies handling.
The new calculated column named FullPath uses the PATH function:
Persons[FullPath] = PATH ( Persons[PersonKey], Persons[ParentKey] )
PATH is a function that receives two parameters. The fi rst parameter is the key of the table (in this
case, Persons[PersonKey]), and the second parameter is the name of the column that holds the parent
key. PATH performs a recursive traversal of the table, and for each node it builds the path as a list of
keys separated by the pipe (|) character. In Figure 11-9, you can see the FullPath calculated column.
FIGURE 11-9 The FullPath column contains the complete path to reach each node, respectively.
The FullPath column by itself is not useful. However, it is important because it acts as the basis for
another set of calculated columns required to build the hierarchy. The next step is to build three calculated columns, one for each level of the hierarchy:
Persons[Level1] = LOOKUPVALUE(
Persons[Name],
Persons[PersonKey], PATHITEM ( Persons[FullPath], 1, INTEGER )
)
Persons[Level2] = LOOKUPVALUE(
Persons[Name],
Persons[PersonKey], PATHITEM ( Persons[FullPath], 2, INTEGER )
)
Persons[Level3] = LOOKUPVALUE(
Persons[Name],
Persons[PersonKey], PATHITEM ( Persons[FullPath], 3, INTEGER )
)

CHAPTER 11 Handling hierarchies 353
PersonKey equals the result of PATHITEM. PATHITEM returns the nth item in a column built with PATH,
or it returns blank if there is no such item when we request a number greater than the length of the
path. The resulting table is shown in Figure 11-10.
FIGURE 11-10 The Level columns contain the values to show in the hierarchy.
In this example, we used three columns because the maximum depth of the hierarchy is three. In a
real-world scenario, one needs to count the maximum number of levels of the hierarchy and to build
a number of columns big enough to hold all the levels. Thus, although the number of levels in a P/C
hierarchy should be fl exible, in order to implement hierarchies in a data model, that maximum number
needs to be set. It is a good practice to add a couple more levels to create space and enable any future
growth of the hierarchy without needing to update the data model.
Now, we need to transform the set of level columns into a hierarchy. Also, because none of the other
columns in the P/C is useful, we should hide everything else from the client tools. At this point, we can
create a report using the hierarchy on the rows and the sum of amounts on the values—but the result is
not yet as desired. Figure 11-11 displays the result in a matrix.
There are a couple problems with this report:

> **Note:** Under Annabel, two blank rows contain the value of Annabel herself.

> **Note:** Under Catherine, a blank row contains the value of Catherine herself. The same happens for
many other rows.
The hierarchy always shows three levels, even for paths where the maximum depth should be two
such as Harry, who has no children.
The three columns will be Level1, Level2, and Level3 and the only change is in the second PATHITEM
parameter, which is 1, 2, and 3. The calculated column uses LOOKUPVALUE to search a row where the

354 CHAPTER 11 Handling hierarchies
FIGURE 11-11 The P/C hierarchy is not exactly what we want because it shows too many rows.
These issues pertain to the visualization of the results. Other than that, the hierarchy computes the
correct values, because under Annabel’s row, you can see the values of all of Annabel’s children. The
important aspect of this solution is that we were able to mimic a self-referencing relationship (also
known as a recursive relationship) by using the PATH function to create a calculated column. The remaining part is solving the presentation issues, but at least things are moving toward the correct solution.
Our fi rst challenge is the removal of all the blank values. For example, the second row of the matrix
in the report accounts for an amount of 600 that should be visible for Annabel and not for blank. We
can solve this by modifying the formula for the Level columns. First, we remove all the blanks, repeating
the previous level if we reached the end of the path. Here, you see the pattern for Level2:
PC[Level2] =
IF ( PATHLENGTH ( Persons[FullPath] ) >= 2,

## Lookupvalue(

Persons[Name],
Persons[PersonKey], PATHITEM ( Persons[FullPath], 2, INTEGER )
),
Persons[Level1]
)

CHAPTER 11 Handling hierarchies 355
Level1 does not need to be modifi ed because there is always a fi rst level. Columns from Level3 must
follow the same pattern as Level2. With this new formula, the table looks like Figure 11-12.
FIGURE 11-12 With the new formula, the Level columns never contain a blank.
At this point, if you look at the report, the blank rows are gone. Yet, there are still too many rows. In
Figure 11-13, you can see the report with two rows highlighted.
FIGURE 11-13 The new report does not have blank rows.
Pay attention to the second and third rows of the report. In both cases, the matrix shows a single
row of the hierarchy (that is, the row of Annabel). We might want to show the second row because it
contains a relevant value for Annabel. However, we certainly do not want to see the third row because
the hierarchy is browsing too deep and the path of Annabel is no longer helpful. As you see, the decision whether to show or hide a node of the hierarchy depends on the depth of the node. We can let a
user expand Annabel up to the second level of the hierarchy, but we surely want to remove the third
level of Annabel.

356 CHAPTER 11 Handling hierarchies
We can store the length of the path needed to reach the row into a calculated column. The length
of the path shows that Annabel is a root node. Indeed, it is a node of level 1 with a path containing one
value only. Catherine, on the other hand, is a node of level 2 because she is a daughter of Annabel and
the path of Catherine is of length 2. Moreover, although it might not be so evident, Catherine is visible
at level 1 because her value is aggregated under the fi rst node of Annabel. In other words, even though
the name of Catherine is not present in the report at level 1, her amount is aggregated under her parent, which is Annabel. The name of Catherine is visible because her row contains Annabel as Level1.
Once we know the level of each node in the hierarchy, we can defi ne that each node be visible
whenever the report browses the hierarchy up to its level. When the report shows a level that is too
deep, then the node needs to be hidden. To implement this algorithm, two values are needed:

> **Note:** The depth of each node; this is a fi xed value for each row of the hierarchy, and as such, it can
safely be stored in a calculated column.

> **Note:** The current browsing depth of the report visual; this is a dynamic value that depends on the
current fi lter context. It needs to be a measure because its value changes depending on the
report and it has a different value for each row of the report. For example, Annabel is a node at
level 1, but she appears in three rows because the current depth of the report has three different values.
The depth of each node is easy to compute. We can add a new calculated column to the Persons
table with this simple expression:
Persons[NodeDepth] = PATHLENGTH ( Persons[FullPath] )
PATHLENGTH returns the length of a value computed by PATH. You can see the resulting calculated
column in Figure 11-14.
FIGURE 11-14 The NodeDepth column stores the depth of each node in a calculated column.
The NodeDepth column is easy to create. Computing the browsing depth is more diffi cult because
it needs to be computed in a measure. Nevertheless, the logic behind it is not very complex, and it is

CHAPTER 11 Handling hierarchies 357
similar to the technique you have already learned for standard hierarchies. The measure uses ISINSCOPE to discover which of the hierarchy columns is fi ltered versus not.
Moreover, the formula takes advantage of the fact that a Boolean value can be converted to a number, where TRUE has a value of 1 and FALSE has a value of 0:
BrowseDepth :=
ISINSCOPE ( Persons[Level1] ) +
ISINSCOPE ( Persons[Level2] ) +
ISINSCOPE ( Persons[Level3] )
Thus, if only Level1 is fi ltered, then the result is 1. If both Level1 and Level2 are fi ltered, but not Level3,
then the result is 2, and so on. You can see the result for the BrowseDepth measure in Figure 11-15.
FIGURE 11-15 The BrowseDepth measure computes the depth of browsing in the report.
We are nearing the resolution of the scenario. The last piece of information we need is that by
default, a report will hide rows that result in a blank value for all the displayed measures. Specifi cally,
we are going to use this behavior to hide the unwanted rows. By transforming the value of Amount into
a blank when we do not want it to appear in the report, we will be able to hide rows from the matrix.
Thus, the solution is going to use these elements:

> **Note:** The depth of each node, in the NodeDepth calculated column,

> **Note:** The depth of the current cell in the report, in the BrowseDepth measure,

> **Note:** A way to hide unwanted rows, by means of blanking the value of the result.
It is time to merge all this information into a single measure, as follows:
PC Amount :=

## If (

MAX (Persons[NodeDepth]) < [BrowseDepth],

## Blank (),

SUM(Sales[Amount])
)

358 CHAPTER 11 Handling hierarchies
To understand how this measure works, look at the report in Figure 11-16. It contains all the values
that are useful to grasp the behavior of the formula.
FIGURE 11-16 This report shows the result and all the partial measures used by the formula.
If you look at Annabel in the first row, you see that BrowseDepth equals 1 because this is the
root of the hierarchy. MaxNodeDepth, which is defined as MAX ( Persons[NodeDepth] ), has a
value of 2—meaning that the current node is not only showing data at level 1, but also data for
some children that are at level 2. Thus, the current node is showing data for some children too,
and for this reason it needs to be visible. The second line of Annabel, on the other hand, has a
BrowseDepth of 2 and a MaxNodeDepth of 1. The reason is that the filter context filters all the rows
where Level1 equals Annabel and Level2 equals Annabel, and there is only one row in the hierarchy
satisfying this condition—this is Annabel herself. But Annabel has a NodeDepth of 1, and because
the report is browsing at level 2, we need to hide the node. Indeed, the PC Amount measure
returns a blank.
It is useful to verify the behavior for other nodes by yourself. This way you can improve your
understanding of how the formula is working. Although one can simply return to this part of the
book and copy the formula whenever they need to, understanding it is a good exercise because it
forces you to think in terms of how the fi lter context interacts with various parts of the formula.
To reach the result, the last step is to remove all the columns that are not needed from the
report, leaving PC Amount alone. The visualization becomes the one we wanted, as you can see in
Figure 11-17.

CHAPTER 11 Handling hierarchies 359
FIGURE 11-17 Once the measure is left alone in the report, all unwanted rows disappear.
The biggest drawback of this approach is that the same pattern has to be used for any measure a
user may add to the report after the P/C hierarchy is in place. If a measure that does not have a blank for
unwanted rows is being used as a value, then all the rows will suddenly appear and disrupt the pattern.
At this point, the result is already satisfactory. Yet there is still a small problem. Indeed, if you look
at the total of Annabel, it is 3,200. Summed up, her children show a total of 2,600. There is a missing
amount of 600, which is the value of Annabel herself. Some might already be satisfi ed by this visualization: The value of a node is easy to calculate, by simply looking at the difference between its total and
the total of its children. However, if you compare this fi gure to the original goal, you see that in the fi nal
formula, the value of each node is clearly visible as a child of the node itself. The comparison is visible in
Figure 11-18, which shows the current and the desired results together.
FIGURE 11-18 The original goal has not been reached yet. We still need to show some rows.

360 CHAPTER 11 Handling hierarchies
At this point, the technique should be clear enough. To show a value for Annabel, we need to fi nd
a condition that lets us identify it as a node that should be made visible. In this case, the condition is
somewhat complex. The nodes that need to be made visible are non-leaf nodes—that is, they have
children—that have values for themselves. The code will make those nodes visible for one additional
level. All other nodes—that is, leaf nodes or nodes with no value associated—will follow the original
rule and be hidden when the hierarchy is browsing over their depth.
First, we need to create a calculated column in the PC table that indicates whether a node is a leaf.
The DAX expression is easy: leaves are nodes that are not parents of any other node. In order to check
the condition, we can count the number of nodes that have the current node as their parent. If it equals
zero, then we know that the current node is a leaf. The following code does this:
Persons[IsLeaf] =

```dax
VAR CurrentPersonKey = Persons[PersonKey]
VAR PersonsAtParentLevel =
CALCULATE (
COUNTROWS ( Persons ),
ALL ( Persons ),
Persons[ParentKey] = CurrentPersonKey
)
VAR Result = ( PersonsAtParentLevel = 0 )
RETURN Result
```

In Figure 11-19, the IsLeaf column has been added to the data model.
FIGURE 11-19 The IsLeaf column indicates which nodes are leaves of the hierarchy.
Now that we can identify leaves, it is time to write the fi nal formula for handling the P/C hierarchy:
FinalFormula =

```dax
VAR TooDeep = [MaxNodeDepth] + 1 < [BrowseDepth]
VAR AdditionalLevel = [MaxNodeDepth] + 1 = [BrowseDepth]
VAR Amount =
SUM ( Sales[Amount] )
VAR HasData =
NOT ISBLANK ( Amount )
VAR Leaf =
```

CHAPTER 11 Handling hierarchies 361

## Selectedvalue (

Persons[IsLeaf],

## False

)

```dax
VAR Result =
IF (
NOT TooDeep,
IF (
AdditionalLevel,
IF (
NOT Leaf && HasData,
Amount
),
Amount
)
)
RETURN
Result
```

The use of variables makes the formula easier to read. Here are some comments about their usage:

> **Note:** TooDeep checks if the browsing depth is greater than the maximum node depth plus one; that
is, it checks whether the browsing of the report is over the additional level.

> **Note:** AdditionalLevel checks if the current browsing level is the additional level for nodes that have
values for themselves and that are not leaves.

> **Note:** HasData checks if a node itself has a value.

> **Note:** Leaf checks whether a node is a leaf or not.

> **Note:** Result is the fi nal result of the formula, making it easy to change the measure result to inspect
intermediate steps during development.
The remaining part of the code is just a set of IF statements that check the various scenarios and
behave accordingly.
It is clear that if the data model had the ability to handle P/C hierarchies natively, then all this hard
work would have been avoided. After all, this is not an easy formula to digest because it requires a full
understanding of evaluation contexts and data modeling.

Important If the model is in compatibility level 1400, you can enable the behavior of a special property called Hide Members. Hide Members automatically hides blank members. This
property is unavailable in Power BI and in Power Pivot as of April 2019. A complete description of how to use this property in a Tabular model is available at https://docs.microsoft.com/
en-us/sql/analysis-services/what-s-new-in-sql-server-analysis- services-2017?view=sqlserver-2017. In case the tool you are using implements this important feature, then we
strongly suggest using the Hide Members property instead of implementing the complex
DAX code shown above to hide levels of an unbalanced hierarchy.

362 CHAPTER 11 Handling hierarchies
Conclusions
In this chapter you learned how to correctly handle calculations over hierarchies. As usual, we now
recap the most relevant topics covered in the chapter:

> **Note:** Hierarchies are not part of DAX. They can be built in the model, but from a DAX point of view
there is no way to reference a hierarchy and use it inside an expression.

> **Note:** In order to detect the level of a hierarchy, one needs to use ISINSCOPE. Although it is a simple
workaround, ISINSCOPE does not actually detect the browsing level; rather, it detects the presence of a fi lter on a column.

> **Note:** Computing simple percentages over the parent requires the ability to both analyze the current
level of a hierarchy and create a suitable set of fi lters to recreate the fi lter of the parent.

> **Note:** Parent/child hierarchies can be handled in DAX by using the predefi ned PATH function and by
building a proper set of columns, one for each level of the hierarchy.

> **Note:** Unary operators, often used in parent/child hierarchies, can prove to be a challenge; they can
therefore be handled in their simpler version (only +/-) by authoring rather complex DAX code.
Handling more complex scenarios requires even more complicated DAX code, which is beyond
the scope of this chapter.