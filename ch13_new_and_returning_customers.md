# Chapter 13: New and returning customers

> **Sample files:** https://sql.bi/dax-218


The New and returning customers pattern helps in understanding
how many customers in a period are new, returning, lost, or
recovered. There are several variations to this pattern, each with
different performance and results depending on the requirements.
Moreover, it is a very flexible pattern that allows the identification of
new and returning customers, or the computation of these
customers’ purchase volume – also known as sales amount.
Before using this pattern, you need to clearly define the meaning of
new and returning customers, as well as when a customer is lost or
recovered. Indeed, depending on the definition you give to these
calculations, the formulas are quite different both in their writing and
– most important – in performance. Even though you could use the
most flexible formula to compute any variation, we would advise you
to spend some time experimenting in order to find the best version
that fits your needs. The most flexible formula is very expensive from

a computational point of view. Therefore, it might be slow even on
small datasets.


Introduction
Given a certain time period, you want to compute these formulas:

Customers: the number of customers who made a purchase
within that time period.
New customers: the number of customers who made their first
purchase within that time period.
Returning customers: the number of customers who have
already purchased something in the past, and are returning in
that time period.
Lost customers: the number of customers whose last purchase
occurred at least 2 months before the start of the current
period.
Recovered customers: the number of customers who were
considered lost in a previous time period, and then made a
purchase in the current period.

The report looks like the one in Figure 13-1.


*📊 FIGURE 13-1 The report shows the main calculations of the pattern.*
*As shown in the report, in January 2007 all customers were new. In*

February, 116 customers were returning and 1,037 were new, for a
total of 1,153 customers. In March, 603 customers were lost.
While the measures computing the number of customers and the
number of new customers are easy to describe, calculating the
number of lost customers is already complex. In the example, let us
look at a customer lost two months after their last purchase.
Therefore, the number reported (603) is made up of customers who
made their last purchase in January. In other words, out of the 1,375
customers in January 2007, 603 did not buy anything in February,
March, and the following months; for this reason, we consider them
lost at the end of March.
The definition of lost customers may be different in your business.
For example, you might define a customer as lost if they made their
last purchase two months ago, even though you already know that

they will be making another purchase next month. Imagine a
customer who bought something in January and April: are they lost
at the end of March or not? The answer leads to different
formulations of the same calculation. Indeed, we consider the
customer as being temporarily lost at the end of March, because we
know the same customer will be recovered later. A report counting
the temporarily-lost customers (who did not buy anything for two
months, but then made a purchase afterwards) is visible in Figure
13-2.


*📊 FIGURE 13-2 The report shows temporarily-lost customers, along with the number of*
*recovered customers.*


The number of temporarily-lost customers is higher than the
number of lost customers previously shown. The reason is that many
of the temporarily-lost customers will buy something in future
months. In that case, the report counts them as recovered

customers in the month when they make a new purchase.
Another important element to take into account when selecting the
right pattern is how you want to look at filters on the report. If the
user selects a category of products, how does this filter affect the
calculation? Let us say that you filter the Cell Phones category. Do
you consider a customer as new the first time they buy a cell phone?
If so, then a single customer will be new multiple times, depending
on the filter. Otherwise, if you want to consider a customer as new
only once, then you need to ignore the filters when computing the
number of new customers. Similarly, all the remaining measures
might or might not be affected by the filters.
Let us clarify the concept with another example. Figure 13-3 shows
the raw data of a reduced version of Contoso with only three
customers.


*📊 FIGURE 13-3 The report shows three customers along with their purchase history.*
*Considering the data in Figure 13-3, can you tell when Dale Lal is a*

new customer, if a user added a filter for Games and Toys? He
bought a toy for the first time in April, even though he was already a
customer for Cameras and camcorders products. Now focus on
Tammy Metha: is she to be considered lost two months after her
game purchase in February? She did not buy any other game
product, even though she bought products of other categories.
Answering these questions is of paramount importance to support

