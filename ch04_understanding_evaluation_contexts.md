# Chapter 4: Understanding evaluation contexts

At this point in the book, you have learned the basics of the DAX language. You know how to create
calculated columns and measures, and you have a good understanding of common functions used in
DAX. This is the chapter where you move to the next level in this language: After learning a solid theoretical background of the DAX language, you become a real DAX champion.
With the knowledge you have gained so far, you can already create many interesting reports, but
you need to learn evaluation contexts in order to create more complex formulas. Indeed, evaluation
contexts are the basis of all the advanced features of DAX.
We want to give a few words of warning to our readers. The concept of evaluation contexts is simple,
and you will learn and understand it soon. Nevertheless, you need to thoroughly understand several
subtle considerations and details. Otherwise, you will feel lost at a certain point on your DAX learning
path. We have been teaching DAX to thousands of users in public and private classes, so we know that
this is normal. At a certain point, you have the feeling that formulas work like magic because they work,
but you do not understand why. Do not worry: you will be in good company. Most DAX students reach
that point, and many others will reach it in the future. It simply means that evaluation contexts are not
clear enough to them. The solution, at that point, is easy: Come back to this chapter, read it again, and
you will probably fi nd something new that you missed during your fi rst read.
Moreover, evaluation contexts play an important role when using the CALCULATE function—which
is probably the most powerful and hard-to-learn DAX function. We introduce CALCULATE in
Chapter 5, “Understanding CALCULATE and CALCULATETABLE,” and then we use it throughout the rest
of the book. Understanding CALCULATE without having a solid understanding of evaluation contexts
is problematic. On the other hand, understanding the importance of evaluation contexts without having ever tried to use CALCULATE is nearly impossible. Thus, in our experience with previous books we
have written, this chapter and the subsequent one are the two that are always marked up and have the
corners of pages folded over.
In the rest of the book we will use these concepts. Then in Chapter 14, “Advanced DAX concepts,”
you will complete your learning of evaluation contexts with expanded tables. Beware that the content
of this chapter is not the defi nitive description of evaluation contexts just yet. A more detailed description of evaluation contexts is the description based on expanded tables, but it would be too hard to
learn about expanded tables before having a good understanding of the basics of evaluation contexts.
Therefore, we introduce the whole theory in different steps.

80 CHAPTER 4 Understanding Evaluation Contexts
Introducing evaluation contexts
There are two evaluation contexts: the fi lter context and the row context. In the next sections, you learn
what they are and how to use them to write DAX code. Before learning what they are, it is important to
state one point: They are different concepts, with different functionalities and a completely different usage.
The most common mistake of DAX newbies is that of confusing the two contexts as if the row context was a slight variation of a fi lter context. This is not the case. The fi lter context fi lters data, whereas
the row context iterates tables. When DAX is iterating, it is not fi ltering; and when it is fi ltering, it is not
iterating. Even though this is a simple concept, we know from experience that it is hard to imprint in
the mind. Our brain seems to prefer a short path to learning—when it believes there are some similarities, it uses them by merging the two concepts into one. Do not be fooled. Whenever you have the
feeling that the two evaluation contexts look the same, stop and repeat this sentence in your mind like
a mantra: “The fi lter context fi lters, the row context iterates, they are not the same.”
An evaluation context is the context under which a DAX expression is evaluated. In fact, any DAX
expression can provide different values in different contexts. This behavior is intuitive, and this is
the reason why one can write DAX code without learning about evaluation contexts in advance. You
probably reached this point in the book having authored DAX code without learning about evaluation
contexts. Because you want more, it is now time to be more precise, to set up the foundations of DAX
the right way, and to prepare yourself to unleash the full power of DAX.
Understanding fi lter contexts
Let us begin by understanding what an evaluation context is. All DAX expressions are evaluated inside a
context. The context is the “environment” within which the formula is evaluated. For example, consider
a measure such as
Sales Amount := SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
This formula computes the sum of quantity multiplied by price in the Sales table. We can use this
measure in a report and look at the results, as shown in Figure 4-1.
FIGURE 4-1 The measure Sales Amount, without a context, shows the grand total of sales.
This number alone does not look interesting. However, if you think carefully, the formula computes
exactly what one would expect: the sum of all sales amounts. In a real report, one is likely to slice the
value by a certain column. For example, we can select the product brand, use it on the rows, and the
matrix report starts to reveal interesting business insights as shown in Figure 4-2.

CHAPTER 4 Understanding Evaluation Contexts 81
FIGURE 4-2 Sum of Sales Amount, sliced by brand, shows the sales of each brand in separate rows.
The grand total is still there, but now it is the sum of smaller values. Each value, together with all the
others, provides more detailed insights. However, you should note that something weird is happening:
The formula is not computing what we apparently asked. In fact, inside each cell of the report, the
formula is no longer computing the sum of all sales. Instead, it computes the sales of a given brand.
Finally, note that nowhere in the code does it say that it can (or should) work on subsets of data. This
fi ltering happens outside of the formula.
Each cell computes a different value because of the evaluation context under which DAX executes
the formula. You can think of the evaluation context of a formula as the surrounding area of the cell
where DAX evaluates the formula.
DAX evaluates all formulas within a respective context. Even though the formula is
the same, the result is different because DAX executes the same code against different
subsets of data.
This context is named Filter Context and, as the name suggests, it is a context that fi lters tables.
Any formula ever authored will have a different value depending on the fi lter context used to perform
its evaluation. This behavior, although intuitive, needs to be well understood because it hides many
complexities.
Every cell of the report has a different fi lter context. You should consider that every cell has a different evaluation—as if it were a different query, independent from the other cells in the same report.
The engine might perform some level of internal optimization to improve computation speed, but you
should assume that every cell has an independent and autonomous evaluation of the underlying DAX
expression. Therefore, the computation of the Total row in Figure 4-2 is not computed by summing the
other rows of the report. It is computed by aggregating all the rows of the Sales table, although this
means other iterations were already computed for the other rows in the same report. Consequently,

82 CHAPTER 4 Understanding Evaluation Contexts
depending on the DAX expression, the result in the Total row might display a different result, unrelated
to the other rows in the same report.
Note In these examples, we are using a matrix for the sake of simplicity. We can defi ne an
evaluation context with queries too, and you will learn more about it in future chapters. For
now, it is better to keep it simple and only think of reports, to have a simplifi ed and visual
understanding of the concepts.
When Brand is on the rows, the fi lter context fi lters one brand for each cell. If we increase the complexity of the matrix by adding the year on the columns, we obtain the report in Figure 4-3.
FIGURE 4-3 Sales amount is sliced by brand and year.
Now each cell shows a subset of data pertinent to one brand and one year. The reason for this is that
the fi lter context of each cell now fi lters both the brand and the year. In the Total row, the fi lter is only
on the brand, whereas in the Total column the fi lter is only on the year. The grand total is the only cell
that computes the sum of all sales because—there—the fi lter context does not apply any fi lter to the
model.
The rules of the game should be clear at this point: The more columns we use to slice and dice,
the more columns are being fi ltered by the fi lter context in each cell of the matrix. If one adds the
Store[Continent] column to the rows, the result is—again—different, as shown in Figure 4-4.

