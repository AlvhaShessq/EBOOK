# Chapter 5: Understanding CALCULATE and


## Calculatetable

In this chapter we continue our journey in discovering the power of the DAX language with a detailed
explanation of a single function: CALCULATE. The same considerations apply for CALCULATETABLE,
which evaluates and returns a table instead of a scalar value. For simplicity’s sake, we will refer to
CALCULATE in the examples, but remember that CALCULATETABLE displays the same behavior.
CALCULATE is the most important, useful, and complex function in DAX, so it deserves a full chapter.
The function itself is simple to learn; it only performs a few tasks. Complexity comes from the fact that
CALCULATE and CALCULATETABLE are the only functions in DAX that can create new fi lter contexts.
Thus, although they are simple functions, using CALCULATE or CALCULATETABLE in a formula instantly
increases its complexity.
This chapter is as tough as the previous chapter was. We suggest you carefully read it once, get a
general feeling for CALCULATE, and move on to the remaining part of the book. Then, as soon as you
feel lost in a specifi c formula, come back to this chapter and read it again from the beginning. You will
probably discover new information each time you read it.
Introducing CALCULATE and CALCULATETABLE
The previous chapter described the two evaluation contexts: the row context and the fi lter context.
The row context automatically exists for a calculated column, and one can create a row context programmatically by using an iterator. The fi lter context, on the other hand, is created by the report, and
we have not described yet how to programmatically create a fi lter context. CALCULATE and CALCULATETABLE are the only functions required to operate on the fi lter context. Indeed, CALCULATE and
CALCULATETABLE are the only functions that can create a new fi lter context by manipulating the
existing one. From here onwards, we will show examples based on CALCULATE only, but remember
that CALCULATETABLE performs the same operation for DAX expressions returning a table. Later in the
book there are more examples using CALCULATETABLE, as in Chapter 12, “Working with tables,” and in
Chapter 13, “Authoring queries.”
Creating fi lter contexts
Here we will introduce the reason why one would want to create new fi lter contexts with a practical example. As described in the next sections, writing code without being able to create new fi lter

116 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
contexts results in verbose and unreadable code. What follows is an example of how creating a new
fi lter context can drastically improve code that, at fi rst, looked rather complex.
Contoso is a company that sells electronic products all around the world. Some products are
branded Contoso, whereas others have different brands. One of the reports requires a comparison of
the gross margins, both as an amount and as a percentage, of Contoso-branded products against their
competitors. The fi rst part of the report requires the following calculations:
Sales Amount := SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
Gross Margin := SUMX ( Sales, Sales[Quantity] * ( Sales[Net Price] - Sales[Unit Cost] ) )
GM % := DIVIDE ( [Gross Margin], [Sales Amount] )
One beautiful aspect of DAX is that you can build more complex calculations on top of existing
measures. In fact, you can appreciate this in the defi nition of GM %, the measure that computes the
percentage of the gross margin against the sales. GM % simply invokes the two original measures as it
divides them. If you already have a measure that computes a value, you can call the measure instead of
rewriting the full code.
Using the three measures defi ned above, one can build the fi rst report, as shown in Figure 5-1.
FIGURE 5-1 The three measures provide quick insights in the margin of different categories.
The next step in building the report is more intricate. In fact, the fi nal report we want is the one in
Figure 5-2 that shows two additional columns: the gross margin for Contoso-branded products, both
as amount and as percentage.
FIGURE 5-2 The last two columns of the report show gross margin amount and gross margin percentage for
Contoso-branded products.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 117
With the knowledge acquired so far, you are already capable of authoring the code for these two
measures. Indeed, because the requirement is to restrict the calculation to only one brand, a solution is
to use FILTER to restrict the calculation of the gross margin to Contoso products only:
Contoso GM :=

```dax
VAR ContosoSales = -- Saves the rows of Sales which are related
FILTER ( -- to Contoso-branded products into a variable
Sales,
RELATED ( 'Product'[Brand] ) = "Contoso"
)
VAR ContosoMargin = -- Iterates over ContosoSales
SUMX ( -- to only compute the margin for Contoso
ContosoSales,
Sales[Quantity] * ( Sales[Net Price] - Sales[Unit Cost] )
)
RETURN
ContosoMargin
```

The ContosoSales variable contains the rows of Sales related to all the Contoso-branded products. Once the variable is computed, SUMX iterates on ContosoSales to compute the margin. Because
the iteration is on the Sales table and the fi lter is on the Product table, one needs to use RELATED to
retrieve the related product for each row in Sales. In a similar way, one can compute the gross margin of
Contoso by iterating the ContosoSales variable twice:
Contoso GM % :=

```dax
VAR ContosoSales = -- Saves the rows of Sales which are related
FILTER ( -- to Contoso-branded products into a variable
Sales,
RELATED ( 'Product'[Brand] ) = "Contoso"
)
VAR ContosoMargin = -- Iterates over ContosoSales
SUMX ( -- to only compute the margin for Contoso
ContosoSales,
Sales[Quantity] * ( Sales[Net Price] - Sales[Unit Cost] )
)
VAR ContosoSalesAmount = -- Iterates over ContosoSales
SUMX ( -- to only compute the sales amount for Contoso
ContosoSales,
Sales[Quantity] * Sales[Net Price]
)
VAR Ratio =
DIVIDE ( ContosoMargin, ContosoSalesAmount )
RETURN
Ratio
```

The code for Contoso GM % is a bit longer but, from a logical point of view, it follows the same pattern as Contoso GM. Although these measures work, it is easy to note that the initial elegance of DAX
is lost. Indeed, the model already contains one measure to compute the gross margin and another
measure to compute the gross margin percentage. However, because the new measures needed to be
fi ltered, we had to rewrite the expression to add the condition.

118 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
It is worth stressing that the basic measures Gross Margin and GM % can already compute the
values for Contoso. In fact, from Figure 5-2 you can note that the gross margin for Contoso is equal to
3,877,070.65 and the percentage is equal to 52.73%. One can obtain the very same numbers by slicing
the base measures Gross Margin and GM % by Brand, as shown in Figure 5-3.
FIGURE 5-3 When sliced by brand, the base measures compute the value of Gross Margin and GM % for Contoso.
In the highlighted cells, the fi lter context created by the report is fi ltering the Contoso brand.
The fi lter context fi lters the model. Therefore, a fi lter context placed on the Product[Brand] column
fi lters the Sales table because of the relationship linking Sales to Product. Using the fi lter context, one
can fi lter a table indirectly because the fi lter context operates on the whole model.
Thus, if we could make DAX compute the Gross Margin measure by creating a fi lter context programmatically, which only fi lters the Contoso-branded products, then our implementation of the last
two measures would be much easier. This is possible by using CALCULATE.
The complete description of CALCULATE comes later in this chapter. First, we examine the syntax of

## Calculate:


```dax
CALCULATE ( Expression, Condition1, … ConditionN )
CALCULATE can accept any number of parameters. The only mandatory parameter is the fi rst one,
that is, the expression to evaluate. The conditions following the fi rst parameter are called fi lter arguments. CALCULATE creates a new fi lter context based on the set of fi lter arguments. Once the new fi lter
context is computed, CALCULATE applies it to the model, and it proceeds with the evaluation of the
expression. Thus, by leveraging CALCULATE, the code for Contoso Margin and Contoso GM % becomes
```

much simpler:
Contoso GM :=

## Calculate (

[Gross Margin], -- Computes the gross margin

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 119
'Product'[Brand] = "Contoso" -- In a filter context where brand = Contoso
)
Contoso GM % :=

## Calculate (

[GM %], -- Computes the gross margin percentage
'Product'[Brand] = "Contoso" -- In a filter context where brand = Contoso
)
Welcome back, simplicity and elegance! By creating a fi lter context that forces the brand to be Contoso, one can rely on existing measures and change their behavior without having to rewrite the code
of the measures.
CALCULATE lets you create new fi lter contexts by manipulating the fi lters in the current context. As
you have seen, this leads to simple and elegant code. In the next sections we provide a complete and
more formal defi nition of the behavior of CALCULATE, describing in detail what CALCULATE does and
how to take advantage of its features. Indeed, so far we have kept the example rather high-level when,
in fact, the initial defi nition of the Contoso measures is not semantically equivalent to the fi nal defi nition. There are some differences that one needs to understand well.
Introducing CALCULATE
Now that you have had an initial exposure to CALCULATE, it is time to start learning the details of this
function. As introduced earlier, CALCULATE is the only DAX function that can modify the fi lter context;
and remember, when we mention CALCULATE, we also include CALCULATETABLE. CALCULATE does
not modify a fi lter context: It creates a new fi lter context by merging its fi lter parameters with the existing fi lter context. Once CALCULATE ends, its fi lter context is discarded and the previous fi lter context
becomes effective again.
We have introduced the syntax of CALCULATE as

```dax
CALCULATE ( Expression, Condition1, … ConditionN )
The fi rst parameter is the expression that CALCULATE will evaluate. Before evaluating the expression, CALCULATE computes the fi lter arguments and uses them to manipulate the fi lter context.
The fi rst important thing to note about CALCULATE is that the fi lter arguments are not Boolean
```

conditions: The fi lter arguments are tables. Whenever you use a Boolean condition as a fi lter argument
of CALCULATE, DAX translates it into a table of values.
In the previous section we used this code:
Contoso GM :=

## Calculate (

[Gross Margin], -- Computes the gross margin
'Product'[Brand] = "Contoso" -- In a filter context where brand = Contoso
)

120 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
Using a Boolean condition is only a shortcut for the complete CALCULATE syntax. This is known as
syntax sugar. It reads this way:
Contoso GM :=

## Calculate (

[Gross Margin], -- Computes the gross margin

```dax
FILTER ( -- Using as valid values for Product[Brand]
ALL ( 'Product'[Brand] ), -- any value for Product[Brand]
'Product'[Brand] = "Contoso" -- which is equal to "Contoso"
)
)
```

The two syntaxes are equivalent, and there are no performance or semantic differences between
them. That being said, particularly when you are learning CALCULATE for the fi rst time, it is useful to
always read fi lter arguments as tables. This makes the behavior of CALCULATE more apparent. Once
you get used to CALCULATE semantics, the compact version of the syntax is more convenient. It is
shorter and easier to read.
A fi lter argument is a table, that is, a list of values. The table provided as a fi lter argument defi nes
the list of values that will be visible—for the column—during the evaluation of the expression. In the
previous example, FILTER returns a table with one row only, containing a value for Product[Brand] that
equals “Contoso”. In other words, “Contoso” is the only value that CALCULATE will make visible for the
Product[Brand] column. Therefore, CALCULATE fi lters the model including only products of the
Contoso brand. Consider these two defi nitions:
Sales Amount :=

## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
Contoso Sales :=

## Calculate (

[Sales Amount],

## Filter (

ALL ( 'Product'[Brand] ),
'Product'[Brand] = "Contoso"
)
)

```dax
The fi lter parameter of FILTER in the CALCULATE of Contoso Sales scans ALL(Product[Brand]); therefore, any previously existing fi lter on the product brand is overwritten by the new fi lter. This is more evident when you use the measures in a report that slices by brand. You can see in Figure 5-4 that Contoso
```

Sales reports on all the rows/brands the same value as Sales Amount did for Contoso specifi cally.
In every row, the report creates a fi lter context containing the relevant brand. For example, in
the row for Litware the original fi lter context created by the report contains a fi lter that only shows
Litware products. Then, CALCULATE evaluates its fi lter argument, which returns a table containing
only Contoso. The newly created fi lter overwrites the previously existing fi lter on the same column.
You can see a graphic representation of the process in Figure 5-5.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 121
FIGURE 5-4 Contoso Sales overwrites the existing fi lter with the new fi lter for Contoso.
Contoso Sales :=

## Calculate (

[Sales Amount],

## Filter (

ALL ( 'Product'[Brand] ),
'Product'[Brand] = "Contoso"
)
)
Brand
Litware
Brand
Contoso
Contoso

## Overwrite