your choice of the pattern that will best suit your specific business
needs.
Additionally, counting customers is useful, but sometimes you are
interested in analyzing the amounts sold to new, returning, and
recovered customers. Or you might want to estimate the amount lost
because of customer losses, in a report like the one in Figure 13-4.
In the report we used the average sales volumes of our lost
customers over the last 12 months, as an estimate for lost sales.


*📊 FIGURE 13-4 The report shows the sales amount of new, returning, recovered, and lost*
*customers.*


Another important note is to think about how the formulas count the
different statuses of a customer inside each time period. For
example, if you consider a full year, then it is possible that the same
customer is new, temporarily lost, returning, and then permanently
lost – all within the same period. On a given day, the status of a
customer is well defined. However, throughout longer time frames
the same customer can be in different statuses. Our formulas are
designed to account for the customer in all their statuses. Figure 13-
5 shows a sample report that only filters and shows one customer:
Lal Dale.


*📊 FIGURE 13-5 The same customer is both new and lost in the same year.*
*The customer is both new and lost in the same year. Lal Dale was*

a returning customer for a few months, but not at the year level
because he was new during the year. In Figure 13-6 the same report
filters out January, thus showing the customer as returning three
times within the period, and never showing them as a new customer.


*📊 FIGURE 13-6 Filtering out January, when the customer was new, the customer now*
*appears as returning in the period.*


### If we were to describe all the possible combinations of measures in


this pattern, this alone would require an entire book. Instead, we
show some of the most common patterns, leaving to the reader the
task of changing the formulas in case their scenario is different from
any of the patterns described.
Finally, the New and returning customers pattern requires heavy
calculations. Therefore, we present both a dynamic and a snapshot
version of the formulas.


Pattern description
The pattern is based on two types of formulas:

Internal formulas: their goal is to compute the relevant dates
for a given customer.
External formulas: these are the formulas used in reports.
They use the internal formulas to compute the number of
customers, the sales amount, or any other measure.

For example, in order to compute the number of new customers, for
each customer the internal formula computes the date of their first
purchase. The external formula then computes the number of
customers whose first purchase happens to fall within the time
period currently filtered.
An example is helpful to better understand this technique. Look at

*📊 Figure 13-7, which shows the reduced dataset we use in order to*
*explain the different formulas.*


*📊 FIGURE 13-7 The report shows a few customers, along with their purchase history.*
*Using this data as an example, think about how you can compute*

the number of new customers in March. The new customers external
measure checks how many customers made their first purchase in
March. To obtain its result, the external formula queries the internal
formulas on a customer-by-customer basis, checking their first
purchase. The internal formula returns March 14 for the first
purchase of Gerald Suri, whereas the first purchases of the other
customers occurred earlier than that. Consequently, the external
formula returns 1 as the number of new customers.
Other measures behave the same way, although each comes with
peculiarities worthy of a more complete description.
As a first example of code, look at the internal formula that
computes the date when a customer must be considered new. Be
mindful, each example has different formulas and we provide greater
detail on this code in subsequent sections. This first example of DAX
is reported here only as an introduction:


> *Measure (hidden) in the Sales table*

```dax
The internal formula is then used by the external formula, which
```

computes the number of customers who are new in the given period:


> *Measure in the Sales table*

Using this approach, the pattern is more flexible. Indeed, if you
need to change the logic that determines when a customer is to be
considered new, lost, or temporarily lost, you only need to update the
internal formulas – thus leaving the external formula untouched. Still,

we need to raise a big warning for our readers: the formulas shown
in this pattern are extremely complex and delicate in the way the
filter context is handled. You will certainly need to change them to
suit your needs. But do so only after having thoroughly understood
their behavior; indeed, each line of DAX in this pattern is the result of
hours of thinking and endless tests, as we systematically had to
make sure that it was the correct way to write it. In other words, get
ready to walk on eggshells with this pattern; we certainly had to!
We organized the patterns in two families: dynamic and snapshot.
The dynamic version computes the measures in a dynamic way,
considering all the filters of the report. The snapshot version
precomputes the values of the internal measures in calculated
tables, in order to speed up the calculation of the external measures.
Therefore, the snapshot version provides less flexibility, albeit with
improved speed.
We also provide three different implementations, depending on how
the measure should consider the active filters in the report:

Relative: a customer is considered new the first time they buy
one of the products selected in the report.
Absolute: a customer is considered new the first time they buy
a product, regardless of any filter present in the report.
By category: a customer is considered new the first time they
buy a product from any of the product categories selected in
the report. If they buy two products of the same category then
they are considered new only once, whereas if they buy two
products of different categories then they are considered new
twice.


### You can find a more complete explanation of the various


calculations in the corresponding section of each pattern. Our
suggestion is to read the chapter start-to-finish before attempting an
implementation on your model. It is better to understand your
requirements well before proceeding with the implementation, rather
than only finding out at the end that you chose the wrong pattern.
Finally, the demo files of this pattern include two versions: the full
version includes the complete database, whereas the base version
only includes three customers. The base version is useful to better
understand the pattern, because you can easily check the numbers
thanks to the limited number of rows in the model. The full version is
more useful to evaluate the performance of the different calculations.

Internal measures
There are three internal measures:

Date New Customer: returns the date when the customer is to
be considered new.
Date Lost Customer: returns the date when the customer is to
be considered permanently lost, checking that there are no
sales in following time periods.
Date Temporary Lost Customer: returns the date when the
customer might be lost, without checking whether the
customer comes back in a following period.

These measures are not intended to be used in reports – they exist
only to be used by the external measures. The code of the internal
measures is different for each pattern.

External measures
Each pattern defines several measures to count customers and

evaluate sales in the various customer states:

# New Customers: counts the number of customers who are
new.
# Returning Customers: counts the number of customers who
were new in a previous period and made a new purchase
within the time period considered.
# Lost Customers: counts the number of customers permanently
lost.
# Temporarily Lost Customers: counts the number of customers
who are only lost when we look at the current time period,
even though they might return in a later period.
# Recovered Customers: counts the number of customers who
were temporarily lost and then made a new purchase within
the time period considered.
Sales New Customers: returns the value of Sales Amount by
filtering only the new customers.
Sales Returning Customers: computes the value of Sales Amount
by filtering only the customers who were new in a previous
period and made a new purchase in the period considered.
Sales Lost Customers (12M): computes the value of Sales Amount
for 12 months prior to the start of the selected time period,
filtering only customers permanently lost in the selected
period.
Sales Recovered Customers: returns the value of Sales Amount
filtering only customers who were previously temporarily lost
and who made a new purchase in the period considered.


### The code of the external measures is very similar in all the


patterns. There are minor variations for some scenarios that are
highlighted when we describe the individual patterns.

How to use pattern measures
The formulas presented in the pattern can be grouped into two
categories. The measures starting with the # prefix compute the
number of unique customers by applying a certain filter. Usually
these measures are used as-is and are optimized for this purpose.
For example, the following measure returns the number of new
customers:


> *Measure in the Sales table*

The measures that do not start with the # prefix create a filter of
customers that is applied to another measure. For example, the
measures with the Sales prefix are measures that apply a filter of
customers to the Sales Amount measure. The following measure can
be reused to compute other measures by just changing the Sales


### Amount measure reference in the last CALCULATE function:


> *Measure in the Sales table*

In each pattern we show the two measures (with the # and Sales
prefixes) when there are differences in the measure structure, even
just for performance optimization. If the two measures only differ by
the calculation made in the last CALCULATE function, then we only
include the # prefix version of the measure.

Dynamic relative
The Dynamic relative pattern takes into account all the filters in the
report for the calculation. Therefore, if the report filters one category
(Audio, for example), a customer is reported as new the first time
they buy a product of the Audio category. Similarly, a customer is
considered lost a certain number of days after they last purchased a
product of the Audio category. Figure 13-8 is useful to better
understand the behavior of this pattern.


*📊 FIGURE 13-8 The only customer visible (Lal Dale) is considered as new multiple times, for*
*different categories.*


The report only takes one customer into account: Lal Dale. He is
reported as new in January, when Cameras and camcorders is
selected, and he is also considered new in April, for the Games and
Toys category. All the other measures behave similarly, by
considering the filter where they are evaluated.

