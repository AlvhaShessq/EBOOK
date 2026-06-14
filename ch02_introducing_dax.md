# Chapter 2: Introducing DAX

In this chapter, we start talking about the DAX language. Here you learn the syntax of the language, the
difference between a calculated column and a measure (also called calculated fi eld, in certain old Excel
versions), and the most commonly used functions in DAX.
Because this is an introductory chapter, it does not cover many functions in depth. In later chapters,
we explain them in more detail. For now, introducing the functions and starting to look at the DAX language in general are enough. When we reference features of the data model in Power BI, Power Pivot,
or Analysis Services, we use the term Tabular even when the feature is not present in all the products.
For example, “DirectQuery in Tabular” refers to the DirectQuery mode feature available in Power BI and
Analysis Services but not in Excel.
Understanding DAX calculations
Before working on more complex formulas, you need to learn the basics of DAX. This includes DAX
syntax, the different data types that DAX can handle, the basic operators, and how to refer to columns
and tables. These concepts are discussed in the next few sections.
We use DAX to compute values over columns in tables. We can aggregate, calculate, and search for
numbers, but in the end, all the calculations involve tables and columns. Thus, the fi rst syntax to learn is
how to reference a column in a table.
The general format is to write the table name enclosed in single quotation marks, followed by the
column name enclosed in square brackets, as follows:
'Sales'[Quantity]
We can omit the single quotation marks if the table name does not start with a number, does not
contain spaces, and is not a reserved word (like Date or Sum).
The table name is also optional in case we are referencing a column or a measure within the table
where we defi ne the formula. Thus, [Quantity] is a valid column reference, if written in a calculated
column or in a measure defi ned in the Sales table. Although this option is available, we strongly
discourage you from omitting the table name. At this point, we do not explain why this is so important, but the reason will become clear when you read Chapter 5, “Understanding CALCULATE and
CALCULATETABLE.” Nevertheless, it is of paramount importance to be able to distinguish between

18 CHAPTER 2 Introducing DAX
measures (discussed later) and columns when you read DAX code. The de facto standard is to always
use the table name in column references and always avoid it in measure references. The earlier you
start adopting this standard, the easier your life with DAX will be. Therefore, you should get used to
this way of referencing columns and measures:
Sales[Quantity] * 2 -- This is a column reference
[Sales Amount] * 2 -- This is a measure reference
You will learn the rationale behind this standard after learning about context transition, which
comes in Chapter 5. For now, just trust us and adhere to this standard.

Comments in DAX
The preceding code example shows comments in DAX for the fi rst time. DAX supports
single-line comments and multiline comments. Single-line comments start with either --
or //, and the remaining part of the line is considered a comment.
= Sales[Quantity] * Sales[Net Price] -- Single-line comment
= Sales[Quantity] * Sales[Unit Cost] // Another example of single-line comment
A multiline comment starts with /* and ends with */. The DAX parser ignores
everything included between these markers and considers them a comment.

## = If (

Sales[Quantity] > 1,
/* First example of a multiline comment
Anything can be written here and is ignored by DAX
*/
"Multi",
/* A common use case of multiline comments is to comment-out a part of
the existing code
The next IF statement is ignored because it falls within a multiline comment

## If (

Sales[Quantity] = 1,
"Single",
"Special note"
)
*/
"Single"
)
It is better to avoid comments at the end of a DAX expression in a measure, calculated
column, or calculated table defi nition. These comments might be not visible at fi rst, and they
might not be supported by tools such as DAX Formatter, which is discussed later in this
chapter.

CHAPTER 2 Introducing DAX 19
DAX data types
DAX can perform computations with different numeric types, of which there are seven. Over time,
Microsoft introduced different names for the same data types, creating some sort of confusion.
Table 2-1 provides the different names under which you might fi nd each DAX data type.
TABLE 2-1 Data Types
DAX Data Type
Power BI
Data Type
Power Pivot and
Analysis Services
Data Type
Correspondent
Conventional Data Type
(e.g., SQL Server)
Tabular Object
Model (TOM)
Data Type
Integer Whole Number Whole Number Integer / INT int64
Decimal Decimal Number Decimal Number Floating point / DOUBLE double
Currency Fixed Decimal
Number
Currency Currency / MONEY decimal
DateTime DateTime, Date,
Time
Date Date / DATETIME dateTime
Boolean True/False True/False Boolean / BIT boolean
String Text Text String / NVARCHAR(MAX) string
Variant - - - variant
Binary Binary Binary Blob / VARBINARY(MAX) binary
In this book, we use the names in the fi rst column of Table 2-1 adhering to the de facto standards
in the database and Business Intelligence community. For example, in Power BI, a column containing
either TRUE or FALSE would be called TRUE/FALSE, whereas in SQL Server, it would be called a BIT.
Nevertheless, the historical and most common name for this type of value is Boolean.
DAX comes with a powerful type-handling system so that we do not have to worry about data
types. In a DAX expression, the resulting type is based on the type of the term used in the expression.
You need to be aware of this in case the type returned from a DAX expression is not the expected type;
you would then have to investigate the data type of the terms used in the expression itself.
For example, if one of the terms of a sum is a date, the result also is a date; likewise, if the same
operator is used with integers, the result is an integer. This behavior is known as operator overloading,
and an example is shown in Figure 2-1, where the OrderDatePlusOneWeek column is calculated by
adding 7 to the value of the Order Date column.
Sales[OrderDatePlusOneWeek] = Sales[Order Date] + 7
The result is a date.

20 CHAPTER 2 Introducing DAX
FIGURE 2-1 Adding an integer to a date results in a date increased by the corresponding number of days.
In addition to operator overloading, DAX automatically converts strings into numbers and
numbers into strings whenever required by the operator. For example, if we use the & operator,
which concatenates strings, DAX converts its arguments into strings. The following formula returns
“54” as a string:

## = 5 & 4

On the other hand, this formula returns an integer result with the value of 9:

## = "5" + "4"

The resulting value depends on the operator and not on the source columns, which are
converted following the requirements of the operator. Although this behavior looks convenient,
later in this chapter you see what kinds of errors might happen during these automatic
conversions. Moreover, not all the operators follow this behavior. For example, comparison
operators cannot compare strings with numbers. Consequently, you can add one number with a
string, but you cannot compare a number with a string. You can fi nd a complete reference here:
https://docs.microsoft.com/en-us/power-bi/desktop-data-types. Because the rules are so
complex, we suggest you avoid automatic conversions altogether. If a conversion needs to
happen, we recommend that you control it and make the conversion explicit. To be more explicit,
the previous example should be written like this:

## = Value ( "5" ) + Value ( "4" )

People accustomed to working with Excel or other languages might be familiar with DAX data types.
Some details about data types depend on the engine, and they might be different for Power BI, Power

CHAPTER 2 Introducing DAX 21
Pivot, or Analysis Services. You can fi nd more detailed information about Analysis Services DAX data
types at http://msdn.microsoft.com/en-us/library/gg492146.aspx, and Power BI information is available
at https://docs.microsoft.com/en-us/power-bi/desktop-data-types. However, it is useful to share a few
considerations about each of these data types.
Integer
DAX has only one Integer data type that can store a 64-bit value. All the internal calculations between
integer values in DAX also use a 64-bit value.
Decimal
A Decimal number is always stored as a double-precision fl oating-point value. Do not confuse this DAX
data type with the decimal and numeric data type of Transact-SQL. The corresponding data type of a
DAX decimal number in SQL is Float.
Currency
The Currency data type, also known as Fixed Decimal Number in Power BI, stores a fi xed decimal
number. It can represent four decimal points and is internally stored as a 64-bit integer value divided
by 10,000. Summing or subtracting Currency data types always ignores decimals beyond the fourth
decimal point, whereas multiplication and division produce a fl oating-point value, thus increasing the
precision of the result. In general, if we need more accuracy than the four digits provided, we must use
a Decimal data type.
The default format of the Currency data type includes the currency symbol. We can also apply the
currency formatting to Integer and decimal numbers, and we can use a format without the currency
symbol for a Currency data type.
DateTime
DAX stores dates as a DateTime data type. This format uses a fl oating-point number internally, wherein
the integer corresponds to the number of days since December 30, 1899, and the decimal part identifi es the fraction of the day. Hours, minutes, and seconds are converted to decimal fractions of a day.
Thus, the following expression returns the current date plus one day (exactly 24 hours):

## = Today () + 1

The result is tomorrow’s date at the time of the evaluation. If you need to take only the date part of a
DateTime, always remember to use TRUNC to get rid of the decimal part.
Power BI offers two additional data types: Date and Time. Internally, they are a simple variation of DateTime. Indeed, Date and Time store only the integer or the decimal part of the DateTime,
respectively.

22 CHAPTER 2 Introducing DAX

