# Chapter 12: ABC classification

> **Sample files:** https://sql.bi/dax-213


The ABC classification pattern classifies entities based on values,
grouping entities together that contribute to a certain percentage of
the total. A typical example of ABC classification is the segmentation
of products (entity) based on sales (value). The best-selling products
that contribute to up to 70% of the total sales belong to cluster A.
The products making up the next 20% of sales are in cluster B,
whereas the products representing the last 10% of sales, belong to
class C. Hence, the pattern is named after the three clusters (ABC).
You can use this pattern to determine the core business of a
company, typically in terms of best performing products or best
customers. You can find more information on ABC classification at
http://en.wikipedia.org/wiki/ABC_analysis.
ABC classification can be either static or dynamic. Static ABC
classification assigns a class to each product statically, so that the
class of a product does not change depending on the filters being
applied to the report. Dynamic ABC classification computes the class

of each product dynamically, based on the report filters. As such, in
the dynamic ABC classification the clustering of product needs to be
done in measures, resulting in a less efficient – albeit more flexible –
algorithm.
There is also a third pattern for this type of clustering, which lies in-
between the static and the dynamic versions: the snapshot ABC. For
example, if one needs to update the ABC class to a product on a
yearly basis, they can accomplish this by creating a snapshot table
containing the ABC class of a product for every year.


Static ABC classification
In the example, we cluster products based on sales. Each product is
statically assigned to a class that can be used on the rows and
columns of a report. The report in Figure 12-1 shows that there are
493 products in class A, making over 21M in sales, whereas 1,455
products in class C only generate 3M in sales.


*📊 FIGURE 12-1 The ABC class can be used to filter the products into a given class.*
*The static ABC classification is based on calculated columns. You*


need four new calculated columns, as shown in Figure 12-2.


*📊 FIGURE 12-2 The ABC static pattern requires four calculated columns.*
*The four calculated columns are:*


Product Sales: the total sales for the product (current row).
Cumulated Sales: the running total of Product Sales ranked
from largest to smallest.
Cumulated Pct: the percentage of Cumulated Sales against
the grand total of sales.
ABC Class: the class of the product, which could be A, B, or C.

You define the calculated columns using the following DAX
formulas:


> *Calculated Column in the Product table*


> *Calculated Column in the Product table*

> *Calculated Column in the Product table*


> *Calculated Column in the Product table*


### The product class is determined by the value of Cumulated Pct. As


you can see in Figure 12-3, when the value is below 70% the
product class is still A, when it is over 70% the product class
becomes B.


*📊 FIGURE 12-3 Products that fall over the 70% threshold in cumulated values are in class B.*
*The four columns can be replaced with a single calculated column*

containing the complete logic, using several variables:


> *Calculated Column in the Product table*

Using this version of the code reduces the size of the model,
because it creates one column in place of the four needed in the
earlier version. Nevertheless, on databases with a large number of
products, the column calculation might require an excessive amount
of memory.

Snapshot ABC classification
You might need to assign the ABC class to each product on a yearly
basis, so that the same product can fall into different ABC classes in
different years. In this case, you should build a solution with an
additional snapshot table containing the correct ABC class for each
product and year. The goal is to produce a report like the one in

*📊 Figure 12-4 – showing for each year, the number of products that fell*
*in class A, B or C.*


*📊 FIGURE 12-4 The ABC classification evaluates the product class every year.*
*The model requires an additional table to store the ABC class for*

each year and product. The ABC by Year table does not have
relationships with other tables in the model and it contains the
product key, the year, and the assigned class, as shown in Figure
12-5.


*📊 FIGURE 12-5 The calculated table that computes the ABC class has one row for each year*
*and product.*


### The code that computes the table is the following:


> *Calculated table*

The result of this code is the final one, shown in Figure 12-5. It
helps to visualize the content of the ClassByYearProduct variable,
which shows the columns added to the intermediate calculation
through several steps. You can see this in Figure 12-6.


*📊 FIGURE 12-6 The intermediate calculation evaluated in the ClassByYearProduct variable.*
*Once the table is loaded in the model, the ABC by Year table can be*

used as a filter remapping the data lineage of ProductKey and
Calendar Year to the corresponding columns in the Product and Date
tables. For example, the report shown at the beginning of the section
uses these two measures:


> *Measure in the Sales table*

> *Measure in the Sales table*

By using TREATAS both measures move the filter from the
snapshot to the Product and the Date tables, obtaining the desired
result. It is important to apply ProductKey and Calendar Year in the
same filter, otherwise the measure could include combinations of
products and years that are not included in the selected ABC

classes.
There is an alternative solution that works better in models with a
larger number of products – by using expanded tables. As you can
see in Figure 12-7, this requires an intermediate Years table linked to
Date through a relationship with a bidirectional filter (so it is not
available in the Excel Power Pivot sample).


*📊 FIGURE 12-7 The Years calculated table enables a relationship propagation from Date to*
*ABC by Year.*


### The Years table is easily computed using a DISTINCT function:


The measures are simpler – though harder to understand –
because they rely on table expansion:


> *Measure in the Sales table*


> *Measure in the Sales table*

The snapshot ABC classification is more dynamic than the static
version. The calculated table requires some computational effort.
Nevertheless, it is computed at data refresh time and it is very quick
at query time. Therefore, the snapshot ABC classification is a very
good compromise between speed and flexibility. If flexibility is the
main goal, then the slower dynamic ABC classification pattern is a
better fit.


Dynamic ABC classification
The dynamic ABC pattern is the most flexible of the three patterns
presented, and consequently it is the slowest and most memory-
hungry. The goal is to dynamically compute the number of products,
the sales amount or any other measure determining the set of
products that belong to the given ABC class in the context of the
report. For example, in Figure 12-8 the classes are determined
considering only the Cell phones category; when the user selects a

different category, the whole report is computed with the new filters.


*📊 FIGURE 12-8 The ABC classification segments the products dynamically, based on the*
*current selection.*


Being dynamic, the whole logic is defined in a measure that
retrieves the list of products in the desired class, and then uses this
list as a filter over the required calculation. Moreover, from the model
point of view, there is the need to create an additional ABC Classes
table that contains the three classes with their boundaries. This is
shown in Figure 12-9.


*📊 FIGURE 12-9 The ABC Classes table contains the definition of the boundaries for each class.*
*The measure that computes the ABC Sales Amount is the following:*


> *Measure in the Sales table*

The complexity of the formula mainly depends on the number of
products – the larger the number of products, the slower and more
memory-intensive it becomes. Over around ten thousand products,
the code will likely start to be too slow to produce an interactive
report. This defeats the initial purpose of obtaining a dynamic report.


Finding the ABC class
This pattern describes how to find the ABC class of a product
dynamically, producing the result in a measure instead of using a
column to classify an existing item. Other ABC segmentation
patterns aim to split products into different classes and compute a
value, like the sales amount or the number of products. This pattern
is useful when you need to show the ABC class of a product
dynamically, producing a report like Figure 12-10: The report shows
for each product of the Computers category, its ABC class in 2008.


*📊 FIGURE 12-10 The ABC classification segments the products dynamically, based on the*
*current selection.*


The measure that computes the ABC class is a variation of the
dynamic ABC classification. This time, the measure does not need to

compute the ABC class of all the products – it is enough to compute
the ABC class of the selected product. Therefore, once it computes
the list of all products along with their sales, the measure uses the
information to compute the correct values only for the current
product:


> *Measure in the Sales table*