Internal measures
The internal measures are the following:


> *Measure (hidden) in the Sales table*


> *Measure (hidden) in the Sales table*

> *Measure (hidden) in the Sales table*

New customers
The measure that computes the number of new customers is the
following:


> *Measure in the Sales table*

```dax
The code computes the date when each customer is new.
```

ALLSELECTED is useful for optimization purposes: it lets the engine
reuse the value of the CustomersWithNewDate variable in multiple
executions of the same expression.
Then, in CustomersWithLineage the formula updates the lineage of
CustomersWithNewDate to let the variable filter Sales[CustomerKey] and
Date[Date]. When used as a filter, CustomersWithLineage makes the
customers only visible on dates when they are considered new. The
final CALCULATE applies the CustomersWithLineage filter using
KEEPFILTERS to intersect with the current filter context. This way
the new filter context ignores customers that are not new in the
range of dates considered.
In order to apply the new customers as a filter for another measure
like Sales Amount we need a slightly different approach, as shown in
the following Sales New Customers measure:


> *Measure in the Sales table*

```dax
The NewCustomers variable holds a list of the values in
Sales[CustomerKey] corresponding to the new customers, obtained by
```

checking whether the @NewCustomerDate is within the filter context
of the current evaluation. The NewCustomers variable obtained this
way is then applied as a filter to compute the Sales Amount measure.
Even though the variable contains two columns (Sales[CustomerKey]
and @NewCustomerDate), the only column actively filtering the model
is Sales[CustomerKey], because the newly added column does not
share the lineage with any other column in the model.

Lost customers
The measure computing the number of lost customers needs to
count customers that are not part of the current filter context. Indeed,
in March we might lose a customer who made a purchase in
January. Therefore, when filtering March the customer is not visible.
The formula must look back at January to find that customer. This is
the reason why the structure of the code is different from the New


### Customers measure:


> *Measure in the Sales table*

The CustomersWithLostDate variable computes the date of loss for
each customer. LostCustomers filters out customers whose date of
loss is not in the current period. Eventually, the measure computes
the number of customers left by counting the rows in LostCustomers
that correspond to the customers whose date of loss falls within the
period visible in the current filter context.

Temporarily-lost customers
The measure computing the number of temporarily-lost customers is
a major variation of the measure computing the lost customers. The
measure must check that in the current context the customer who is
potentially lost did not make a purchase prior to the date when they
would have been lost. This is the code that implements this

calculation:


> *Measure in the Sales table*

The measure first computes the potential date of loss of each
customer; it applies a filter on the date so that it only considers

transactions made before the start of the current time period. Then, it
checks which customers have a loss date that falls within the current
period.
The resulting table (PotentialTemporarilyLostCustomers) contains the
customers that can be potentially lost in the current period. Before
returning a result, a final check is required: these customers must
not have purchased anything in the current period before the date
when they would be considered lost. This validation happens by
computing TemporarilyLostCustomers, which checks for each
customer whether there are sales in the current period before the
date when the customer would be considered lost.

Recovered customers
The number of recovered customers is the number of customers that
were temporarily lost before a purchase was made in the current
period. It is computed by the following measure:


> *Measure in the Sales table*

```dax
The CustomersWithLostDateComplete variable computes the
```

temporarily-lost date for the customers. Out of this list, the
CustomersWithLostDate variable removes the customers who do not
have a temporarily-lost date. The ActiveCustomers variable retrieves
the first purchase date for the customers in the current selection.
The RecoveredCustomers variable filters customers that are in both
ActiveCustomers and CustomersWithLostDate lists and have a
transaction date greater than the temporarily-lost date.
Finally, the Result variable counts the recovered customers.

Returning customers
The last measure in the set of counting measures is # Returning
Customers:


> *Measure in the Sales table*

The measure first prepares a table in CustomersWithNewDate with
the first purchase date for every customer. The ExistingCustomers
variable filters out all the customers whose date is not strictly earlier
than the start of the currently selected period. What remains in
ExistingCustomers is the set of customers who already purchased
products before the current period started. Therefore, if those
customers also made purchases within the current period, then they
are returning customers. This last condition is obtained by combining
ExistingCustomers with the customers active in the selected period.