The leap year bug
Lotus 1-2-3, a popular spreadsheet released in 1983, presented a bug in the handling of
the DateTime data type. It considered 1900 as a leap year, even though it was not. The
fi nal year in a century is a leap year only if the fi rst two digits can be divided by 4 without
a remainder. At that time, the development team of the fi rst version of Excel deliberately
replicated the bug, to maintain compatibility with Lotus 1-2-3. Since then, each new
version of Excel has maintained the bug for compatibility.
At the time of printing in 2019, the bug is still there in DAX, introduced for backward compatibility with Excel. The presence of the bug (should we call it a feature?) might lead to errors
on time periods prior to March 1, 1900. Thus, by design, the fi rst date offi cially supported by
DAX is March 1, 1900. Date calculations executed on time periods prior to that date might lead
to errors and should be considered as inaccurate.
Boolean
The Boolean data type is used to express logical conditions. For example, a calculated column defi ned
by the following expression is of Boolean type:
= Sales[Unit Price] > Sales[Unit Cost]
You will also see Boolean data types as numbers where TRUE equals 1 and FALSE equals 0. This
notation sometimes proves useful for sorting purposes because TRUE > FALSE.
String
Every string in DAX is stored as a Unicode string, where each character is stored in 16 bits. By default,
the comparison between strings is not case sensitive, so the two strings “Power BI” and “POWER BI” are
considered equal.
Variant
The Variant data type is used for expressions that might return different data types, depending on the
conditions. For example, the following statement can return either an integer or a string, so it returns a
variant type:
IF ( [measure] > 0, 1, "N/A" )
The Variant data type cannot be used as a data type for a column in a regular table. A DAX measure,
and in general, a DAX expression can be Variant.

CHAPTER 2 Introducing DAX 23
Binary
The Binary data type is used in the data model to store images or other nonstructured types of
information. It is not available in DAX. It was mainly used by Power View, but it might not be available
in other tools such as Power BI.
DAX operators
Now that you have seen the importance of operators in determining the type of an expression, see
Table 2-2, which provides a list of the operators available in DAX.
TABLE 2-2 Operators
Operator Type Symbol Use Example
Parenthesis ( ) Precedence order and grouping
of arguments

## (5 + 2) * 3

Arithmetic +
−
*
/
Addition
Subtraction/negation
Multiplication
Division
4 + 2
5 − 3
4 * 2
4 / 2
Comparison =
<>
>
>=
<
<=
Equal to
Not equal to
Greater than
Greater than or equal to
Less than
Less than or equal to
[CountryRegion] = "USA"
[CountryRegion] <> "USA"
[Quantity] > 0
[Quantity] >= 100
[Quantity] < 0
[Quantity] <= 100
Text concatenation & Concatenation of strings "Value is" & [Amount]
Logical &&
||
IN
NOT
AND condition between two
Boolean expressions
OR condition between two Boolean expressions
Inclusion of an element in a list
Boolean negation
[CountryRegion] = "USA" && [Quantity]>0
[CountryRegion] = "USA" || [Quantity] > 0
[CountryRegion] IN {"USA", "Canada"}
NOT [Quantity] > 0
Moreover, the logical operators are also available as DAX functions, with a syntax similar to Excel’s.
For example, we can write expressions like these:
AND ( [CountryRegion] = "USA", [Quantity] > 0 )
OR ( [CountryRegion] = "USA", [Quantity] > 0 )
These examples are equivalent, respectively, to the following:
[CountryRegion] = "USA" && [Quantity] > 0
[CountryRegion] = "USA" || [Quantity] > 0
Using functions instead of operators for Boolean logic becomes helpful when writing complex
conditions. In fact, when it comes to formatting large sections of code, functions are much easier to
format and to read than operators are. However, a major drawback of functions is that we can pass in
only two parameters at a time. Therefore, we must nest functions if we have more than two conditions
to evaluate.

24 CHAPTER 2 Introducing DAX
Table constructors
In DAX we can defi ne anonymous tables directly in the code. If the table has a single column, the syntax
requires only a list of values—one for each row—delimited by curly braces. We can delimit multiple
rows by parentheses, which are optional if the table is made of a single column. The two following
defi nitions, for example, are equivalent:
{ "Red", "Blue", "White" }
{ ( "Red" ), ( "Blue" ), ( "White" ) }
If the table has multiple columns, parentheses are mandatory. Every column should have the same
data type throughout all its rows; otherwise, DAX will automatically convert the column to a data type
that can accommodate all the data types provided in different rows for the same column.
{

## ( "A", 10, 1.5, Date ( 2017, 1, 1 ), Currency ( 199.99 ), True ),


## ( "B", 20, 2.5, Date ( 2017, 1, 2 ), Currency ( 249.99 ), False ),


## ( "C", 30, 3.5, Date ( 2017, 1, 3 ), Currency ( 299.99 ), False )

}
The table constructor is commonly used with the IN operator. For example, the following are
possible, valid syntaxes in a DAX predicate:
'Product'[Color] IN { "Red", "Blue", "White" }
( 'Date'[Year], 'Date'[MonthNumber] ) IN { ( 2017, 12 ), ( 2018, 1 ) }
This second example shows the syntax required to compare a set of columns (tuple) using the IN
operator. Such syntax cannot be used with a comparison operator. In other words, the following syntax
is not valid:
( 'Date'[Year], 'Date'[MonthNumber] ) = ( 2007, 12 )
However, we can rewrite it using the IN operator with a table constructor that has a single row, as in
the following example:
( 'Date'[Year], 'Date'[MonthNumber] ) IN { ( 2007, 12 ) }
Conditional statements
In DAX we can write a conditional expression using the IF function. For example, we can write an
expression returning MULTI or SINGLE depending on the quantity value being greater than one or not,
respectively.

## If (

Sales[Quantity] > 1,

## "Multi",


## "Single"

)

CHAPTER 2 Introducing DAX 25
The IF function has three parameters, but only the fi rst two are mandatory. The third is optional, and
it defaults to BLANK. Consider the following code:

## If (

Sales[Quantity] > 1,
Sales[Quantity]
)
It corresponds to the following explicit version:

## If (

Sales[Quantity] > 1,
Sales[Quantity],

## Blank ()

)
Understanding calculated columns and measures
Now that you know the basics of DAX syntax, you need to learn one of the most important concepts in
DAX: the difference between calculated columns and measures. Even though calculated columns and
measures might appear similar at fi rst sight because you can make certain calculations using either,
they are, in reality, different. Understanding the difference is key to unlocking the power of DAX.
Calculated columns
Depending on the tool you are using, you can create a calculated column in different ways. Indeed, the
concept remains the same: a calculated column is a new column added to your model, but instead of
being loaded from a data source, it is created by resorting to a DAX formula.
A calculated column is just like any other column in a table, and we can use it in rows, columns,
fi lters, or values of a matrix or any other report. We can also use a calculated column to defi ne a relationship, if needed. The DAX expression defi ned for a calculated column operates in the context of the
current row of the table that the calculated column belongs to. Any reference to a column returns the
value of that column for the current row. We cannot directly access the values of other rows.
If you are using the default Import Mode of Tabular and are not using DirectQuery, one important
concept to remember about calculated columns is that these columns are computed during database
processing and then stored in the model. This concept might seem strange if you are accustomed to
SQL-computed columns (not persisted), which are evaluated at query time and do not use memory.
In Tabular, however, all calculated columns occupy space in memory and are computed during table
processing.
This behavior is helpful whenever we create complex calculated columns. The time required to
compute complex calculated columns is always process time and not query time, resulting in a better
user experience. Nevertheless, be mindful that a calculated column uses precious RAM. For example,

26 CHAPTER 2 Introducing DAX
if we have a complex formula for a calculated column, we might be tempted to separate the steps of
computation into different intermediate columns. Although this technique is useful during project
development, it is a bad habit in production because each intermediate calculation is stored in RAM
and wastes valuable space.
If a model is based on DirectQuery instead, the behavior is hugely different. In DirectQuery
mode, calculated columns are computed on the fl y when the Tabular engine queries the data source.
This might result in heavy queries executed by the data source, therefore producing slow models.

Computing the duration of an order
Imagine we have a Sales table containing both the order and the delivery dates. Using these
two columns, we can compute the number of days involved in delivering the order. Because
dates are stored as number of days after 12/30/1899, a simple subtraction computes the
difference in days between two dates:
Sales[DaysToDeliver] = Sales[Delivery Date] - Sales[Order Date]
Nevertheless, because the two columns used for subtraction are dates, the result also
is a date. To produce a numeric result, convert the result to an integer this way:
Sales[DaysToDeliver] = INT ( Sales[Delivery Date] - Sales[Order Date] )
The result is shown in Figure 2-2.
FIGURE 2-2 By subtracting two dates and converting the result to an integer, DAX computes the number of
days between the two dates.
Measures
Calculated columns are useful, but you can defi ne calculations in a DAX model in another way.
Whenever you do not want to compute values for each row but rather want to aggregate values
from many rows in a table, you will fi nd these calculations useful; they are called measures.