Brand
FIGURE 5-5 The fi lter with Litware is overwritten by the fi lter with Contoso evaluated by CALCULATE.

122 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
CALCULATE does not overwrite the whole original fi lter context. It only replaces previously existing
fi lters on the columns contained in the fi lter argument. In fact, if one changes the report to now slice by
Product[Category], the result is different, as shown in Figure 5-6.
FIGURE 5-6 If the report fi lters by Category, the fi lter on Brand will be merged and no overwrite happens.
Now the report is fi ltering Product[Category], whereas CALCULATE applies a fi lter on Product[Brand]
to evaluate the Contoso Sales measure. The two fi lters do not work on the same column of the Product
table. Therefore, no overwriting happens, and the two fi lters work together as a new fi lter context. As
a result, each cell is showing the sales of Contoso for the given category. The scenario is depicted in
Figure 5-7.
Contoso Sales :=

## Calculate (

[Sales Amount],

## Filter (

ALL ( 'Product'[Brand] ),
'Product'[Brand] = "Contoso"
)
)
Cell phones
Category
Contoso
Brand
Brand
Contoso
Category
Cell phones
FIGURE 5-7 CALCULATE overwrites fi lters on the same column. It merges fi lters if they are on different columns.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 123
Now that you have seen the basics of CALCULATE, we can summarize its semantics:

> **Note:** CALCULATE makes a copy of the current fi lter context.

> **Note:** CALCULATE evaluates each fi lter argument and produces, for each condition, the list of valid
values for the specifi ed columns.

> **Note:** If two or more fi lter arguments affect the same column, they are merged together using an
AND operator (or using the set intersection in mathematical terms).

> **Note:** CALCULATE uses the new condition to replace existing fi lters on the columns in the model.
If a column already has a fi lter, then the new fi lter replaces the existing one. On the other
hand, if the column does not have a fi lter, then CALCULATE adds the new fi lter to the fi lter
context.

> **Note:** Once the new fi lter context is ready, CALCULATE applies the fi lter context to the model, and it
computes the fi rst argument: the expression. In the end, CALCULATE restores the original fi lter
context, returning the computed result.
Note CALCULATE does another very important task: It transforms any existing row context into an equivalent fi lter context. You fi nd a more detailed discussion on this topic later
in this chapter, under “Understanding context transition.” Should you do a second reading
of this section, do remember: CALCULATE creates a fi lter context out of the existing row
contexts.
CALCULATE accepts fi lters of two types:

> **Note:** Lists of values, in the form of a table expression. In that case, you provide the exact list of values you want to make visible in the new fi lter context. The fi lter can be a table with any number
of columns. Only the existing combinations of values in different columns will be considered in
the fi lter.

> **Note:** Boolean conditions, such as Product[Color] = “White”. These fi lters need to work on a single
column because the result needs to be a list of values for a single column. This type of fi lter
argument is also known as predicate.
If you use the syntax with a Boolean condition, DAX transforms it into a list of values. Thus, whenever you write this code:
Sales Amount Red Products :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red"
)

124 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
DAX transforms the expression into this:
Sales Amount Red Products :=

## Calculate (

[Sales Amount],

## Filter (

ALL ( 'Product'[Color] ),
'Product'[Color] = "Red"
)
)
For this reason, you can only reference one column in a fi lter argument with a Boolean condition.
DAX needs to detect the column to iterate in the FILTER function, which is generated in the background
automatically. If the Boolean expression references two or more columns, then you must explicitly write
the FILTER iteration, as you learn later in this chapter.
Using CALCULATE to compute percentages
Now that we have introduced CALCULATE, we can use it to defi ne several calculations. The goal of this
section is to bring your attention to some details about CALCULATE that are not obvious at fi rst sight.
Later in this chapter, we will cover more advanced aspects of CALCULATE. For now, we focus on some
of the issues you might encounter when you start using CALCULATE.
A pattern that appears often is that of percentages. When working with percentages, it is very
important to defi ne exactly the calculation required. In this set of examples, you learn how different
uses of CALCULATE and ALL functions provide different results.
We can start with a simple percentage calculation. We want to build the following report showing
the sales amount along with the percentage over the grand total. You can see in Figure 5-8 the result
we want to obtain.
FIGURE 5-8 Sales Pct shows the percentage of the current category against the grand total.
To compute the percentage, one needs to divide the value of Sales Amount in the current fi lter context by the value of Sales Amount in a fi lter context that ignores the existing fi lter on Category. In fact,
the value of 1.26% for Audio is computed as 384,518.16 divided by 30,591,343.98.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 125
In each row of the report, the fi lter context already contains the current category. Thus, for Sales
Amount, the result is automatically fi ltered by the given category. The denominator of the ratio needs
to ignore the current fi lter context, so that it evaluates the grand total. Because the fi lter arguments of
CALCULATE are tables, it is enough to provide a table function that ignores the current fi lter context on
the category and always returns all the categories—regardless of any fi lter. You previously learned that
this function is ALL. Look at the following measure defi nition:
All Category Sales :=

```dax
CALCULATE ( -- Changes the filter context of
[Sales Amount], -- the sales amount
ALL ( 'Product'[Category] ) -- making ALL categories visible
)
ALL removes the fi lter on the Product[Category] column from the fi lter context. Thus, in any cell of
```

the report, it ignores any fi lter existing on the categories. The effect is that the fi lter on the category
applied by the row of the report is removed. Look at the result in Figure 5-9. You can see that each row
of the report for the All Category Sales measure returns the same value all the way through—the grand
total of Sales Amount.
ALL ( 'Product'[Category] )
removes the current filter
on the category
Category
Audio
All Category Sales :=

## Calculate (

[Sales Amount],
ALL ( 'Product'[Category] )
)

## Remove


## Filter

Category
FIGURE 5-9 ALL removes the fi lter on Category, so CALCULATE defi nes a fi lter context without any fi lter on
Category.
The All Category Sales measure is not useful by itself. It is unlikely a user would want to create a
report that shows the same value on all the rows. However, that value is perfect as the denominator of
the percentage we are looking to compute. In fact, the formula computing the percentage can be written this way:

126 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
Sales Pct :=

```dax
VAR CurrentCategorySales = -- CurrentCategorySales contains
[Sales Amount] -- the sales in the current context
VAR AllCategoriesSales = -- AllCategoriesSales contains
CALCULATE ( -- the sales amount in a filter context
[Sales Amount], -- where all the product categories
ALL ( 'Product'[Category] ) -- are visible
)
VAR Ratio =
DIVIDE (
CurrentCategorySales,
AllCategoriesSales
)
RETURN
Ratio
As you have seen in this example, mixing table functions and CALCULATE makes it possible to author useful measures easily. We use this technique a lot in the book because it is the primary calculation tool in DAX.
Note ALL has specifi c semantics when used as a fi lter argument of CALCULATE. In fact,
it does not replace the fi lter context with all the values. Instead, CALCULATE uses ALL to
```

remove the fi lter on the category column from the fi lter context. The side effects of this
behavior are somewhat complex to follow and do not belong in this introductory section.
We will cover them in more detail later in this chapter.
As we said in the introduction of this section, it is important to pay attention to small details when
authoring percentages like the one we are currently writing. In fact, the percentage works fi ne if the
report is slicing by category. The code removes the fi lter from the category, but it does not touch any
other existing fi lter. Therefore, if the report adds other fi lters, the result might not be exactly what one
wants to achieve. For example, look at the report in Figure 5-10 where we added the Product[Color]
column as a second level of detail in the rows of the report.
FIGURE 5-10 Adding the color to the report produces unexpected results at the color level.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 127
Looking at percentages, the value at the category level is correct, whereas the value at the color
level looks wrong. In fact, the color percentages do not add up—neither to the category level nor to
100%. To understand the meaning of these values and how they are evaluated, it is always of great help
to focus on one cell and understand exactly what happened to the fi lter context. Focus on Figure 5-11.
Black
Color
Audio
Category
Black
Color

## Remove


## Filter

Category
Sales Pct :=

```dax
VAR CurrentCategorySales =
[Sales Amount]
CALCULATE (
)
[Sales Amount],
ALL ( 'Product'[Category] )
```

CurrentCategorySales,
AllCategoriesSales

```dax
VAR AllCategoriesSales =
VAR Ratio =
)
RETURN Ratio
```


## Divide (

FIGURE 5-11 ALL on Product[Category] removes the fi lter on category, but it leaves the fi lter on color intact.
The original fi lter context created by the report contained both a fi lter on category and a fi lter on
color. The fi lter on Product[Color] is not overwritten by CALCULATE, which only removes the fi lter from
Product[Category]. As a result, the fi nal fi lter context only contains the color. Therefore, the denominator of the ratio contains the sales of all the products of the given color—Black—and of any category.
The calculation being wrong is not an unexpected behavior of CALCULATE. The problem here is that
the formula has been designed to specifi cally work with a fi lter on a category, leaving any other fi lter
untouched. The same formula makes perfect sense in a different report. Look at what happens if one
switches the order of the columns, building a report that slices by color fi rst and category second, as in
Figure 5-12.

128 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
FIGURE 5-12 The result looks more reasonable once color and category are interchanged.
The report in Figure 5-12 makes a lot more sense. The measure computes the same result, but it is
more intuitive thanks to the layout of the report. The percentage shown is the percentage of the category inside the given color. Color by color, the percentage always adds up to 100%.
In other words, when the user is required to compute a percentage, they should pay special attention in determining the denominator of the percentage. CALCULATE and ALL are the primary tools to
use, but the specifi cation of the formula depends on the business requirements.
Back to the example: The goal is to fi x the calculation so that it computes the percentage against a
fi lter on either the category or the color. There are multiple ways of performing the operation, all leading to slightly different results that are worth examining deeper.
One possible solution is to let CALCULATE remove the fi lter from both the category and the color.
Adding multiple fi lter arguments to CALCULATE accomplishes this goal:
Sales Pct :=

```dax
VAR CurrentCategorySales =
[Sales Amount]
VAR AllCategoriesAndColorSales =
CALCULATE (
[Sales Amount],
ALL ( 'Product'[Category] ), -- The two ALL conditions could also be replaced
ALL ( 'Product'[Color] ) -- by ALL ( 'Product'[Category], 'Product'[Color] )
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 129
VAR Ratio =
DIVIDE (
CurrentCategorySales,
AllCategoriesAndColorSales
)
RETURN
Ratio
```

This latter version of Sales Pct works fi ne with the report containing the color and the category, but
it still suffers from limitations similar to the previous versions. In fact, it produces the right percentage with color and category—as you can see in Figure 5-13—but it will fail as soon as one adds other
columns to the report.
FIGURE 5-13 With ALL on product category and color, the percentages now sum up correctly.
Adding another column to the report would create the same inconsistency noticed so far. If the user
wants to create a percentage that removes all the fi lters on the Product table, they could still use the
ALL function passing a whole table as an argument:
Sales Pct All Products :=

```dax
VAR CurrentCategorySales =
[Sales Amount]
VAR AllProductSales =
CALCULATE (
[Sales Amount],
ALL ( 'Product' )
)
VAR Ratio =
DIVIDE (
CurrentCategorySales,
AllProductSales
)
RETURN
Ratio

130 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
```

ALL on the Product table removes any fi lter on any column of the Product table. In Figure 5-14 you
can see the result of that calculation.
FIGURE 5-14 ALL used on the product table removes the fi lters from all the columns of the Product table.
So far, you have seen that by using CALCULATE and ALL together, you can remove fi lters—from a
column, from multiple columns, or from a whole table. The real power of CALCULATE is that it offers
many options to manipulate a fi lter context, and its capabilities do not end there. In fact, one might
want to analyze the percentages by also slicing columns from different tables. For example, if the
report is sliced by product category and customer continent, the last measure we created is not perfect
yet, as you can see in Figure 5-15.
FIGURE 5-15 Slicing with columns of multiple tables still shows unexpected results.
At this point, the problem might be evident to you. The measure at the denominator removes
any fi lter from the Product table, but it leaves the fi lter on Customer[Continent] intact. Therefore, the
denominator computes the total sales of all products in the given continent.
As in the previous scenario, the fi lter can be removed from multiple tables by putting several fi lters
as arguments of CALCULATE:

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 131
Sales Pct All Products and Customers :=

