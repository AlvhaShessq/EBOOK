# Chapter 6: Variables

Variables are important for at least two reasons: code readability and performance. In this chapter, we
provide detailed information about variables and their usage, whereas considerations about performance and readability are found all around the book. Indeed, we use variables in almost all the code
examples, sometimes showing the version with and without variables to let you appreciate how using
variables improves readability.
Later in Chapter 20, “Optimizing DAX,” we will also show how the use of variables can dramatically
improve the performance of your code. In this chapter, we are mainly interested in providing all the
useful information about variables in a single place.
Introducing VAR syntax
What introduces variables in an expression is fi rst the keyword VAR, which defi nes the variable,
followed by the RETURN part, which defi nes the result. You can see a typical expression containing a
variable in the following code:

```dax
VAR SalesAmt =
SUMX (
Sales,
Sales[Quantity] * Sales[Net Price]
)
RETURN
IF (
SalesAmt > 100000,
SalesAmt,
SalesAmt * 1.2
)
Adding more VAR defi nitions within the same block allows for the defi nition of multiple variables,
whereas the RETURN block needs to be unique. It is important to note that the VAR/RETURN block is,
```

indeed, an expression. As such, a variable defi nition makes sense wherever an expression can be used.
This makes it possible to defi ne variables during an iteration, or as part of more complex expressions,
like in the following example:

```dax
VAR SalesAmt =
SUMX (
Sales,
```

176 CHAPTER 6 Variables

```dax
VAR Quantity = Sales[Quantity]
VAR Price = Sales[Price]
RETURN
Quantity * Price
)
RETURN
...
```

Variables are commonly defi ned at the beginning of a measure defi nition and then used throughout the measure code. Nevertheless, this is only a writing habit. In complex expressions, defi ning local
variables deeply nested inside other function calls is common practice. In the previous code example,
the Quantity and Price variables are assigned for every row of the Sales table iterated by SUMX. These
variables are not available outside of the expression executed by SUMX for each row.
A variable can store either a scalar value or a table. The variables can be—and often are—of a different type than the expression returned after RETURN. Multiple variables in the same VAR/RETURN
block can be of different types too—scalar values or tables.
A very frequent usage of variables is to divide the calculation of a complex formula into logical
steps, by assigning the result of each step to a variable. For example, in the following code variables are
used to store partial results of the calculation:
Margin% :=

```dax
VAR SalesAmount =
SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
VAR TotalCost =
SUMX ( Sales, Sales[Quantity] * Sales[Unit Cost] )
VAR Margin =
SalesAmount - TotalCost
VAR MarginPerc =
DIVIDE ( Margin, TotalCost )
RETURN
MarginPerc
```

The same expression without variables takes a lot more attention to read:
Margin% :=

## Divide (


## Sumx (

Sales,
Sales[Quantity] * Sales[Net Price]

## ) - Sumx (

Sales,
Sales[Quantity] * Sales[Unit Cost]
),

## Sumx (

Sales,
Sales[Quantity] * Sales[Unit Cost]
)
)

CHAPTER 6 Variables 177
Moreover, the version with variables has the advantage that each variable is only evaluated once.
For example, TotalCost is used in two different parts of the code but, because it is defi ned as a variable,
DAX guarantees that its evaluation only happens once.
You can write any expression after RETURN. However, using a single variable for the RETURN part
is considered best practice. For example, in the previous code, it would be possible to remove the

```dax
MarginPerc variable defi nition by writing DIVIDE right after RETURN. However, using RETURN followed by a single variable (like in the example) allows for an easy change of the value returned by the
```

measure. This is useful when inspecting the value of intermediate steps. In our example, if the total is
not correct, it would be a good idea to check the value returned in each step, by using a report that
includes the measure. This means replacing MarginPerc with Margin, then with TotalCost, and then with
SalesAmount in the fi nal RETURN. You would execute the report each time to see the result produced
in the intermediate steps.
Understanding that variables are constant
Despite its name, a DAX variable is a constant. Once assigned a value, the variable cannot be modifi ed.
For example, if a variable is assigned within an iterator, it is created and assigned for every row iterated.
Moreover, the value of the variable is only available within the expression of the iterator it is defi ned in.
Amount at Current Price :=

