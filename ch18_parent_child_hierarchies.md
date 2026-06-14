# Chapter 18: Parent-child hierarchies

> **Sample files:** https://sql.bi/dax-221


Parent-child hierarchies are often used to represent charts of accounts,
stores, salespersons and such. Parent-child hierarchies have a peculiar way
of storing the hierarchy in the sense that they have a variable depth. In this
pattern we show how to use parent-child hierarchies to show budget, actual
and forecast values in a report using both a chart of accounts and a
geographic hierarchy.


### Introduction


In the Parent-child pattern the hierarchy is not defined by the presence of
columns in the table of the original data source. The hierarchy is based on a
structure where each node of the hierarchy is related to the key of its parent
node. For example, Figure 18-1 shows the first few rows of a parent-child
hierarchy that defines a geographic structure for sales.


*📊 Figure 18-1 The Entity table stores the key of the parent for each entity.*
*Based on this data structure, we need to display a hierarchy showing*

Contoso United States under Contoso North America, as shown in Figure
18-2.


*📊 Figure 18-2 The parent-child hierarchy derives from the parent keys.*
*The Parent-child pattern implements some sort of self-join of the table*

containing the entities, which is not supported in Tabular. Because of their
nature, parent-child hierarchies may also have a variable depth: the number
of levels traversing the hierarchy top to bottom can be different depending
on the navigated path. For these reasons, a parent-child hierarchy should be
implemented following the technique described in this pattern.

Parent-child hierarchies are often used with charts of accounts. In this case,
the nodes also define the sign to use to aggregate a value to its parent. The
chart of accounts in Figure 18-3 shows expenses that are subtracted from
the total – despite the numbers displayed being all positive – whereas
incomes are added.


*📊 Figure 18-3 A parent-child hierarchy used with charts of accounts may*
*define the sign to use to aggregate values.*


The DAX expressions aggregating data over a parent-child hierarchy must
consider the sign used to aggregate data at lower level of a hierarchy node.


### Basic Parent-child pattern


Neither hierarchies of variable depth nor self-joins are directly supported in
a Tabular model. The first step in handling parent-child hierarchies is to
flatten the hierarchical structure to a regular hierarchy made up of one
column for each possible level of the hierarchy. We must move from the
data structure of Figure 18-4 to that of Figure 18-5. In Figure 18-4 we only
have the three columns required to define a parent-child hierarchy.


*📊 Figure 18-4 The parent-child hierarchy shows a row for each node of the*
*hierarchy and a single column with the name, regardless of the number of*

levels in the hierarchy.

The full expansion of the parent-child hierarchy in this example requires
four levels. Figure 18-5 shows that there is one column for each level of the
hierarchy, named Level1 to Level4. The number of columns required
depends on the data, so it is possible to add additional levels to
accommodate for future changes in the data.


*📊 Figure 18-5 Flattened hierarchy where each level of the original parent-*
*child hierarchy is stored in a separate column.*


The first step is to create a technical column called EntityPath by using the
PATH function:


> *Calculated column in the Entity table*

The EntityPath column contains the full path to reach the node
corresponding to the row of the table, as shown in Figure 18-6. This
technical column is useful to define the Level columns.


*📊 Figure 18-6 The EntityPath technical column contains the traversal path to*
*reach the node from the root level.*


The code for all the Level columns is similar, and only differs in the value
assigned to the LevelNumber variable. This is the code for the Level1
column:


> *Calculated column in the Entity table*

The other columns have a different name and a different value assigned to
LevelNumber, corresponding to the relative position of their level in the
hierarchy. Once all the Level columns are defined, we hide them and create
a regular hierarchy in the table that includes all of them – all the Level
columns. Only exposing these columns through a hierarchy is important in
order to make sure they are used in properly by the user navigating a report.

If used straight in a report, the hierarchy still does not provide an optimal
result. Indeed, all the levels are always shown, even though they might
contain no value. Figure 18-7 shows a blank row under Contoso Asia
Online Store, even though the Level4 column for that node is blank - thus
meaning that the node can be expanded only three levels, not four.


*📊 Figure 18-7 Rows with blank names should be hidden, but this does not*
*happen by default.*


To hide the unwanted rows, for each row we must check whether the
current level is available by the visited node. This can be accomplished by
checking the depth of each node. We need a calculated column in the
hierarchy table containing the depth of the node defined by each row:


> *Calculated column in the Entity table*

We need two measures: EntityRowDepth returns the maximum depth of the
current node, whereas EntityBrowseDepth returns the current depth of the
matrix by leveraging the ISINSCOPE function:


> *Measure in the Entity table*


> *Measure in the Entity table*

Finally, we use these two measures to blank out the result if the
EntityRowDepth is greater than the browsing depth:


> *Measure in the StrategyPlan table*


> *Measure in the StrategyPlan table*

The report obtained by using the Total Base measure no longer contains
rows with an empty description, as shown in Figure 18-8.


*📊 Figure 18-8 The rows with a blank name have disappeared because Total*
*Base also returns blank in those cases.*


The same pattern must be applied to any measure that could be reported by
using the parent-child hierarchy.


### Chart of accounts hierarchy