CHAPTER 2 Introducing DAX 27
For example, you can defi ne a few calculated columns in the Sales table to compute the gross
margin amount:
Sales[SalesAmount] = Sales[Quantity] * Sales[Net Price]
Sales[TotalCost] = Sales[Quantity] * Sales[Unit Cost]
Sales[GrossMargin] = Sales[SalesAmount] – Sales[TotalCost]
What happens if you want to show the gross margin as a percentage of the sales amount? You could
create a calculated column with the following formula:
Sales[GrossMarginPct] = Sales[GrossMargin] / Sales[SalesAmount]
This formula computes the correct value at the row level—as you can see in Figure 2-3—but at the
grand total level the result is clearly wrong.
FIGURE 2-3 The GrossMarginPct column shows a correct value on each row, but the grand total is incorrect.
The value shown at the grand total level is the sum of the individual percentages computed row by
row within the calculated column. When we compute the aggregate value of a percentage, we cannot
rely on calculated columns. Instead, we need to compute the percentage based on the sum of individual columns. We must compute the aggregated value as the sum of gross margin divided by the
sum of sales amount. In this case, we need to compute the ratio on the aggregates; you cannot use an
aggregation of calculated columns. In other words, we compute the ratio of the sums, not the sum of
the ratios.
It would be equally wrong to simply change the aggregation of the GrossMarginPct column to an
average and rely on the result because doing so would provide an incorrect evaluation of the percentage, not considering the differences between amounts. The result of this averaged value is visible in
Figure 2-4, and you can easily check that (330.31 / 732.23) is not equal to the value displayed, 45.96%;
it should be 45.11% instead.

28 CHAPTER 2 Introducing DAX
FIGURE 2-4 Changing the aggregation method to AVERAGE does not provide the correct result.
The correct implementation for GrossMarginPct is with a measure:
GrossMarginPct := SUM ( Sales[GrossMargin] ) / SUM (Sales[SalesAmount] )
As we have already stated, the correct result cannot be achieved with a calculated column. If you
need to operate on aggregated values instead of operating on a row-by-row basis, you must create
measures. You might have noticed that we used := to defi ne a measure instead of the equal sign (=).
This is a standard we used throughout the book to make it easier to differentiate between measures
and calculated columns in code.
After you defi ne GrossMarginPct as a measure, the result is correct, as you can see in Figure 2-5.
FIGURE 2-5 GrossMarginPct defi ned as a measure shows the correct grand total.

CHAPTER 2 Introducing DAX 29
Measures and calculated columns both use DAX expressions; the difference is the context of
evaluation. A measure is evaluated in the context of a visual element or in the context of a DAX query.
However, a calculated column is computed at the row level of the table it belongs to. The context of the
visual element (later in the book, you will learn that this is a fi lter context) depends on user selections
in the report or on the format of the DAX query. Therefore, when using SUM(Sales[SalesAmount]) in a
measure, we mean the sum of all the rows that are aggregated under a visualization. However, when
we use Sales[SalesAmount] in a calculated column, we mean the value of the SalesAmount column in
the current row.
A measure needs to be defi ned in a table. This is one of the requirements of the DAX language.
However, the measure does not really belong to the table. Indeed, we can move a measure from one
table to another table without losing its functionality.

Differences between calculated columns and measures
Although they look similar, there is a big difference between calculated columns and
measures. The value of a calculated column is computed during data refresh, and it uses
the current row as a context. The result does not depend on user activity on the report.
A measure operates on aggregations of data defi ned by the current context. In a matrix
or in a pivot table, for example, source tables are fi ltered according to the coordinates of
cells, and data is aggregated and calculated using these fi lters. In other words, a measure
always operates on aggregations of data under the evaluation context. The evaluation
context is explained further in Chapter 4, “Understanding evaluation contexts.”
Choosing between calculated columns and measures
Now that you have seen the difference between calculated columns and measures, it is useful to
discuss when to use one over the other. Sometimes either is an option, but in most situations, the
computation requirements determine the choice.
As a developer, you must defi ne a calculated column whenever you want to do the following:

> **Note:** Place the calculated results in a slicer or see results in rows or columns in a matrix or in a pivot
table (as opposed to the Values area), or use the calculated column as a fi lter condition in a DAX
query.

> **Note:** Defi ne an expression that is strictly bound to the current row. For example, Price * Quantity
cannot work on an average or on a sum of those two columns.

> **Note:** Categorize text or numbers. For example, a range of values for a measure, a range of ages of
customers, such as 0–18, 18–25, and so on. These categories are often used as fi lters or to slice
and dice values.

30 CHAPTER 2 Introducing DAX
However, it is mandatory to defi ne a measure whenever one wants to display calculation values that
refl ect user selections, and the values need to be presented as aggregates in a report, for example:

> **Note:** To calculate the profi t percentage of a report selection

> **Note:** To calculate ratios of a product compared to all products but keep the fi lter both by year and by
region
We can express many calculations both with calculated columns and with measures, although we
need to use different DAX expressions for each. For example, one can defi ne the GrossMargin as a
calculated column:
Sales[GrossMargin] = Sales[SalesAmount] - Sales[TotalProductCost]
However, it can also be defi ned as a measure:
GrossMargin := SUM ( Sales[SalesAmount] ) - SUM ( Sales[TotalProductCost] )
We suggest you use a measure in this case because, being evaluated at query time, it does not consume memory and disk space. As a rule, whenever you can express a calculation both ways, measures
are the preferred way to go. You should limit the use of calculated columns to the few cases where
they are strictly needed. Users with Excel experience typically prefer calculated columns over measures
because calculated columns closely resemble the way of performing calculations in Excel. Nevertheless,
the best way to compute a value in DAX is through a measure.
Using measures in calculated columns
It is obvious that a measure can refer to one or more calculated columns. Although less
intuitive, the opposite is also true. A calculated column can refer to a measure. This way, the
calculated column forces the calculation of a measure for the context defi ned by the current
row. This operation transforms and consolidates the result of a measure into a column, which
will not be infl uenced by user actions. Obviously, only certain operations can produce meaningful results because a measure usually makes computations that strongly depend on the
selection made by the user in the visualization. Moreover, whenever you, as the developer,
use measures in a calculated column, you rely on a feature called context transition, which is
an advanced calculation technique in DAX. Before you use a measure in a calculated column,
we strongly suggest you read and understand Chapter 4, which explains in detail evaluation
contexts and context transitions.
Introducing variables
When writing a DAX expression, one can avoid repeating the same expression and greatly enhance the
code readability by using variables. For example, look at the following expression:

```dax
VAR TotalSales = SUM ( Sales[SalesAmount] )
VAR TotalCosts = SUM ( Sales[TotalProductCost] )
```

CHAPTER 2 Introducing DAX 31

```dax
VAR GrossMargin = TotalSales - TotalCosts
RETURN
GrossMargin / TotalSales
Variables are defi ned with the VAR keyword. After you defi ne a variable, you need to provide a
RETURN section that defi nes the result value of the expression. One can defi ne many variables, and the
```

variables are local to the expression in which they are defi ned.
A variable defi ned in an expression cannot be used outside the expression itself. There is no such
thing as a global variable defi nition. This means that you cannot defi ne variables used through the
whole DAX code of the model.
Variables are computed using lazy evaluation. This means that if one defi nes a variable that, for any
reason, is not used in the code, the variable itself will never be evaluated. If it needs to be computed,
this happens only once. Later uses of the variable will read the value computed previously. Thus, variables are also useful as an optimization technique when used in a complex expression multiple times.
Variables are an important tool in DAX. As you will learn in Chapter 4, variables are extremely useful
because they use the defi nition evaluation context instead of the context where the variable is used.
In Chapter 6, “Variables,” we will fully cover variables and how to use them. We will also use variables
extensively throughout the book.
Handling errors in DAX expressions
Now that you have seen some of the basics of the syntax, it is time to learn how to handle invalid calculations gracefully. A DAX expression might contain invalid calculations because the data it references
is not valid for the formula. For example, the formula might contain a division by zero or reference a
column value that is not a number while being used in an arithmetic operation such as multiplication.
It is good to learn how these errors are handled by default and how to intercept these conditions for
special handling.
Before discussing how to handle errors, though, we describe the different kinds of errors that might
appear during a DAX formula evaluation. They are

> **Note:** Conversion errors

> **Note:** Arithmetic operations errors

> **Note:** Empty or missing values
Conversion errors
The fi rst kind of error is the conversion error. As we showed previously in this chapter, DAX automatically converts values between strings and numbers whenever the operator requires it. All these
examples are valid DAX expressions:

## "10" + 32 = 42


## "10" & 32 = "1032"