```dax
VAR CurrentCategorySales =
[Sales Amount]
VAR AllProductAndCustomersSales =
CALCULATE (
[Sales Amount],
ALL ( 'Product' ),
ALL ( Customer )
)
VAR Ratio =
DIVIDE (
CurrentCategorySales,
AllProductAndCustomersSales
)
RETURN
Ratio
By using ALL on two tables, now CALCULATE removes the fi lters from both tables. The result, as
```

expected, is a percentage that adds up correctly, as you can appreciate in Figure 5-16.
FIGURE 5-16 Using ALL on two tables removes the fi lter context on both tables at the same time.
As with two columns, the same challenge comes up with two tables. If a user adds another column
from a third table to the context, the measure will not remove the fi lter from the third table. One possible solution when they want to remove the fi lter from any table that might affect the calculation is to
remove any fi lter from the fact table itself. In our model the fact table is Sales. Here is a measure that
computes an additive percentage no matter what fi lter is interacting with the Sales table:
Pct All Sales :=

```dax
VAR CurrentCategorySales =
[Sales Amount]
VAR AllSales =
CALCULATE (
[Sales Amount],
ALL ( Sales )
)

132 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
VAR Ratio =
DIVIDE (
CurrentCategorySales,
AllSales
)
RETURN
Ratio
```

This measure leverages relationships to remove the fi lter from any table that might fi lter Sales. At
this stage, we cannot explain the details of how it works because it leverages expanded tables, which
we introduce in Chapter 14, “Advanced DAX concepts.” You can appreciate its behavior by inspecting
Figure 5-17, where we removed the amount from the report and added the calendar year on the columns. Please note that the Calendar Year belongs to the Date table, which is not used in the measure.
Nevertheless, the fi lter on Date is removed as part of the removal of fi lters from Sales.
FIGURE 5-17 ALL on the fact table removes any fi lter from related tables as well.
Before leaving this long exercise with percentages, we want to show another fi nal example of fi lter
context manipulation. As you can see in Figure 5-17, the percentage is always against the grand total,
exactly as expected. What if the goal is to compute a percentage over the grand total of only the current year? In that case, the new fi lter context created by CALCULATE needs to be prepared carefully.
Indeed, the denominator needs to compute the total of sales regardless of any fi lter apart from the
current year. This requires two actions:

> **Note:** Removing all fi lters from the fact table

> **Note:** Restoring the fi lter for the year
Beware that the two conditions are applied at the same time, although it might look like the two
steps come one after the other. You have already learned how to remove all the fi lters from the fact
table. The last step is learning how to restore an existing fi lter.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 133
Note The goal of this section is to explain basic techniques for manipulating the
fi lter context. Later in this chapter you see another easier approach to solve this specifi c
requirement—percentage over the visible grand total—by using ALLSELECTED.
In Chapter 3, “Using basic table functions,” you learned the VALUES function. VALUES returns the
list of values of a column in the current fi lter context. Because the result of VALUES is a table, it can be
used as a fi lter argument for CALCULATE. As a result, CALCULATE applies a fi lter on the given column,
restricting its values to those returned by VALUES. Look at the following code:
Pct All Sales CY :=

```dax
VAR CurrentCategorySales =
[Sales Amount]
VAR AllSalesInCurrentYear =
CALCULATE (
[Sales Amount],
ALL ( Sales ),
VALUES ( 'Date'[Calendar Year] )
)
VAR Ratio =
DIVIDE (
CurrentCategorySales,
AllSalesInCurrentYear
)
RETURN
Ratio
```

Once used in the report the measure accounts for 100% for every year, still computing the percentage against any other fi lter apart from the year. You see this in Figure 5-18.
FIGURE 5-18 By using VALUES, you can restore part of the fi lter context, reading it from the original fi lter context.
Figure 5-19 depicts the full behavior of this complex formula.

134 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
Cell phones
Category

## Cy 2007

Calendar Year REMOVE

## Filter

Category CY 2007
Calendar Year
Pct All Sales CY :=

```dax
VAR CurrentCategorySales =
[Sales Amount]
CALCULATE (
)
[Sales Amount],
ALL ( Sales ),
VALUES ( 'Date'[Calendar Year] )
```

CurrentCategorySales,
AllSalesInCurrentYear

```dax
VAR AllSalesInCurrentYear =
VAR Ratio =
)
RETURN Ratio
```


## Divide (


## Cy 2007


## Overwrite

Calendar Year
FIGURE 5-19 The key of this diagram is that VALUES is still evaluated in the original fi lter context.
Here is a review of the diagram:

> **Note:** The cell containing 4.22% (sales of Cell Phones for Calendar Year 2007) has a fi lter context that
fi lters Cell phones for CY 2007.

> **Note:** CALCULATE has two fi lter arguments: ALL ( Sales ) and VALUES ( Date[Calendar Year] ).
• ALL ( Sales ) removes the fi lter from the Sales table.
• VALUES ( Date[Calendar Year] ) evaluates the VALUES function in the original fi lter context,
still affected by the presence of CY 2007 on the columns. As such, it returns the only year visible in the current fi lter context—that is, CY 2007.
The two fi lter arguments of CALCULATE are applied to the current fi lter context, resulting in a fi lter
context that only contains a fi lter on Calendar Year. The denominator computes the total sales in a fi lter
context with CY 2007 only.
It is of paramount importance to understand clearly that the fi lter arguments of CALCULATE are
evaluated in the original fi lter context where CALCULATE is called. In fact, CALCULATE changes the
fi lter context, but this only happens after the fi lter arguments are evaluated.
Using ALL over a table followed by VALUES over a column is a technique used to replace the fi lter
context with a fi lter over that same column.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 135
Note The previous example could also have been obtained by using ALLEXCEPT. The
semantics of ALL/VALUES is different from ALLEXCEPT. In Chapter 10, “Working with the fi lter context,” you will see a complete description of the differences between the ALLEXCEPT
and the ALL/VALUES techniques.
As you have seen in these examples, CALCULATE, in itself, is not a complex function. Its behavior is
simple to describe. At the same time, as soon as you start using CALCULATE, the complexity of the code
becomes much higher. Indeed, you need to focus on the fi lter context and understand exactly how CALCULATE generates the new fi lter context. A simple percentage hides a lot of complexity, and that complexity
is all in the details. Before one really masters the handling of evaluation contexts, DAX is a bit of a mystery.
The key to unlocking the full power of the language is all in mastering evaluation contexts. Moreover, in all
these examples we only had to manage one CALCULATE. In a complex formula, having four or fi ve different
contexts in the same code is not unusual because of the presence of many instances of CALCULATE.
It is a good idea to read this whole section about percentages at least twice. In our experience, a
second read is much easier and lets you focus on the important aspects of the code. We wanted to
show this example to stress the importance of theory, when it comes to CALCULATE. A small change in
the code has an important effect on the numbers computed by the formula. After your second read,
proceed with the next sections where we focus more on theory than on practical examples.
Introducing KEEPFILTERS
You learned in the previous sections that the fi lter arguments of CALCULATE overwrite any previously
existing fi lter on the same column. Thus, the following measure returns the sales of Audio regardless of
any previously existing fi lter on Product[Category]:
Audio Sales :=

## Calculate (

[Sales Amount],
'Product'[Category] = "Audio"
)
As you can see in Figure 5-20, the value of Audio is repeated on all the rows of the report.
FIGURE 5-20 Audio Sales always shows the sales of Audio products, regardless of the current fi lter context.

136 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
CALCULATE overwrites the existing fi lters on the columns where a new fi lter is applied. All the
remaining columns of the fi lter context are left intact. In case you do not want to overwrite existing fi lters, you can wrap the fi lter argument with KEEPFILTERS. For example, if you want to show the amount
of Audio sales when Audio is present in the fi lter context and a blank value if Audio is not present in the
fi lter context, you can write the following measure:
Audio Sales KeepFilters :=

## Calculate (

[Sales Amount],

```dax
KEEPFILTERS ( 'Product'[Category] = "Audio" )
)
KEEPFILTERS is the second CALCULATE modifi er that you learn—the fi rst one was ALL. We further
cover CALCULATE modifi ers later in this chapter. KEEPFILTERS alters the way CALCULATE applies a fi lter
```

to the new fi lter context. Instead of overwriting an existing fi lter over the same column, it adds the new
fi lter to the existing ones. Therefore, only the cells where the fi ltered category was already included in
the fi lter context will produce a visible result. You see this in Figure 5-21.
FIGURE 5-21 Audio Sales KeepFilters shows the sales of Audio products only for the Audio row and for the Grand
Total.
KEEPFILTERS does exactly what its name implies. Instead of overwriting the existing fi lter, it
keeps the existing fi lter and adds the new fi lter to the fi lter context. We can depict the behavior
with Figure 5-22.
Because KEEPFILTERS avoids overwriting, the new fi lter generated by the fi lter argument of CALCULATE is added to the context. If we look at the cell for the Audio Sales KeepFilters measure in the
Cell Phones row, there the resulting fi lter context contains two fi lters: one fi lters Cell Phones; the other
fi lters Audio. The intersection of the two conditions results in an empty set, which produces a blank
result.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 137
Audio Sales KeepFilters :=

## Calculate (

[Sales Amount],

```dax
KEEPFILTERS ( 'Product'[Category] = "Audio" )
)
```

Category
Cell phones
Category
Audio

## Keepfilters

Cell phones
Category
Audio
Category
FIGURE 5-22 The fi lter context generated with KEEPFILTERS fi lters at the same time as both Cell phones and Audio.
The behavior of KEEPFILTERS is clearer when there are multiple elements selected in a column.
For example, consider the following measures; they fi lter Audio and Computers with and without

## Keepfilters:

Always Audio-Computers :=

## Calculate (

[Sales Amount],
'Product'[Category] IN { "Audio", "Computers" }
)
KeepFilters Audio-Computers :=

## Calculate (

[Sales Amount],

```dax
KEEPFILTERS ( 'Product'[Category] IN { "Audio", "Computers" } )
)
The report in Figure 5-23 shows that the version with KEEPFILTERS only computes the sales amount
```

values for Audio and for Computers, leaving all other categories blank. The Total row only takes Audio
and Computers into account.

138 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
FIGURE 5-23 Using KEEPFILTERS, the original and the new fi lter contexts are merged together.
KEEPFILTERS can be used either with a predicate or with a table. Indeed, the previous code could
also be written in a more verbose way:
KeepFilters Audio-Computers :=

## Calculate (

[Sales Amount],

## Keepfilters (


## Filter (

ALL ( 'Product'[Category] ),
'Product'[Category] IN { "Audio", "Computers" }
)
)
)
This is just an example for educational purposes. You should use the simplest predicate syntax available for a fi lter argument. When fi ltering a single column, you can avoid writing the FILTER explicitly.
Later however, you will see that more complex fi lter conditions require an explicit FILTER. In those
cases, the KEEPFILTERS modifi er can be used around the explicit FILTER function, as you see in the next
section.
Filtering a single column
In the previous section, we introduced fi lter arguments referencing a single column in CALCULATE.
It is important to note that you can have multiple references to the same column in one expression.
For example, the following is a valid syntax because it references the same column (Sales[Net Price])
twice.
Sales 10-100 :=

## Calculate (

[Sales Amount],
Sales[Net Price] >= 10 && Sales[Net Price] <= 100
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 139
In fact, this is converted into the following syntax:
Sales 10-100 :=

## Calculate (

[Sales Amount],

## Filter (