## Sumx (

Sales,

```dax
VAR Quantity = Sales[Quantity]
VAR CurrentPrice = RELATED ( 'Product'[Unit Price] )
VAR AmountAtCurrentPrice = Quantity * CurrentPrice
RETURN
AmountAtCurrentPrice
)
```

-- Any reference to Quantity, CurrentPrice, or AmountAtCurrentPrice
-- would be invalid outside of SUMX
Variables are evaluated once in the scope of the defi nition (VAR) and not when their value is used.
For example, the following measure always returns 100% because the SalesAmount variable is not
affected by CALCULATE. Its value is only computed once. Any reference to the variable name returns
the same value regardless of the fi lter context where the variable value is used.
% of Product :=

```dax
VAR SalesAmount = SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
RETURN
DIVIDE (
SalesAmount,
CALCULATE (
SalesAmount,
ALL ( 'Product' )
)
)
```

178 CHAPTER 6 Variables
In this latter example, we used a variable where we should have used a measure. Indeed, if the goal
is to avoid the duplication of the code of SalesAmount in two parts of the expression, the right solution
requires using a measure instead of a variable to obtain the expected result. In the following code, the
correct percentage is obtained by defi ning two measures:
Sales Amount :=
SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
% of Product :=

## Divide (

[Sales Amount],

## Calculate (

[Sales Amount],
ALL ( 'Product' )
)
)
In this case the Sales Amount measure is evaluated twice, in two different fi lter contexts—leading as
expected to two different results.
Understanding the scope of variables
Each variable defi nition can reference the variables previously defi ned within the same VAR/RETURN
statement. All the variables already defi ned in outer VAR statements are also available.
A variable defi nition can access the variables defi ned in previous VAR statements, but not the
variables defi ned in following statements. Thus, this code works fi ne:
Margin :=

```dax
VAR SalesAmount =
SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
VAR TotalCost =
SUMX ( Sales, Sales[Quantity] * Sales[Unit Cost] )
VAR Margin = SalesAmount - TotalCost
RETURN
Margin
```

Whereas if one moves the defi nition of Margin at the beginning of the list, as in the following
example, DAX will not accept the syntax. Indeed, Margin references two variables that are not yet
defi ned—SalesAmount and TotalCost:
Margin :=

```dax
VAR Margin = SalesAmount - TotalCost -- Error: SalesAmount and TotalCost are not defined
VAR SalesAmount =
SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
VAR TotalCost =
SUMX ( Sales, Sales[Quantity] * Sales[Unit Cost] )
RETURN
Margin
```

CHAPTER 6 Variables 179
Because it is not possible to reference a variable before its defi nition, it is also impossible to create
either a circular dependency between variables, or any sort of recursive defi nition.
It is possible to nest VAR/RETURN statements inside each other, or to have multiple VAR/RETURN
blocks in the same expression. The scope of variables differs in the two scenarios. For example, in the
following measure the two variables LineAmount and LineCost are defi ned in two different scopes that
are not nested. Thus, at no point in the code can LineAmount and LineCost both be accessed within the
same expression:
Margin :=

## Sumx (