32 CHAPTER 2 Introducing DAX
10 & 32 = "1032"

## Date (2010,3,25) = 3/25/2010


## Date (2010,3,25) + 14 = 4/8/2010


## Date (2010,3,25) & 14 = "3/25/201014"

These formulas are always correct because they operate with constant values. However, what about
the following formula if VatCode is a string?
Sales[VatCode] + 100
Because the fi rst operand of this sum is a column that is of Text data type, you as a developer must be
confi dent that DAX can convert all the values in that column into numbers. If DAX fails in converting some
of the content to suit the operator needs, a conversion error will occur. Here are some typical situations:
"1 + 1" + 0 = Cannot convert value '1 + 1' of type Text to type Number
DATEVALUE ("25/14/2010") = Type mismatch
If you want to avoid these errors, it is important to add error detection logic in DAX expressions
to intercept error conditions and return a result that makes sense. One can obtain the same result by
intercepting the error after it has happened or by checking the operands for the error situation beforehand. Nevertheless, checking for the error situation proactively is better than letting the error happen
and then catching it.
Arithmetic operations errors
The second category of errors is arithmetic operations, such as the division by zero or the square root
of a negative number. These are not conversion-related errors: DAX raises them whenever we try to call
a function or use an operator with invalid values.
The division by zero requires special handling because its behavior is not intuitive (except, maybe,
for mathematicians). When one divides a number by zero, DAX returns the special value Infi nity. In
the special cases of 0 divided by 0 or Infi nity divided by Infi nity, DAX returns the special NaN (not a
number) value.
Because this is unusual behavior, it is summarized in Table 2-3.
TABLE 2-3 Special Result Values for Division by Zero
Expression Result
10 / 0 Infi nity
7 / 0 Infi nity
0 / 0 NaN
(10 / 0) / (7 / 0) NaN
It is important to note that Infi nity and NaN are not errors but special values in DAX. In fact, if one
divides a number by Infi nity, the expression does not generate an error. Instead, it returns 0:
9954 / ( 7 / 0 ) = 0

CHAPTER 2 Introducing DAX 33
Apart from this special situation, DAX can return arithmetic errors when calling a function with an
incorrect parameter, such as the square root of a negative number:
SQRT ( -1 ) = An argument of function 'SQRT' has the wrong data type or the result is too
large or too small
If DAX detects errors like this, it blocks any further computation of the expression and raises an
error. One can use the ISERROR function to check if an expression leads to an error. We show this
scenario later in this chapter.
Keep in mind that special values like NaN are displayed in the user interface of several tools such as
Power BI as regular values. They can, however, be treated as errors when shown by other client tools
such as an Excel pivot table. Finally, these special values are detected as errors by the error detection
functions.
Empty or missing values
The third category that we examine is not a specifi c error condition but rather the presence of empty
values. Empty values might result in unexpected results or calculation errors when combined with other
elements in a calculation.
DAX handles missing values, blank values, or empty cells in the same way, using the value BLANK.
BLANK is not a real value but instead is a special way to identify these conditions. We can obtain the
value BLANK in a DAX expression by calling the BLANK function, which is different from an empty
string. For example, the following expression always returns a blank value, which can be displayed as
either an empty string or as “(blank)” in different client tools:

## = Blank ()

On its own, this expression is useless, but the BLANK function itself becomes useful every time there
is the need to return an empty value. For example, one might want to display an empty result instead
of 0. The following expression calculates the total discount for a sale transaction, leaving the blank
value if the discount is 0:

## =If (

Sales[DiscountPerc] = 0, -- Check if there is a discount
BLANK (), -- Return a blank if no discount is present
Sales[DiscountPerc] * Sales[Amount] -- Compute the discount otherwise
)
BLANK, by itself, is not an error; it is just an empty value. Therefore, an expression containing a
BLANK might return a value or a blank, depending on the calculation required. For example, the
following expression returns BLANK whenever Sales[Amount] is BLANK:
= 10 * Sales[Amount]

34 CHAPTER 2 Introducing DAX
In other words, the result of an arithmetic product is BLANK whenever one or both terms are
BLANK. This creates a challenge when it is necessary to check for a blank value. Because of the implicit
conversions, it is impossible to distinguish whether an expression is 0 (or empty string) or BLANK using
an equal operator. Indeed, the following logical conditions are always true:
BLANK () = 0 -- Always returns TRUE
BLANK () = "" -- Always returns TRUE
Therefore, if the columns Sales[DiscountPerc] or Sales[Clerk] are blank, the following conditions
return TRUE even if the test is against 0 and empty string, respectively:
Sales[DiscountPerc] = 0 -- Returns TRUE if DiscountPerc is either BLANK or 0
Sales[Clerk] = "" -- Returns TRUE if Clerk is either BLANK or ""
In such cases, one can use the ISBLANK function to check whether a value is BLANK or not:
ISBLANK ( Sales[DiscountPerc] ) -- Returns TRUE only if DiscountPerc is BLANK
ISBLANK ( Sales[Clerk] ) -- Returns TRUE only if Clerk is BLANK
The propagation of BLANK in a DAX expression happens in several other arithmetic and logical
operations, as shown in the following examples:

## Blank () + Blank () = Blank ()

10 * BLANK () = BLANK ()

## Blank () / 3 = Blank ()


## Blank () / Blank () = Blank ()

However, the propagation of BLANK in the result of an expression does not happen for all
formulas. Some calculations do not propagate BLANK. Instead, they return a value depending on
the other terms of the formula. Examples of these are addition, subtraction, division by BLANK, and
a logical operation including a BLANK. The following expressions show some of these conditions
along with their results:

## Blank () − 10 = −10

18 + BLANK () = 18
4 / BLANK () = Infinity
0 / BLANK () = NaN

## Blank () || Blank () = False


## Blank () && Blank () = False


## ( Blank () = Blank () ) = True


## ( Blank () = True ) = False


## ( Blank () = False ) = True


## ( Blank () = 0 ) = True


## ( Blank () = "" ) = True


## Isblank ( Blank() ) = True


## False || Blank () = False


## False && Blank () = False


## True || Blank () = True


## True && Blank () = False


CHAPTER 2 Introducing DAX 35

Empty values in Excel and SQL
Excel has a different way of handling empty values. In Excel, all empty values are considered 0 whenever they are used in a sum or in a multiplication, but they might return an
error if they are part of a division or of a logical expression.
In SQL, null values are propagated in an expression differently from what happens with
BLANK in DAX. As you can see in the previous examples, the presence of a BLANK in a DAX
expression does not always result in a BLANK result, whereas the presence of NULL in SQL
often evaluates to NULL for the entire expression. This difference is relevant whenever you use
DirectQuery on top of a relational database because some calculations are executed in SQL
and others are executed in DAX. The different semantics of BLANK in the two engines might
result in unexpected behaviors.
Understanding the behavior of empty or missing values in a DAX expression and using BLANK to
return an empty cell in a calculation are important skills to control the results of a DAX expression.
One can often use BLANK as a result when detecting incorrect values or other errors, as we demonstrate in the next section.
Intercepting errors
Now that we have detailed the various kinds of errors that can occur, we still need to show you the
techniques to intercept errors and correct them or, at least, produce an error message containing
meaningful information. The presence of errors in a DAX expression frequently depends on the value
of columns used in the expression itself. Therefore, one might want to control the presence of these
error conditions and return an error message. The standard technique is to check whether an expression returns an error and, if so, replace the error with a specifi c message or a default value. There are a
few DAX functions for this task.
The fi rst of them is the IFERROR function, which is similar to the IF function, but instead of evaluating a Boolean condition, it checks whether an expression returns an error. Two typical uses of the
IFERROR function are as follows:
= IFERROR ( Sales[Quantity] * Sales[Price], BLANK () )
= IFERROR ( SQRT ( Test[Omega] ), BLANK () )
In the fi rst expression, if either Sales[Quantity] or Sales[Price] is a string that cannot be converted
into a number, the returned expression is an empty value. Otherwise, the product of Quantity and Price
is returned.
In the second expression, the result is an empty cell every time the Test[Omega] column contains a
negative number.

36 CHAPTER 2 Introducing DAX
Using IFERROR this way corresponds to a more general pattern that requires using ISERROR and IF:

## = If (

ISERROR ( Sales[Quantity] * Sales[Price] ),

## Blank (),

Sales[Quantity] * Sales[Price]
)

## = If (

ISERROR ( SQRT ( Test[Omega] ) ),

## Blank (),

SQRT ( Test[Omega] )
)
In these cases, IFERROR is a better option. One can use IFERROR whenever the result is the same
expression tested for an error; there is no need to duplicate the expression in two places, and the code
is safer and more readable. However, a developer should use IF when they want to return the result of a
different expression.
Besides, one can avoid raising the error altogether by testing parameters before using them. For
example, one can detect whether the argument for SQRT is positive, returning BLANK for negative
values:

## = If (

Test[Omega] >= 0,
SQRT ( Test[Omega] ),

## Blank ()

)
Considering that the third argument of an IF statement defaults to BLANK, one can also write the
same expression more concisely:

## = If (

Test[Omega] >= 0,
SQRT ( Test[Omega] )
)
A frequent scenario is to test against empty values. ISBLANK detects empty values, returning TRUE if
its argument is BLANK. This capability is important especially when a value being unavailable does not
imply that it is 0. The following example calculates the cost of shipping for a sale transaction, using a
default shipping cost for the product if the transaction itself does not specify a weight:

## = If (

ISBLANK ( Sales[Weight] ), -- If the weight is missing
Sales[DefaultShippingCost], -- then return the default cost
Sales[Weight] * Sales[ShippingPrice] -- otherwise multiply weight by shipping price
)
If we simply multiply product weight by shipping price, we get an empty cost for all the sales transactions without weight data because of the propagation of BLANK in multiplications.

CHAPTER 2 Introducing DAX 37
When using variables, errors must be checked at the time of variable defi nition rather than where
we use them. In fact, the fi rst formula in the following code returns zero, the second formula always
throws an error, and the last one produces different results depending on the version of the product
using DAX (the latest version throws an error also):
IFERROR ( SQRT ( -1 ), 0 ) -- This returns 0

```dax
VAR WrongValue = SQRT ( -1 ) -- Error happens here, so the result is
RETURN -- always an error
IFERROR ( WrongValue, 0 ) -- This line is never executed
```

IFERROR ( -- Different results depending on versions

```dax
VAR WrongValue = SQRT ( -1 ) -- IFERROR throws an error in 2017 versions
RETURN -- IFERROR returns 0 in versions until 2016
WrongValue,
0
)
```

The error happens when WrongValue is evaluated. Thus, the engine will never execute the IFERROR
function in the second example, whereas the outcome of the third example depends on product
versions. If you need to check for errors, take some extra precautions when using variables.
Avoid using error-handling functions
Although we will cover optimizations later in the book, you need to be aware that errorhandling functions might create severe performance issues in your code. It is not that
they are slow in and of themselves. The problem is that the DAX engine cannot use
optimized paths in its code when errors happen. In most cases, checking operands for
possible errors is more effi cient than using the error-handling engine. For example,
instead of writing this:

## Iferror (

SQRT ( Test[Omega] ),

## Blank ()

)
It is much better to write this:

## If (

Test[Omega] >= 0,
SQRT ( Test[Omega] ),

## Blank ()

)
This second expression does not need to detect the error and is faster than the
previous expression. This, of course, is a general rule. For a detailed explanation, see
Chapter 19, “Optimizing DAX.”

38 CHAPTER 2 Introducing DAX
Generating errors
Sometimes, an error is just an error, and the formula should not return a default value in case of an
error. Indeed, returning a default value would end up producing an actual result that would be
incorrect. For example, a confi guration table that contains inconsistent data should produce an
invalid report rather than numbers that are unreliable, and yet it might be considered correct.
Moreover, instead of a generic error, one might want to produce an error message that is more
meaningful to the users. Such a message would help users fi nd where the problem is.
Consider a scenario that requires the computation of the square root of the absolute temperature
measured in Kelvin, to approximately adjust the speed of sound in a complex scientifi c calculation.
Obviously, we do not expect that temperature to be a negative number. If that happens due to a
problem in the measurement, we need to raise an error and stop the calculation.
Another reason to avoid IFERROR is that it cannot intercept errors happening at a deeper
level of execution. For example, the following code intercepts any error happening in the
conversion of the Table[Amount] column considering a blank value in case Amount does
not contain a number. As discussed previously, this execution is expensive because it is
evaluated for every row in Table.

## Sumx (

Table,
IFERROR ( VALUE ( Table[Amount] ), BLANK () )
)
Be mindful that, due to optimizations in the DAX engine, the following code does not
intercept the same errors intercepted by the preceding example. If Table[Amount] contains
a string that is not a number in just one row, the entire expression generates an error that
is not intercepted by IFERROR.

## Iferror (


## Sumx (

Table,
VALUE ( Table[Amount] )
),

## Blank ()

)
ISERROR has the same behavior as IFERROR. Be sure to use them carefully and only to
intercept errors raised directly by the expression evaluated within IFERROR/ISERROR and not
in nested calculations.

CHAPTER 2 Introducing DAX 39
In that case, this code is dangerous because it hides the problem:

## = Iferror (

SQRT ( Test[Temperature] ),
0
)
Instead, to protect the calculations, one should write the formula like this:

## = If (

Test[Temperature] >= 0,
SQRT ( Test[Temperature] ),
ERROR ( "The temperature cannot be a negative number. Calculation aborted." )
)
Formatting DAX code
Before we continue explaining the DAX language, we would like to cover an important aspect of DAX—
that is, formatting the code. DAX is a functional language, meaning that no matter how complex it is, a
DAX expression is like a single function call. The complexity of the code translates into the complexity
of the expressions that one uses as parameters for the outermost function.
For this reason, it is normal to see expressions that span over 10 lines or more. Seeing a 20-line DAX
expression is common, so you will become acquainted with it. Nevertheless, as formulas start to grow
in length and complexity, it is extremely important to format the code to make it human-readable.
There is no “offi cial” standard to format DAX code, yet we believe it is important to describe the
standard that we use in our code. It is likely not the perfect standard, and you might prefer something
different. We have no problem with that: fi nd your optimal standard and use it. The only thing you
need to remember is: format your code and never write everything on a single line; otherwise, you will be
in trouble sooner than you expect.
To understand why formatting is important, look at a formula that computes a time intelligence
calculation. This somewhat complex formula is still not the most complex you will write. Here is how the
expression looks if you do not format it in some way:

```dax
IF(CALCULATE(NOT ISEMPTY(Balances), ALLEXCEPT (Balances, BalanceDate)),SUMX (ALL(Balances
[Account]), CALCULATE(SUM (Balances[Balance]),LASTNONBLANK(DATESBETWEEN(BalanceDate[Date],
BLANK(),MAX(BalanceDate[Date])),CALCULATE(COUNTROWS(Balances))))),BLANK())
```

Trying to understand what this formula computes in its present form is nearly impossible. There is
no clue which is the outermost function and how DAX evaluates the different parameters to create the
complete fl ow of execution. We have seen too many examples of formulas written this way by students
who, at some point, ask for help in understanding why the formula returns incorrect results. Guess
what? The fi rst thing we do is format the expression; only later do we start working on it.

40 CHAPTER 2 Introducing DAX
The same expression, properly formatted, looks like this:

## If (


## Calculate (

NOT ISEMPTY ( Balances ),

## Allexcept (

Balances,
BalanceDate
)
),

## Sumx (

ALL ( Balances[Account] ),

## Calculate (

SUM ( Balances[Balance] ),

## Lastnonblank (


## Datesbetween (

BalanceDate[Date],

## Blank (),

MAX ( BalanceDate[Date] )
),

## Calculate (

COUNTROWS ( Balances )
)
)
)
),

## Blank ()

)
The code is the same, but this time it is much easier to see the three parameters of IF. Most important, it is easier to follow the blocks that arise naturally from indenting lines and how they compose
the complete fl ow of execution. The code is still hard to read, but now the problem is DAX, not poor
formatting. A more verbose syntax using variables can help you read the code, but even in this case,
the formatting is important in providing a correct understanding of the scope of each variable:

## If (


## Calculate (

NOT ISEMPTY ( Balances ),

## Allexcept (

Balances,
BalanceDate
)
),

## Sumx (

ALL ( Balances[Account] ),

```dax
VAR PreviousDates =
DATESBETWEEN (
BalanceDate[Date],
BLANK (),
MAX ( BalanceDate[Date] )
)
```

CHAPTER 2 Introducing DAX 41

```dax
VAR LastDateWithBalance =
LASTNONBLANK (
PreviousDates,
CALCULATE (
COUNTROWS ( Balances )
)
)
RETURN
CALCULATE (
SUM ( Balances[Balance] ),
LastDateWithBalance
)
),
BLANK ()
)
```

DAXFormatter.com
We created a website dedicated to formatting DAX code. We created this site for ourselves
because formatting code is a time-consuming operation and we did not want to spend our
time doing it for every formula we write. After the tool was working, we decided to donate
it to the public domain so that users can format their own DAX code (by the way, we have
been able to promote our formatting rules this way).
You can fi nd the website at www.daxformatter.com. The user interface is simple: just copy your DAX
code, click FORMAT, and the page refreshes showing a nicely formatted version of your code, which
you can then copy and paste in the original window.
This is the set of rules that we use to format DAX:

> **Note:** Always separate function names such as IF, SUMX, and CALCULATE from any other term using a
space and always write them in uppercase.