CHAPTER 4 Understanding Evaluation Contexts 83
FIGURE 4-4 The context is defi ned by the set of fi elds on rows and on columns.
Now the fi lter context of each cell is fi ltering brand, country, and year. In other words, the fi lter context contains the complete set of fi elds that one uses on rows and columns of the report.
Note Whether a fi eld is on the rows or on the columns of the visual, or on the slicer and/or
page/report/visual fi lter, or in any other kind of fi lter we can create with a report—all this
is irrelevant. All these fi lters contribute to defi ne a single fi lter context, which DAX uses to
evaluate the formula. Displaying a fi eld on rows or columns is useful for aesthetic purposes,
but nothing changes in the way DAX computes values.
Visual interactions in Power BI compose a fi lter context by combining different elements from a
graphical interface. Indeed, the fi lter context of a cell is computed by merging together all the fi lters
coming from rows, columns, slicers, and any other visual used for fi ltering. For example, look at
Figure 4-5.

84 CHAPTER 4 Understanding Evaluation Contexts
FIGURE 4-5 In a typical report, the context is defi ned in many ways, including slicers, fi lters, and other visuals.
The fi lter context of the top-left cell (A.Datum, CY 2007, 57,276.00) not only fi lters the row and the
column of the visual, but it also fi lters the occupation (Professional) and the continent (Europe), which
are coming from different visuals. All these fi lters contribute to the defi nition of a single fi lter context
valid for one cell, which DAX applies to the whole data model prior to evaluating the formula.
A more formal defi nition of a fi lter context is to say that a fi lter context is a set of fi lters. A fi lter,
in turn, is a list of tuples, and a tuple is a set of values for some defi ned columns. Figure 4-6 shows a
visual representation of the fi lter context under which the highlighted cell is evaluated. Each element
of the report contributes to creating the fi lter context, and every cell in the report has a different fi lter
context.
Calendar Year

## Cy 2007

Education
High School
Partial College
Brand
Contoso
FIGURE 4-6 The fi gure shows a visual representation of a fi lter context in a Power BI report.
The fi lter context of Figure 4-6 contains three fi lters. The fi rst fi lter contains a tuple for Calendar Year
with the value CY 2007. The second fi lter contains two tuples for Education with the values High School
and Partial College. The third fi lter contains a single tuple for Brand, with the value Contoso. You might

CHAPTER 4 Understanding Evaluation Contexts 85
notice that each fi lter contains tuples for one column only. You will learn how to create tuples with
multiple columns later. Multi-column tuples are both powerful and complex tools in the hand of a DAX
developer.
Before leaving this introduction, let us recall the measure used at the beginning of this section:
Sales Amount := SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
Here is the correct way of reading the previous measure: The measure computes the sum of Quantity
multiplied by Net Price for all the rows in Sales which are visible in the current fi lter context.
The same applies to simpler aggregations. For example, consider this measure:
Total Quantity := SUM ( Sales[Quantity] )
It sums the Quantity column of all the rows in Sales that are visible in the current fi lter context. You
can better understand its working by considering the corresponding SUMX version:
Total Quantity := SUMX ( Sales, Sales[Quantity] )
Looking at the SUMX defi nition, we might consider that the fi lter context affects the evaluation of
the Sales expression, which only returns the rows of the Sales table that are visible in the current fi lter
context. This is true, but you should consider that the fi lter context also applies to the following measures, which do not have a corresponding iterator:
Customers := DISTINCTCOUNT ( Sales[CustomerKey] ) -- Count customers in filter context
Colors :=

```dax
VAR ListColors = DISTINCT ( 'Product'[Color] ) -- Unique colors in filter context
RETURN COUNTROWS ( ListColors ) -- Count unique colors
```

It might look pedantic, at this point, to spend so much time stressing the concept that a fi lter context is always active, and that it affects the formula result. Nevertheless, keep in mind that DAX requires
you to be extremely precise. Most of the complexity of DAX is not in learning new functions. Instead,
the complexity comes from the presence of many subtle concepts. When these concepts are mixed
together, what emerges is a complex scenario. Right now, the fi lter context is defi ned by the report. As
soon as you learn how to create fi lter contexts by yourself (a critical skill described in the next chapter),
being able to understand which fi lter context is active in each part of your formula will be of paramount importance.
Understanding the row context
In the previous section, you learned about the fi lter context. In this section, you now learn the second
type of evaluation context: the row context. Remember, although both the row context and the fi lter
context are evaluation contexts, they are not the same concept. As you learned in the previous section,
the purpose of the fi lter context is, as its name implies, to fi lter tables. On the other hand, the row context is not a tool to fi lter tables. Instead, it is used to iterate over tables and evaluate column values.

86 CHAPTER 4 Understanding Evaluation Contexts
This time we use a different formula for our considerations, defi ning a calculated column to compute the gross margin:
Sales[Gross Margin] = Sales[Quantity] * ( Sales[Net Price] - Sales[Unit Cost] )
There is a different value for each row in the resulting calculated column, as shown in Figure 4-7.
FIGURE 4-7 There is a different value in each row of Gross Margin, depending on the value of other columns.
As expected, for each row of the table there is a different value in the calculated column. Indeed,
because there are given values in each row for the three columns used in the expression, it comes as a
natural consequence that the fi nal expression computes different values. As it happened with the fi lter
context, the reason is the presence of an evaluation context. This time, the context does not fi lter a
table. Instead, it identifi es the row for which the calculation happens.
Note The row context references a row in the result of a DAX table expression. It should
not be confused with a row in the report. DAX does not have a way to directly reference a
row or a column in the report. The values displayed in a matrix in Power BI and in a PivotTable in Excel are the result of DAX measures computed in a fi lter context, or are values
stored in the table as native or calculated columns.
In other words, we know that a calculated column is computed row by row, but how does DAX know
which row it is currently iterating? It knows the row because there is another evaluation context providing the row—it is the row context. When we create a calculated column over a table with one million
rows, DAX creates a row context that evaluates the expression iterating over the table row by row,
using the row context as the cursor.

CHAPTER 4 Understanding Evaluation Contexts 87
When we create a calculated column, DAX creates a row context by default. In that case, there is no
need to manually create a row context: A calculated column is always executed in a row context. You
have already learned how to create a row context manually—by starting an iteration. In fact, one can
write the gross margin as a measure, like in the following code:
Gross Margin :=

## Sumx (

Sales,
Sales[Quantity] * ( Sales[Net Price] - Sales[Unit Cost] )
)
In this case, because the code is for a measure, there is no automatic row context. SUMX, being an
iterator, creates a row context that starts iterating over the Sales table, row by row. During the iteration,
it executes the second expression of SUMX inside the row context. Thus, during each step of the iteration, DAX knows which value to use for the three column names used in the expression.
The row context exists when we create a calculated column or when we are computing an expression inside an iteration. There is no other way of creating a row context. Moreover, it helps to think
that a row context is needed whenever we want to obtain the value of a column for a certain row. For
example, the following measure defi nition is invalid. Indeed, it tries to compute the value of Sales[Net
Price] and there is no row context providing the row for which the calculation needs to be executed:
Gross Margin := Sales[Quantity] * ( Sales[Net Price] - Sales[Unit Cost] )
This same expression is valid when executed for a calculated column, and it is invalid if used in a
measure. The reason is not that measures and calculated columns have different ways of using DAX.
The reason is that a calculated column has an automatic row context, whereas a measure does not. If
one wants to evaluate an expression row by row inside a measure, one needs to start an iteration to
create a row context.
Note A column reference requires a row context to return the value of the column from a
table. A column reference can be also used as an argument for several DAX functions without a row context. For example, DISTINCT and DISTINCTCOUNT can have a column reference as a parameter, without defi ning a row context. Nonetheless, a column reference in a
DAX expression requires a row context to be evaluated.
At this point, we need to repeat one important concept: A row context is not a special kind of fi lter
context that fi lters one row. The row context is not fi ltering the model in any way; the row context only
indicates to DAX which row to use out of a table. If one wants to apply a fi lter to the model, the tool to
use is the fi lter context. On the other hand, if the user wants to evaluate an expression row by row, then
the row context will do the job.