Sales,
(

```dax
VAR LineAmount = Sales[Quantity] * Sales[Net Price]
RETURN
LineAmount
) -- The parenthesis closes the scope of LineAmount
-- The LineAmount variable is not accessible from here on in
-
(
VAR LineCost = Sales[Quantity] * Sales[Unit Cost]
RETURN
LineCost
)
)
```

Clearly, this example is only for educational purposes. A better way of defi ning the two variables and
of using them is the following defi nition of Margin:
Margin :=

## Sumx (

Sales,

```dax
VAR LineAmount = Sales[Quantity] * Sales[Net Price]
VAR LineCost = Sales[Quantity] * Sales[Unit Cost]
RETURN
LineAmount - LineCost
)
```

As a further educational example, it is interesting to consider the real scope where a variable is
accessible when the parentheses are not used and an expression defi nes and reads several variables in
separate VAR/RETURN statements. For example, consider the following code:
Margin :=

## Sumx (

Sales,

```dax
VAR LineAmount = Sales[Quantity] * Sales[Net Price]
RETURN LineAmount
-
VAR LineCost = Sales[Quantity] * Sales[Unit Cost]
RETURN LineCost -- Here LineAmount is still accessible
)
```

180 CHAPTER 6 Variables
The entire expression after the fi rst RETURN is part of a single expression. Thus, the LineCost defi nition is nested within the LineAmount defi nition. Using the parentheses to delimit each RETURN
expression and indenting the code appropriately makes this concept more visible:
Margin :=

## Sumx (

Sales,

```dax
VAR LineAmount = Sales[Quantity] * Sales[Net Price]
RETURN (
LineAmount
- VAR LineCost = Sales[Quantity] * Sales[Unit Cost]
RETURN (
LineCost
-- Here LineAmount is still accessible
)
)
)
```

As shown in the previous example, because a variable can be defi ned for any expression, a variable
can also be defi ned within the expression assigned to another variable. In other words, it is possible to
defi ne nested variables. Consider the following example:
Amount at Current Price :=

## Sumx (

'Product',

```dax
VAR CurrentPrice = 'Product'[Unit Price]
RETURN -- CurrentPrice is available within the inner SUMX
SUMX (
RELATEDTABLE ( Sales ),
VAR Quantity = Sales[Quantity]
VAR AmountAtCurrentPrice = Quantity * CurrentPrice
RETURN
AmountAtCurrentPrice
)
-- Any reference to Quantity, or AmountAtCurrentPrice
-- would be invalid outside of the innermost SUMX
)
```

-- Any reference to CurrentPrice
-- would be invalid outside of the outermost SUMX
The rules pertaining to the scope of variables are the following:

> **Note:** A variable is available in the RETURN part of its VAR/RETURN block. It is also available in all the
variables defi ned after the variable itself, within that VAR/RETURN block. The VAR/RETURN
block replaces any DAX expression, and in such expression the variable can be read. In other
words, the variable is accessible from its declaration point until the end of the expression
following the RETURN statement that is part of the same VAR/RETURN block.

> **Note:** A variable is never available outside of its own VAR/RETURN block defi nition. After the expression following the RETURN statement, the variables declared within the VAR/RETURN block are
no longer visible. Referencing them generates a syntax error.

CHAPTER 6 Variables 181
Using table variables
A variable can store either a table or a scalar value. The type of the variable depends on its defi nition;
for instance, if the expression used to defi ne the variable is a table expression, then the variable contains a table. Consider the following code:
Amount :=

## If (

HASONEVALUE ( Slicer[Factor] ),
VAR
Factor = VALUES ( Slicer[Factor] )

## Return


## Divide (

[Sales Amount],
Factor
)
)
If Slicer[Factor] is a column with a single value in the current fi lter context, then it can be used as a
scalar expression. The Factor variable stores a table because it contains the result of VALUES, which is
a table function. If the user does not check for the presence of a single row with HASONEVALUE, the
variable assignment works fi ne; the line raising an error is the second parameter of DIVIDE, where the
variable is used, and conversion fails.
When a variable contains a table, it is likely because one wants to iterate on it. It is important to note
that, during such iteration, one should access the columns of a table variable by using their original
names. In other words, a variable name is not an alias of the underlying table in column references:
Filtered Amount :=
VAR

```dax
MultiSales = FILTER ( Sales, Sales[Quantity] > 1 )
RETURN
SUMX (
MultiSales,
-- MultiSales is not a table name for column references
-- Trying to access MultiSales[Quantity] would generate an error
Sales[Quantity] * Sales[Net Price]
)
```

Although SUMX iterates over MultiSales, you must use the Sales table name to access the Quantity
and Net Price columns. A column reference such as MultiSales[Quantity] is invalid.
One current DAX limitation is that a variable cannot have the same name as any table in the data
model. This prevents the possible confusion between a table reference and a variable reference.
Consider the following code:

## Sumx (

LargeSales,
Sales[Quantity] * Sales[NetPrice]
)