The result in the ReturningCustomers variable can be used to count
the returning customers – as in this measure – or to filter them in a
different calculation.

Dynamic absolute
The Dynamic absolute pattern ignores the filters on the report when
computing the relevant dates for the customer. Its implementation is
a variation of the basic Dynamic relative pattern, with a different set
of CALCULATE modifiers to explicitly ignore filters.
The result is an absolute assignment of the status of a customer
regardless of report filters, as shown in Figure 13-9: Dale Lal is
considered new in January when Games and Toys is selected, even
though he purchased cameras and no games.


*📊 FIGURE 13-9 Lal Dale is considered new, returning, and lost regardless of the category*
*used in the visual.*


The only measure that changes depending on the category is #
Customers, which shows when Lal Dale purchased products. All the
other measures ignore the filter on the product: customers are new
only the first time they make a purchase regardless of the report
filter.

Internal measures
The internal measures are the following:


> *Measure (hidden) in the Sales table*


> *Measure (hidden) in the Sales table*


> *Measure (hidden) in the Sales table*

```dax
As shown in the previous code, the internal measures are designed
```

to ignore all filters other than the ones on Customer – with the
noticeable exception of Date Temporary Lost Customer which needs to
also consider the filters on Date.
Please note that the internal measures have been designed to
behave properly when called from the external measures. This is the
reason why ALLEXCEPT explicitly keeps the filter on
Sales[CustomerKey] in a somewhat unusual way. If called within an
iteration that includes that column, the internal measures do not
remove the filter, thereby observing the requirements of the external
measure.

New customers
The measure that computes the new customers is the following:


> *Measure in the Sales table*

```dax
There are two things to note about this measure. First, the filter in
```

the calculation of CustomersWithNewDate uses ALLEXCEPT to ignore
any filter apart from the ones on the Customer table. Second, in order
to check whether a customer is new, the measure filters the content
of CustomersWithNewDate. It then counts the row in the NewCustomers
variable, instead of using TREATAS as the corresponding measure
in the Dynamic relative pattern. This technique may turn out to be
slower than the one used in the Dynamic relative pattern; it is still
required because it needs to count a customer even though they
might not be visible due to the current filter context.

Lost customers
The measure computing the number of lost customers is the
following:


> *Measure in the Sales table*

```dax
Its structure is close to the New Customer measure, the main
```

difference being in the calculation of the CustomersWithLostDate
variable.

Temporarily-lost customers
The measure computing the number of temporarily-lost customers is
a variation of the measure computing the lost customers:


> *Measure in the Sales table*

```dax
Its behavior is very close to the corresponding measure in the
```

Dynamic relative pattern. The main differences are the use of
ALLEXCEPT in the evaluation of CustomersWithLostDateComplete and
ActiveCustomers. In CustomersWithLostDateComplete all the filters other

than Customer are removed, whereas in ActiveCustomers the filters
are not removed from Date and Customer.

Recovered customers
The number of recovered customers is the number of customers that
were temporarily lost before a purchase made in the current period.
It is computed by the following measure:


> *Measure in the Sales table*

```dax
Its behavior is very close to the corresponding measure in the
Dynamic relative pattern. The main difference is the use of
```

ALLEXCEPT in the evaluation of the CustomersWithLostDateComplete
and ActiveCustomer variables to correctly set the required filter.

Returning customers
The last measure in the set of counting ones is the # Returning
Customers:


> *Measure in the Sales table*

Its behavior is very close to the corresponding measure in the
Dynamic relative pattern. The main difference is the use of

ALLEXCEPT in the evaluation of the CustomersWithNewDate and
ActiveCustomers variables, to accurately set the required filter.