88 CHAPTER 4 Understanding Evaluation Contexts
Testing your understanding of evaluation contexts
Before moving on to more complex descriptions about evaluation contexts, it is useful to test your
understanding of contexts with a couple of examples. Please do not look at the explanation immediately; stop after the question and try to answer it. Then read the explanation to make sense of it. As a
hint, try to remember, while thinking, ”The fi lter context fi lters; the row context iterates. This means that
the row context does not fi lter, and the fi lter context does not iterate.”
Using SUM in a calculated column
The fi rst test uses an aggregator inside a calculated column. What is the result of the following expression, used in a calculated column, in Sales?
Sales[SumOfSalesQuantity] = SUM ( Sales[Quantity] )
Remember, this internally corresponds to this equivalent syntax:
Sales[SumOfSalesQuantity] = SUMX ( Sales, Sales[Quantity] )
Because it is a calculated column, it is computed row by row in a row context. What number do you
expect to see? Choose from these three answers:

> **Note:** The value of Quantity for that row, that is, a different value for each row.

> **Note:** The total of Quantity for all the rows, that is, the same value for all the rows.

> **Note:** An error; we cannot use SUM inside a calculated column.
Stop reading, please, while we wait for your educated guess before moving on.
Here is the correct reasoning. You have learned that the formula means, “the sum of quantity for all
the rows visible in the current fi lter context.” Moreover, because the code is executed for a calculated
column, DAX evaluates the formula row by row, in a row context. Nevertheless, the row context is not
fi ltering the table. The only context that can fi lter the table is the fi lter context. This turns the question
into a different one: What is the fi lter context, when the formula is evaluated? The answer is straightforward: The fi lter context is empty. Indeed, the fi lter context is created by visuals or by queries, and a
calculated column is computed at data refresh time when no fi ltering is happening. Thus, SUM works
on the whole Sales table, aggregating the value of Sales[Quantity] for all the rows of Sales.
The correct answer is the second answer. This calculated column computes the same value for each
row, that is, the grand total of Sales[Quantity] repeated for all the rows. Figure 4-8 shows the result of
the SumOfSalesQuantity calculated column.

CHAPTER 4 Understanding Evaluation Contexts 89
FIGURE 4-8 SUM ( Sales[Quantity] ), in a calculated column, is computed against the entire database.
This example shows that the two evaluation contexts exist at the same time, but they do not interact.
The evaluation contexts both work on the result of a formula, but they do so in different ways.
Aggregators like SUM, MIN, and MAX only use the fi lter context, and they ignore the row context. If
you have chosen the fi rst answer, as many students typically do, it is perfectly normal. The thing is that
you are still confusing the fi lter context and the row context. Remember, the fi lter context fi lters; the
row context iterates. The fi rst answer is the most common, when using intuitive logic, but it is wrong—
now you know why. However, if you chose the correct answer ... then we are glad this section helped
you in learning the important difference between the two contexts.
Using columns in a measure
The second test is slightly different. Imagine we defi ne the formula for the gross margin in a measure
instead of in a calculated column. We have a column with the net price, another column for the product
cost, and we write the following expression:
GrossMargin% := ( Sales[Net Price] - Sales[Unit Cost] ) / Sales[Unit Cost]
What will the result be? As it happened earlier, choose among the three possible answers:

> **Note:** The expression works correctly, time to test the result in a report.

> **Note:** An error, we should not even write this formula.

> **Note:** We can defi ne the formula, but it will return an error when used in a report.
As in the previous test, stop reading, think about the answer, and then read the following
explanation.

90 CHAPTER 4 Understanding Evaluation Contexts
The code references Sales[Net Price] and Sales[Unit Cost] without any aggregator. As such, DAX
needs to retrieve the value of the columns for a certain row. DAX has no way of detecting which row
the formula needs to be computed for because there is no iteration happening and the code is not in a
calculated column. In other words, DAX is missing a row context that would make it possible to retrieve
a value for the columns that are part of the expression. Remember that a measure does not have an
automatic row context; only calculated columns do. If we need a row context in a measure, we should
start an iteration.
Thus, the second answer is the correct one. We cannot write the formula because it is syntactically
wrong, and we get an error when trying to enter the code.
Using the row context with iterators
You learned that DAX creates a row context whenever we defi ne a calculated column or when we start
an iteration with an X-function. When we use a calculated column, the presence of the row context is
simple to use and understand. In fact, we can create simple calculated columns without even knowing
about the presence of the row context. The reason is that the row context is created automatically by
the engine. Therefore, we do not need to worry about the presence of the row context. On the other
hand, when using iterators we are responsible for the creation and the handling of the row context.
Moreover, by using iterators we can create multiple nested row contexts; this increases the complexity
of the code. Therefore, it is important to understand more precisely the behavior of row contexts with
iterators.
For example, look at the following DAX measure:
IncreasedSales := SUMX ( Sales, Sales[Net Price] * 1.1 )
Because SUMX is an iterator, SUMX creates a row context on the Sales table and uses it during the
iteration. The row context iterates the Sales table (fi rst parameter) and provides the current row to the
second parameter during the iteration. In other words, DAX evaluates the inner expression (the second
parameter of SUMX) in a row context containing the currently iterated row on the fi rst parameter.
Please note that the two parameters of SUMX use different contexts. In fact, any piece of DAX code
works in the context where it is called. Thus, when the expression is executed, there might already be a
fi lter context and one or many row contexts active. Look at the same expression with comments:

## Sumx (

Sales, -- External filter and row contexts
Sales[Net Price] * 1.1 -- External filter and row contexts + new row context
)
The fi rst parameter, Sales, is evaluated using the contexts coming from the caller. The second
parameter (the expression) is evaluated using both the external contexts plus the newly created row
context.

CHAPTER 4 Understanding Evaluation Contexts 91
All iterators behave the same way:
1. Evaluate the fi rst parameter in the existing contexts to determine the rows to scan.
2. Create a new row context for each row of the table evaluated in the previous step.
3. Iterate the table and evaluate the second parameter in the existing evaluation context, including the newly created row context.
4. Aggregate the values computed during the previous step.
Be mindful that the original contexts are still valid inside the expression. Iterators add a new row
context; they do not modify existing fi lter contexts. For example, if the outer fi lter context contains a
fi lter for the color Red, that fi lter is still active during the whole iteration. Besides, remember that the
row context iterates; it does not fi lter. Therefore, no matter what, we cannot override the outer fi lter
context using an iterator.
This rule is always valid, but there is an important detail that is not trivial. If the previous contexts
already contained a row context for the same table, then the newly created row context hides the
previous existing row context on the same table. For DAX newbies, this is a possible source of mistakes.
Therefore, we discuss row context hiding in more detail in the next two sections.
Nested row contexts on different tables
The expression evaluated by an iterator can be very complex. Moreover, the expression can, on its own,
contain further iterations. At fi rst sight, starting an iteration inside another iteration might look strange.
Still, it is a common DAX practice because nesting iterators produce powerful expressions.
For example, the following code contains three nested iterators, and it scans three tables: Categories, Products, and Sales.

## Sumx (