ALL ( Sales[Net Price] ),
Sales[Net Price] >= 10 && Sales[Net Price] <= 100
)
)
The resulting fi lter context produced by CALCULATE only adds one fi lter over the Sales[Net Price]
column. One important note about predicates as fi lter arguments in CALCULATE is that although they
look like conditions, they are tables. If you read the fi rst of the last two code snippets, it looks as though
CALCULATE evaluates a condition. Instead, CALCULATE evaluates the list of all the values of Sales[Net
Price] that satisfy the condition. Then, CALCULATE uses this table of values to apply a fi lter to the
model.
When two conditions are in a logical AND, they can be represented as two separate fi lters. Indeed,
the previous expression is equivalent to the following one:
Sales 10-100 :=

## Calculate (

[Sales Amount],
Sales[Net Price] >= 10,
Sales[Net Price] <= 100
)
However, keep in mind that the multiple fi lter arguments of CALCULATE are always merged with a
logical AND. Thus, you must use a single fi lter in case of a logical OR statement, such as in the following
measure:
Sales Blue+Red :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red" || 'Product'[Color] = "Blue"
)
By writing multiple fi lters, you would combine two independent fi lters in a single fi lter context. The
following measure always produces a blank result because there are no products that are both Blue
and Red at the same time:
Sales Blue and Red :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red",
'Product'[Color] = "Blue"
)

140 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
In fact, the previous measure corresponds to the following measure with a single fi lter:
Sales Blue and Red :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red" && 'Product'[Color] = "Blue"
)
The fi lter argument always returns an empty list of colors allowed in the fi lter context. Therefore, the
measure always returns a blank value.
Whenever a fi lter argument refers to a single column, you can use a predicate. We suggest you do
so because the resulting code is much easier to read. You should do so for logical AND conditions too.
Nevertheless, never forget that you are relying on syntax-sugaring only. CALCULATE always works with
tables, although the compact syntax might suggest otherwise.
On the other hand, whenever there are two or more different column references in a fi lter argument,
it is necessary to write the FILTER condition as a table expression. You learn this in the following section.
Filtering with complex conditions
A fi lter argument referencing multiple columns requires an explicit table expression. It is important to
understand the different techniques available to write such fi lters. Remember that creating a fi lter with
the minimum number of columns required by the predicate is usually a best practice.
Consider a measure that sums the sales for only the transactions with an amount greater than or
equal to 1,000. Getting the amount of each transaction requires the multiplication of the Quantity and
Net Price columns. This is because you do not have a column that stores that amount for each row of
the Sales table in the sample Contoso database. You might be tempted to write something like the following expression, which unfortunately will not work:
Sales Large Amount :=

## Calculate (

[Sales Amount],
Sales[Quantity] * Sales[Net Price] >= 1000
)
This code is not valid because the fi lter argument references two different columns in the same
expression. As such, it cannot be converted automatically by DAX into a suitable FILTER condition. The
best way to write the required fi lter is by using a table that only has the existing combinations of the
columns referenced in the predicate:
Sales Large Amount :=

## Calculate (

[Sales Amount],

## Filter (

ALL ( Sales[Quantity], Sales[Net Price] ),
Sales[Quantity] * Sales[Net Price] >= 1000
)
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 141
This results in a fi lter context that has a fi lter with two columns and a number of rows that correspond to the unique combinations of Quantity and Net Price that satisfy the fi lter condition. This is
shown in Figure 5-24.
Quantity
Net Price
1000.00
1 1001.00
1 1199.00
… …
2 500.00
2 500.05
… …
3 333.34
…
FIGURE 5-24 The multi-column fi lter only includes combinations of Quantity and Net Price producing a result
greater than or equal to 1,000.
This fi lter produces the result in Figure 5-25.
FIGURE 5-25 Sales Large Amount only shows sales of transactions with a large amount.
Be mindful that the slicer in Figure 5-25 is not fi ltering any value: The two displayed values are the
minimum and the maximum values of Net Price. The next step is showing how the measure is interacting with the slicer. In a measure like Sales Large Amount, you need to pay attention when you overwrite
existing fi lters over Quantity or Net Price. Indeed, because the fi lter argument uses ALL on the two
columns, it ignores any previously existing fi lter on the same columns including, in this example, the
fi lter of the slicer. The report in Figure 5-26 is the same as Figure 5-25 but, this time, the slicer fi lters for
net prices between 500 and 3,000. The result is surprising.

142 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
FIGURE 5-26 There are no sales for Audio in the current price range; still Sales Large Amount is showing a result.
The presence of value of Sales Large Amount for Audio and Music, Movies and Audio Books is unexpected. Indeed, for these two categories there are no sales in the net price range between 500 and 3,000,
which is the fi lter context generated by the slicer. Still, the Sales Large Amount measure is showing a result.
The reason is that the fi lter context of Net Price created by the slicer is ignored by the Sales Large
Amount measure, which overwrites the existing fi lter over both Quantity and Net Price. If you carefully
compare fi gures 5-25 and 5-26, you will notice that the value of Sales Large Amount is identical, as if the
slicer was not added to the report. Indeed, Sales Large Amount is completely ignoring the slicer.
If you focus on a cell, like the value of Sales Large Amount for Audio, the code executed to compute
its value is the following:
Sales Large Amount :=

## Calculate (


## Calculate (

[Sales Amount],

## Filter (

ALL ( Sales[Quantity], Sales[Net Price] ),
Sales[Quantity] * Sales[Net Price] >= 1000
)
),
'Product'[Category] = "Audio",
Sales[Net Price] >= 500
)
From the code, you can see that the innermost ALL ignores the fi lter on Sales[Net Price] set by the
outer CALCULATE. In that scenario, you can use KEEPFILTERS to avoid the overwrite of existing fi lters:
Sales Large Amount KeepFilter :=

## Calculate (

[Sales Amount],

## Keepfilters (


## Filter (

ALL ( Sales[Quantity], Sales[Net Price] ),
Sales[Quantity] * Sales[Net Price] >= 1000
)
)
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 143
The new Sales Large Amount KeepFilter measure produces the result shown in Figure 5-27.
FIGURE 5-27 Using KEEPFILTERS, the calculation takes into account the outer slicer too.
Another way of specifying a complex fi lter is by using a table fi lter instead of a column fi lter. This is
one of the preferred techniques of DAX newbies, although it is very dangerous to use. In fact, the previous measure can be written using a table fi lter:
Sales Large Amount Table :=

## Calculate (

[Sales Amount],

## Filter (

Sales,
Sales[Quantity] * Sales[Net Price] >= 1000
)
)
As you may remember, all the fi lter arguments of CALCULATE are evaluated in the fi lter context that
exists outside of the CALCULATE itself. Thus, the iteration over Sales only considers the rows fi ltered in
the existing fi lter context, which contains a fi lter on Net Price. Therefore, the semantic of the Sales Large
Amount Table measure corresponds to the Sales Large Amount KeepFilter measure.
Although this technique looks easy, you should be careful in using it because it could have serious
consequences on performance and on result accuracy. We will cover the details of these issues in
Chapter 14. For now, just remember that the best practice is to always use a fi lter with the smallest
possible number of columns.
Moreover, you should avoid table fi lters because they usually are more expensive. The Sales table
might be very large, and scanning it row by row to evaluate a predicate can be a time-consuming
operation. The fi lter in Sales Large Amount KeepFilter, on the other hand, only iterates the number of
unique combinations of Quantity and Net Price. That number is usually much smaller than the number
of rows of the entire Sales table.

144 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
Evaluation order in CALCULATE
Whenever you look at DAX code, the natural order of evaluation is innermost fi rst. For example, look at
the following expression:
Sales Amount Large :=

## Sumx (


```dax
FILTER ( Sales, Sales[Quantity] >= 100 ),
Sales[Quantity] * Sales[Net Price]
)
DAX needs to evaluate the result of FILTER before starting the evaluation of SUMX. In fact, SUMX
iterates a table. Because that table is the result of FILTER, SUMX cannot start executing before FILTER
has fi nished its job. This rule is true for all DAX functions, except for CALCULATE and CALCULATETABLE.
Indeed, CALCULATE evaluates its fi lter arguments fi rst and only at the end does it evaluate the fi rst
parameter, which is the expression to evaluate to provide the CALCULATE result.
Moreover, things are a bit more intricate because CALCULATE changes the fi lter context. All the
fi lter arguments are executed in the fi lter context outside of CALCULATE, and each fi lter is evaluated
independently. The order of fi lters within the same CALCULATE does not matter. Consequently, all the
```

following measures are completely equivalent:
Sales Red Contoso :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red",

```dax
KEEPFILTERS ( 'Product'[Brand] = "Contoso" )
)
```

Sales Red Contoso :=

## Calculate (

[Sales Amount],

```dax
KEEPFILTERS ( 'Product'[Brand] = "Contoso" ),
'Product'[Color] = "Red"
)
```

Sales Red Contoso :=

```dax
VAR ColorRed =
FILTER (
ALL ( 'Product'[Color] ),
'Product'[Color] = "Red"
)
VAR BrandContoso =
FILTER (
ALL ( 'Product'[Brand] ),
'Product'[Brand] = "Contoso"
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 145
VAR SalesRedContoso =
CALCULATE (
[Sales Amount],
ColorRed,
KEEPFILTERS ( BrandContoso )
)
RETURN
SalesRedContoso
```

The version of Sales Red Contoso defi ned using variables is more verbose than the other versions,
but you might want to use it in case the fi lters are complex expressions with explicit fi lters. This way, it is
easier to understand that the fi lter is evaluated “before” CALCULATE.
This rule becomes more important in case of nested CALCULATE statements. In fact, the outermost
fi lters are applied fi rst, and the innermost are applied later. Understanding the behavior of nested
CALCULATE statements is important, because you encounter this situation every time you nest
measures calls. For example, consider the following measures, where Sales Green calls Sales Red:
Sales Red :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red"
)
Green calling Red :=

## Calculate (

[Sales Red],
'Product'[Color] = "Green"
)
To make the nested measure call more evident, we can expand Sales Green this way:
Green calling Red Exp :=

## Calculate (


## Calculate (

[Sales Amount],
'Product'[Color] = "Red"
),
'Product'[Color] = "Green"
)
The order of evaluation is the following:

```dax
1. First, the outer CALCULATE applies the fi lter, Product[Color] = “Green”.
2. Second, the inner CALCULATE applies the fi lter, Product[Color] = “Red”. This fi lter overwrites
```

the previous fi lter.
3. Last, DAX computes [Sales Amount] with a fi lter for Product[Color] = “Red”.
Therefore, the result of both Red and Green calling Red is still Red, as shown in Figure 5-28.

146 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
FIGURE 5-28 The last three measures return the same result, which is always the sales of red products.
Note The description we provided is for educational purposes only. In reality the engine
uses lazy evaluation for the fi lter context. So, in the presence of fi lter argument overwrites
such as the previous code, the outer fi lter might never be evaluated because it would have
been useless. Nevertheless, this behavior is for optimization only. It does not change the
semantics of CALCULATE in any way.
We can review the order of the evaluation and how the fi lter context is evaluated with another
example. Consider the following measure:
Sales YB :=

## Calculate (


## Calculate (

[Sales Amount],
'Product'[Color] IN { "Yellow", "Black" }
),
'Product'[Color] IN { "Black", "Blue" }
)
The evaluation of the fi lter context produced by Sales YB is visible in Figure 5-29.
As seen before, the innermost fi lter over Product[Color] overwrites the outermost fi lters. Therefore,
the result of the measure shows the sum of products that are Yellow or Black. By using KEEPFILTERS
in the innermost CALCULATE, the fi lter context is built by keeping the two fi lters instead of overwriting
the existing fi lter:
Sales YB KeepFilters :=

## Calculate (


## Calculate (

[Sales Amount],

```dax
KEEPFILTERS ( 'Product'[Color] IN { "Yellow", "Black" } )
),
'Product'[Color] IN { "Black", "Blue" }
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 147
CALCULATE (
CALCULATE (
…,
Product[Color] IN { "Yellow", "Black" }
),
Product[Color] { "Black", "Blue" }
)
```

Color
Yellow
Black
Color
Black
Blue
Color
Yellow
Black