Generic dynamic pattern (dynamic by
category)
The generic dynamic pattern is an intermediate level between the
absolute and the dynamic patterns. The pattern ignores all the filters
from the report except for attributes determined by the business
logic. In the examples used in this section, the measures are local to
each product category. The result is dynamic for product category
and absolute for all the other attributes in the data model. For
instance, one customer can be new for a product category and a
returning customer for another product category within the same
month. The same customers might be considered new multiple times
if they buy different categories of products over time. In other words,
the analysis of new and returning customers is made by product
category. You can customize the pattern by replacing product
category with one or more other attributes, so that it fits your
business logic.
We purposely avoided excessive optimizations when writing the
code of this pattern: the primary goal of this set of measures is to
make them easier to update. If you plan on modifying the pattern to
fit your needs, this set of measures should be a good starting point.
The rules of this pattern are the following:

The same customer might be considered a new customer
multiple times, one for each combination of dynamic attributes
(product category in the example).
Customers are considered returning customers if they already
purchased the same combination of dynamic attributes
(product category in the example) they are purchasing in the

selected period.
Customers are temporarily lost if they did not purchase a
combination of dynamic attributes (product category in the
example) for two months, even though they may have
purchased different combinations of dynamic attributes
(product category in the example) in the meantime.
Customers are considered recovered customers if they make
a new purchase of products of the very combination of
dynamic attributes (product category in the example) for which
they were temporarily lost.

It is important to note that the pattern detects the customers, not
the combination of dynamic attributes and customers – like customer
and product category in the example. Therefore, the measures with
the # prefix always return the number of unique customers, whereas
the measures with the Sales prefix always evaluate the Sales Amount
measure regardless of the combination of dynamic attributes
(product category in the example) for which a customer is
considered new/lost/recovered. The difference is visible by filtering
two or more combinations of the dynamic attributes. For example, by
filtering two product categories, the Sales measures for new and
returning customers could add up to more than the value of Sales
Amount; indeed, the same amount can be computed considering the
same customer both new and returning, because of their having
different states for different categories.
Your requirements might be different from those assumed in this
example. In that case, as we already stated in the introduction, you
need to very carefully understand the filtering happening in all the
measures before implementing any change. These measures are
quite complex and easy to break with small changes.

Internal measures
The internal measures are the following:


> *Measure (hidden) in the Sales table*


> *Measure (hidden) in the Sales table*


> *Measure (hidden) in the Sales table*

```dax
As shown in this code, the internal measures are designed to
ignore filters other than the ones on Customer and Product[Category].

New customers
```


### The measure that computes the new customers is the following:


> *Measure in the Sales table*

```dax
In this version of the measure, the CustomersWithNewDate variable
```

might compute a different date for each product category. Indeed,
SUMMARIZE uses the Product[Category] column as a group-by

condition. Consequently, TREATAS specifies the lineage for the
three    columns   in    CustomersWithNewDate so    that    the
@NewCustomerDate column can be used later to filter or join the
Date[Date] column.
For performance reasons, the CustomersWithNewDate and
CustomersCategoryNewDate variables are invariant to the filter context
of cells in a report, so their result is computed only once for a single
visualization. In order to get the actual new customers, it is
necessary to filter those combinations that are not visible in the filter
context where # New Customer is evaluated. This is accomplished by
the NATURALINNERJOIN in ActiveNewCustomers, which joins the
combinations of customer, date, and category visible in the filter
context (ActiveCustomersCategories) with the combinations in
CustomersCategoryNewDate.
The NewCustomers variable removes the duplicated customers that
could be new for different categories in the same period. This way,
NewCustomers can be used as a filter in following calculations or it
can be counted to obtain the number of new customers, as the # New
Customers measure does.
The Sales New Customers measure is similar to # New Customers, the
only difference is the Result variable that uses the NewCustomersCat
as a filter in CALCULATE instead of just counting the rows of the
NewCustomer variable. Therefore, we show here only the last part of
the code, using ellipsis for the unchanged sections:


> *Measure in the Sales table*

```dax
Lost customers
The measure computing the number of lost customers is the
following:
```


> *Measure in the Sales table*

```dax
In this version of the measure, the CustomersWithLostDate variable
```

might compute a different date for each product category. The
reason is that SUMMARIZE uses the Product[Category] column as a
group-by condition and that the customer might have different dates