'Product Category', -- Scans the Product Category table
SUMX ( -- For each category
RELATEDTABLE ( 'Product' ), -- Scans the category products
SUMX ( -- For each product
RELATEDTABLE ( Sales ) -- Scans the sales of that product
Sales[Quantity] --
* 'Product'[Unit Price] -- Computes the sales amount of that sale
* 'Product Category'[Discount]
)
)
)
The innermost expression—the multiplication of three factors—references three tables. In fact,
three row contexts are opened during that expression evaluation: one for each of the three tables that
are currently being iterated. It is also worth noting that the two RELATEDTABLE functions return the
rows of a related table starting from the current row context. Thus, RELATEDTABLE ( Product), being

92 CHAPTER 4 Understanding Evaluation Contexts
executed in a row context from the Categories table, returns the products of the given category. The
same reasoning applies to RELATEDTABLE ( Sales ), which returns the sales of the given product.
The previous code is suboptimal in terms of both performance and readability. As a rule, it is fi ne to
nest iterators provided that the number of rows to scan is not too large: hundreds is good, thousands
is fi ne, millions is bad. Otherwise, we may easily hit performance issues. We used the previous code to
demonstrate that it is possible to create multiple nested row contexts; we will see more useful examples
of nested iterators later in the book. One can express the same calculation in a much faster and readable way by using the following code, which relies on one individual row context and the RELATED
function:

## Sumx (

Sales,
Sales[Quantity]
* RELATED ( 'Product'[Unit Price] )
* RELATED ( 'Product Category'[Discount] )
)
Whenever there are multiple row contexts on different tables, one can use them to reference the
iterated tables in a single DAX expression. There is one scenario, however, which proves to be challenging.
This happens when we nest multiple row contexts on the same table, which is the topic covered in the
following section.
Nested row contexts on the same table
The scenario of having nested row contexts on the same table might seem rare. However, it does happen quite often, and more frequently in calculated columns. Imagine we want to rank products based
on the list price. The most expensive product should be ranked 1, the second most expensive product
should be ranked 2, and so on. We could solve the scenario using the RANKX function. But for educational purposes, we show how to solve it using simpler DAX functions.
To compute the ranking, for each product we can count the number of products whose price is
higher than the current product’s. If there is no product with a higher price than the current product
price, then the current product is the most expensive and its ranking is 1. If there is only one product
with a higher price, then the ranking is 2. In fact, what we are doing is computing the ranking of a
product by counting the number of products with a higher price and adding 1 to the result.
Therefore, one can author a calculated column using this code, where we used PriceOfCurrentProduct as a placeholder to indicate the price of the current product.
1. 'Product'[UnitPriceRank] =
2. COUNTROWS (

```dax
3. FILTER (
```

4. 'Product',
5. 'Product'[Unit Price] > PriceOfCurrentProduct
6. )
7. ) + 1

CHAPTER 4 Understanding Evaluation Contexts 93
FILTER returns the products with a price higher than the current products’ price, and COUNTROWS
counts the rows of the result of FILTER. The only remaining issue is fi nding a way to express the price of
the current product, replacing PriceOfCurrentProduct with a valid DAX syntax. By “current,” we mean
the value of the column in the current row when DAX computes the column. It is harder than you might
expect.
Focus your attention on line 5 of the previous code. There, the reference to Product[Unit Price] refers
to the value of Unit Price in the current row context. What is the active row context when DAX executes
row number 5? There are two row contexts. Because the code is written in a calculated column, there is
a default row context automatically created by the engine that scans the Product table. Moreover,
FILTER being an iterator, there is the row context generated by FILTER that scans the product table
again. This is shown graphically in Figure 4-9.
Product[UnitPriceRank] =

## Countrows (


## Filter (

Product,
Product[Unit Price] >= PriceOfCurrentProduct
)

## ) + 1

Row context of the
calculated column
Row context of the
FILTER function
FIGURE 4-9 During the evaluation of the innermost expression, there are two row contexts on the
same table.
The outer box includes the row context of the calculated column, which is iterating over Product.
However, the inner box shows the row context of the FILTER function, which is iterating over Product
too. The expression Product[Unit Price] depends on the context. Therefore, a reference to Product[Unit
Price] in the inner box can only refer to the currently iterated row by FILTER. The problem is that, in that
box, we need to evaluate the value of Unit Price that is referenced by the row context of the calculated
column, which is now hidden.
Indeed, when one does not create a new row context using an iterator, the value of Product[Unit
Price] is the desired value, which is the value in the current row context of the calculated column, as in
this simple piece of code:
Product[Test] = Product[Unit Price]
To further demonstrate this, let us evaluate Product[Unit Price] in the two boxes, with some dummy
code. What comes out are different results as shown in Figure 4-10, where we added the evaluation of
Product[Unit Price] right before COUNTROWS, only for educational purposes.

94 CHAPTER 4 Understanding Evaluation Contexts
Products[UnitPriceRank] =
Product[UnitPrice] +

## Countrows (


## Filter (

Product,
Product[Unit Price] >= PriceOfCurrentProduct
)

## ) + 1

This is the value of the current
product in the calculated column
This is the value of the
product iterated by FILTER
FIGURE 4-10 Outside of the iteration, Product[Unit Price] refers to the row context of the calculated column.
Here is a recap of the scenario so far:

> **Note:** The inner row context, generated by FILTER, hides the outer row context.

> **Note:** We need to compare the inner Product[Unit Price] with the value of the outer Product[Unit
Price].

> **Note:** If we write the comparison in the inner expression, we are unable to access the outer
Product[Unit Price].
Because we can retrieve the current unit price, if we evaluate it outside of the row context of
FILTER, the best approach to this problem is saving the value of the Product[Unit Price] inside a variable.
Indeed, one can evaluate the variable in the row context of the calculated column using this code:
'Product'[UnitPriceRank] =
VAR
PriceOfCurrentProduct = 'Product'[Unit Price]

## Return


## Countrows (


## Filter (

'Product',
'Product'[Unit Price] > PriceOfCurrentProduct
)

## ) + 1

Moreover, it is even better to write the code in a more descriptive way by using more variables to
separate the different steps of the calculation. This way, the code is also easier to follow:
'Product'[UnitPriceRank] =

```dax
VAR PriceOfCurrentProduct = 'Product'[Unit Price]
VAR MoreExpensiveProducts =
FILTER (
'Product',
'Product'[Unit Price] > PriceOfCurrentProduct
)
RETURN
COUNTROWS ( MoreExpensiveProducts ) + 1
```

CHAPTER 4 Understanding Evaluation Contexts 95
Figure 4-11 shows a graphical representation of the row contexts of this latter formulation of the
code, which makes it easier to understand which row context DAX computes each part of the formula in.
Product[UnitPriceRank] =

```dax
VAR PriceOfCurrentProduct = Product[Unit Price]
VAR MoreExpensiveProducts =
FILTER (
Product,
Product[Unit Price] > PriceOfCurrentProduct
)
RETURN
COUNTROWS ( MoreExpensiveProducts ) + 1
```

This is the value of the current
product in the calculated column
This is the value of the
product iterated by FILTER
FIGURE 4-11 The value of PriceOfCurrentProduct is evaluated in the outer row context.
Figure 4-12 shows the result of this calculated column.
FIGURE 4-12 UnitPriceRank is a useful example of how to use variables to navigate within nested row contexts.