## Overwrite

FIGURE 5-29 The innermost fi lter overwrites the outer fi lter.
The evaluation of the fi lter context produced by Sales YB KeepFilters is visible in Figure 5-30.

## Calculate (


## Calculate (

…,

```dax
KEEPFILTERS ( Product[Color] IN { "Yellow", "Black" } )
),
Product[Color] { "Black", "Blue" }
)
```

Color
Yellow
Black
Color
Black
Blue
Color
Yellow
Black
Color
Black
Blue

## Keepfilters

FIGURE 5-30 By using KEEPFILTERS, CALCULATE does not overwrite the previous fi lter context.
Because the two fi lters are kept together, they are intersected. Therefore, in the new fi lter context
the only visible color is Black because it is the only value present in both fi lters.

148 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
However, the order of the fi lter arguments within the same CALCULATE is irrelevant because they
are applied to the fi lter context independently.
Understanding context transition
In Chapter 4, “Understanding evaluation contexts,” we evoked multiple times that the row context and
the fi lter context are different concepts. This still holds true. However, there is one operation performed
by CALCULATE that can transform a row context into a fi lter context. It is the operation of context transition, defi ned as follows:
CALCULATE invalidates any row context. It automatically adds as fi lter arguments all the
columns that are currently being iterated in any row context—fi ltering their actual value in the
row being iterated.
Context transition is hard to understand at the beginning, and even seasoned DAX coders fi nd it
complex to follow all the implications of context transition. We are more than confi dent that the previous defi nition does not suffi ce to fully understand context transition.
Therefore, we are going to describe context transition through several examples of increasing complexity. But before discussing such a delicate concept, let us make sure we thoroughly understand row
context and fi lter context.
Row context and fi lter context recap
We can recap some important facts about row context and fi lter context with the aid of Figure 5-31,
which shows a report with the Brand on the rows and a diagram describing the evaluation process.
Products and Sales in the diagram are not displaying real data. They only contain a few rows to make
the points clearer.
Sales Amount =

## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
Filter Context
Row Context
Product Brand
A
B
Contoso
Litware

## A 1 11.00


## B 2 25.00


## A 2 10.99

1 11.00
1*11.00
21.98
Products
Sales
2*10.99
SUMX Iterations
Contoso
Brand
Product Quantity Net Price
Iteration Operation Result
FIGURE 5-31 The diagram depicts the full fl ow of execution of a simple iteration with SUMX.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 149
The following comments on Figure 5-31 are helpful to monitor your understanding of the whole
process for evaluating the Sales Amount measure for the Contoso row:

> **Note:** The report creates a fi lter context containing a fi lter for Product[Brand] = “Contoso”.

> **Note:** The fi lter works on the entire model, fi ltering both the Product and the Sales tables.

> **Note:** The fi lter context reduces the number of rows iterated by SUMX while scanning Sales. SUMX
only iterates the Sales rows that are related to a Contoso product.

> **Note:** In the fi gure there are two rows in Sales with product A, which is branded Contoso.

> **Note:** Consequently, SUMX iterates two rows. In the fi rst row it computes 1*11.00 with a partial result
of 11.00. In the second row it computes 2*10.99 with a partial result of 21.98.

> **Note:** SUMX returns the sum of the partial results gathered during the iteration.

> **Note:** During the iteration of Sales, SUMX only scans the visible portion of the Sales table, generating
a row context for each visible row.

> **Note:** When SUMX iterates the fi rst row, Sales[Quantity] equals 1, whereas Sales[Net Price] equals 11.
On the second row, the values are different. Columns have a current value that depends on the
iterated row. Potentially, each row iterated has a different value for all the columns.

> **Note:** During the iteration, there is a row context and a fi lter context. The fi lter context is still the same
that fi lters Contoso because no CALCULATE has been executed to modify it.
Speaking about context transition, the last statement is the most important. During the iteration the
fi lter context is still active, and it fi lters Contoso. The row context, on the other hand, is currently iterating the Sales table. Each column of Sales has a given value. The row context is providing the value via
the current row. Remember that the row context iterates; the fi lter context does not.
This is an important detail. We invite you to double-check your understanding in the following
scenario. Imagine you create a measure that simply counts the number of rows in the Sales table, with
the following code:
NumOfSales := COUNTROWS ( Sales )
Once used in the report, the measure counts the number of Sales rows that are visible in the current
fi lter context. The result shown in Figure 5-32 is as expected: a different number for each brand.

150 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
FIGURE 5-32 NumOfSales counts the number of rows visible in the current fi lter context in the Sales table.
Because there are 37,984 rows in Sales for the Contoso brand, this means that an iteration over Sales
for Contoso will iterate exactly 37,984 rows. The Sales Amount measure we used so far would complete
its execution after 37,984 multiplications.
With the understanding you have obtained so far, can you guess the result of the following measure
on the Contoso row?
Sum Num Of Sales := SUMX ( Sales, COUNTROWS ( Sales ) )
Do not rush in deciding your answer. Take your time, study this simple code carefully, and make an
educated guess. In the following paragraph we provide the correct answer.
The fi lter context is fi ltering Contoso. From the previous examples, it is understood that SUMX
iterates 37,984 times. For each of these 37,984 rows, SUMX computes the number of rows visible
in Sales in the current fi lter context. The fi lter context is still the same, so for each row the result of
COUNTROWS is always 37,984. Consequently, SUMX sums the value of 37,984 for 37,984 times.
The result is 37,984 squared. You can confi rm this by looking at Figure 5-33, where the measure is
displayed in the report.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 151
FIGURE 5-33 Sum Num Of Sales computes NumOfSales squared because it counts all the rows for each iteration.
Now that we have refreshed the main ideas about row context and fi lter context, we can further
discuss the impact of context transition.
Introducing context transition
A row context exists whenever an iteration is happening on a table. Inside an iteration are expressions
that depend on the row context itself. The following expression, which you have studied multiple times
by now, comes in handy:
Sales Amount :=

## Sumx (

Sales,
Sales[Quantity] * Sales[Unit Price]
)
The two columns Quantity and Unit Price have a value in the current row context. In the previous
section we showed that if the expression used inside an iteration is not strictly bound to the row context, then it is evaluated in the fi lter context. As such the results are surprising, at least for beginners.
Nevertheless, one is completely free to use any function inside a row context. Among the many functions available, one appears to be more special: CALCULATE.
If executed in a row context, CALCULATE invalidates the row context before evaluating its expression. Inside the expression evaluated by CALCULATE, all the previous row contexts will no longer be
valid. Thus, the following code produces a syntax error:
Sales Amount :=

## Sumx (

Sales,

```dax
CALCULATE ( Sales[Quantity] ) -- No row context inside CALCULATE, ERROR !
)

152 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
The reason is that the value of the Sales[Quantity] column cannot be retrieved inside CALCULATE
because CALCULATE invalidates the row context that exists outside of CALCULATE itself. Nevertheless,
```

this is only part of what context transition performs. The second—and most relevant—operation is that
CALCULATE adds as fi lter arguments all the columns of the current row context with their current value.
For example, look at the following code:
Sales Amount :=

## Sumx (

Sales,

```dax
CALCULATE ( SUM ( Sales[Quantity] ) ) -- SUM does not require a row context
)
There are no fi lter arguments in CALCULATE. The only CALCULATE argument is the expression to
evaluate. Thus, it looks like CALCULATE will not overwrite the existing fi lter context. The point is that
CALCULATE, because of context transition, is silently creating many fi lter arguments. It creates a fi lter
```

for each column in the iterated table. You can use Figure 5-34 to obtain a fi rst look at the behavior of
context transition. We used a reduced set of columns for visual purposes.
Test :=

## Sumx (

Sales,

```dax
CALCULATE ( SUM ( Sales[Quantity] ) )
)
```

A 1 11.00 Row Context

## B 2 25.00


## A 2 10.99

Sales
SUMX Iteration
1 1
1
2 2
3 2 2
Row
Iterated
Product Quantity Net Price
Sales[Quantity]
Value
Row
Result The result of
SUMX is 5
A
Product
Quantity
11.00
Net Price
Filter Context
FIGURE 5-34 When CALCULATE is executed in a row context, it creates a fi lter context with a fi lter for each of the
columns in the currently iterated table.

```dax
During the iteration CALCULATE starts on the fi rst row, and it computes SUM ( Sales[Quantity] ).
Even though there are no fi lter arguments, CALCULATE adds one fi lter argument for each of the
```

columns of the iterated table. Namely, there are three columns in the example: Product, Quantity, and
Net Price. As a result, the fi lter context generated by the context transition contains the current value
(A, 1, 11.00) for each of the columns (Product, Quantity, Net Price). The process, of course, continues for
each one of the three rows during the iteration made by SUMX.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 153
In other words, the execution of the previous SUMX results in these three CALCULATE executions:

## Calculate (

SUM ( Sales[Quantity] ),
Sales[Product] = "A",
Sales[Quantity] = 1,
Sales[Net Price] = 11
) +

## Calculate (

SUM ( Sales[Quantity] ),
Sales[Product] = "B",
Sales[Quantity] = 2,
Sales[Net Price] = 25
) +

## Calculate (

SUM ( Sales[Quantity] ),
Sales[Product] = "A",
Sales[Quantity] = 2,
Sales[Net Price] = 10.99
)
These fi lter arguments are hidden. They are added by the engine automatically, and there is no way
to avoid them. In the beginning, context transition seems very strange. Nevertheless, once one gets
used to context transition, it is an extremely powerful feature. Hard to master, but extremely powerful.
We summarize the considerations presented earlier, before we further discuss a few of them
specifi cally:

> **Note:** Context transition is expensive. If context transition is used during an iteration on a table
with 10 columns and one million rows, then CALCULATE needs to apply 10 fi lters, one million
times. No matter what, it will be a slow operation. This is not to say that relying on context transition should be avoided. However, it does make CALCULATE a feature that needs to be used
carefully.

> **Note:** Context transition does not only fi lter one row. The original row context existing outside
of CALCULATE always only points to one row. The row context iterates on a row-by-row basis.
When the row context is moved to a fi lter context through context transition, the newly created
fi lter context fi lters all the rows with the same set of values. Thus, you should not assume that
the context transition creates a fi lter context with one row only. This is very important, and we
will return to this topic in the next sections.

> **Note:** Context transition uses columns that are not present in the formula. Although the columns used in the fi lter are hidden, they are part of the expression. This makes any formula with
CALCULATE much more complex than it fi rst seems. If a context transition is used, then all the
columns of the table are part of the expression as hidden fi lter arguments. This behavior might
create unexpected dependencies. This topic is also described later in this section.

154 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE

> **Note:** Context transition creates a fi lter context out of a row context. You might remember the
evaluation context mantra, “the row context iterates a table, whereas the fi lter context fi lters
the model.” Once context transition transforms a row context into a fi lter context, it changes the
nature of the fi lter. Instead of iterating a single row, DAX fi lters the whole model; relationships
become part of the equation. In other words, context transition happening on one table might
propagate its fi ltering effects far from the table the row context originated from.

> **Note:** Context transition is invoked whenever there is a row context. For example, if one uses
CALCULATE in a calculated column, context transition occurs. There is an automatic row context
inside a calculated column, and this is enough for context transition to occur.

> **Note:** Context transition transforms all the row contexts. When nested iterations are being
performed on multiple tables, context transition considers all the row contexts. It invalidates all
of them and adds fi lter arguments for all the columns that are currently being iterated by all the
active row contexts.

