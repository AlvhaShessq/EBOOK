# Chapter 10: Static segmentation

> **Sample files:** https://sql.bi/dax-211


The static segmentation pattern classifies numerical values into
ranges. A typical example is the analysis of sales by price range.
You do not want to slice the data by individual price; instead you
want to simplify the analysis by grouping prices within ranges of
prices. The price ranges are stored in a configuration table and the
pattern requires the model to be entirely data-driven. In other words,
when the configuration table is updated, the model is updated
automatically without requiring any change to the DAX code.
Depending on the size of the data model, there are different options
for this pattern. On small models (up to a few million rows) the best
option is to use calculated columns and/or calculated relationships.
On larger models with hundreds of millions of rows, calculated
columns might increase the processing time of the model. Therefore,
for large models the best option is to build a calculated table
expanding the prices, thereby reducing to a minimum the number of
calculated columns in the larger tables.

Basic pattern
You need to analyze sales sliced by price range. To attain this goal,
you build a configuration table that stores the price ranges; the price
should be greater than or equal to the Min Price and less than the
Max Price, as shown in Figure 10-1.


*📊 FIGURE 10-1 The configuration table defines the price ranges.*
*Then, you want to analyze sales by price range, obtaining a report*

like Figure 10-2.


*📊 FIGURE 10-2 The report shows sales sliced by price range.*
*In the report, the VERY LOW row contains the sales with a net*

price between 0 and 100.
In order to obtain the desired result, you need a relationship
between the configuration table (Price Ranges) and the Sales table. In
the example, we use Sales[Net Price] instead of Sales[Unit Price] to
determine the sales price, so to consider possible discounts. Indeed,
Sales[Net Price] might be different than Sales[Unit price] because of
discounts. The required relationship should use a “between”
condition for the join, which is not natively supported by the Tabular

engine. Nevertheless, in the Sales table we can add a calculated
column that stores the key of the price range for each specific row,
by using the following code:


> *Calculated column in the Sales table*

When building the calculated column, you need to be careful not to
use functions that might reference the blank row, such as ALL and
VALUES. This is the reason we used DISTINCT instead of VALUES
to retrieve the price range key.
Next, you build a relationship between Sales and Price Ranges based
on the new calculated column, like in Figure 10-3.


*📊 FIGURE 10-3 The relationship is based on a calculated column.*
*Once the relationship is in place, you can slice sales by ‘Price*

Ranges’[Price Range].
You need to make sure that the configuration table is properly
designed, so that each price belongs to only one price range. The
presence of overlapping segments in the configuration table can
generate errors in the evaluation of the PriceRangeKey calculated
column. If you want to make sure there are no mistakes in the
configuration table – such as overlapping ranges – you can generate
the Max Price column using a calculated column that retrieves the
value of Min Price for the next segment. This is shown in the
following sample code.


> *Calculated column in the Price Ranges table*

```dax
You can also write a safer version of the calculated column that
```

writes a blank or generates an error in the event there are multiple
ranges active for one price, as in the following example:


> *Calculated column in the Sales table*

```dax
The code shown in this pattern must satisfy the requirements for
calculated columns used in a relationship, in order to avoid circular
dependencies.
```


Price ranges by category
A variation of the static segmentation pattern is when the condition
to check is not a simple between, but rather a more complex
condition. For example, the requirement might be to use different
price ranges for different product categories: The LOW price range
for games and toys needs to be different from the LOW price range
for home appliances.
In this scenario, the configuration table contains an additional

column that indicates the category the price range must be applied
to. Different categories might have different price ranges, as in

*📊 Figure 10-4.*

*📊 FIGURE 10-4 The configuration table also contains the categories.*
*The pattern here is very similar to the basic pattern, the only*

noticeable change being in the condition used to find the correct
price range key. Indeed, the search must be limited to the row in the
Price Ranges table with the category of the product being sold and
where the net price falls within the desired range:


> *Calculated column in the Sales table*

```dax
Similarly, you can use any other condition if it is guaranteed that
```

only one row remains visible in the configuration table. In order to
make sure that the configuration table does not contain overlapping
ranges, you can generate the Max Price column using a calculated
column similar to the one used in the basic pattern. The important
difference is the use of ALLEXCEPT instead of REMOVEFILTERS,
so that the filter over ‘Price Ranges’[Category] coming from the
context transition is kept in the filter context:


> *Calculated column in the Price Ranges table*

```dax
Price ranges on large tables
The static segmentation pattern requires the creation of a calculated
column in the Sales table. The column itself is typically rather small in
size, because it contains few distinct values. However, on very large
tables the column size might start to grow and you may face another
problem: the column needs to be computed for the entire table at
every data refresh. On a multi-billion-row table that is likely to be
partitioned, the column needs to be recomputed for the entire table
whenever one partition is refreshed. This slows down every refresh
operation.
In this scenario, it is possible to use a variation of the static
segmentation that works without adding any column in the Sales
table. Instead of building the relationship with the new calculated
column, this pattern uses Sales[Net Price] as the key for a
relationship with a new calculated table. Indeed, it is not possible to
create a relationship between Sales and the Price Ranges table
because the Price Ranges table is missing a suitable column.
Nevertheless, such column can be created by increasing the number

of rows in the configuration table.
The table we want to generate contains one row for each value of
Sales[Net Price] with the corresponding price range, like in Figure 10-
5.
```


*📊 FIGURE 10-5 The expanded configuration contains one row for each value in Net Price.*
*We renamed the original configuration table to Price Ranges*

Configuration. The Price Ranges table can be created as a calculated
table using the following code:


> *Calculated table*

```dax
This new table contains exactly one row for each distinct value of
the Sales[Net Price] column. Therefore, it is possible to create a
```

relationship between Sales and the new Price Ranges calculated table
based on the Net Price column, as shown in Figure 10-6.


*📊 FIGURE 10-6 The relationship is based on the Net Price column.*
*With this optimization, there is no need to create a new column in*


### Sales, because the model uses the existing Sales[Net Price] column to


setup the relationship. Therefore, no calculated column in Sales must
be recomputed during data refresh. The original Price Ranges
Configuration table should be hidden in the model in order to avoid
any possible confusion for the end users.
On smaller models, creating a calculated column is not an issue.
Therefore, the basic solution that does not involve new tables is to
be preferred. On larger models, this version reduces the processing
time.