96 CHAPTER 4 Understanding Evaluation Contexts
Because there are 14 products with the same unit price, their rank is always 1; the fi fteenth product
has a rank of 15, shared with other products with the same price. It would be great if we could rank 1, 2,
3 instead of 1, 15, 19 as is the case in the fi gure. We will fi x this soon but, before that, it is important to
make a small digression.
To solve a scenario like the one proposed, it is necessary to have a solid understanding of what a
row context is, to be able to detect which row context is active in different parts of the formula and,
most importantly, to conceive how the row context affects the value returned by a DAX expression. It
is worth stressing that the same expression Product[Unit Price], evaluated in two different parts of the
formula, returns different values because of the different contexts under which it is evaluated. When
one does not have a solid understanding of evaluation contexts, it is extremely hard to work on such
complex code.
As you have seen, a simple ranking expression with two row contexts proves to be a challenge. Later
in Chapter 5 you learn how to create multiple fi lter contexts. At that point, the complexity of the code
increases a lot. However, if you understand evaluation contexts, these scenarios are simple. Before
moving to the next level in DAX, you need to understand evaluation contexts well. This is the reason
why we urge you to read this whole section again—and maybe the whole chapter so far—until these
concepts are crystal clear. It will make reading the next chapters much easier and your learning experience much smoother.
Before leaving this example, we need to solve the last detail—that is, ranking using a sequence of 1,
2, 3 instead of the sequence obtained so far. The solution is easier than expected. In fact, in the previous code we focused on counting the products with a higher price. By doing that, the formula counted
14 products ranked 1 and assigned 15 to the second ranking level. However, counting products is not
very useful. If the formula counted the prices higher than the current price, rather than the products,
then all 14 products would be collapsed into a single price.
'Product'[UnitPriceRankDense] =

```dax
VAR PriceOfCurrentProduct = 'Product'[Unit Price]
VAR HigherPrices =
FILTER (
VALUES ( 'Product'[Unit Price] ),
'Product'[Unit Price] > PriceOfCurrentProduct
)
RETURN
COUNTROWS ( HigherPrices ) + 1
```

Figure 4-13 shows the new calculated column, along with UnitPriceRank.

CHAPTER 4 Understanding Evaluation Contexts 97
FIGURE 4-13 UnitPriceRankDense returns a more useful ranking because it counts prices, not products.
This fi nal small step is counting prices instead of counting products, and it might seem harder than
expected. The more you work with DAX, the easier it will become to start thinking in terms of ad hoc
temporary tables created for the purpose of a calculation.
In this example you learned that the best technique to handle multiple row contexts on the same
table is by using variables. Keep in mind that variables were introduced in the DAX language as late as
2015. You might fi nd existing DAX code—written before the age of variables—that uses another technique to access outer row contexts: the EARLIER function, which we describe in the next section.
Using the EARLIER function
DAX provides a function that accesses the outer row contexts: EARLIER. EARLIER retrieves the value of
a column by using the previous row context instead of the last one. Therefore, we can express the value
of PriceOfCurrentProduct using EARLIER ( Product[UnitPrice] ).
Many DAX newbies feel intimidated by EARLIER because they do not understand row contexts well
enough and they do not realize that they can nest row contexts by creating multiple iterations over the

98 CHAPTER 4 Understanding Evaluation Contexts
same table. EARLIER is a simple function, once you understand the concept of row context and nesting.
For example, the following code solves the previous scenario without using variables:
'Product'[UnitPriceRankDense] =

## Countrows (


## Filter (

VALUES ( 'Product'[Unit Price] ),
'Product'[UnitPrice] > EARLIER ( 'Product'[UnitPrice] )
)

## ) + 1

Note EARLIER accepts a second parameter, which is the number of steps to skip, so that
one can skip two or more row contexts. Moreover, there is also a function named EARLIEST
that lets a developer access the outermost row context defi ned for a table. In the real world,
neither EARLIEST nor the second parameter of EARLIER is used often. Though having two
nested row contexts is a common scenario in calculated columns, having three or more of
them is something that rarely happens. Besides, since the advent of variables, EARLIER has
virtually become useless because variable usage superseded EARLIER.
The only reason to learn EARLIER is to be able to read existing DAX code. There are no further reasons to use EARLIER in newer DAX code because variables are a better way to save the required value
when the right row context is accessible. Using variables for this purpose is a best practice and results in
more readable code.
Understanding FILTER, ALL, and context interactions
In the preceding examples, we used FILTER as a convenient way of fi ltering a table. FILTER is a common
function to use whenever one wants to apply a fi lter that further restricts the existing fi lter context.
Imagine that we want to create a measure that counts the number of red products. With the knowledge gained so far, the formula is easy:
NumOfRedProducts :=

```dax
VAR RedProducts =
FILTER (
'Product',
'Product'[Color] = "Red"
)
RETURN
COUNTROWS ( RedProducts )
```

We can use this formula inside a report. For example, put the product brand on the rows to produce
the report shown in Figure 4-14.

CHAPTER 4 Understanding Evaluation Contexts 99
FIGURE 4-14 We can count the number of red products using the FILTER function.
Before moving on with this example, stop for a moment and think carefully about how DAX computed these values. Brand is a column of the Product table. Inside each cell of the report, the fi lter
context fi lters one given brand. Therefore, each cell shows the number of products of the given brand
that are also red. The reason for this is that FILTER iterates the Product table as it is visible in the current
fi lter context, which only contains products with that specifi c brand. It might seem trivial, but it is better
to repeat this a few times than there being a chance of forgetting it.
This is more evident if we add a slicer to the report fi ltering the color. In Figure 4-15 there are two
identical reports with two slicers fi ltering color, where each slicer only fi lters the report on its immediate right. The report on the left fi lters Red and the numbers are the same as in Figure 4-14, whereas the
report on the right is empty because the slicer is fi ltering Azure.
FIGURE 4-15 DAX evaluates NumOfRedProducts taking into account the outer context defi ned by the slicer.
In the report on the right, the Product table iterated by FILTER only contains Azure products,
and, because FILTER can only return Red products, there are no products to return. As a result, the
NumOfRedProducts measure always evaluates to blank.

100 CHAPTER 4 Understanding Evaluation Contexts
The important part of this example is the fact that in the same formula, there are both a fi lter
context coming from the outside—the cell in the report, which is affected by the slicer selection—and
a row context introduced in the formula by the FILTER function. Both contexts work at the same time
and modify the result. DAX uses the fi lter context to evaluate the Product table, and the row context to
evaluate the fi lter condition row by row during the iteration made by FILTER.
We want to repeat this concept again: FILTER does not change the fi lter context. FILTER is an iterator
that scans a table (already fi ltered by the fi lter context) and it returns a subset of that table, according to
the fi ltering condition. In Figure 4-14, the fi lter context is fi ltering the brand and, after FILTER returned the
result, it still only fi ltered the brand. Once we added the slicer on the color in Figure 4-15, the fi lter context contained both the brand and the color. For this reason, in the left-hand side report FILTER returned
all the products iterated, and in the right-hand side report it did not return any product. In both reports,
FILTER did not change the fi lter context. FILTER only scanned a table and returned a fi ltered result.
At this point, one might want to defi ne another formula that returns the number of red products
regardless of the selection done on the slicer. In other words, the code needs to ignore the selection
made on the slicer and must always return the number of all the red products.
To accomplish this, the ALL function comes in handy. ALL returns the content of a table ignoring the
fi lter context. We can defi ne a new measure, named NumOfAllRedProducts, by using this expression:
NumOfAllRedProducts :=

```dax
VAR AllRedProducts =
FILTER (
ALL ( 'Product' ),
'Product'[Color] = "Red"
)
RETURN
COUNTROWS ( AllRedProducts )
This time, FILTER does not iterate Product. Instead, it iterates ALL ( Product ).
ALL ignores the fi lter context and always returns all the rows of the table, so that FILTER returns the
```

