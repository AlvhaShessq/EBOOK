# Chapter 14: Related distinct count

> **Sample files:** https://sql.bi/dax-208


The Related distinct count pattern is useful whenever you have one
or more fact tables related to a dimension, and you need to perform
the distinct count of column values in a dimension table only
considering items related to transactions in the fact table. For
demonstration purposes we use the distinct count of the product
name in a model with two fact tables: Sales and Receipts.
Because the product name is not unique – we artificially introduced
duplicated names by removing the color description from the product
name – a simple distinct count of the product key in the Sales or
Receipts table does not work. Finally, we show how to compute the
distinct count of product names that appear in both tables and in at
least one of the two.

Pattern description
The Product[Product Name] column is not unique in the Product table
and we need the distinct count of the product names that have
related sales transactions. The model contains two tables with
transactions related to products: Sales and Receipts. Figure 14-1
shows this data model.


*📊 FIGURE 14-1 The data model contains two fact tables: Sales and Receipts.*
*Based on this model we want to compute the distinct count of*

product names appearing:

In Sales.

In Receipts.
In both the Sales and Receipts tables.
In at least one of the Sales and Receipts tables.

The report is visible in Figure 14-2.


*📊 FIGURE 14-2 The report shows the four measures demonstrated in the pattern.*
*The code for the first two measures is the following:*


> *Measure in the Sales table*


> *Measure in the Receipts table*

```dax
Using SUMMARIZE, the # Prods from Sales and # Prods from Receipts
```

measures retrieve the distinct product names referenced in the
relevant table. SUMX just counts the number of those products and
it is used instead of COUNTROWS or DISTINCTCOUNT for
performance reasons – more details in the article Analyzying the
performance of DISTINCTCOUNT in DAX.
Despite being longer than a solution using DISTINCTCOUNT and
bidirectional cross-filtering, this version of the code is faster in the
most frequent case – where the number of products is significantly
smaller than the number of transactions.
NOTE The natural syntax to compute the Result variable in the #
Prods from Sales and # Prods from Receipts measures should use
COUNTROWS. The SUMX version is only suggested for
performance reasons in the simple measures. The following
measures of this pattern use COUNTROWS because there would
be no advantage in using SUMX in more complex expressions.

The formulation using SUMMARIZE and COUNTROWS can be
easily extended to accommodate for the next formulas that produce
the intersection (# Prods from Both) or the union (# Prods from Any) of
the product names:


> *Measure in the Receipts table*

> *Measure in the Receipts table*

We provided the examples for INTERSECT and UNION. But the
pattern can easily be adapted to perform more complex calculations.
As a further example, the # Prods in Sales and not in Receipts measure
computes the number of product names that exist in Sales but not in
Receipts by using the set function EXCEPT instead of the
INTERSECT or UNION functions used in previous measures:


> *Measure in the Sales table*

```dax
The result of the # Prods in Sales and not in Receipts measure is
visible in Figure 14-3.
```


*📊 FIGURE 14-3 The # Prods in Sales and not in Receipts measure counts the products present in*
*Sales but not in Receipts.*


The pattern can be extended to compute the distinct count of any
column in a table that can be reached through a many-to-one chain
of relationships from the fact tables. This is because SUMMARIZE is

able to group by any of those columns.
