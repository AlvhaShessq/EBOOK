# Chapter 21: Survey

> **Sample files:** https://sql.bi/dax-216


The Survey pattern uses a data model to analyze correlations
between different events related to the same entity, such as
customer answers to survey questions. For example, in healthcare
organizations the Survey pattern can be used to analyze data about
patient status, diagnoses, and medicine prescribed.


Pattern description
You have a model that stores answers to questions. Therefore,
consider a Questions table containing questions and possible
answers shown in Figure 21-1.


*📊 FIGURE 21-1 Every question has several possible answers.*
*The answers are stored in an Answers table containing in each row*

the survey target (the Customer in this case), one question and one
answer. There are multiple rows in case the same customer provides
multiple answers to the same question. The real model would store
information with integer keys; In Figure 21-2 we are using strings to
clarify the concept.


*📊 FIGURE 21-2 Every row in the Answers table contains the answer of a customer to one*
*specific question.*


By using a DAX formula, we can answer a request like, “How many
customers enjoy cartoons, broken down by job and gender?”
Consider Figure 21-3 as an example. In this table, totals are not
strict totals. This is explained later.


*📊 FIGURE 21-3 Every cell shows the number of customers who like one kind of movie and*
*provided different answers to the job and gender questions.*


The report includes two slicers to select the questions to intersect
in the report. The columns in the matrix have the answers to the
question selected in the Question 1 slicer, whereas the rows of the
matrix provide the details of questions and answers corresponding to
the selection made in the Question 2 slicer. The highlighted cell
shows that 9 customers who answered Cartoons to the Movie
Preferences question also answered Female to the Gender
question.
In order to implement this pattern, you need to load the Questions
table twice. This way you can use two slicers for the questions to
analyze. Moreover, the relationship between the two copies of the
questions must be inactive. Because we use the tables as filters, we
named them Filter1 and Filter2. You can see the resulting diagram in

*📊 Figure 21-4.*

*📊 FIGURE 21-4 The Survey data model includes inactive relationships between the Filter and*
*Answers tables.*


To compute the number of customers who answered Q1 (the
question filtered by Filter1) and Q2 (the question filtered by Filter2)
you can use the following formula:


> *Measure in the Customers table*

The formula activates the correct relationship when computing
CustomersQ1 and CustomersQ2. It then uses the two variables as
filters for the Answers table, which filters the customers through the

CROSSFILTER modifier.
You can compute any calculation using the previous formula -
provided that the CROSSFILTER modifier makes the Answers table
filter the table you are basing your code on. Therefore, you can
replace COUNTROWS ( Customer ) with any expression involving the
Customers table. For example, the RevenueQ1andQ2 measure
provides the total revenue made off of the customers included in the
selection; The only difference with the CustomersQ1andQ2 measure is
the Revenue Amount measure reference that replaces the previous
COUNTROWS ( Customer ) expression:


> *Measure in the Customers table*

The result of the RevenueQ1andQ2 measure is visible in Figure 21-5.


*📊 FIGURE 21-5 Every cell shows the revenue made off of customers who like one kind of*
*movie and provided different answers to the job and gender questions.*


If you only count the number of customers, then the previous code
can be simplified and sped up by using the following variation:


> *Measure in the Customers table*

It is important to understand the condition computed in each cell.
We use Figure 21-6 to explain this further, where we labeled a few
cells from A to E.


*📊 FIGURE 21-6 Each cell computes a different number, the explanation is in the text below.*
*Here is what is computed in each cell:*


### A Female AND prefers Cartoons


### B ( Female OR Male ) AND prefers Cartoons


C ( Female OR Male OR Consultant OR IT Pro OR Teacher ) AND prefers Cartoons


### D ( Female OR Male ) AND prefers ( Cartoons OR Comedy OR Horror )


( Female OR Male OR Consultant OR IT Pro OR Teacher ) AND prefers ( Cartoons OR
E
Comedy OR Horror )

The formula uses an AND condition for the intersection between
questions selected in Question 1 and Question 2, whereas it uses an
OR condition for the answers provided to one same question.
Remember that the OR condition means “any combination” (do not
confuse it with an “exclusive or”) and the OR condition also implies a
non-additive behavior of the measure.