red products even if products were previously fi ltered by another brand or color.
The result shown in Figure 4-16—although correct—might be surprising.
FIGURE 4-16 NumOfAllRedProducts returns strange results.

CHAPTER 4 Understanding Evaluation Contexts 101
There are a couple of interesting things to note here, and we want to describe both in more detail:

> **Note:** The result is always 99, regardless of the brand selected on the rows.

> **Note:** The brands in the left matrix are different from the brands in the right matrix.
First, 99 is the total number of red products, not the number of red products of any given brand.
ALL—as expected—ignores the fi lters on the Product table. It not only ignores the fi lter on the color,
but it also ignores the fi lter on the brand. This might be an undesired effect. Nonetheless, ALL is easy
and powerful, but it is an all-or-nothing function. If used, ALL ignores all the fi lters applied to the
table specifi ed as its argument. With the knowledge you have gained so far, you cannot yet choose to
only ignore part of the fi lter. In the example, it would have been better to only ignore the fi lter on the
color. Only after the next chapter, with the introduction of CALCULATE, will you have better options to
achieve the selective ignoring of fi lters.
Let us now describe the second point: The brands on the two reports are different. Because the
slicer is fi ltering one color, the full matrix is computed with the fi lter on the color. On the left the color
is Red, whereas on the right the color is Azure. This determines two different sets of products, and
consequently, of brands. The list of brands used to populate the axis of the report is computed in the
original fi lter context, which contains a fi lter on color. Once the axes have been computed, then DAX
computes values for the measure, always returning 99 as a result regardless of the brand and color.
Thus, the report on the left shows the brands of red products, whereas the report on the right shows
the brands of azure products, although in both reports the measure shows the total of all the red products, regardless of their brand.
Note The behavior of the report is not specifi c to DAX, but rather to the SUMMARIZECOLUMNS function used by Power BI. We cover SUMMARIZECOLUMNS in Chapter 13,
“Authoring queries.”
We do not want to further explore this scenario right now. The solution comes later when you learn

```dax
CALCULATE, which offers a lot more power (and complexity) for the handling of fi lter contexts. As
```

of now, we used this example to show that you might fi nd unexpected results from relatively simple
formulas because of context interactions and the coexistence, in the same expression, of fi lter and row
contexts.
Working with several tables
Now that you have learned the basics of evaluation contexts, we can describe how the context behaves
when it comes to relationships. In fact, few data models contain just one single table. There would most
likely be several tables, linked by relationships. If there is a relationship between Sales and Product,
does a fi lter context on Product fi lter Sales, too? And what about a fi lter on Sales, is it fi ltering Product?
Because there are two types of evaluation contexts (the row context and the fi lter context) and relationships have two sides (a one-side and a many-side), there are four different scenarios to analyze.

102 CHAPTER 4 Understanding Evaluation Contexts
The answer to these questions is already found in the mantra you are learning in this chapter, “The
fi lter context fi lters; the row context iterates” and in its consequence, “The fi lter context does not iterate;
the row context does not fi lter.”
To examine the scenario, we use a data model containing six tables, as shown in Figure 4-17.
FIGURE 4-17 Data model used to learn the interaction between contexts and relationships.
The model presents a couple of noteworthy details:

> **Note:** There is a chain of relationships starting from Sales and reaching Product Category, through
Product and Product Subcategory.

> **Note:** The only bidirectional relationship is between Sales and Product. All remaining relationships are
set to be single cross-fi lter direction.
This model is going to be useful when looking at the details of evaluation contexts and relationships
in the next sections.
Row contexts and relationships
The row context iterates; it does not fi lter. Iteration is the process of scanning a table row by row and of
performing an operation in the meantime. Usually, one wants some kind of aggregation like sum or
average. During an iteration, the row context is iterating an individual table, and it provides a value to

CHAPTER 4 Understanding Evaluation Contexts 103
all the columns of the table, and only that table. Other tables, although related to the iterated table, do
not have a row context on them. In other words, the row context does not interact automatically with
relationships.
Consider as an example a calculated column in the Sales table containing the difference between
the unit price stored in the fact table and the unit price stored in the Product table. The following DAX
code does not work because it uses the Product[UnitPrice] column and there is no row context on
Product:
Sales[UnitPriceVariance] = Sales[Unit Price] – 'Product'[Unit Price]
This being a calculated column, DAX automatically generates a row context on the table containing the column, which is the Sales table. The row context on Sales provides a row-by-row evaluation
of expressions using the columns in Sales. Even though Product is on the one-side of a one-to-many
relationship with Sales, the iteration is happening on the Sales table only.
When we are iterating on the many-side of a relationship, we can access columns on the one-side
of the relationship, but we must use the RELATED function. RELATED accepts a column reference as
the parameter and retrieves the value of the column in the corresponding row in the target table.
RELATED can only reference one column and multiple RELATED functions are required to access more
than one column on the one-side of the relationship. The correct version of the previous code is the
following:
Sales[UnitPriceVariance] = Sales[Unit Price] - RELATED ( 'Product'[Unit Price] )
RELATED requires a row context (that is, an iteration) on the table on the many-side of a relationship. If the row context were active on the one-side of a relationship, then RELATED would no longer
be useful because RELATED would fi nd multiple rows by following the relationship. In this case, that
is, when iterating the one-side of a relationship, the function to use is RELATEDTABLE. RELATEDTABLE
returns all the rows of the table on the many-side that are related with the currently iterated table. For
example, if one wants to compute the number of sales of each product, the following formula defi ned
as a calculated column on Product solves the problem:
Product[NumberOfSales] =

```dax
VAR SalesOfCurrentProduct = RELATEDTABLE ( Sales )
RETURN
COUNTROWS ( SalesOfCurrentProduct )
```

This expression counts the number of rows in the Sales table that corresponds to the current
product. The result is visible in Figure 4-18.

104 CHAPTER 4 Understanding Evaluation Contexts
FIGURE 4-18 RELATEDTABLE is useful in a row context on the one-side of the relationship.
Both RELATED and RELATEDTABLE can traverse a chain of relationships; they are not limited to a
single hop. For example, one can create a column with the same code as before but, this time, in the
Product Category table:
'Product Category'[NumberOfSales] =

```dax
VAR SalesOfCurrentProductCategory = RELATEDTABLE ( Sales )
RETURN
COUNTROWS ( SalesOfCurrentProductCategory )
```

The result is the number of sales for the category, which traverses the chain of relationships from
Product Category to Product Subcategory, then to Product to fi nally reach the Sales table.
In a similar way, one can create a calculated column in the Product table that copies the category
name from the Product Category table.
'Product'[Category] = RELATED ( 'Product Category'[Category] )
In this case, a single RELATED function traverses the chain of relationships from Product to Product
Subcategory to Product Category.
Note The only exception to the general rule of RELATED and RELATEDTABLE is for oneto-one relationships. If two tables share a one-to-one relationship, then both RELATED and
RELATEDTABLE work in both tables and they result either in a column value or in a table with
a single row, depending on the function used.
Regarding chains of relationships, all the relationships need to be of the same type—that is, oneto-many or many-to-one. If the chain links two tables through a one-to-many relationship to a bridge
table, followed by a many-to-one relationship to the second table, then neither RELATED nor RELATEDTABLE works with single-direction fi lter propagation. Only RELATEDTABLE can work using bidirectional