182 CHAPTER 6 Variables
A human reader immediately understands that LargeSales should be a variable because the column
references in the iterator reference another table name: Sales. However, DAX disambiguates at the language level through the distinctiveness of the name. A certain name can be either a table or a variable,
but not both at the same time.
Although this looks like a convenient limitation because it reduces confusion, it might be problematic in the long run. Indeed, whenever you defi ne the name of a variable, you should use a name that
will never be used as a table name in the future. Otherwise, if at some point you create a new table
whose name confl icts with variables used in any measure, you will obtain an error. Any syntax limitation
that requires you to predict what will happen in the future—like choosing the name of a table—is an
issue to say the least.
For this reason, when Power BI generates DAX queries, it uses variable names adopting a prefi x with
two underscores (__). The rationale is that a user is unlikely to use the same name in a data model.
Note This behavior could change in the future, thus enabling a variable name to override
the name of an existing table. When this change is implemented, there will no longer be a
risk of breaking an existing DAX expression by giving a new table the name of a variable.
When a variable name overrides a table name, the disambiguation will be possible by using
the single quote to delimit the table identifi er using the following syntax:
variableName
'tableName'
Should a developer design a DAX code generator to be injected in existing expressions,
they can use the single quote to disambiguate table identifi ers. This is not required in regular DAX code, if the code does not include ambiguous names between variables and tables.
Understanding lazy evaluation
As you have learned, DAX evaluates the variable within the evaluation context where it is defi ned, and
not where it is being used. Still, the evaluation of the variable itself is delayed until its fi rst use. This
technique is known as lazy evaluation. Lazy evaluation is important for performance reasons: a variable
that is never used in an expression will never be evaluated. Moreover, once a variable is computed for
the fi rst time, it will never be computed again in the same scope.
For example, consider the following code:
Sales Amount :=

```dax
VAR SalesAmount =
SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
VAR DummyError =
ERROR ( "This error will never be displayed" )
RETURN
SalesAmount
```

CHAPTER 6 Variables 183
The variable DummyError is never used, so its expression is never executed. Therefore, the error
never happens and the measure works correctly.
Obviously, nobody would ever write code like this. The goal of the example is to show that DAX does
not spend precious CPU time evaluating a variable if it is not useful to do so, and you can rely on this
behavior when writing code.
If a sub-expression is used multiple times in a complex expression, then creating a variable to store
its value is always a best practice. This guarantees that evaluation only happens once. Performancewise, this is more important than you might think. We will discuss this in more detail in Chapter 20, but
we cover the general idea here.
The DAX optimizer features a process called sub-formula detection. In a complex piece of code,
sub-formula detection checks for repeating sub-expressions that should only be computed once.
For example, look at the following code:
SalesAmount := SUMX ( Sales, Sales[Quantity] * Sales[Net Price] )
TotalCost := SUMX ( Sales, Sales[Quantity] * Sales[Unit Cost] )
Margin := [SalesAmount] – [TotalCost]
Margin% := DIVIDE ( [Margin], [TotalCost] )
The TotalCost measure is called twice—once in Margin and once in Margin%. Depending on the
quality of the optimizer, it might be able to detect that both measure calls refer to the same value, so it
might be able to compute TotalCost only once. Nevertheless, the optimizer is not always able to detect
that a sub-formula exists and that it can be evaluated only once. As a human, and being the author of
your own code, you always have a much better understanding of when part of the code can be used in
multiple parts of your formula.
If you get used to using variables whenever you can, defi ning sub-formulas as variables will come
naturally. When you use their value multiple times, you will greatly help the optimizer in fi nding the
best execution path for your code.
Common patterns using variables
In this section, you fi nd practical uses of variables. It is not an exhaustive list of scenarios where variables become useful, and although there are many other situations where a variable would be a good
fi t, these are relevant and frequent uses.
The fi rst and most relevant reason to use variables is to provide documentation in your code.
A good example is when you need to use complex fi lters in a CALCULATE function. Using variables
as CALCULATE fi lters only improves readability. It does not change semantics or performance. Filters
would be executed outside of the context transition triggered by CALCULATE in any case, and DAX also

184 CHAPTER 6 Variables
uses lazy evaluation for fi lter contexts. Nevertheless, improving readability is an important task for any
DAX developer. For example, consider the following measure defi nition:
Sales Large Customers :=