> **Note:** Context transition invalidates the row contexts. Though we have repeated this concept
multiple times, it is worth bringing to your attention again. None of the outer row contexts are
valid inside the expression evaluated by CALCULATE. All the outer row contexts are transformed
into equivalent fi lter contexts.
As anticipated earlier in this section, most of these considerations require further explanation. In
the remaining part of this section about context transition, we provide a deeper analysis of these main
points. Although all these considerations are shown as warnings, in reality they are important features.
Being ignorant of certain behaviors can ensure surprising results. Nevertheless, once you master the
behavior, you start leveraging it as you see fi t. The only difference between a strange behavior and a
useful feature—at least in DAX—is your level of knowledge.
Context transition in calculated columns
A calculated column is evaluated in a row context. Therefore, using CALCULATE in a calculated column
triggers a context transition. We use this feature to create a calculated column in Product that marks
as “High Performance” all the products that—alone—sold more than 1% of the total sales of all the
products.
To produce this calculated column, we need two values: the sales of the current product and the
total sales of all the products. The former requires fi ltering the Sales table so that it only computes sales
amount for the current product, whereas the latter requires scanning the Sales table with no active
fi lters. Here is the code:
'Product'[Performance] =

```dax
VAR TotalSales = -- Sales of all the products
SUMX (
Sales, -- Sales is not filtered
Sales[Quantity] * Sales[Net Price] -- thus here we compute all sales
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 155
VAR CurrentSales =
CALCULATE ( -- Performs context transition
SUMX (
Sales, -- Sales of the current product only
Sales[Quantity] * Sales[Net Price] -- thus here we compute sales of the
) -- current product only
)
VAR Ratio = 0.01 -- 1% expressed as a real number
VAR Result =
IF (
CurrentSales >= TotalSales * Ratio,
"High Performance product",
"Regular product"
)
RETURN
Result
```

You note that there is only one difference between the two variables: TotalSales is executed as a
regular iteration, whereas CurrentSales computes the same DAX code within a CALCULATE function.
Because this is a calculated column, the row context is transformed into a fi lter context. The fi lter context propagates through the model and it reaches Sales, only fi ltering the sales of the current product.
Thus, even though the two variables look similar, their content is completely different. TotalSales
computes the sales of all the products because the fi lter context in a calculated column is empty and
does not fi lter anything. CurrentSales computes the sales of the current product only thanks to the
context transition performed by CALCULATE.
The remaining part of the code is a simple IF statement that checks whether the condition is met
and marks the product appropriately. One can use the resulting calculated column in a report like the
one visible in Figure 5-35.
FIGURE 5-35 Only four products are marked High Performance.
In the code of the Performance calculated column, we used CALCULATE and context transition as a
feature. Before moving on, we must check that we considered all the implications. The Product table is

156 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
small, containing just a few thousand rows. Thus, performance is not an issue. The fi lter context generated by CALCULATE fi lters all the columns. Do we have a guarantee that CurrentSales only contains the
sales of the current product? In this special case, the answer is yes. The reason is that each row of Product
is unique because Product contains a column with a different value for each row—ProductKey. Consequently, the fi lter context generated by the context transition is guaranteed to only fi lter one product.
In this case, we could rely on context transition because each row of the iterated table is unique.
Beware that this is not always true. We want to demonstrate that with an example that is purposely
wrong. We create a calculated column, in Sales, containing this code:
Sales[Wrong Amt] =

## Calculate (


## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
)
Being a calculated column, it runs in a row context. CALCULATE performs the context transition,
so SUMX iterates all the rows in Sales with an identical set of values corresponding to the current row
in Sales. The problem is that the Sales table does not have any column with unique values. Therefore,
there is a chance that multiple identical rows exist and, if they exist, they will be fi ltered together. In
other words, there is no guarantee that SUMX always iterates only one row in the Wrong Amt column.
If you are lucky, there are many duplicated rows, and the value computed by this calculated column
is totally wrong. This way, the problem would be clearly visible and immediately recognized. In many
real-world scenarios, the number of duplicated rows in tables is tiny, making these inaccurate calculations hard to spot and debug. The sample database we use in this book is no exception. Look at the
report in Figure 5-36 showing the correct value for Sales Amount and the wrong value computed by
summing the Wrong Amt calculated column.
FIGURE 5-36 Most results are correct; only two rows have different values.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 157
You can see that the difference only exists at the total level and for the Fabrikam brand. There are some
duplicates in the Sales table—related to some Fabrikam product—that perform the calculation twice. The
presence of these rows might be legitimate: The same customer bought the same product in the same store
on the same day in the morning and in the afternoon, but the Sales table only stores the date and not the
time of the transaction. Because the number of duplicates is small, most numbers look correct. However,
the calculation is wrong because it depends on the content of the table. Inaccurate numbers might appear
at any time because of duplicated rows. The more duplicates there are, the worse the result turns out.
In this case, relying on context transition is the wrong choice. Because the table is not guaranteed to
only have unique rows, context transition is not safe to use. An expert DAX coder should know this in
advance. Besides, the Sales table might contain millions of rows; thus, this calculated column is not only
wrong, it is also very slow.
Context transition with measures
Understanding context transition is very important because of another important aspect of DAX.
Every measure reference always has an implicit CALCULATE surrounding it.
Because of CALCULATE, a measure reference generates an implicit context transition if executed in
the presence of any row context. This is why in DAX, it is important to use the correct naming convention when writing column references (always including the table name) and measure references (always
without the table name). You want to be aware of any implicit context transition writing and reading a
DAX expression.
This simple initial defi nition deserves a longer explanation with several examples. The fi rst one is
that translating a measure reference always requires wrapping the expression of the measure within a
CALCULATE function. For example, consider the following defi nition of the Sales Amount measure and
of the Product Sales calculated column in the Product table:
Sales Amount :=

## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
'Product'[Product Sales] = [Sales Amount]
The Product Sales column correctly computes the sum of Sales Amount only for the current product
in the Product table. Indeed, expanding the Sales Amount measure in the defi nition of Product Sales
requires the CALCULATE function that wraps the defi nition of Sales Amount:
'Product'[Product Sales] =

## Calculate


## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
)

158 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
Without CALCULATE, the result of the calculated column would produce the same value for all the
products. This would correspond to the sales amount of all the rows in Sales without any fi ltering by
product. The presence of CALCULATE means that context transition occurs, producing in this case the
desired result. A measure reference always calls CALCULATE. This is very important and can be used to
write short and powerful DAX expressions. However, it could also lead to big mistakes if you forget that
the context transition takes place every time the measure is called in a row context.
As a rule of thumb, you can always replace a measure reference with the expression that defi nes the
measure wrapped inside CALCULATE. Consider the following defi nition of a measure called Max Daily
Sales, which computes the maximum value of Sales Amount computed day by day:
Max Daily Sales :=

## Maxx (

'Date',
[Sales Amount]
)
This formula is intuitive to read. However, Sales Amount must be computed for each date, only
fi ltering the sales of that day. This is exactly what context transition performs. Internally, DAX replaced
the Sales Amount measure reference with its defi nition wrapped by CALCULATE, as in the following
example:
Max Daily Sales :=

## Maxx (

'Date',

## Calculate (


## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
)
)
We will use this feature extensively in Chapter 7, “Working with iterators and CALCULATE,” when we
start writing complex DAX code to solve specifi c scenarios. This initial description just completes the
explanation of context transition, which happens in these cases:

> **Note:** When a CALCULATE or CALCULATETABLE function is called in the presence of any row context.

> **Note:** When there is a measure reference in the presence of any row context because the measure
reference internally executes its DAX code within a CALCULATE function.
This powerful behavior might lead to mistakes, mainly due to the incorrect assumption that you
can replace a measure reference with the DAX code of its defi nition. You cannot. This could work
when there are no row contexts, like in a measure, but this is not possible when the measure reference
appears within a row context. It is easy to forget this rule, so we provide an example of what could happen by making an incorrect assumption.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 159
You may have noticed that in the previous example, we wrote the code for a calculated column
repeating the iteration over Sales twice. Here is the code we already presented in the previous example:
'Product'[Performance] =

```dax
VAR TotalSales = -- Sales of all the products
SUMX (
Sales, -- Sales is not filtered
Sales[Quantity] * Sales[Net Price] -- thus here we compute all sales
)
VAR CurrentSales =
CALCULATE ( -- Performs the context transition
SUMX (
Sales, -- Sales of the current product only
Sales[Quantity] * Sales[Net Price] -- thus here we compute sales of the
) -- current product only
)
VAR Ratio = 0.01 -- 1% expressed as a real number
VAR Result =
IF (
CurrentSales >= TotalSales * Ratio,
"High Performance product",
"Regular product"
)
RETURN
Result
The iteration executed by SUMX is the same code for the two variables: One is surrounded by CALCULATE, whereas the other is not. It might seem like a good idea to rewrite the code and use a measure
```

to host the code of the iteration. This could be even more relevant in case the expression is not a simple
SUMX but, rather, some more complex code. Unfortunately, this approach will not work because the
measure reference will always include a CALCULATE around the expression that the measure replaced.
Imagine creating a measure, Sales Amount, and then a calculated column that calls the measure surrounding it—once with CALCULATE and once without CALCULATE.
Sales Amount :=

## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]
)
'Product'[Performance] =

```dax
VAR TotalSales = [Sales Amount]
VAR CurrentSales = CALCULATE ( [Sales Amount] )
VAR Ratio = 0.01
VAR Result =
IF (
CurrentSales >= TotalSales * Ratio,
"High Performance product",
"Regular product"
)
RETURN
Result

160 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
```

Though it looked like a good idea, this calculated column does not compute the expected result.
The reason is that both measure references will have their own implicit CALCULATE around them. Thus,
TotalSales does not compute the sales of all the products. Instead, it only computes the sales of the
current product because the hidden CALCULATE performs a context transition. CurrentSales computes
the same value. In CurrentSales, the extra CALCULATE is redundant. Indeed, CALCULATE is already
there, only because it is referencing a measure. This is more evident by looking at the code resulting by
expanding the Sales Amount measure:
'Product'[Performance] =

```dax
VAR TotalSales =
CALCULATE (
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
)
VAR CurrentSales =
CALCULATE (
CALCULATE (
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
)
)
VAR Ratio = 0.01
VAR Result =
IF (
CurrentSales >= TotalSales * Ratio,
"High Performance product",
"Regular product"
)
RETURN
Result
Whenever you read a measure call in DAX, you should always read it as if CALCULATE were there.
```

Because it is there. We introduced a rule in Chapter 2, “Introducing DAX,” where we said that it is a best
practice to always use the table name in front of columns, and never use the table name in front of
measures. The reason is what we are discussing now.
When reading DAX code, it is of paramount importance that the user be immediately able to understand whether the code is referencing a measure or a column. The de facto standard that nearly every
DAX coder adopts is to omit the table name in front of measures.
The automatic CALCULATE makes it easy to author formulas that perform complex calculations with
iterations. We will use this feature extensively in Chapter 7 when we start writing complex DAX code to
solve specifi c scenarios.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 161
Understanding circular dependencies
When you design a data model, you should pay attention to the complex topic of circular dependencies in formulas. In this section, you learn what circular dependencies are and how to avoid them in
your model. Before introducing circular dependencies, it is worth discussing simple, linear dependencies with the aid of an example. Look at the following calculated column:
Sales[Margin] = Sales[Net Price] - Sales[Unit Cost]
The new calculated column depends on two columns: Net Price and Unit Cost. This means that to
compute the value of Margin, DAX needs to know in advance the values of the two other columns.
Dependencies are an important part of the DAX model because they drive the order in which
calculated columns and calculated tables are processed. In the example, Margin can only be
computed after Net Price and Unit Cost already have a value. The coder does not need to worry
about dependencies. Indeed, DAX handles them gracefully, building a complex graph that drives the
order of evaluation of all its internal objects. However, it is possible to write code in such a way that
circular dependencies appear in the graph. Circular dependencies happen when DAX cannot
determine the order of evaluation of an expression because there is a loop in the chain of
dependencies.
For example, consider two calculated columns with the following formulas:
Sales[MarginPct] = DIVIDE ( Sales[Margin], Sales[Unit Cost] )
Sales[Margin] = Sales[MarginPct] * Sales[Unit Cost]
In this code, MarginPct depends on Margin and, at the same time, Margin depends on MarginPct.
There is a loop in the chain of dependencies. In that scenario, DAX refuses to accept the last formula
and raises the error, “A circular dependency was detected.”
Circular dependencies do not happen frequently because as humans we understand the problem
well. B cannot depend on A if, at the same time, A depends on B. Nevertheless, there is a scenario
where circular dependency occurs—not because it is one’s intention to do so, but only because
one does not consider certain implications by reading DAX code. This scenario includes the use of