of loss – one for each category.
The LostCustomersCategories variable only filters the combinations of
customers and categories that have a lost date included in the
selected time period. Similarly to the New Customers measure, the
LostCustomers variable removes the duplicated customers so it can
be used both as a filter and to count the lost customers.

Temporarily-lost customers
The measure computing the number of temporarily-lost customers is
a variation of the measure computing the lost customers:


> *Measure in the Sales table*

The CustomersWithLostDateComplete variable needs to enforce the
filter on the Product[Category] column by using the VALUES function
– though the filter might not be directly applied to that column but
rather, to other columns cross-filtering Product[Category].
Similarly, the ActiveCustomersCategories variable creates a table of
combinations of Sales[CustomerKey] and Product[Category] along with
the first purchase date for each combination of customers and
product category. This table is then joined to the
PotentialTemporarilyLostCustomers variable, which contains the
content of CustomersWithLostDate visible in the current selection. The
result of the join filtered by date over the limit of the temporarily-lost
date is returned in the TemporarilyLostCustomersCategories variable.
Finally, to avoid counting the same customer multiple times, the
measure extracts the customer key before finally counting the
number of temporarily-lost customers.

Recovered customers
The number of recovered customers is the number of customers that
were temporarily lost before a purchase was made in the current
period. It is computed by using the following measure:


> *Measure in the Sales table*

The measure first determines the customers that were temporarily
lost before the current date, also summarizing by Product[Category].
Because the Sales[CustomerKey] and Product[Category] columns are
part of the tables stored in the CustomersWithLostDateComplete and

ActiveCustomers       variables,       the      join    made       in
RecoveredCustomersCategories returns a table that has both columns.
This ensures that a customer that was to be considered lost for a
given category is recovered only if they buy a product of the same
category. The customer might appear multiple times in this table, so
duplicated customers are removed in RecoveredCustomersCategories
in order to count or filter only the unique recovered customers. The
Sales Recovered Customers measure is similar to # Recovered
Customers; the only difference is the Result variable that uses
RecoveredCustomersCat as a filter in CALCULATE instead of just
counting the rows of the corresponding RecoveredCustomersCategories
variable in the # Recovered Customers measure. Therefore, here we
only show the last part of the code, using ellipsis for the identical
sections:


> *Measure in the Sales table*

Returning customers
The last measure in the set of counting measures is # Returning


### Customers:


> *Measure in the Sales table*

```dax
The measure creates a CustomersWithNewDate variable which
```

obtains the first sale date for each combination of customer and
product category. This result is joined to the combination of
customers and product category that is present in the current filter
context over Sales. The result is the set of returning customers in the
ReturningCustomers variable that is counted in the # Returning
Customer measure. The Sales Returning Customers measure uses the
following ReturningCustomersCat variable as a filter instead of the
ReturningCustomers variable. Here we only write its final lines of code,
all the remaining code being identical to the previous formula:


> *Measure in the Sales table*

```dax
Snapshot absolute
Computing new and returning customers dynamically is a very
expensive operation. Therefore, this pattern is oftentimes
implemented by using precomputed tables (snapshots) to store the
most relevant dates at the desired granularity.
By using precomputed tables, we get a much faster solution albeit
with reduced flexibility. In the pre-calculated absolute pattern, the
state of new and returning customers does not depend on the filters
applied to the report. The results obtained by using this pattern
correspond to those of the Dynamic absolute pattern.
The pattern uses a snapshot table containing the relevant states of
each customer (New, Lost, Temporarily lost, and Recovered) shown
in Figure 13-10.
```


*📊 FIGURE 13-10 The snapshot table contains the full history for each customer.*
*The New and Lost events are unique for each customer, whereas*

the Temporarily lost and Recovered events can have multiple
occurrences over time for each customer.
The resulting table is linked to Customer and Date through regular
relationships. The resulting model is visible in Figure 13-11.


*📊 FIGURE 13-11 The CustomerEvents snapshot table is connected to Customer and Date.*
*Building the CustomerEvents table is a critical step. Creating this*