```dax
VAR LargeCustomers =
FILTER (
Customer,
[Sales Amount] > 10000
)
VAR WorkingDaysIn2008 =
CALCULATETABLE (
ALL ( 'Date'[IsWorkingDay], 'Date'[Calendar Year] ),
'Date'[IsWorkingDay] = TRUE (),
'Date'[Calendar Year] = "CY 2008"
)
RETURN
CALCULATE (
[Sales Amount],
LargeCustomers,
WorkingDaysIn2008
)
```

Using the two variables for the fi ltered customers and the fi ltered dates splits the full execution fl ow
into three distinct parts: the defi nition of what a large customer is, the defi nition of the period one
wants to consider, and the actual calculation of the measure with the two fi lters applied.
Although it might look like we are only talking about style, you should never forget that a more
elegant and simple formula is more likely to also be an accurate formula. Writing a simpler formula, the
author is more likely to have understood the code and fi xed any possible fl aws. Whenever an expression takes more than 10 lines of code, it is time to split its execution path with multiple variables.
This allows the author to focus on smaller fragments of the full formula.
Another scenario where variables are important is when nesting multiple row contexts on the same
table. In this scenario, variables let you save data from hidden row contexts and avoid the use of the
EARLIER function:
'Product'[RankPrice] =

```dax
VAR CurrentProductPrice = 'Product'[Unit Price]
VAR MoreExpensiveProducts =
FILTER (
'Product',
'Product'[Unit Price] > CurrentProductPrice
)
RETURN
COUNTROWS ( MoreExpensiveProducts ) + 1
```

Filter contexts can be nested too. Nesting multiple fi lter contexts does not create syntax problems
as it does with multiple row contexts. One frequent scenario with nested fi lter contexts is needing to
save the result of a calculation to use it later in the code when the fi lter context changes.

CHAPTER 6 Variables 185
For example, if one needs to search for the customers who bought more than the average customer,
this code is not going to work:
AverageSalesPerCustomer :=
AVERAGEX ( Customer, [Sales Amount] )
CustomersBuyingMoreThanAverage :=

## Countrows (


## Filter (

Customer,
[Sales Amount] > [AverageSalesPerCustomer]
)
)
The reason is that the AverageSalesPerCustomer measure is evaluated inside an iteration over
Customer. As such, there is a hidden CALCULATE around the measure that performs a context transition. Thus, AverageSalesPerCustomer evaluates the sales of the current customer inside the iteration
every time, instead of the average over all the customers in the fi lter context. There is no customer
whose sales amount is strictly greater than the sales amount itself. The measure always returns blank.
To obtain the correct behavior, one needs to evaluate AverageSalesPerCustomer outside of the
iteration. A variable fi ts this requirement perfectly:
AverageSalesPerCustomer :=
AVERAGEX ( Customer, [Sales Amount] )
CustomersBuyingMoreThanAverage :=

```dax
VAR AverageSales = [AverageSalesPerCustomer]
RETURN
COUNTROWS (
FILTER (
Customer,
[Sales Amount] > AverageSales
)
)
```

In this example DAX evaluates the variable outside of the iteration, computing the correct average
sales for all the selected customers. Moreover, the optimizer knows that the variable can (and must) be
evaluated only once, outside of the iteration. Thus, the code is likely to be faster than any other possible
implementation.
Conclusions
Variables are useful for multiple reasons: readability, performance, and elegance of the code. Whenever you need to write a complex formula, split it into multiple variables. You will appreciate having
done so the next time you review your code.

186 CHAPTER 6 Variables
It is true that expressions using variables tend to be longer than the same expressions without
variables. A longer expression is not a bad thing if it means that each part is easier to understand.
Unfortunately, in several tools the user interface to author DAX code makes it hard to write expressions over 10 lines long. You might think that a shorter formulation of the same code without variables
is preferable because it is easier to author in a specifi c tool—for example Power BI. That is incorrect.
We certainly need better tools to author longer DAX code that includes comments and many
variables. These tools will come eventually. In the meantime, rather than authoring shorter and
confusing code directly into a small text box, it is wiser to use external tools like DAX Studio to author
longer DAX code. You would then copy and paste the resulting code into Power BI or Visual Studio.