> **Note:** Write all column references in the form TableName[ColumnName], with no space between the
table name and the opening square bracket. Always include the table name.

> **Note:** Write all measure references in the form [MeasureName], without any table name.

> **Note:** Always use a space following commas and never precede them with a space.

> **Note:** If the formula fi ts one single line, do not apply any other rule.

> **Note:** If the formula does not fi t a single line, then
• Place the function name on a line by itself, with the opening parenthesis.
• Keep all parameters on separate lines, indented with four spaces and with the comma at the
end of the expression except for the last parameter.
• Align the closing parenthesis with the function call so that the closing parenthesis stands on
its own line.

42 CHAPTER 2 Introducing DAX
These are the basic rules we use. A more detailed list of these rules is available at
http://sql.bi/daxrules.
If you fi nd a way to express formulas that best fi ts your reading method, use it. The goal of formatting is to make the formula easier to read, so use the technique that works best for you. The most
important point to remember when defi ning your personal set of formatting rules is that you always
need to be able to see errors as soon as possible. If, in the unformatted code shown previously, DAX
complained about a missing closing parenthesis, it would be hard to spot where the error is. In the
formatted formula, it is much easier to see how each closing parenthesis matches the opening
function call.

Help on formatting DAX
Formatting DAX is not an easy task because often we write it using a small font in a text
box. Depending on the version, Power BI, Excel, and Visual Studio provide different text
editors for DAX. Nevertheless, a few hints might help in writing DAX code:

> **Note:** To increase the font size, hold down Ctrl while rotating the wheel button on the
mouse, making it easier to look at the code.

> **Note:** To add a new line to the formula, press Shift+Enter.

> **Note:** If editing in the text box is not for you, copy the code into another editor, such as
Notepad or DAX Studio, and then copy and paste the formula back into the text box.
When you look at a DAX expression, at fi rst glance it may be hard to understand
whether it is a calculated column or a measure. Thus, in our books and articles we use an
equal sign (=) whenever we defi ne a calculated column and the assignment operator (:=)
to defi ne measures:
CalcCol = SUM ( Sales[SalesAmount] ) -- is a calculated column
Store[CalcCol] = SUM ( Sales[SalesAmount] ) -- is a calculated column in Store table
CalcMsr := SUM ( Sales[SalesAmount] ) -- is a measure
Finally, when using columns and measures in code, we recommend to always put a table
name before a column and never before a measure, as we do in every example.
Introducing aggregators and iterators
Almost every data model needs to operate on aggregated data. DAX offers a set of functions that
aggregate the values of a column in a table and return a single value. We call this group of functions
aggregation functions. For example, the following measure calculates the sum of all the numbers in the
SalesAmount column of the Sales table:
Sales := SUM ( Sales[SalesAmount] )

CHAPTER 2 Introducing DAX 43
SUM aggregates all the rows of the table if it is used in a calculated column. Whenever it is used in
a measure, it considers only the rows that are being fi ltered by slicers, rows, columns, and fi lter conditions in the report.
There are many aggregation functions (SUM, AVERAGE, MIN, MAX, and STDEV), and their behavior
changes only in the way they aggregate values: SUM adds values, whereas MIN returns the minimum
value. Nearly all these functions operate only on numeric values or on dates. Only MIN and MAX can
operate on text values also. Moreover, DAX never considers empty cells when it performs the aggregation, and this behavior is different from their counterpart in Excel (more on this later in this chapter).
Note MIN and MAX offer another behavior: if used with two parameters, they return
the minimum or maximum of the two parameters. Thus, MIN (1, 2) returns 1 and MAX (1, 2)
returns 2. This functionality is useful when one needs to compute the minimum or maximum of complex expressions because it saves having to write the same expression multiple
times in IF statements.
All the aggregation functions we have described so far work on columns. Therefore, they aggregate
values from a single column only. Some aggregation functions can aggregate an expression instead of
a single column. Because of the way they work, they are known as iterators. This set of functions is useful, especially when you need to make calculations using columns of different related tables, or when
you need to reduce the number of calculated columns.
Iterators always accept at least two parameters: the fi rst is a table that they scan; the second is typically an expression that is evaluated for each row of the table. After they have completed scanning the
table and evaluating the expression row by row, iterators aggregate the partial results according to
their semantics.
For example, if we compute the number of days needed to deliver an order in a calculated column
called DaysToDeliver and build a report on top of that, we obtain the report shown in Figure 2-6. Note
that the grand total shows the sum of all the days, which is not useful for this metric:
Sales[DaysToDeliver] = INT ( Sales[Delivery Date] - Sales[Order Date] )
FIGURE 2-6 The grand total is shown as a sum, when you might want an average instead.

44 CHAPTER 2 Introducing DAX
A grand total that we can actually use requires a measure called AvgDelivery showing the delivery
time for each order and the average of all the durations at the grand total level:
AvgDelivery := AVERAGE ( Sales[DaysToDeliver] )
The result of this new measure is visible in the report shown in Figure 2-7.
FIGURE 2-7 The measure aggregating by average shows the average delivery days at the grand total level.
The measure computes the average value by averaging a calculated column. One could remove the
calculated column, thus saving space in the model, by leveraging an iterator. Indeed, although it is true
that AVERAGE cannot average an expression, its counterpart AVERAGEX can iterate the Sales table and
compute the delivery days row by row, averaging the results at the end. This code accomplishes the
same result as the previous defi nition:
AvgDelivery :=

## Averagex (

Sales,
INT ( Sales[Delivery Date] - Sales[Order Date] )
)
The biggest advantage of this last expression is that it does not rely on the presence of a calculated
column. Thus, we can build the entire report without creating expensive calculated columns.
Most iterators have the same name as their noniterative counterpart. For example, SUM has a corresponding SUMX, and MIN has a corresponding MINX. Nevertheless, keep in mind that some iterators
do not correspond to any aggregator. Later in this book, you will learn about FILTER, ADDCOLUMNS,
GENERATE, and other functions that are iterators even if they do not aggregate their results.
When you fi rst learn DAX, you might think that iterators are inherently slow. The concept of performing calculations row by row looks like a CPU-intensive operation. Actually, iterators are fast, and no
performance penalty is caused by using iterators instead of standard aggregators. Aggregators are just
a syntax-sugared version of iterators.
Indeed, the basic aggregation functions are a shortened version of the corresponding X-suffi xed
function. For example, consider the following expression:
SUM ( Sales[Quantity] )

CHAPTER 2 Introducing DAX 45
It is internally translated into this corresponding version of the same code:
SUMX ( Sales, Sales[Quantity] )
The only advantage in using SUM is a shorter syntax. However, there are no differences in performance between SUM and SUMX aggregating a single column. They are in all respects the same function.
We will cover more details about this behavior in Chapter 4. There we introduce the concept of
evaluation contexts to describe properly how iterators work.
Using common DAX functions
Now that you have seen the fundamentals of DAX and how to handle error conditions, what follows is a
brief tour through the most commonly used functions and expressions of DAX.
Aggregation functions
In the previous sections, we described the basic aggregators like SUM, AVERAGE, MIN, and MAX. You
learned that SUM and AVERAGE, for example, work only on numeric columns.
DAX also offers an alternative syntax for aggregation functions inherited from Excel, which adds the
suffi x A to the name of the function, just to get the same name and behavior as Excel. However, these
functions are useful only for columns containing Boolean values because TRUE is evaluated as 1 and
FALSE as 0. Text columns are always considered 0. Therefore, no matter what is in the content of a column, if one uses MAXA on a text column, the result will always be a 0. Moreover, DAX never considers
empty cells when it performs the aggregation. Although these functions can be used on nonnumeric
columns without retuning an error, their results are not useful because there is no automatic conversion to numbers for text columns. These functions are named AVERAGEA, COUNTA, MINA, and MAXA.
We suggest that you do not use these functions, whose behavior will be kept unchanged in the future
because of compatibility with existing code that might rely on current behavior.
Note Despite the names being identical to statistical functions, they are used differently in
DAX and Excel because in DAX a column has a data type, and its data type determines the
behavior of aggregation functions. Excel handles a different data type for each cell, whereas
DAX handles a single data type for the entire column. DAX deals with data in tabular form
with well-defi ned types for each column, whereas Excel formulas work on heterogeneous
cell values without well-defi ned types. If a column in Power BI has a numeric data type, all
the values can be only numbers or empty cells. If a column is of a text type, it is always 0
for these functions (except for COUNTA), even if the text can be converted into a number,
whereas in Excel the value is considered a number on a cell-by-cell basis. For these reasons,
these functions are not very useful for Text columns. Only MIN and MAX also support text
values in DAX.

46 CHAPTER 2 Introducing DAX
The functions you learned earlier are useful to perform the aggregation of values. Sometimes, you
might not be interested in aggregating values but only in counting them. DAX offers a set of functions
that are useful to count rows or values:

> **Note:** COUNT operates on any data type, apart from Boolean.

> **Note:** COUNTA operates on any type of column.

> **Note:** COUNTBLANK returns the number of empty cells (blanks or empty strings) in a column.

> **Note:** COUNTROWS returns the number of rows in a table.

> **Note:** DISTINCTCOUNT returns the number of distinct values of a column, blank value included if
present.

> **Note:** DISTINCTCOUNTNOBLANK returns the number of distinct values of a column, no blank value
included.
COUNT and COUNTA are nearly identical functions in DAX. They return the number of values of the
column that are not empty, regardless of their data type. They are inherited from Excel, where COUNTA
accepts any data type including strings, whereas COUNT accepts only numeric columns. If we want to
count all the values in a column that contain an empty value, you can use the COUNTBLANK function.
Both blanks and empty values are considered empty values by COUNTBLANK. Finally, if we want to
count the number of rows of a table, you can use the COUNTROWS function. Beware that COUNTROWS requires a table as a parameter, not a column.
The last two functions, DISTINCTCOUNT and DISTINCTCOUNTNOBLANK, are useful because they
do exactly what their names suggest: count the distinct values of a column, which it takes as its only
parameter. DISTINCTCOUNT counts the BLANK value as one of the possible values, whereas DISTINCTCOUNTNOBLANK ignores the BLANK value.
Note DISTINCTCOUNT is a function introduced in the 2012 version of DAX. The earlier
versions of DAX did not include DISTINCTCOUNT; to compute the number of distinct values
of a column, we had to use COUNTROWS ( DISTINCT ( table[column] ) ). The two patterns
return the same result although DISTINCTCOUNT is easier to read, requiring only a single
function call. DISTINCTCOUNTNOBLANK is a function introduced in 2019 and it provides
the same semantic of a COUNT DISTINCT operation in SQL without having to write a longer
expression in DAX.
Logical functions
Sometimes we want to build a logical condition in an expression—for example, to implement different
calculations depending on the value of a column or to intercept an error condition. In these cases,
we can use one of the logical functions in DAX. The earlier section titled “Handling errors in DAX
expressions” described the two most important functions of this group: IF and IFERROR. We described
the IF function in the “Conditional statements” section, earlier in this chapter.

CHAPTER 2 Introducing DAX 47
Logical functions are very simple and do what their names suggest. They are AND, FALSE, IF,
IFERROR, NOT, TRUE, and OR. For example, if we want to compute the amount as quantity multiplied
by price only when the Price column contains a numeric value, we can use the following pattern:
Sales[Amount] = IFERROR ( Sales[Quantity] * Sales[Price], BLANK ( ) )
If we did not use IFERROR and if the Price column contained an invalid number, the result for the
calculated column would be an error because if a single row generates a calculation error, the error
propagates to the whole column. The use of IFERROR, however, intercepts the error and replaces it with
a blank value.
Another interesting function in this category is SWITCH, which is useful when we have a column
containing a low number of distinct values, and we want to get different behaviors depending on its
value. For example, the column Size in the Product table contains S, M, L, XL, and we might want to
decode this value in a more explicit column. We can obtain the result by using nested IF calls:
'Product'[SizeDesc] =

## If (

'Product'[Size] = "S",
"Small",

## If (

'Product'[Size] = "M",
"Medium",

## If (

'Product'[Size] = "L",
"Large",

## If (

'Product'[Size] = "XL",
"Extra Large",
"Other"
)
)
)
)
A more convenient way to express the same formula, using SWITCH, is like this:
'Product'[SizeDesc] =

## Switch (

'Product'[Size],
"S", "Small",
"M", "Medium",
"L", "Large",
"XL", "Extra Large",
"Other"
)
The code in this latter expression is more readable, though not faster, because internally DAX
translates SWITCH statements into a set of nested IF functions.

48 CHAPTER 2 Introducing DAX
Note SWITCH is often used to check the value of a parameter and defi ne the result of a
measure. For example, one might create a parameter table containing YTD, MTD, QTD as
three rows and let the user choose from the three available which aggregation to use in a
measure. This was a common scenario before 2019. Now it is no longer needed thanks to
the introduction of calculation groups, covered in Chapter 9, “Calculation groups.” Calculation groups are the preferred way of computing values that the user can parameterize.
Tip Here is an interesting way to use the SWITCH function to check for multiple conditions in the same expression. Because SWITCH is converted into a set of nested IF functions,
where the fi rst one that matches wins, you can test multiple conditions using this pattern:

## Switch (


## True (),

Product[Size] = "XL" && Product[Color] = "Red", "Red and XL",
Product[Size] = "XL" && Product[Color] = "Blue", "Blue and XL",
Product[Size] = "L" && Product[Color] = "Green", "Green and L"
)
Using TRUE as the fi rst parameter means, “Return the fi rst result where the condition
evaluates to TRUE.”
Information functions
Whenever there is the need to analyze the type of an expression, you can use one of the information
functions. All these functions return a Boolean value and can be used in any logical expression. They
are ISBLANK, ISERROR, ISLOGICAL, ISNONTEXT, ISNUMBER, and ISTEXT.
It is important to note that when a column is passed as a parameter instead of an expression, the
functions ISNUMBER, ISTEXT, and ISNONTEXT always return TRUE or FALSE depending on the data
type of the column and on the empty condition of each cell. This makes these functions nearly useless
in DAX; they have been inherited from Excel in the fi rst DAX version.
You might be wondering whether you can use ISNUMBER with a text column just to check whether
a conversion to a number is possible. Unfortunately, this approach is not possible. If you want to test
whether a text value is convertible to a number, you must try the conversion and handle the error if it
fails. For example, to test whether the column Price (which is of type string) contains a valid number,
one must write
Sales[IsPriceCorrect] = NOT ISERROR ( VALUE ( Sales[Price] ) )
DAX tries to convert from a string value to a number. If it succeeds, it returns TRUE (because
ISERROR returns FALSE); otherwise, it returns FALSE (because ISERROR returns TRUE). For example,
the conversion fails if some of the rows have an “N/A” string value for price.

CHAPTER 2 Introducing DAX 49
However, if we try to use ISNUMBER, as in the following expression, we always receive FALSE as a
result:
Sales[IsPriceCorrect] = ISNUMBER ( Sales[Price] )
In this case, ISNUMBER always returns FALSE because, based on the defi nition in the model, the
Price column is not a number but a string, regardless of the content of each row.
Mathematical functions
The set of mathematical functions available in DAX is similar to the set available in Excel, with the same
syntax and behavior. The mathematical functions of common use are ABS, EXP, FACT, LN, LOG, LOG10,
MOD, PI, POWER, QUOTIENT, SIGN, and SQRT. Random functions are RAND and RANDBETWEEN. By
using EVEN and ODD, you can test numbers. GCD and LCM are useful to compute the greatest common denominator and least common multiple of two numbers. QUOTIENT returns the integer division
of two numbers.
Finally, several rounding functions deserve an example; in fact, we might use several approaches to
get the same result. Consider these calculated columns, along with their results in Figure 2-8:
FLOOR = FLOOR ( Tests[Value], 0.01 )
TRUNC = TRUNC ( Tests[Value], 2 )
ROUNDDOWN = ROUNDDOWN ( Tests[Value], 2 )
MROUND = MROUND ( Tests[Value], 0.01 )
ROUND = ROUND ( Tests[Value], 2 )
CEILING = CEILING ( Tests[Value], 0.01 )
ISO.CEILING = ISO.CEILING ( Tests[Value], 0.01 )
ROUNDUP = ROUNDUP ( Tests[Value], 2 )
INT = INT ( Tests[Value] )
FIXED = FIXED ( Tests[Value], 2, TRUE )
FIGURE 2-8 This summary shows the results of using different rounding functions.
FLOOR, TRUNC, and ROUNDDOWN are similar except in the way we can specify the number of
digits to round. In the opposite direction, CEILING and ROUNDUP are similar in their results. You can
see a few differences in the way the rounding is done between MROUND and ROUND function.