CHAPTER 4 Understanding Evaluation Contexts 105
fi lter propagation, as explained later. On the other hand, a one-to-one relationship behaves as a
one-to-many and as a many-to-one relationship at the same time. Thus, there can be a one-to-one
relationship in a chain of one-to-many (or many-to-one) without interrupting the chain.
For example, in the model we chose as a reference, Customer is related to Sales and Sales is related
to Product. There is a one-to-many relationship between Customer and Sales, and then a many-to-one
relationship between Sales and Product. Thus, a chain of relationships links Customer to Product.
However, the two relationships are not in the same direction. This scenario is known as a many-tomany relationship. A customer is related to many products bought and a product is in turn related to
many customers who bought that product. We cover many-to-many relationships later in Chapter 15,
“Advanced relationships”; let us focus on row context, for the moment. If one uses RELATEDTABLE
through a many-to-many relationship, the result would be wrong. Consider a calculated column in
Product with this formula:
Product[NumOfBuyingCustomers] =

```dax
VAR CustomersOfCurrentProduct = RELATEDTABLE ( Customer )
RETURN
COUNTROWS ( CustomersOfCurrentProduct )
```

The result of the previous code is not the number of customers who bought that product. Instead,
the result is the total number of customers, as shown in Figure 4-19.
FIGURE 4-19 RELATEDTABLE does not work over a many-to-many relationship.
RELATEDTABLE cannot follow the chain of relationships because they are not going in the same
direction. The row context from Product does not reach Customers. It is worth noting that if we try
the formula in the opposite direction, that is, if we count the number of products bought for each
customer, the result is correct: a different number for each row representing the number of products
bought by the customer. The reason for this behavior is not the propagation of a row context but,
rather, the context transition generated by RELATEDTABLE. We added this fi nal note for full disclosure.
It is not time to elaborate on this just yet. You will have a better understanding of this after reading
Chapter 5.

106 CHAPTER 4 Understanding Evaluation Contexts
Filter context and relationships
In the previous section, you learned that the row context iterates and, as such, that it does not use
relationships. The fi lter context, on the other hand, fi lters. A fi lter context is not applied to an individual
table. Instead, it always works on the whole model. At this point, you can update the evaluation context
mantra to its complete formulation:
The fi lter context fi lters the model; the row context iterates one table.
Because a fi lter context fi lters the model, it uses relationships. The fi lter context interacts with
relationships automatically, and it behaves differently depending on how the cross-fi lter direction of
the relationship is set. The cross-fi lter direction is represented with a small arrow in the middle of a
relationship, as shown in Figure 4-20.
FIGURE 4-20 Behavior of fi lter context and relationships.
The fi lter context uses a relationship by going in the direction allowed by the arrow. In all relationships the arrow allows propagation from the one-side to the many-side, whereas when the cross-fi lter
direction is BOTH, propagation is allowed from the many-side to the one-side too.
A relationship with a single cross-fi lter is a unidirectional relationship, whereas a relationship with
BOTH cross-fi lter directions is a bidirectional relationship.
This behavior is intuitive. Although we have not explained this sooner, all the reports we have used
so far relied on this behavior. Indeed, in a typical report fi ltering by Product[Color] and aggregating
the Sales[Quantity], one would expect the fi lter from Product to propagate to Sales. This is exactly
what happens: Product is on the one-side of a relationship; thus a fi lter on Product propagates to Sales,
regardless of the cross-fi lter direction.

CHAPTER 4 Understanding Evaluation Contexts 107
Because our sample data model contains both a bidirectional relationship and many unidirectional
relationships, we can demonstrate the fi ltering behavior by using three different measures that count
the number of rows in the three tables: Sales, Product, and Customer.
[NumOfSales] := COUNTROWS ( Sales )
[NumOfProducts] := COUNTROWS ( Product )
[NumOfCustomers] := COUNTROWS ( Customer )
The report contains the Product[Color] on the rows. Therefore, each cell is evaluated in a fi lter context that fi lters the product color. Figure 4-21 shows the result.
FIGURE 4-21 This shows the behavior of fi lter context and relationships.
In this fi rst example, the fi lter is always propagating from the one-side to the many-side of relationships. The fi lter starts from Product[Color]. From there, it reaches Sales, which is on the many-side
of the relationship with Product, and Product, because it is the very same table. On the other hand,
NumOfCustomers always shows the same value—the total number of customers. This is because the
relationship between Customer and Sales does not allow propagation from Sales to Customer. The fi lter
is moved from Product to Sales, but from there it does not reach Customer.
You might have noticed that the relationship between Sales and Product is a bidirectional relationship. Thus, a fi lter context on Customer also fi lters Sales and Product. We can prove it by changing the
report, slicing by Customer[Education] instead of Product[Color]. The result is visible in Figure 4-22.

108 CHAPTER 4 Understanding Evaluation Contexts
FIGURE 4-22 Filtering by customer education, the Product table is fi ltered too.
This time the fi lter starts from Customer. It can reach the Sales table because Sales is on the manyside of the relationship. Furthermore, it propagates from Sales to Product because the relationship
between Sales and Product is bidirectional—its cross-fi lter direction is BOTH.
Beware that a single bidirectional relationship in a chain does not make the whole chain bidirectional. In fact, a similar measure that counts the number of subcategories, such as the following one,
demonstrates that the fi lter context starting from Customer does not reach Product Subcategory:
NumOfSubcategories := COUNTROWS ( 'Product Subcategory' )
Adding the measure to the previous report produces the results shown in Figure 4-23, where the
number of subcategories is the same for all the rows.
FIGURE 4-23 If the relationship is unidirectional, customers cannot fi lter subcategories.
Because the relationship between Product and Product Subcategory is unidirectional, the fi lter does
not propagate to Product Subcategory. If we update the relationship, setting the cross-fi lter direction to
BOTH, the result is different as shown in Figure 4-24.
FIGURE 4-24 If the relationship is bidirectional, customers can fi lter subcategories too.

CHAPTER 4 Understanding Evaluation Contexts 109
With the row context, we use RELATED and RELATEDTABLE to propagate the row context through
relationships. On the other hand, with the fi lter context, no functions are needed to propagate the
fi lter. The fi lter context fi lters the model, not a table. As such, once one applies a fi lter context, the
entire model is subject to the fi lter according to the relationships.
Important From the examples, it may look like enabling bidirectional fi ltering on all the
relationships is a good option to let the fi lter context propagate to the whole model. This is
defi nitely not the case. We will cover advanced relationships in depth later, in Chapter 15.
Bidirectional fi lters come with a lot more complexity than what we can share with this
introductory chapter, and you should not use them unless you have a clear idea of the
consequences. As a rule, you should enable bidirectional fi lters in specifi c measures by using
the CROSSFILTER function, and only when strictly required.
Using DISTINCT and SUMMARIZE in fi lter contexts
Now that you have a solid understanding of evaluation contexts, we can use this knowledge to solve a
scenario step-by-step. In the meantime, we provide the analysis of a few details that—hopefully—will
shed more light on the fundamental concepts of row context and fi lter context. Besides, in this example
we also further describe the SUMMARIZE function, briefl y introduced in Chapter 3, “Using basic table
functions.”
Before going into more details, please note that this example shows several inaccurate calculations
before reaching the correct solution. The purpose is educational because we want to teach the process
of writing DAX code rather than give a solution. In the process of authoring a measure, it is likely you
will make several initial errors. In this guided example, we describe the correct way of reasoning, which
helps you solve similar errors by yourself.
The requirement is to compute the average age of customers of Contoso. Even though this looks
like a legitimate requirement, it is not complete. Are we speaking about their current age or their age
at the time of the sale? If a customer buys three times, should it count as one event or as three events
in the average? What if they buy three times at different ages? We need to be more precise. Here is the
more complete requirement: “Compute the average age of customers at the time of sale, counting each
customer only once if they made multiple purchases at the same age.”
The solution can be split into two steps:

> **Note:** Computing the age of the customer when the sale happened

> **Note:** Averaging it

110 CHAPTER 4 Understanding Evaluation Contexts
The age of the customer changes for every sale. Thus, the age needs to be stored in the Sales table.
For each row in Sales, one can compute the age of the customer at the time when the sale happened.
A calculated column perfectly fi ts this need:
Sales[Customer Age] =
DATEDIFF ( -- Compute the difference between
RELATED ( Customer[Birth Date] ), -- the customer’s birth date
Sales[Order Date], -- and the date of the sale
YEAR -- in years
)
Because Customer Age is a calculated column, it is evaluated in a row context that iterates Sales.
The formula needs to access Customer[Birth Date], which is a column in Customer, on the one-side of a
relationship with Sales. In this case, RELATED is needed to let DAX access the target table. In the sample
database Contoso, there are many customers for whom the birth date is blank. DATEDIFF returns blank
if the fi rst parameter is blank.
Because the requirement is to provide the average, a fi rst—and inaccurate—solution might be a
measure that averages this column:
Avg Customer Age Wrong := AVERAGE ( Sales[Customer Age] )
The result is incorrect because Sales[Customer Age] contains multiple rows with the same age if a
customer made multiple purchases at a certain age. The requirement is to compute each customer
only once, and this formula is not following such a requirement. Figure 4-25 shows the result of this last
measure side-by-side with the expected result.
FIGURE 4-25 A simple average computes the wrong result for the customer’s age.

CHAPTER 4 Understanding Evaluation Contexts 111
Here is the problem: The age of each customer must be counted only once. A possible solution—
still inaccurate—would be to perform a DISTINCT of the customer ages and then average it, with the
following measure:
Avg Customer Age Wrong Distinct :=
AVERAGEX ( -- Iterate on the distinct values of
DISTINCT ( Sales[Customer Age] ), -- Sales[Customer Age] and compute the
Sales[Customer Age] -- average of the customer’s age
)
This solution is not the correct one yet. In fact, DISTINCT returns the distinct values of the customer
age. Two customers with the same age would be counted only once by this formula. The requirement
is to count each customer once, whereas this formula is counting each age once. In fact, Figure 4-26
shows the report with the new formulation of Avg Customer Age. You see that this solution is still
inaccurate.
FIGURE 4-26 The average of the distinct customer ages still provides a wrong result.
In the last formula, one might try to replace Customer Age with CustomerKey as the parameter of
DISTINCT, as in the following code:
Avg Customer Age Invalid Syntax :=
AVERAGEX ( -- Iterate on the distinct values of
DISTINCT ( Sales[CustomerKey] ), -- Sales[CustomerKey] and compute the
Sales[Customer Age] -- average of the customer’s age
)
This code contains an error and DAX will not accept it. Can you spot the reason, without reading the
solution we provide in the next paragraph?

112 CHAPTER 4 Understanding Evaluation Contexts
AVERAGEX generates a row context that iterates a table. The table provided as the fi rst parameter
to AVERAGEX is DISTINCT ( Sales[CustomerKey] ). DISTINCT returns a table with one column only, and
all the unique values of the customer key. Therefore, the row context generated by AVERAGEX only
contains one column, namely Sales[CustomerKey]. DAX cannot evaluate Sales[Customer Age] in a row
context that only contains Sales[CustomerKey].
What is needed is a row context that has the granularity of Sales[CustomerKey] but that also contains Sales[Customer Age]. SUMMARIZE, introduced in Chapter 3, can generate the existing unique
combinations of two columns. Now we can fi nally show a version of this code that implements all the
requirements:
Correct Average :=
AVERAGEX ( -- Iterate on
SUMMARIZE ( -- all the existing combinations
Sales, -- that exist in Sales
Sales[CustomerKey], -- of the customer key and
Sales[Customer Age] -- the customer age

## ), --

Sales[Customer Age] -- and average the customer’s age
)
As usual, it is possible to use a variable to split the calculation in multiple steps. Note that the access
to the Customer Age column still requires a reference to the Sales table name in the second argument
of the AVERAGEX function. A variable can contain a table, but it cannot be used as a table reference.
Correct Average :=

```dax
VAR CustomersAge =
SUMMARIZE ( -- Existing combinations
Sales, -- that exist in Sales
Sales[CustomerKey], -- of the customer key and
Sales[Customer Age] -- the customer age
)
RETURN
```

AVERAGEX ( -- Iterate on list of
CustomersAge, -- Customers/age in Sales
Sales[Customer Age] -- and average the customer’s age
)
SUMMARIZE generates all the combinations of customer and age available in the current fi lter context. Thus, multiple customers with the same age will duplicate the age, once per customer. AVERAGEX
ignores the presence of CustomerKey in the table; it only uses the customer age. CustomerKey is only
needed to count the correct number of occurrences of each age.
It is worth stressing that the full measure is executed in the fi lter context generated by the report.
Thus, only the customers who bought something are evaluated and returned by SUMMARIZE. Every
cell of the report has a different fi lter context, only considering the customers who purchased at least
one product of the color displayed in the report.

CHAPTER 4 Understanding Evaluation Contexts 113
Conclusions
It is time to recap the most relevant topics you learned in this chapter about evaluation contexts.

> **Note:** There are two evaluation contexts: the fi lter context and the row context. The two evaluation
contexts are not variations of the same concept: the fi lter context fi lters the model; the row context iterates one table.

> **Note:** To understand a formula’s behavior, you always need to consider both evaluation contexts
because they operate at the same time.

> **Note:** DAX creates a row context automatically for a calculated column. One can also create a row
context programmatically by using an iterator. Every iterator defi nes a row context.

> **Note:** You can nest row contexts and, in case they are on the same table, the innermost row context
hides the previous row contexts on the same table. Variables are useful to store values retrieved
when the required row context is accessible. In earlier versions of DAX where variables were not
available, the EARLIER function was used to get access to the previous row context. As of today,
using EARLIER is discouraged.

> **Note:** When iterating over a table that is the result of a table expression, the row context only contains
the columns returned by the table expression.

> **Note:** Client tools like Power BI create a fi lter context when you use fi elds on rows, columns, slicers,
and fi lters. A fi lter context can also be created programmatically by using CALCULATE, which we
introduce in the next chapter.

> **Note:** The row context does not propagate through relationships automatically. One needs to force
the propagation by using RELATED and RELATEDTABLE. You need to use these functions in a
row context on the correct side of a one-to-many relationship: RELATED on the many-side,
RELATEDTABLE on the one-side.

> **Note:** The fi lter context fi lters the model, and it uses relationships according to their cross-fi lter
direction. It always propagates from the one-side to the many-side. In addition, if you use the
cross-fi ltering direction BOTH, then the propagation also happens from the many-side to the
one-side.
At this point, you have learned the most complex conceptual topics of the DAX language. These
points rule all the evaluation fl ows of your formulas, and they are the pillars of the DAX language.
Whenever you encounter an expression that does not compute what you want, there is a huge chance
that was because you have not fully understood these rules.
As we said in the introduction, at fi rst glance all these topics look simple. In fact, they are. What
makes them complex is the fact that in a DAX expression you might have several evaluation contexts
active in different parts of the formula. Mastering evaluation contexts is a skill that you will gain with
experience, and we will try to help you on this by showing many examples in the next chapters. After
writing some DAX formulas of your own, you will intuitively know which contexts are used and which
functions they require, and you will fi nally master the DAX language.