## Calculate.

Imagine a calculated column in Sales with the following code:

```dax
Sales[AllSalesQty] = CALCULATE ( SUM ( Sales[Quantity] ) )
```

The interesting question is, which columns does AllSalesQty depend on? Intuitively, one would
answer that the new column depends solely on Sales[Quantity] because it is the only column used
in the expression. However, it is all too easy to forget the real semantics of CALCULATE and context
transition. Because CALCULATE runs in a row context, all current values of all the columns of the

162 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
table are included in the expression, though hidden. Thus, the real expression evaluated by DAX is
the following:
Sales[AllSalesQty] =

## Calculate (

SUM ( Sales[Quantity] ),
Sales[ProductKey] = <CurrentValueOfProductKey>,
Sales[StoreKey] = <CurrentValueOfStoreKey>,
...,
Sales[Margin] = <CurrentValueOfMargin>
)
As you see, the list of columns AllSalesQty depends on is actually the full set of columns of the table.
Once CALCULATE is being used in a row context, the calculation suddenly depends on all the columns
of the iterated table. This is much more evident in calculated columns, where the row context is present
by default.
If one authors a single calculated column using CALCULATE, everything still works fi ne. The problem
appears if one tries to author two separate calculated columns in a table, with both columns using
CALCULATE, thus fi ring context transition in both cases. In fact, the following new calculated column will fail:

```dax
Sales[NewAllSalesQty] = CALCULATE ( SUM ( Sales[Quantity] ) )
The reason for this is that CALCULATE adds all the columns of the table as fi lter arguments.
```

Adding a new column to a table changes the defi nition of existing columns too. If one were able to
create NewAllSalesQty, the code of the two calculated columns would look like this:
Sales[AllSalesQty] =

## Calculate (

SUM ( Sales[Quantity] ),
Sales[ProductKey] = <CurrentValueOfProductKey>,
...,
Sales[Margin] = <CurrentValueOfMargin>,
Sales[NewAllSalesQty] = <CurrentValueOfNewAllSalesQty>
)
Sales[NewAllSalesQty] =

## Calculate (

SUM ( Sales[Quantity] ),
Sales[ProductKey] = <CurrentValueOfProductKey>,
...,
Sales[Margin] = <CurrentValueOfMargin>,
Sales[AllSalesQty] = <CurrentValueOfAllSalesQty>
)
You can see that the two highlighted rows reference each other. AllSalesQty depends on the value of
NewAllSalesQty and, at the same time, NewAllSalesQty depends on the value of AllSalesQty. Although
very well hidden, a circular dependency does exist. DAX detects the circular dependency, preventing
the code from being accepted.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 163
The problem, although somewhat complex to detect, has a simple solution. If the table on which
CALCULATE performs the context transition contains one column with unique values and DAX is aware
of that, then the context transition only fi lters that column from a dependency point of view.
For example, consider a calculated column in the Product table with the following code:

```dax
'Product'[ProductSales] = CALCULATE ( SUM ( Sales[Quantity] ) )
```

In this case, there is no need to add all the columns as fi lter arguments. In fact, Product contains
one column that has a unique value for each row of the Product table—that is ProductKey. This is
well-known by the DAX engine because that column is on the one-side of a one-to-many relationship.
Consequently, when the context transition occurs, the engine knows that it would be pointless to add a
fi lter to each column. The code would be translated into the following:
'Product'[ProductSales] =

## Calculate (

SUM ( Sales[Quantity] ),
'Product'[ProductKey] = <CurrentValueOfProductKey>
)
As you can see, the ProductSales calculated column in the Product table depends solely on ProductKey. Therefore, one could create many calculated columns using CALCULATE because all of them would
only depend on the column with unique values.
Note The last CALCULATE equivalent statement for the context transition is not totally
accurate. We used it for educational purposes only. CALCULATE adds all the columns of the
table as fi lter arguments, even if a row identifi er is present. Nevertheless, the internal dependency is only created on the unique column. The presence of the unique column lets DAX
evaluate multiple columns with CALCULATE. Still, the semantics of CALCULATE is the same
with or without the unique column: All the columns of the iterated table are added as fi lter
arguments.
We already discussed the fact that relying on context transition on a table that contains duplicates is
a serious problem. The presence of circular dependencies is another very good reason why one should
avoid using CALCULATE and context transition whenever the uniqueness of rows is not guaranteed.
Resorting to a column with unique values for each row is not enough to ensure that CALCULATE
only depends on it for the context transition. The data model must be aware of that. How does DAX
know that a column contains unique values? There are multiple ways to provide this information to the
engine:

> **Note:** When a table is the target (one-side) of a relationship, then the column used to build the relationship is marked as unique. This technique works in any tool.

> **Note:** When a column is selected in the Mark As Date Table setting, then the column is implicitly
unique—more on this in Chapter 8, “Time intelligence calculations.”

164 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE

> **Note:** You can manually set the property of a row identifi er for the unique column by using the Table
Behavior properties. This technique only works in Power Pivot for Excel and Analysis Services
Tabular; it is not available in Power BI at the time of writing.
Any one of these operations informs the DAX engine that the table has a row identifi er, stopping the
process of a table that does not respect that constraint. When a table has a row identifi er, you can use
CALCULATE without worrying about circular dependencies. The reason is that the context transition
depends on the key column only.
Note Though described as a feature, this behavior is actually a side effect of an optimization. The semantics of DAX require the dependency from all the columns. A specifi c optimization introduced very early in the engine only creates the dependency on the primary
key of the table. Because many users rely on this behavior today, it has become part of the
language. Still, it remains an optimization. In borderline scenarios—for example when using
USERELATIONSHIP as part of the formula—the optimization does not kick in, thus recreating the circular dependency error.
CALCULATE modifi ers
As you have learned in this chapter, CALCULATE is extremely powerful and produces complex DAX
code. So far, we have only covered fi lter arguments and context transition. There is still one concept
required to provide the set of rules to fully understand CALCULATE. It is the concept of CALCULATE
modifi er.
We introduced two modifi ers earlier, when we talked about ALL and KEEPFILTERS. While ALL can
be both a modifi er and a table function, KEEPFILTERS is always a fi lter argument modifi er—meaning
that it changes the way one fi lter is merged with the original fi lter context. CALCULATE accepts several
different modifi ers that change how the new fi lter context is prepared. However, the most important
of all these modifi ers is a function that you already know very well: ALL. When ALL is directly used in a
CALCULATE fi lter argument, it acts as a CALCULATE modifi er instead of being a table function. Other
important modifi ers include USERELATIONSHIP, CROSSFILTER, and ALLSELECTED, which have separate
descriptions. The ALLEXCEPT, ALLSELECTED, ALLCROSSFILTERED and ALLNOBLANKROW modifi ers
have the same precedence rules of ALL.
In this section we introduce these modifi ers; then we will discuss the order of precedence of the
different CALCULATE modifi ers and fi lter arguments. At the end, we will present the fi nal schema of
CALCULATE rules.
Understanding USERELATIONSHIP
The fi rst CALCULATE modifi er you learn is USERELATIONSHIP. CALCULATE can activate a relationship
during the evaluation of its expression by using this modifi er. A data model might contain both active

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 165
and inactive relationships. One might have inactive relationships in the model because there are several relationships between two tables, and only one of them can be active.
As an example, one might have order date and delivery date stored in the Sales table for each order.
Typically, the requirement is to perform sales analysis based on the order date, but one might need to
consider the delivery date for some specifi c measures. In that scenario, an option is to create two relationships between Sales and Date: one based on Order Date and another one based on Delivery Date.
The model looks like the one in Figure 5-37.
FIGURE 5-37 Sales and Date are linked through two relationships, although only one can be active.
Only one of the two relationships can be active at a time. For example, in this demo model the relationship with Order Date is active, whereas the one linked to Delivery Date is kept inactive. To author
a measure that shows the delivered value in a given time period, the relationship with Delivery Date
needs to be activated for the duration of the calculation. In this scenario, USERELATIONSHIP is of great
help as in the following code:
Delivered Amount:=

## Calculate (

[Sales Amount],
USERELATIONSHIP ( Sales[Delivery Date], 'Date'[Date] )
)
The relationship between Delivery Date and Date is activated during the evaluation of Sales Amount.
In the meantime, the relationship with Order Date is deactivated. Keep in mind that at a given point in

166 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
time, only one relationship can be active between any two tables. Thus, USERELATIONSHIP temporarily
activates one relationship, deactivating the one active outside of CALCULATE.
Figure 5-38 shows the difference between Sales Amount based on the Order Date, and the new
Delivered Amount measure.
FIGURE 5-38 The fi gure illustrates the difference between ordered and delivered sales.
When using USERELATIONSHIP to activate a relationship, you need to be aware of an important
aspect: Relationships are defi ned when a table reference is used, not when RELATED or other relational
functions are invoked. We will cover the details of this in Chapter 14 by using expanded tables. For
now, an example should suffi ce. To compute all amounts delivered in 2007, the following formula will
not work:
Delivered Amount 2007 v1 :=

## Calculate (

[Sales Amount],

## Filter (

Sales,

## Calculate (

RELATED ( 'Date'[Calendar Year] ),
USERELATIONSHIP ( Sales[Delivery Date], 'Date'[Date] )

## ) = "Cy 2007"

)
)

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 167
In fact, CALCULATE would inactivate the row context generated by the FILTER iteration. Thus, inside
the CALCULATE expression, one cannot use the RELATED function at all. One option to author the code
would be the following:
Delivered Amount 2007 v2 :=

## Calculate (

[Sales Amount],

## Calculatetable (


## Filter (

Sales,
RELATED ( 'Date'[Calendar Year] ) = "CY 2007"
),

## Userelationship (

Sales[Delivery Date],
'Date'[Date]
)
)
)
In this latter formulation, Sales is referenced after CALCULATE has activated the required relationship. Therefore, the use of RELATED inside FILTER happens with the relationship with Delivery Date
active. The Delivered Amount 2007 v2 measure works, but a much better formulation of the same measure relies on default fi lter context propagation rather than relying on RELATED:
Delivered Amount 2007 v3 :=

## Calculate (

[Sales Amount],
'Date'[Calendar Year] = "CY 2007",

## Userelationship (

Sales[Delivery Date],
'Date'[Date]
)
)
When you use USERELATIONSHIP in a CALCULATE statement, all the fi lter arguments are evaluated
using the relationship modifi ers that appear in the same CALCULATE statement—regardless of their
order. For example, in the Delivered Amount 2007 v3 measure, the USERELATIONSHIP modifi er affects
the predicate fi ltering Calendar Year, although it is the previous parameter within the same CALCULATE
function call.
This behavior makes the use of nondefault relationships a complex operation in calculated column expressions. The invocation of the table is implicit in a calculated column defi nition. Therefore,
you do not have control over it, and you cannot change that behavior by using CALCULATE and

## Userelationship.

One important note is the fact that USERELATIONSHIP does not introduce any fi lter by itself. Indeed,
USERELATIONSHIP is not a fi lter argument. It is a CALCULATE modifi er. It only changes the way other
fi lters are applied to the model. If you carefully look at the defi nition of Delivered Amount in 2007 v3,
you might notice that the fi lter argument applies a fi lter on the year 2007, but it does not indicate

168 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
which relationship to use. Is it using Order Date or Delivery Date? The relationship to use is defi ned by

## Userelationship.