50 CHAPTER 2 Introducing DAX
Trigonometric functions
DAX offers a rich set of trigonometric functions that are useful for certain calculations: COS, COSH,
COT, COTH, SIN, SINH, TAN, and TANH. Prefi xing them with A computes the arc version (arcsine,
arccosine, and so on). We do not go into the details of these functions because their use is
straightforward.
DEGREES and RADIANS perform conversion to degrees and radians, respectively, and SQRTPI
computes the square root of its parameter after multiplying it by pi.
Text functions
Most of the text functions available in DAX are similar to those available in Excel, with only a few
exceptions. The text functions are CONCATENATE, CONCATENATEX, EXACT, FIND, FIXED, FORMAT,
LEFT, LEN, LOWER, MID, REPLACE, REPT, RIGHT, SEARCH, SUBSTITUTE, TRIM, UPPER, and VALUE. These
functions are useful for manipulating text and extracting data from strings that contain multiple values.
For example, Figure 2-9 shows an example of the extraction of fi rst and last names from a string that
contains these values separated by commas, with the title in the middle that we want to remove.
FIGURE 2-9 This example shows fi rst and last names extracted using text functions.
To achieve this result, you start calculating the position of the two commas. Then we use these
numbers to extract the right part of the text. The SimpleConversion column implements a formula that
might return inaccurate values if there are fewer than two commas in the string, and it raises an error
if there are no commas at all. The FirstLastName column implements a more complex expression that
does not fail in case of missing commas:
People[Comma1] = IFERROR ( FIND ( ",", People[Name] ), BLANK ( ) )
People[Comma2] = IFERROR ( FIND ( " ,", People[Name], People[Comma1] + 1 ), BLANK ( ) )
People[SimpleConversion] =
MID ( People[Name], People[Comma2] + 1, LEN ( People[Name] ) )

## & " "

& LEFT ( People[Name], People[Comma1] - 1 )
People[FirstLastName] =

## Trim (


## Mid (

People[Name],
IF ( ISNUMBER ( People[Comma2] ), People[Comma2], People[Comma1] ) + 1,
LEN ( People[Name] )
)
)

## & If (


CHAPTER 2 Introducing DAX 51
ISNUMBER ( People[Comma1] ),
" " & LEFT ( People[Name], People[Comma1] - 1 ),
""
)
As you can see, the FirstLastName column is defi ned by a long DAX expression, but you must use it
to avoid possible errors that would propagate to the whole column if even a single value generates an
error.
Conversion functions
You learned previously that DAX performs automatic conversions of data types to adjust them to
operator needs. Although the conversion happens automatically, a set of functions can still perform
explicit data type conversions.
CURRENCY can transform an expression into a Currency type, whereas INT transforms an expression into an Integer. DATE and TIME take the date and time parts as parameters and return a correct
DateTime. VALUE transforms a string into a numeric format, whereas FORMAT gets a numeric value as
its fi rst parameter and a string format as its second parameter, and it can transform numeric values into
strings. FORMAT is commonly used with DateTime. For example, the following expression returns “2019
Jan 12”:
= FORMAT ( DATE ( 2019, 01, 12 ), "yyyy mmm dd" )
The opposite operation, that is, converting strings into DateTime values, is performed using the
DATEVALUE function.
DATEVALUE with dates in different format
DATEVALUE displays a special behavior regarding dates in different formats. In the European
standard, dates are written with the format “dd/mm/yy”, whereas Americans prefer to use
“mm/dd/yy”. For example, the 28th of February has different string representations in the
two cultures. If you provide to DATEVALUE a date that cannot be converted using the default
regional setting, instead of immediately raising an error, it tries a second conversion switching months and days. DATEVALUE also supports the unambiguous format “yyyy-mm-dd”.
As an example, the following three expressions evaluate to February 28, no matter which
regional settings you have:
DATEVALUE ( "28/02/2018" ) -- This is February 28 in European format
DATEVALUE ( "02/28/2018" ) -- This is February 28 in American format
DATEVALUE ( "2018-02-28" ) -- This is February 28 (format is not ambiguous)
Sometimes, DATEVALUE does not raise errors when you would expect them. However,
this is the behavior of the function by design.

52 CHAPTER 2 Introducing DAX
Date and time functions
In almost every type of data analysis, handling time and dates is an important part of the job. Many
DAX functions operate on date and time. Some of them correspond to similar functions in Excel and
make simple transformations to and from a DateTime data type. The date and time functions are DATE,
DATEVALUE, DAY, EDATE, EOMONTH, HOUR, MINUTE, MONTH, NOW, SECOND, TIME, TIMEVALUE,
TODAY, WEEKDAY, WEEKNUM, YEAR, and YEARFRAC.
These functions are useful to compute values on top of dates, but they are not used to perform
typical time intelligence calculations such as comparing aggregated values year over year or calculating the year-to-date value of a measure. To perform time intelligence calculations, you use another
set of functions called time intelligence functions, which we describe in Chapter 8, “Time intelligence
calculations.”
As we mentioned earlier in this chapter, a DateTime data type internally uses a fl oating-point
number wherein the integer part corresponds to the number of days after December 30, 1899, and the
decimal part indicates the fraction of the day in time. Hours, minutes, and seconds are converted into
decimal fractions of the day. Thus, adding an integer number to a DateTime value increments the value
by a corresponding number of days. However, you will probably fi nd it more convenient to use the
conversion functions to extract the day, month, and year from a date. The following expressions used
in Figure 2-10 show how to extract this information from a table containing a list of dates:
'Date'[Day] = DAY ( Calendar[Date] )
'Date'[Month] = FORMAT ( Calendar[Date], "mmmm" )
'Date'[MonthNumber] = MONTH ( Calendar[Date] )
'Date'[Year] = YEAR ( Calendar[Date] )
FIGURE 2-10 This example shows how to extract date information using date and time functions.

CHAPTER 2 Introducing DAX 53
Relational functions
Two useful functions that you can use to navigate through relationships inside a DAX formula are
RELATED and RELATEDTABLE.
You already know that a calculated column can reference column values of the table in which it is
defi ned. Thus, a calculated column defi ned in Sales can reference any column of Sales. However, what if
one must refer to a column in another table? In general, one cannot use columns in other tables unless
a relationship is defi ned in the model between the two tables. If the two tables share a relationship, you
can use the RELATED function to access columns in the related table.
For example, one might want to compute a calculated column in the Sales table that checks whether
the product that has been sold is in the “Cell phones” category and, in that case, apply a reduction factor to the standard cost. To compute such a column, one must use a condition that checks the value of
the product category, which is not in the Sales table. Nevertheless, a chain of relationships starts from
Sales, reaching Product Category through Product and Product Subcategory, as shown in Figure 2-11.
FIGURE 2-11 Sales has a chained relationship with Product Category.
Regardless of how many steps are necessary to travel from the original table to the related table,
DAX follows the complete chain of relationships, and it returns the related column value. Thus, the
formula for the AdjustedCost column can look like this:
Sales[AdjustedCost] =

## If (

RELATED ( 'Product Category'[Category] ) = "Cell Phone",
Sales[Unit Cost] * 0.95,
Sales[Unit Cost]
)
In a one-to-many relationship, RELATED can access the one-side from the many-side because in that
case, only one row in the related table exists, if any. If no such row exists, RELATED returns BLANK.
If an expression is on the one-side of the relationship and needs to access the many-side, RELATED
is not helpful because many rows from the other side might be available for a single row. In that case,
we can use RELATEDTABLE. RELATEDTABLE returns a table containing all the rows related to the current
row. For example, if we want to know how many products are in each category, we can create a column
in Product Category with this formula:
'Product Category'[NumOfProducts] = COUNTROWS ( RELATEDTABLE ( Product ) )

54 CHAPTER 2 Introducing DAX
For each product category, this calculated column shows the number of products related, as shown
in Figure 2-12.
FIGURE 2-12 You can count the number of products by using RELATEDTABLE.
As is the case for RELATED, RELATEDTABLE can follow a chain of relationships always starting from
the one-side and going toward the many-side. RELATEDTABLE is often used in conjunction with iterators. For example, if we want to compute the sum of quantity multiplied by net price for each category,
we can write a new calculated column as follows:
'Product Category'[CategorySales] =

## Sumx (

RELATEDTABLE ( Sales ),
Sales[Quantity] * Sales[Net Price]
)
The result of this calculated column is shown in Figure 2-13.
FIGURE 2-13 Using RELATEDTABLE and iterators, we can compute the amount of sales per category.
Because the column is calculated, this result is consolidated in the table, and it does not change
according to the user selection in the report, as it would if it were written in a measure.

CHAPTER 2 Introducing DAX 55
Conclusions
In this chapter, you learned many new functions and started looking at some DAX code. You may not
remember all the functions right away, but the more you use them, the more familiar they will become.
The more crucial topics you learned in this chapter are

> **Note:** Calculated columns are columns in a table that are computed with a DAX expression. Calculated
columns are computed at data refresh time and do not change their value depending on user
selection.

> **Note:** Measures are calculations expressed in DAX. Instead of being computed at refresh time like
calculated columns are, measures are computed at query time. Consequently, the value of a
measure depends on the user selection in the report.

> **Note:** Errors might happen at any time in a DAX expression; it is preferable to detect the error condition beforehand rather than letting the error happen and intercepting it after the fact.

> **Note:** Aggregators like SUM are useful to aggregate columns, whereas to aggregate expressions, you
need to use iterators. Iterators work by scanning a table and evaluating an expression row by
row. At the end of the iteration, iterators aggregate a result according to their semantics.
In the next chapter, you will continue on your learning path by studying the most important table
functions available in DAX.