table as a derived snapshot by using a calculated table in DAX is
relatively efficient for the New and Lost states, whereas it can be
very expensive for the Temporarily lost and Recovered states. Keep
in mind that Temporarily lost is needed in order to compute the
Recovered state. In models with hundreds of thousands of
customers or with hundreds of millions of sales you should consider
preparing this table outside of the data model, and importing it as a
simple table.
Once this model is in place, the DAX measures are simple and
efficient. Indeed, for this model there is no need to create external
and internal measures – the external measures are already simple.

The full logic that defines the status of a customer is in the table
itself. This is the reason why the resulting DAX code is much
simpler.
The only calculation that requires some attention is the # Returning
Customers measure, because it computes the number of customers
dynamically while ignoring any filter other than Date and Customer. It
then subtracts the number of new customers obtained by querying
the snapshot table:


> *Measure in the Sales table*


> *Measure in the Sales table*


> *Measure in the Sales table*


> *Measure in the Sales table*

> *Measure in the Sales table*

The measures computing the sales amount for new and returning
customers take advantage of the physical relationship between the
CustomerEvents snapshot table and the Customer table, thus reducing
the DAX code required and providing higher efficiency:


> *Measure in the Sales table*

> *Measure in the Sales table*


> *Measure in the Sales table*

```dax
Creating the derived snapshot table in
DAX
```

We suggest creating the CustomerEvents snapshot table outside of
the data model. Indeed, creating it in DAX is an expensive operation
that requires large amounts of memory and processing power to
refresh the data model. The DAX implementation described in this
section works well on models with up to a few thousand customers
and up to a few million sales transactions. If your model is larger
than that, you can implement a similar business logic using other
tools or languages that are more optimized for data preparation.
The complex part of the calculation is the retrieving of the dates
when a customer is temporarily lost and then possibly recovered.
These events can happen multiple times for each customer. For this
reason, for each transaction we compute two dates in two calculated
columns in the Sales table:

TemporarilyLostDate: this is the date obtained by the Date
Temporary Lost Customer measure when there are no other
transactions between the current row in Sales and the date. If
the same customer put multiple transactions through on the

same date, all of them will have the same value in the
TemporarilyLostDate column.
RecoveredDate: this is the date of the first purchase made by
that same customer after TemporarilyLostDate. This column is
blank if there are no transactions after TemporarilyLostDate.


### The code of these calculated columns is the following:


> *Calculated column in the Sales table*


> *Calculated column in the Sales table*

```dax
We then use these calculated columns to obtain two calculated
tables as an intermediate step to compute the CustomerEvents
```

snapshot table. If you want to leverage an external tool to only
compute the Temporarily Lost and Recovered events, you should
consider importing these two tables from the data source, where you
prepare their content by using dedicated tools for data preparation.
The two tables are visible in Figure 13-12 and Figure 13-13.


*📊 FIGURE 13-12 The TempLostDates table only contains the Temporarily Lost events.*

*📊 FIGURE 13-13 The RecoveredDates table only contains the Recovered events.*
*Using these intermediate tables, the CustomerEvents calculated*

table is obtained with a final UNION of the four states:


> *Calculated table*

```dax
Splitting the calculation into smaller steps is useful for educational
```

purposes and to provide a guide in case you want to implement part
of the calculation outside of the data model. However, if you
implement the calculation entirely in DAX then you can skip the
intermediate TempLostDates and RecoveredDates calculated tables. In
this case you must pay attention to the CALCULATE functions in
order to avoid circular dependencies, by implementing explicit filters
obtained by iterating the result of ALLNOBLANKROW. This results in
a more verbose definition of the CustomerEvents table, proposed here
under the name CustomerEventsSingleTable:


> *Calculated table*

Although     the    sample     file  includes    a   definition    of
CustomerEventsSingleTable, the measures in the report do not use that
table. If you want to use this approach, you can replace the definition
of CustomerEvents with the expression in CustomerEventsSingleTable
and remove the former expression from the model – you also want
to remove the TempLostDates and RecoveredDates calculated tables
that are no longer being used.