The Chart of accounts pattern is a variation of the basic Parent-child
hierarchy pattern, where the hierarchy is also used to drive the calculations.
Each row in the hierarchy is tagged as either Income, Expense or Taxation.
Incomes need to be summed, whereas expenses and taxation must be

subtracted from the total. The Figure 18-9 shows the content of the table
containing the hierarchy items.


*📊 Figure 18-9 Each row in the hierarchy defines an AccountType that drives*
*the calculations.*


The implementation is similar to the Parent-child pattern, grouping the
calculation by AccountType and applying the proper sign to the calculation
depending on the value of AccountType:


> *Measure in the StrategyPlan table*

The Total measure can use both parent-child hierarchies: the hierarchy
defined in the Entity table – shown in the previous example – and the
hierarchy defined in the Account table, which is the subject of this section.

The formula in Total returns the right result for each node of the hierarchy.
However, in these types of reports it is commonly requested that the
numbers be shown as positive despite being expenses. The requirement can
be fulfilled by changing the sign of the result at the report level. The
following Total No Signs measure implements the calculation this way: It
first determines the sign to use for the report, and then it changes the sign
of the result in order to show expenses as positive numbers, even though
they are internally managed as negative numbers:


> *Measure in the StrategyPlan table*

The report obtained using Total No Signs is visible in Figure 18-10.


*📊 Figure 18-10 The result of the parent-child hierarchy using the Total No*
*Signs measure.*


The pattern shown above works fine if the chart of accounts contains the
AccountType column, which defines each item as being either an income or
an expense. Sometimes the chart of accounts has a different way of

defining the sign to use. For example, there could be a column defining the
sign to use when aggregating an account to its parent. This is the case of the
Operator column shown in Figure 18-11.


*📊 Figure 18-11 The operator column indicates the sign to use to aggregate*
*one account to its parent.*


In this case, the code to author is more complex. We need one column for
each level of the hierarchy, stating how that account needs to be shown
when aggregated at any given level of the hierarchy. A single account can
be aggregated at one level with a plus, but at a different level with a minus.

These columns need to be built from the bottom of the hierarchy. In this
example we need seven columns because there are seven levels. The
column indicates the sign to use when aggregating that specific item of the
hierarchy at the desired level. Figure 18-12 shows the result of the seven
columns in this example.


*📊 Figure 18-12 The columns from S L1 to S L7 show the sign required when*
*aggregating the account at the correspondent hierarchical level.*


For instance, examine the rows with AccountKey 4 and 5: account 4 (Sale
Revenue) must be summed when aggregated at levels 1, 2, 3 and 4,
whereas it is not visible at other levels. Account 5 (Cost of Goods Sold)
must be summed when aggregated at level 4, but it must be subtracted
when aggregated at levels 1, 2, and 3.

The DAX formula computing the sign at each level starts from the most
granular level – level 7 in our example. At this most granular level, the sign
to use is just the operator converted into +1 or -1, for convenience in
further calculations:


> *Calculated column in the Account table*

All the other columns (from level 1 to level 6) follow a similar pattern,
though for each level the DAX expression must consider the sign at the

more granular, adjacent level (stored in the PrevSign variable) and invert
the result when that level shows a “-“ sign, as shown in the column for
level 6:


> *Calculated column in the Account table*

Once the level columns are ready, the Signed Total measure computing the
total with custom signs is the following:


> *Measure in the StrategyPlan table*

We can compare the result of this last Signed Total measure with that of the
previous Total measure in Figure 18-13.


*📊 Figure 18-13 The two formulas return a different sign for the same node in*
*the hierarchy.*


The amount for “Internet” is negative in Total, because it is an expense.
However, in Signed Total the same row holds a positive number and it
becomes negative only when it traverses the Expense node, which is
aggregated to the parent with a minus sign.


### Security pattern for a parent-child hierarchy


A common security requirement for parent-child hierarchies is to restrict
the visibility to a node (or a set of nodes) including all of its children. In
that scenario, the PATHCONTAINS function is useful.

By applying the following expression to a security role on the Account
table, we limit the visibility to the node provided in the second argument of
PATHCONTAINS. This way, all the children of the node are made visible
to the user, because the node requested (2, corresponding to Income) is also
part of the AccountPath value of all the children nodes:

If we used the AccountKey column to limit the visibility, we would end up
limiting the visibility to only one row and the user would not see the
children nodes. By leveraging the path column, we can easily select
multiple rows by including all the nodes that can be reached when
traversing a path that includes the filtered node.

When the security role is active, the user can only see the nodes (and the
values) included in the tree starting from the Income node, as shown in

*📊 Figure 18-14.*

*📊 Figure 18-14 The hierarchy is limited to the node that is visible for the*
*active security role.*


The nodes above the Income node (Level3) no longer consider other
children nodes in the Total measure. In case this is misleading in the report,
consider removing the initial levels from the report (in this case Level1 and
Level2) or using different descriptions of the nodes in Level1 and Level2 in
order to better explain the result.

It is worth noting that the security role defined by using PATHCONTAINS
may slow down the performance if used with a hierarchy with thousands of
nodes. The expression in the role security must be evaluated for every node
of the hierarchy when the end user opens a connection, and
PATHCONTAINS can be expensive if it is applied to thousands of rows or
more.