Thus, CALCULATE fi rst modifi es the structure of the model by activating the relationship, and only
later does it apply the fi lter argument. If that were not the case—that is, if the fi lter argument were
always evaluated on the current relationship architecture—then the calculation would not work.
There are precedence rules in the application of fi lter arguments and of CALCULATE modifi ers. The
fi rst rule is that CALCULATE modifi ers are always applied before any fi lter argument, so that the effect
of fi lter arguments is applied on the modifi ed version of the model. We discuss precedence of CALCULATE arguments in more detail later.
Understanding CROSSFILTER
The next CALCULATE modifi er you learn is CROSSFILTER. CROSSFILTER is somewhat similar to
USERELATIONSHIP because it manipulates the architecture of the relationships in the model.
Nevertheless, CROSSFILTER can perform two different operations:

> **Note:** It can change the cross-fi lter direction of a relationship.

> **Note:** It can disable a relationship.
USERELATIONSHIP lets you activate a relationship while disabling the active relationship, but it cannot disable a relationship without activating another one between the same tables. CROSSFILTER works
in a different way. CROSSFILTER accepts two parameters, which are the columns involved in the relationship, and a third parameter that can be either NONE, ONEWAY, or BOTH. For example, the following measure computes the distinct count of product colors after activating the relationship between
Sales and Product as a bidirectional one:
NumOfColors :=

## Calculate (

DISTINCTCOUNT ( 'Product'[Color] ),

```dax
CROSSFILTER ( Sales[ProductKey], 'Product'[ProductKey], BOTH )
)
As is the case with USERELATIONSHIP, CROSSFILTER does not introduce fi lters by itself. It only
```

changes the structure of the relationships, leaving to other fi lter arguments the task of applying fi lters.
In the previous example, the effect of the relationship only affects the DISTINCTCOUNT function
because CALCULATE has no further fi lter arguments.
Understanding KEEPFILTERS
We introduced KEEPFILTERS earlier in this chapter as a CALCULATE modifi er. Technically, KEEPFILTERS is
not a CALCULATE modifi er, it is a fi lter argument modifi er. Indeed, it does not change the entire evaluation of CALCULATE. Instead, it changes the way one individual fi lter argument is applied to the fi nal
fi lter context generated by CALCULATE.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 169
We already discussed in depth the behavior of CALCULATE in the presence of calculations like the
following one:
Contoso Sales :=

## Calculate (

[Sales Amount],

```dax
KEEPFILTERS ( 'Product'[Brand] = "Contoso" )
)
The presence of KEEPFILTERS means that the fi lter on Brand does not overwrite a previously existing
```

fi lter on the same column. Instead, the new fi lter is added to the fi lter context, leaving the previous one
intact. KEEPFILTERS is applied to the individual fi lter argument where it is used, and it does not change
the semantic of the whole CALCULATE function.
There is another way to use KEEPFILTERS that is less obvious. One can use KEEPFILTERS as a modifi er
for the table used for an iteration, like in the following code:
ColorBrandSales :=

## Sumx (


```dax
KEEPFILTERS ( ALL ( 'Product'[Color], 'Product'[Brand] ) ),
[Sales Amount]
)
The presence of KEEPFILTERS as the top-level function used in an iteration forces DAX to use
KEEPFILTERS on the implicit fi lter arguments added by CALCULATE during a context transition. In fact,
during the iteration over the values of Product[Color] and Product[Brand], SUMX invokes CALCULATE
```

as part of the evaluation of the Sales Amount measure. At that point, the context transition occurs, and
the row context becomes a fi lter context by adding a fi lter argument for Color and Brand.
Because the iteration started with KEEPFILTERS, context transition will not overwrite existing fi lters.
It will intersect the existing fi lters instead. It is uncommon to use KEEPFILTERS as the top-level function
in an iteration. We will cover some examples of this advanced use later in Chapter 10.
Understanding ALL in CALCULATE
ALL is a table function, as you learned in Chapter 3. Nevertheless, ALL acts as a CALCULATE modifi er
when used as a fi lter argument in CALCULATE. The function name is the same, but the semantics of ALL
as a CALCULATE modifi er is slightly different than what one would expect.
Looking at the following code, one might think that ALL returns all the years, and that it changes the
fi lter context making all years visible:
All Years Sales :=

## Calculate (

[Sales Amount],
ALL ( 'Date'[Year] )
)
However, this is not true. When used as a top-level function in a fi lter argument of CALCULATE,
ALL removes an existing fi lter instead of creating a new one. A proper name for ALL would have been

170 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
REMOVEFILTER. For historical reasons, the name remained ALL and it is a good idea to know exactly
how the function behaves.
If one considers ALL as a table function, they would interpret the CALCULATE behavior like in Figure 5-39.

## Calculate (


## Calculate (

…
ALL ( 'Date'[Year] )
)
),
'Date'[Year] = 2007
Year
2006
Year
Year
2006

## Overwrite

FIGURE 5-39 It looks like ALL returns all the years and uses the list to overwrite the previous fi lter context.
The innermost ALL over Date[Year] is a top-level ALL function call in CALCULATE. As such, it does
not behave as a table function. It should really be read as REMOVEFILTER. In fact, instead of returning
all the years, in that case ALL acts as a CALCULATE modifi er that removes any fi lter from its argument.
What really happens inside CALCULATE is the diagram of Figure 5-40.

## Calculate (


## Calculate (

…
ALL ( 'Date'[Year] )
)
),
'Date'[Year] = 2007
Year
Year
Empty
filter
Removes Year from the filter

## Remove

FIGURE 5-40 ALL removes a previously existing fi lter from the context, when used as REMOVEFILTER.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 171
The difference between the two behaviors is subtle. In most calculations, the slight difference in
semantics will go unnoticed. Nevertheless, when we start authoring more advanced code, this small
difference will make a big impact. For now, the important detail is that when ALL is used as REMOVEFILTER, it acts as a CALCULATE modifi er instead of acting as a table function.
This is important because of the order of precedence of fi lters in CALCULATE. The CALCULATE
modifi ers are applied to the fi nal fi lter context before explicit fi lter arguments. Thus, consider the presence of ALL on a column where KEEPFILTERS is being used on another explicit fi lter over that column; it
produces the same result as a fi lter applied to that same column without KEEPFILTERS. In other words,
the following defi nitions of the Sales Red measure produce the same result:
Sales Red :=

## Calculate (

[Sales Amount],
'Product'[Color] = "Red"
)
Sales Red :=

## Calculate (

[Sales Amount],

```dax
KEEPFILTERS ( 'Product'[Color] = "Red" ),
ALL ( 'Product'[Color] )
)
The reason is that ALL is a CALCULATE modifi er. Therefore, ALL is applied before KEEPFILTERS.
```

Moreover, the same precedence rule of ALL is shared by other functions with the same ALL prefi x:
These are ALL, ALLSELECTED, ALLNOBLANKROW, ALLCROSSFILTERED, and ALLEXCEPT. We generally
refer to these functions as the ALL* functions. As a rule, ALL* functions are CALCULATE modifi ers when
used as top-level functions in CALCULATE fi lter arguments.
Introducing ALL and ALLSELECTED with no parameters
We introduced ALLSELECTED in Chapter 3. We introduced it early on, mainly because of how useful
it is. Like all the ALL* functions, ALLSELECTED acts as a CALCULATE modifi er when used as a top-level
function in CALCULATE. Moreover, when introducing ALLSELECTED, we described it as a table function
that can return the values of either a column or a table.
The following code computes a percentage over the total number of colors selected outside of the
current visual. The reason is that ALLSELECTED restores the fi lter context outside of the current visual
on the Product[Color] column.
SalesPct :=

## Divide (

[Sales],

## Calculate (

[Sales],
ALLSELECTED ( 'Product'[Color] )
)
)

172 CHAPTER 5 Understanding CALCULATE and CALCULATETABLE
One achieves a similar result using ALLSELECTED ( Product ), which executes ALLSELECTED on top of
a whole table. Nevertheless, when used as a CALCULATE modifi er, both ALL and ALLSELECTED can also
work without any parameter.
Thus, the following is a valid syntax:
SalesPct :=

## Divide (

[Sales],

## Calculate (

[Sales],

## Allselected ( )

)
)
As you can easily notice, in this case ALLSELECTED cannot be a table function. It is a CALCULATE
modifi er that instructs CALCULATE to restore the fi lter context that was active outside of the current
visual. The way this whole calculation works is rather complex. We will take the behavior of ALLSELECTED to the next level in Chapter 14. Similarly, ALL with no parameters clears the fi lter context from
all the tables in the model, restoring a fi lter context with no fi lters active.
Now that we have completed the overall structure of CALCULATE, we can fi nally discuss in detail the
order of evaluation of all the elements involving CALCULATE.
CALCULATE rules
In this fi nal section of a long and diffi cult chapter, we are now able to provide the defi nitive guide to
CALCULATE. You might want to reference this section multiple times, while reading the remaining
part of the book. Whenever you need to recall the complex behavior of CALCULATE, you will fi nd the
answer in this section.
Do not fear coming back here multiple times. We started working with DAX many years ago, and we
must still remind ourselves of these rules for complex formulas. DAX is a clean and powerful language,
but it is easy to forget small details here and there that are actually crucial in determining the calculation outcome of particular scenarios.
To recap, this is the overall picture of CALCULATE:

> **Note:** CALCULATE is executed in an evaluation context, which contains a fi lter context and might contain one or more row contexts. This is the original context.

> **Note:** CALCULATE creates a new fi lter context, in which it evaluates its fi rst argument. This is the new
fi lter context. The new fi lter context only contains a fi lter context. All the row contexts disappear
in the new fi lter context because of the context transition.

CHAPTER 5 Understanding CALCULATE and CALCULATETABLE 173

> **Note:** CALCULATE accepts three kinds of parameters:
• One expression that will be evaluated in the new fi lter context. This is always the fi rst
argument.
• A set of explicit fi lter arguments that manipulate the original fi lter context. Each fi lter argument might have a modifi er, such as KEEPFILTERS.
• A set of CALCULATE modifi ers that can change the model and/or the structure of the original fi lter context, by removing some fi lters or by altering the relationships architecture.

> **Note:** When the original context includes one or more row contexts, CALCULATE performs a context
transition adding implicit and hidden fi lter arguments. The implicit fi lter arguments obtained
by row contexts iterating table expressions marked as KEEPFILTERS are also modifi ed by

## Keepfilters.

When using all these parameters, CALCULATE follows a very precise algorithm. It needs to be well
understood if the developer hopes to be able to make sense of certain complex calculations.
1. CALCULATE evaluates all the explicit fi lter arguments in the original evaluation context. This
includes both the original row contexts (if any) and the original fi lter context. All explicit fi lter
arguments are evaluated independently in the original evaluation context. Once this evaluation is fi nished, CALCULATE starts building the new fi lter context.
2. CALCULATE makes a copy of the original fi lter context to prepare the new fi lter context. It discards
the original row contexts because the new evaluation context will not contain any row context.
3. CALCULATE performs the context transition. It uses the current value of columns in the original row contexts to provide a fi lter with a unique value for all the columns currently being
iterated in the original row contexts. This fi lter may or may not contain one individual row.
There is no guarantee that the new fi lter context contains a single row at this point. If there
are no row contexts active, this step is skipped. Once all implicit fi lters created by the context
transition are applied to the new fi lter context, CALCULATE moves on to the next step.
4. CALCULATE evaluates the CALCULATE modifi ers USERELATIONSHIP, CROSSFILTER, and ALL*.
This step happens after step 3. This is very important because it means that one can remove the
effects of the context transition by using ALL, as described in Chapter 10. The CALCULATE modifi ers are applied after the context transition, so they can alter the effects of the context transition.
5. CALCULATE evaluates all the explicit fi lter arguments in the original fi lter context. It applies
their result to the new fi lter context generated after step 4. These fi lter arguments are applied
to the new fi lter context once the context transition has happened so they can overwrite it,
after fi lter removal—their fi lter is not removed by any ALL* modifi er—and after the relationship architecture has been updated. However, the evaluation of fi lter arguments happens in
the original fi lter context, and it is not affected by any other modifi er or fi lter within the same
CALCULATE function.

```dax
The fi lter context generated after point (5) is the new fi lter context used by CALCULATE in the evaluation of its expression.
```
