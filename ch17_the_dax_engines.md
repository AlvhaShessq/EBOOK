# Chapter 17: The DAX engines

The goal of the book up to this point has been to provide a solid understanding of the DAX language.
On top of gaining further experience through practice, the next goal for you is to write effi cient
DAX and not just DAX that works. Writing effi cient DAX requires understanding the internals of the
engine. The next chapters aim to provide the essential knowledge to measure and improve DAX code
performance.
More specifi cally, this chapter is dedicated to the internal architecture of the engines running DAX
queries. Indeed, a DAX query can run on a model that is stored entirely in memory, or entirely on the
original data source, or on a mix of these two options.
Starting from this chapter, we somewhat deviate from DAX and begin to discuss low-level technical
details about the implementation of products that use DAX. This is an important topic, but you need
to be aware that implementation details change often. We did our best to show information at a level
that is not likely to change soon, carefully balancing detail level and usefulness with consistency over
time. Nevertheless, given the pace at which technology runs these days, the information might be
outdated within a few years. The most up-to-date information is always available online, in blog posts
and articles.
New versions of the engines come out every month, and the query optimizer can change and
improve the query execution. Therefore, we aim to teach how the engines work, rather than just provide a few rules about writing DAX code that would quickly become obsolete. We sometimes provide
best practices, but remember to always double-check how our suggestions apply to your specifi c
scenario.
Understanding the architecture of the DAX engines
The DAX language is used in several Microsoft products based on the Tabular technology. Yet, specifi c
features might only be available in a few editions or license conditions. A Tabular model uses both
DAX and MDX as query languages. This section describes the broader architecture of a Tabular model,
regardless of the query language and of the limitations of specifi c products.

546 CHAPTER 17 The DAX engines
Every report sends queries to Tabular using either DAX or MDX. Despite the query language used,
the Tabular model uses two engines to process a query:

> **Note:** The formula engine (FE), which processes the request, generating and executing a query plan.

> **Note:** The storage engine (SE), which retrieves data out of the Tabular model to answer the requests
made by the Formula Engine. The Storage Engine has two implementations:
• VertiPaq hosts a copy of the data in-memory that is refreshed periodically from the data
source.
• DirectQuery forwards queries directly to the original data source for every request.
DirectQuery does not create an additional copy of data.
Figure 17-1 represents the architecture that executes a DAX or MDX query.
Formula Engine
Tabular Model
Storage Engine
Cached Data
Data Source
DAX

## Calculation


## Engine


## Vertipaq

(xmSQL)

## Query


## (Dax/Mdx)


## Periodic


## Refresh


## Directquery


## (Sql, ...)

FIGURE 17-1 A query is processed by an architecture using a formula engine and a storage engine.
The formula engine is the higher-level execution unit of the query engine in a Tabular model. It
can handle all the operations requested by DAX and MDX functions and can solve complex DAX and
MDX expressions. However, when the formula engine must retrieve data from the underlying tables, it
forwards part of the requests to the storage engine.
The queries sent to the storage engine might vary from a simple retrieval of the raw table data to
more complex queries aggregating data and joining tables. The storage engine only communicates
with the formula engine. The storage engine returns data in an uncompressed format, regardless of the
original format of the data.
A Tabular model usually stores data using either the VertiPaq or the DirectQuery storage engine.
However, composite models can use both technologies within the same data model and for the same
tables. The choice of which engine to use is made by the engine on a by-query basis.
This book is exclusively focused on DAX. Be mindful that MDX uses the same architecture when it
queries a Tabular model. This chapter describes the different types of storage engines available in a
Tabular model, focusing more on the details of the VertiPaq engine because it is the native and faster
engine for DAX.

CHAPTER 17 The DAX engines 547
Introducing the formula engine
The formula engine is the absolute core of the DAX execution. Indeed, the formula engine alone is able
to understand the DAX language, though it understands MDX as well. The formula engine converts a
DAX or MDX query into a query plan describing a list of physical steps to execute. The storage engine
part of Tabular is not aware that its queries originated from a model supporting DAX.
Each step in the query plan corresponds to a specifi c operation executed by the formula engine.
Typical operators of the formula engine include joins between tables, fi ltering with complex conditions,
aggregations, and lookups. These operators typically require data from columns in the data model.
In these cases, the formula engine sends a request to the storage engine, which answers by returning
a datacache. A datacache is a temporary storage area created by the storage engine and read by the
formula engine.
Note Datacaches are not compressed; datacaches are plain in-memory tables stored in an
uncompressed format, regardless of the storage engine they come from.
The formula engine always works with datacaches returned by the storage engine or with data
structures computed by other formula engine operators. The result of a formula engine operation is
not persisted in memory across different executions, even within the same session. On the other hand,
datacaches are kept in memory and can be reused in following queries. The formula engine does not
have a cache system to reuse results between different queries. DAX relies entirely on the cache
features of the storage engine.
Finally, the formula engine is single-threaded. This means that any operation executed in the formula engine uses just one thread and one core, no matter how many cores are available. The formula
engine sends requests to the storage engine sequentially, one query at a time. A certain degree of parallelism is available only within each request to the storage engine, which has a different architecture
and can take advantage of multiple cores available. This is described in the next sections.
Introducing the storage engine
The goal of the storage engine is to scan the Tabular database and produce the datacaches needed by
the formula engine. The storage engine is independent from DAX. For example, DirectQuery on top
of SQL Server uses SQL as the storage engine. SQL was born much earlier than DAX. Although it might
seem strange, the internal storage engine of Tabular (known as VertiPaq) is independent from DAX
too. The overall architecture is very clean and sound. The storage engine executes exclusively queries
allowed by its own set of operators. Depending on the kind of storage engine used, the set of operators
might range from very limited (VertiPaq) to very rich (SQL). This affects the performance and the kind
of optimizations that a developer should consider when analyzing query plans.

548 CHAPTER 17 The DAX engines
A developer can defi ne the storage engine used for each table, using one of these three options:

> **Note:** Import: Also called in-memory, or VertiPaq. The content of the table is stored by the VertiPaq
engine, copying and restructuring the data from the data source during data refresh.

> **Note:** DirectQuery: The content of the table is read from the data source at query time, and it is not
stored in memory during data refresh.

> **Note:** Dual: The table can be queried in both VertiPaq and DirectQuery. During data refresh the table
is loaded in memory, but at query time the table may also be read in DirectQuery mode, with
the most up-to-date information.
Moreover, a table in a Tabular model could be used as an aggregation for another table. Aggregations are useful to optimize storage engine requests, but not to optimize a bottleneck in the formula
engine. Aggregations can be defi ned in both VertiPaq and DirectQuery, though they are commonly
defi ned in VertiPaq to achieve the best query performance.
The storage engine features a parallel implementation. However, it receives requests from the formula engine, which sends them synchronously. Thus, the formula engine waits for one storage engine
query to fi nish before sending the next one. Therefore, parallelism in the storage engine might be
reduced by the lack of parallelism of the formula engine.
Introducing the VertiPaq (in-memory) storage engine
The VertiPaq storage engine is the native lower-level execution unit of the DAX query engine. In certain
products it was offi cially named xVelocity In-Memory Analytical Engine. Nevertheless, it is widely
known as VertiPaq, which is the original code name used during development. VertiPaq stores a copy
of the data read from the data source in a compressed in-memory format based on a columnar
database structure.
VertiPaq queries are expressed using an internal pseudo-SQL language called xmSQL. xmSQL is
not a real query language, but rather a textual representation of a storage engine query. The intent of
xmSQL is to give visibility to humans as to how the formula engine is querying VertiPaq. VertiPaq offers
a very limited set of operators: In case the calculation requires a more complex evaluation within an
internal data scan, VertiPaq can perform a callback to the formula engine.
The VertiPaq storage engine is multithreaded. The operations performed by the VertiPaq storage
engine are very effi cient and can scale up on multiple cores. A single storage engine query can increase
its parallelism up to one thread for each segment of a table. We will describe segments later in this
chapter. Considering that the storage engine can use up to one thread per column segment, one can
benefi t from the parallelism of the storage engine only when there are many segments involved in the
query. In other words, if there are eight storage engine queries, running on a small table (one segment),
they will run sequentially one after the other, instead of all in parallel, because of the synchronous
nature of communication between the formula engine and the storage engine.

CHAPTER 17 The DAX engines 549
A cache system stores the results produced by the VertiPaq storage engine, holding a limited number of results—typically the last 512 internal queries per database, but different versions of the engine
might use a different number. When the storage engine receives an xmSQL query identical to one
already in cache, it returns the corresponding datacache without doing any scan of data in memory.
The cache is not involved in security considerations because the row-level security system only infl uences the formula engine behavior, producing different xmSQL queries in case the user is restricted to
seeing specifi c rows in a table.
A scan operation made by the storage engine is usually faster than the equivalent scan performed
by the formula engine, even with a single thread available. This is because the storage engine is better
optimized for these operations and because it iterates over compressed data; the formula engine, on
the other hand, can only iterate over datacaches, which are uncompressed.
Introducing the DirectQuery storage engine
The DirectQuery storage engine is a generic defi nition, describing the scenario where the data is kept
in the original data source instead of being copied in the VertiPaq storage. When the formula engine
sends a request to the storage engine in DirectQuery mode, it sends a query to the data source in its
specifi c query language. This is SQL most of the time, but it could be different.
The formula engine is aware of the presence of DirectQuery. Therefore, the formula engine generates a different query plan compared to VertiPaq because it can take advantage of more advanced
functions available in the query language used by the data source. For example, SQL can manage string
transformations such as UPPER and LOWER, whereas the VertiPaq engine does not have any string
manipulation functions available.
Any optimization of the storage engine using DirectQuery requires an optimization of the data
source—for example, using indexes in a relational database. More details about DirectQuery and the
possible optimizations are available in the following white paper: https://www.sqlbi.com/whitepapers/
directquery-in-analysis-services-2016/. The considerations are valid for both Power BI and Analysis
Services because they share the same underlying engine.
Understanding data refresh
DAX runs on SQL Server Analysis Services (SSAS) Tabular, Azure Analysis Services (same as SSAS in this
book), Power BI service (both on server and on the local Power BI Desktop), and in the Power Pivot for
Microsoft Excel add-in. Technically, both Power Pivot for Excel and Power BI use a customized version
of SSAS Tabular. Speaking about different engines is thus somewhat artifi cial: Power Pivot and Power BI
are like SSAS although SSAS runs in a hidden mode. In this book, we do not discriminate between these
engines; when we mention SSAS, the reader should always mentally replace SSAS with Power Pivot or
Power BI. If there are differences worth highlighting, then we will note them in that specifi c section.

550 CHAPTER 17 The DAX engines
When SSAS loads the content of a source table in memory, we say that it processes the table. This
takes place during the process operation of SSAS or during the data refresh in Power Pivot for Excel
and Power BI. The table process for DirectQuery simply clears the internal cache without executing any
access to the data source. On the other hand, when processing occurs in VertiPaq mode, the engine
reads the content of the data sources and transforms it into the internal VertiPaq data structure.
VertiPaq processes a table following these few steps:
1. Reading of the source dataset, transformation into the columnar data structure of VertiPaq,
encoding and compressing of each column.
2. Creating of dictionaries and indexes for each column.
3. Creating of the data structures for relationships.
4. Computing and compressing all the calculated columns and calculated tables.
The last two steps are not necessarily sequential. Indeed, a relationship can be based on a calculated
column, or calculated columns can depend on a relationship because they use RELATED or CALCULATE.
Therefore, SSAS creates a complex graph of dependencies to execute the steps in the correct order.
In the next sections, we describe these steps in more detail. We also cover the format of the internal
structures created by SSAS during the transformation of the data source into the VertiPaq model.
Understanding the VertiPaq storage engine
The VertiPaq engine is the most common storage engine used in Tabular models. VertiPaq is used
whenever a table is in Import storage mode. This is the common choice in many data models, and it is
the only choice in Power Pivot for Excel. In composite models, the presence of tables or aggregations in
dual storage mode also implies the use of the VertiPaq storage engine combined with DirectQuery.
For these reasons, a solid knowledge of the VertiPaq storage engine is a basic skill required to
understand how to optimize both the memory consumption of the model and the execution time of
the queries. In this section, we describe how the VertiPaq storage works.
Introducing columnar databases
VertiPaq is an in-memory columnar database. Being in-memory means that all the data handled by a
model reside in RAM. But VertiPaq is not only in-memory; it is also a columnar database. Therefore, it is
relevant to have a good understanding of what a columnar database is in order to correctly understand
VertiPaq.
We think of a table as a list of rows, where each row is divided into columns. For example, consider
the Product table in Figure 17-2.

CHAPTER 17 The DAX engines 551
ID Name Color Unit Price
1 Camcorder Red 112.25
2 Camera Red 97.50
3 Smartphone White 100.00
4 Console Black 112.25
5 TV Blue 1,240.85
6 CD Red 39.99
7 Touch screen Blue 45.12
8 PDA Black 120.25
9 Keyboard Black 120.50
Product
FIGURE 17-2 The fi gure shows the Product table, with four columns and nine rows.
Thinking of a table as a set of rows, we are using the most natural visualization of a table structure.
Technically, this is known as a row store. In a row store, data is organized in rows. When the table is
stored in memory, we might think that the value of the Name column in the fi rst row is adjacent to the
values of the ID and Color columns in the same row. On the other hand, the value in the second row of
the Name column is slightly farther from the Name value in the fi rst row because in between we fi nd
Color and Unit Price in the fi rst row, and the value of the ID column in the second row. As an example,
the following code is a schematic representation of the physical memory layout of a row store:
ID,Name,Color,Unit Price|1,Camcorder,Red,112.25|2,Camera,Red,97.50|3,Smartphone,
White,100.00|4,Console,Black,112.25|5,TV,Blue,1,240.85|6,CD,Red,39.99|7,
Touch screen,Blue,45.12|8,PDA,Black,120.25,9,Keyboard,Black,120.50
Imagine a developer needs to compute the sum of Unit Price: The engine must scan the entire
memory area, reading many irrelevant values in the process. Imagine scanning the memory of the
database sequentially: To read the fi rst value of Unit Price, the engine needs to read (and skip) the fi rst
row of ID, Name, and Color. Only then does it fi nd an interesting value. The same process is repeated
for all the rows. Following this technique, the engine needs to read and ignore many columns to fi nd
the relevant values to sum.
Reading and ignoring values take time. In fact, if we asked someone to compute the sum of Unit
Price, they would not follow that algorithm. Instead, as human beings, they would probably scan the
fi rst row in Figure 17-2 searching for the position of Unit Price, and then move their eyes down, reading
the values one at a time and mentally accumulating them to produce the sum. The reason for this very
natural behavior is that we save time by reading vertically instead of row-by-row.
A columnar database organizes data to optimize vertical scanning. To obtain this result, it needs a
way to make the different values of a column adjacent to one another. In Figure 17-3 you can see the
same Product table as organized by a columnar database.

552 CHAPTER 17 The DAX engines
Color
Red
Red
White
Black
Blue
Red
Blue
Black
Black
ID
2
4
6
8
Name
Camcorder
Camera
Smartphone
Console
TV
CD
Touch screen
PDA
Keyboard
Unit Price
112.25
97.50
100.00
112.25
1,240.85
39.99
45.12
120.25
120.50
Product Columns
FIGURE 17-3 The Product table organized column-by-column.
When stored in a columnar database, each column has its own data structure; it is physically separated from the others. Thus, the different values of Unit Price are adjacent to one another and distant
from Color, Name, and ID. The following code is a schematic representation of the physical memory
layout of a column store:

## Id,1,2,3,4,5,6,7,8,9

Name,Camcorder,Camera,Smartphone,Console,TV,CD,Touch screen,PDA,Keyboard
Color,Red,Red,White,Black,Blue,Red,Blue,Black,Black
Unit Price,112.25,97.50,100.00,112.25,1240.85,39.99,45.12,120.25,120.50
With this data structure, computing the sum of Unit Price is much easier because the engine immediately goes to the structure containing Unit Price. There, it fi nds all the values needed to perform the
computation next to each other. In other words, it does not have to read and ignore other column
values: In a single scan, it obtains exclusively the useful numbers, and it can quickly aggregate them.
In our next scenario, instead of summing Unit Price, we compute the sum of Unit Price just for the
Red products. You are encouraged to give this a try before reading on, in order to better understand
the algorithm.
This is not so easy anymore; indeed, it is no longer possible to obtain the desired number by simply
scanning the Unit Price column. What developers would typically do is scan the Color column, and
whenever it is Red, retrieve the corresponding value in Unit Price. At the end, all the values would be
summed up to compute the result.
Though very intuitive, this algorithm requires a constant move of the eyes from one column to the
other in Figure 17-3, possibly using a fi nger as a guide to save the last scanned position of Color. It is not
an optimized way of computing the value. The reason is that the engine needs to constantly jump from
one memory area to another, resulting in poor performance. A better way—which only computers
use—is to fi rst scan the Color column, fi nd the positions where the color is Red, and then scan the
Unit Price column, summing only the values in the positions identifi ed in the previous step.

CHAPTER 17 The DAX engines 553
This last algorithm is much better because it performs one scan of the fi rst column and one scan of
the second column, always accessing memory locations that are adjacent to one another—other than
the jump between the scan of the fi rst and second column. Sequential reading of memory is much
faster than random access.
For a more complex expression, such as the sum of all products that are either Blue or Black with a
price higher than US$50, things are even worse. This time, there is no possibility of scanning the column
one at a time because the condition depends on way too many columns. As usual, trying on paper
helps better understand the problem.
The simplest algorithm producing the desired result is to scan the table not on a column basis, but
on a row basis instead. We naturally tend to scan the table row-by-row, though the storage organization is column-by-column. Although it is a very simple operation when executed on paper by a human,
the same operation is extremely expensive if executed by a computer in RAM; indeed, it requires a lot
of random reads of memory, leading to poorer performance than if computed doing a sequential scan.
As discussed, a columnar storage presents both pros and cons. Columnar databases provide very
quick access to a single column; but as soon as one needs a calculation involving many columns, they
need to spend some time—after having read the column content—to reorganize the information so
that the fi nal expression can be computed. Even though this example was very simple, it helps highlight
the most important characteristics of column stores:

> **Note:** Single-column access is very fast: It sequentially reads a single block of memory and then
computes whatever aggregation is needed on that memory block.

> **Note:** If an expression uses many columns, the algorithm is more complex because it requires the
engine to access different memory areas at different times, keeping track of the progress in a
temporary area.

> **Note:** The more columns are needed to compute an expression, the harder it becomes to produce a
result. At a certain point it becomes easier to rebuild the row storage out of the column store to
compute the expression.
Column stores aim to reduce the read time. However, they spend more CPU cycles to rearrange the
data when many columns from the same table are used. Row stores, on the other hand, have a more
linear algorithm to scan data, but they result in many useless reads. As a rule, reducing reads at the
cost of increasing CPU usage is a good deal, because with modern computers, it is always easier (and
cheaper) to increase the CPU speed versus reducing I/O (or memory access) time.
Moreover, as we will see in the next sections, columnar databases have more options to reduce the
amount of time spent scanning data. The most relevant technique used by VertiPaq is compression.
Understanding VertiPaq compression
In the previous section, you learned that VertiPaq stores each column in a separate data structure. This
simple fact allows the engine to implement some extremely important compressions and encoding
described in this section.

554 CHAPTER 17 The DAX engines
Note The actual details of the compression algorithm of VertiPaq are proprietary. Thus, we
cannot publish them in a book. Yet what we explain in this chapter is already a good approximation of what takes place in the engine, and we can use it, for all intents and purposes, to
describe how the VertiPaq engine stores data.
VertiPaq compression algorithms aim to reduce the memory footprint of a data model. Reducing
the memory usage is a very important task for two very good reasons:

> **Note:** A smaller model makes better use of the hardware. Why spend money on 1 TB of RAM when the
same model, once compressed, can be hosted in 256 GB? Saving RAM is always a good option,
if feasible.

> **Note:** A smaller model is faster to scan. As simple as this rule is, it is very important when speaking
about performance. If a column is compressed, the engine will scan less RAM to read its content, resulting in better performance.
Understanding value encoding
Value encoding is the fi rst kind of encoding that VertiPaq might use to reduce the memory cost of a
column. Consider a column containing the price of products, stored as integer values. The column
contains many different values and a defi ned number of bits is required to represent all of them.
In the Figure 17-4 example, the maximum value of Unit Price is 216. At least 8 bits are required to
store each integer value up to that number. Nevertheless, by using a simple mathematical operation,
we can reduce the storage to 5 bits.
Reducing the number of bits needed
Unit Price
197
197
197
197
Unit Price - 194
3
3
3
3
Max: 216
8 bits needed
Max: 22
5 bits needed
Value Encoding
FIGURE 17-4 By using simple mathematical operations, VertiPaq reduces the number of bits needed for a column.

CHAPTER 17 The DAX engines 555
In the example, VertiPaq found out that by subtracting the minimum value (194) from all the values
of the column, it could modify the range of the values in the column, reducing it to a range from 0 to
22. Storing numbers up to 22 requires fewer bits than storing numbers up to 216. While 3 bits might
seem like an insignifi cant savings, when we multiply this by a few billion rows, it is easy to see that the
difference can be important.
The VertiPaq engine is much more sophisticated than this. It can discover mathematical relationships between the values of a column, and when it fi nds them, it can use them to modify the storage.
This reduces its memory footprint. Obviously, when using the column, it must reapply the transformation in the opposite direction to obtain the original value. Depending on the transformation, this can
happen before or after aggregating the values. Again, this increases the CPU usage and reduces the
number of reads, which is a very good option.
Value encoding only takes place for integer columns because it cannot be applied on strings or
fl oating-point values. Be mindful that VertiPaq stores the Currency data type of DAX (also called Fixed
Decimal Number) as an integer value. Therefore, currencies can be value-encoded too, whereas fl oating point numbers cannot.
Understanding hash encoding
Hash encoding (also known as dictionary encoding) is another technique used by VertiPaq to reduce
the number of bits required to store a column. Hash encoding builds a dictionary of the distinct values
of a column and then replaces the column values with indexes to the dictionary. In Figure 17-5 you can
see the storage of the Color column, which uses strings and cannot be value-encoded.
Replacing data types with dictionary and indexes
Hash Encoding
Color
Red
Red
White
Black
Blue
Red
Blue
Black
Black
ID Color
0 Red
1 White
2 Black
3 Blue
Color ID
0
2
0
2
FIGURE 17-5 Hash encoding consists of building a dictionary and replacing values with indexes.
When VertiPaq encodes a column with hash encoding, it

> **Note:** Builds a dictionary, containing the distinct values of the column.

> **Note:** Replaces the values with integer numbers, where each number is the dictionary index of the
original value.

556 CHAPTER 17 The DAX engines
There are some advantages in using hash encoding:

> **Note:** All columns only contain integer values; this makes it simpler to optimize the internal code of
the engine. Moreover, it also means that VertiPaq is data type independent.

> **Note:** The number of bits used to store a single value is the minimum number of bits necessary to
store an index entry. In the example provided, 2 bits are enough because there are only four
different values.
These two aspects are of paramount importance for VertiPaq. It does not matter whether a column
uses a string, a 64-bit integer, or a fl oating point to represent a value. All these data types can be hash
encoded, providing the same performance in terms of speed of scanning and of storage space. The
only difference might be in the size of the dictionary, which is typically very small when compared with
the size of the original column itself.
The primary factor to determine the column size is not the data type. Instead, it is the number of
distinct values of the column. We refer to the number of distinct values of a column as its cardinality.
Repeating a concept this important is always a good thing: Of all the various aspects of an individual
column, the most important one when designing a data model is its cardinality.
The lower the cardinality, the smaller the number of bits required to store a single value. Consequently, the smaller the memory footprint of the column. If a column is smaller, not only will it be possible to store more data in the same amount of RAM, but it will also be much faster to scan it whenever
the engine needs to aggregate its values in a DAX expression.
Understanding Run Length Encoding (RLE)
Hash encoding and value encoding are two very good compression techniques. However, there is
another complementary compression technique used by VertiPaq: Run Length Encoding (RLE). This
technique aims to reduce the size of a dataset by avoiding repeated values. For example, consider
a column storing in which quarter the sales took place, stored in the Sales table. This column might
contain the string “Q1” repeated many times in contiguous rows, for all the sales in the same quarter.
In such a case, VertiPaq avoids storing values that are repeated. It replaces them with a slightly more
complex structure that contains the value only once, with the number of contiguous rows having the
same value. This is shown in Figure 17-6.
RLE’s effi ciency strongly depends on the repetition pattern of the column. Some columns have the
same value repeated for many rows, resulting in a great compression ratio. Other columns with quickly
changing values produce a lower compression ratio. Data sorting is extremely important to improve
the compression ratio of RLE. Therefore, fi nding an optimal sort order is an important step of the data
refresh performed by VertiPaq.

CHAPTER 17 The DAX engines 557
Quarter
Q1
Q1
Q1
Q1
Q1
Q1
…
Q2
Q2
Q2
Q2
Q2
Q2
Q2
Q2
Q2
…
Quarter Count

## Q1 310


## Q2 290

… …
RLE
310 times
290 times
Reducing rows using Run Length Encoding (RLE)
FIGURE 17-6 RLE replaces values that are repeated with the number of contiguous rows with the same value.
Finally, there could be columns in which the content changes so often that if VertiPaq tried to
compress them using RLE, the compressed columns would end up using more space than the original
columns. A great example of this is the primary key of a table. It has a different value for each row,
resulting in an RLE version larger than the column itself. In cases like this, VertiPaq skips the RLE compression and stores the column as-is. Thus, the VertiPaq storage of a column never exceeds the original
column size. Worst-case scenario, both would be the same size.
In the example, we have shown RLE working on a Quarter column containing strings. RLE can also
process the already hash-encoded version of a column. Each column can have both RLE and either
hash or value encoding. Therefore, the VertiPaq storage for a column compressed with hash encoding
consists of two distinct entities: the dictionary and the data rows. The latter is the RLE-encoded result of
the hash-encoded version of the original column, as shown in Figure 17-7.

558 CHAPTER 17 The DAX engines
Quarter
Q1
Q1
Q1
Q1
Q2
Q2
…
Q2
Q3
Q3
Q3
Q3
Q4
Q4
Q4
Q4
…
4 unique IDs
2 bits used
Hash Encoding
Q.ID Quarter
0 Q1
1 Q2
2 Q3
3 Q4

## Q.Id

0
0
1
…
2
2
3
3
…
RLE
VertiPaq Store
Q.ID Count
0 310
1 290
2 425
3 350
Q.ID Quarter
0 Q1
1 Q2
2 Q3
3 Q4
Dictionary
Data Rows
FIGURE 17-7 RLE is applied to the dictionary-encoded version of a column.
VertiPaq also applies RLE to value-encoded columns. In this case the dictionary is missing because
the column already contains value-encoded integers.
The factors infl uencing the compression ratio of a Tabular model are, in order of importance:
1. The cardinality of the column, which defi nes the number of bits used to store a value.
2. The number of repetitions, that is, the distribution of data in a column. A column with many
repeated values is compressed more than a column with very frequently changing values.
3. The number of rows in the table.
4. The data type of the column, which only affects the dictionary size.
Given all these considerations, it is nearly impossible to predict the compression ratio of a table.
Moreover, while a developer has full control over certain aspects of a table—they can limit the number
of rows and change the data types—these are the least important aspects. Yet as you learn in the next
chapter, one can work on cardinality and repetitions too. This improves the compression and performance of a model.
Finally, it is worth noting that reducing the cardinality of a column also increases the chances of repetitions. For example, if a time column is stored at the second granularity, then the column contains up

CHAPTER 17 The DAX engines 559
to 86,400 distinct values. If, on the other hand, the developer stores the same time column at the hour
granularity, then not only have they reduced the cardinality, but they also introduced repeating values.
Indeed, 3,600 seconds convert to one same hour. All this results in a much better compression ratio.
On the other hand, changing the data type from DateTime to Integer or even String offers a negligible
impact on column size.
Understanding re-encoding
SSAS must decide which algorithm to use to encode each column. More specifi cally, it needs to decide
whether to use value or dictionary encoding. In order to make an educated decision, it reads a row
sample during the fi rst scan of the source, and it chooses a compression algorithm depending on the
values found.
If the data type of the column is not Integer, then the choice is straightforward: SSAS goes for
dictionary encoding. For integer values, it uses some heuristics, for example:

> **Note:** If the numbers in the column increase linearly, it is probably a primary key and value encoding is
the best option.

> **Note:** If all numbers fall within a defi ned range of values, then value encoding is the way to go.

> **Note:** If the numbers fall within a very wide range of values, with values very different from another,
then dictionary encoding is the best choice.
Once the decision is made, SSAS starts to compress the column using the chosen algorithm. Unfortunately, it sometimes makes the wrong decision and fi nds this out only very late during processing.
For example, SSAS might read a few million rows where the values are in the 100–201 range, so value
encoding is the best choice. After those millions of rows, suddenly an outlier appears, such as a large
number like 60,000,000. Obviously, the initial choice was wrong because the number of bits needed to
store such a large number is huge. What should SSAS do then? Instead of continuing with the wrong
choice, SSAS can decide to re-encode the column. This means that the entire column is re-encoded
using dictionary encoding. This process might take a long time because SSAS needs to reprocess the
whole column.
For very large datasets where processing time is important, a best practice is the following: the data
distribution in the fi rst set of rows read by SSAS should be of such quality that all types of values are
represented. This in turn reduces re-encoding to a minimum. Developers do so by providing a quality
sample in the fi rst partition processed or by providing an encoding hint parameter to the column.
Note The Encoding Hint property was introduced in Analysis Services 2017, and it is not
available in all products.

560 CHAPTER 17 The DAX engines
Finding the best sort order
As we said earlier, RLE’s effi ciency strongly depends on the sort order of the table. All the columns of
the same table are sorted the same way to keep integrity of the data at the table level. In large tables
it is important to determine the best sorting of data to improve the effi ciency of RLE and to reduce the
memory footprint of the model.
When SSAS reads a table, it tries different sort orders to improve the compression. In a table with
many columns, this is a very expensive operation. SSAS then sets an upper limit to the time it can spend
fi nding the best sort order. The default can change with different versions of the engine. At printing
time, the default is currently 10 seconds per million rows. One can modify its value in the ProcessingTimeboxSecPerMRow entry in the confi guration fi le of the SSAS service. Power BI and Power Pivot do
not provide access to this value.
Note SSAS searches for the best sort order in the data, using a heuristic algorithm that certainly also considers the physical order of the rows it receives. For this reason, although one
cannot force the sort order used by VertiPaq for RLE, it is possible to provide the engine with
data sorted arbitrarily. The VertiPaq engine includes this sort order in the options to consider.
To attain maximum compression, one can set the value of ProcessingTimeboxSecPerMRow to 0,
which means SSAS stops searching only when it fi nds the best compression factor. The benefi t in terms
of space usage and query speed can vary. On the other hand, processing will take much longer because
the engine is being instructed to try all the possible sort orders before making a choice.
Generally, developers should put the columns with the least number of unique values fi rst in the sort
order because these columns are likely to generate many repeating values. Still, keep in mind that fi nding the best sort order is a very complex task. It only makes sense to spend time on this when the data
model is really large (in the order of a few billion rows). Otherwise, the benefi t obtained from these
extreme optimizations is limited.
Once all the columns are compressed, SSAS completes the processing by building calculated columns, tables, hierarchies, and relationships. Hierarchies and relationships are additional data structures
needed by VertiPaq to execute queries, whereas calculated columns and tables are added to the model
by using DAX expressions.
Calculated columns, like all other columns, are compressed after they are computed. However, calculated columns are not the same as standard columns. Calculated columns are compressed during the
fi nal stage of processing, when all the other columns have already fi nished their compression. Consequently, VertiPaq does not consider calculated columns when choosing the best sort order for a table.
Consider creating a calculated column that results in a Boolean value. There being only two values,
the calculated column can be compressed very well (1 bit is enough to store a Boolean value), and it is
a very good candidate to be fi rst in the sort order list. Indeed, doing this, the table shows all the True

CHAPTER 17 The DAX engines 561
values fi rst and only later the False values. Being a calculated column, the sort order is already defi ned
by other columns; it might be the case that with the defi ned sort order, the calculated column frequently changes its value. In that case, the column ends up with less-than-optimal compression.
Whenever there is a chance to compute a column in DAX or in the data source (including Power
Query), keep in mind that computing it in the data source results in slightly better compression. Many
other factors may drive the choice of DAX instead of Power Query or SQL to calculate the column. For
example, the engine automatically computes a calculated column in a large table depending on a column in a small table, whenever said small table has a partial or full refresh. This happens without having
to reprocess the entire large table, which would be necessary if the computation were in Power Query
or SQL. This is something to consider when looking for the optimal compression.

Note A calculated table has the same compression as a regular table, without the side
effects described for calculated columns. However, creating a calculated table can be quite
expensive. Indeed, a calculated table requires enough memory to keep a copy of the entire
uncompressed table in memory before it is compressed. Carefully think before creating a
large calculated table because of the memory pressure generated at refresh time.
Understanding hierarchies and relationships
As we said in the previous sections, at the end of table processing, SSAS builds two additional data
structures: hierarchies and relationships.
There are two types of hierarchies: attribute hierarchies and user hierarchies. Hierarchies are data
structures used primarily to improve performance of MDX queries and also to improve certain search
operations in DAX. Because the concept of hierarchy is not present in the DAX language, hierarchies
are not relevant to the topics of this book.
Relationships, on the other hand, play an important role in the VertiPaq engine; it is important to
understand how they work for extreme optimizations. We will describe the role of relationships in a
query in following chapters. Here, we are only interested in defi ning what relationships are, in terms of
VertiPaq storage and behavior.
A relationship is a data structure that maps IDs from one table to row numbers in another table. For
example, consider the columns ProductKey in Sales and ProductKey in Product. These two columns are
used to build the relationship between the two tables. Product[ProductKey] is a primary key. Because
it is a primary key, the engine used value encoding and no compression at all. Indeed, RLE could not
reduce the size of a column in the absence of duplicated values. On the other hand, Sales[ProductKey]
is likely to have been dictionary-encoded and compressed. This is because it probably contains many
repetitions. Therefore, despite the columns having the same name and data type, their internal data
structures are completely different.

562 CHAPTER 17 The DAX engines
Moreover, because they are part of a relationship, VertiPaq knows that queries are likely to use the
columns very often placing a fi lter on Product and also expecting to fi lter Sales. VertiPaq would be very
slow if—every time it needs to move a fi lter from Product to Sales—it had to perform the following:
retrieve values from Product[ProductKey], search them in the dictionary of Sales[ProductKey], and fi nally
retrieve the IDs of Sales[ProductKey] to place the fi lter.
Therefore, to improve query performance, VertiPaq stores relationships as pairs of IDs and row
numbers. Given the ID of a Sales[ProductKey], it can immediately fi nd the corresponding rows of
Product that match the relationship. Relationships are stored in memory, as any other data structure of
VertiPaq. Figure 17-8 shows how the relationship between Sales and Product is stored in VertiPaq.
Amount ProductKey
25.00 1
12.50 2
2.25 3
2.50 3
14.00 4
25.00 5
Relationship
ProductKey Product
1 Coffee
2 Pasta
3 Tomato

## Blank Blank

Row Num
2
4
Sales[ProductKey] Product[Row Num]
1 1
2 2
3 3
4 4
5 4
Sales Product
FIGURE 17-8 The fi gure shows the relationship between Sales and Product.
Even though the structure does not seem to be very intuitive, later in this chapter we describe how
VertiPaq uses relationships and why relationships have this very specifi c structure. It would come
naturally that it is a complex structure optimized for performance.
Understanding segmentation and partitioning
Compressing a table of several billion rows in one single step would be extremely memory-intensive
and time-consuming. Therefore, the table is not processed as a single unit. Instead, during processing, SSAS splits the table into segments that contain 8 million rows each by default. When a segment
is completely read, the engine starts to compress the segment while reading the next segment in the
meantime.
It is possible to confi gure the segment size in SSAS using the DefaultSegmentRowCount entry in the
confi guration fi le of the service (or in the server properties in Management Studio). In Power BI Desktop and Power Pivot, the segment size has a set value of 1 million rows, and it cannot be changed.

CHAPTER 17 The DAX engines 563
Segmentation is important for several reasons, including query parallelisms and compression effi ciency. When querying a table, VertiPaq uses the segments as the basis for parallelism: It uses one core
per segment when scanning a column. By default, SSAS always uses one single thread to scan a table
with 8 million rows or less. We start observing parallelism in action only on much larger tables.
The larger the segment, the better the compression. Having the option of analyzing more rows in
a single compression step, VertiPaq can achieve better compression levels. On very large tables, it is
important to test different segment sizes and measure the memory usage to achieve optimal compression. Keep in mind that increasing the segment size can negatively affect processing time: The larger
the segment, the slower the processing.
Although the dictionary is global to the table, bit-sizing takes place at the segment level. Thus, if a
column has 1,000 distinct values but only two distinct values are used in a specifi c segment, then that
column will be compressed to a single bit for that segment.
If segments are small, then the parallelism at query time is increased. This is not always a good thing.
While it is true that scanning the column is faster because more cores can do that in parallel, VertiPaq
needs more time at the end of the scan to aggregate partial results computed by the different threads.
If a partition is too small, then the time required for managing task switching and fi nal aggregation is
more than the time needed to scan the data, with a negative impact on the overall query performance.
During processing, the treatment of the fi rst segment is particular if the table has only one partition.
Indeed, the fi rst segment can be larger than DefaultSegmentRowCount. VertiPaq reads twice the size
of DefaultSegmentRowCount and starts to segment a table only if the table contains more rows. This
does not apply to a partitioned table. If a table is partitioned, then all the segments are smaller than the
default segment row count. Consequently, in SSAS a nonpartitioned table with 10 million rows is stored
as a single segment. On the other hand, a table with 20 million rows uses three segments: two containing 8 million rows and one containing 4 million rows. In Power BI Desktop and Power Pivot, VertiPaq
uses multiple segments for tables with more than 2 million rows.
Segments cannot exceed the partition size. If the partitioning schema of a model creates partitions
of only 1 million rows, then all the segments will be smaller than 8 million rows; namely, they will be
same as the partition size. Overpartitioning a table is a common mistake made by novices to optimize
performance. What they obtain is the opposite effect: Creating too many small partitions typically
lowers performance.
Using Dynamic Management Views
SSAS enables the discovery of all the information about the data model using Dynamic Management
Views (DMV). DMVs are extremely useful to explore how a model is compressed, the space used by
different columns and tables, the number of segments in a table, or the number of bits used by columns in different segments.
DMVs can run from inside SQL Server Management Studio. Regardless, we suggest you use DAX
Studio; it offers a list of all DMVs in a simpler way without the need to remember them or to reopen this

564 CHAPTER 17 The DAX engines
book looking for the DMV name. However, a more effi cient way to use DMVs is with the free VertiPaq
Analyzer tool (http://www.sqlbi.com/tools/vertipaq-analyzer/), which displays data from DMVs and
organizes them in useful reports, as shown in Figure 17-9.
FIGURE 17-9 VertiPaq Analyzer shows statistics about a data model in an effi cient manner.
Although DMVs use an SQL-like syntax, the full SQL syntax is not available. DMVs do not run inside
SQL Server. They are only a convenient way to discover the status of SSAS and to gather information
about data models.
There are different DMVs, divided into two main categories:

> **Note:** SCHEMA views: These return information about SSAS metadata, such as database names,
tables, and individual columns. They are used to gather information about data types, names,
and similar data, including statistical information about numbers of rows and unique values
stored in columns.

> **Note:** DISCOVER views: They are intended to gather information about the SSAS engine and/or discover statistics information about objects in a database. For example, one can use views in the
discover area to enumerate the DAX keywords, the number of connections and sessions that are
currently open, or the traces running.
In this book, we do not describe the details of all the views because doing so would be going off
topic. More information is available in Microsoft documentation on the web. Instead, we want to
provide a few hints and point out the most useful DMVs related to databases used by DAX. Moreover,
while many DMVs report useful information in many columns, in this book we describe the most
interesting ones related to the internal structure.
A fi rst useful DMV to discover the memory usage of all the objects in the SSAS instance is DISCOVER_OBJECT_MEMORY_USAGE. This DMV returns information about all the objects in all the databases in the SSAS instance. DISCOVER_OBJECT_MEMORY_USAGE is not limited to the current database.
For example, the following query can be run in DAX Studio or SQL Server Management Studio:

## Select * From $System.Discover_Object_Memory_Usage

Figure 17-10 shows a small excerpt of the result of the previous query. There are many more columns
and rows, so analyzing this detailed information can be very time-consuming.

CHAPTER 17 The DAX engines 565
FIGURE 17-10 Partial result of the DISCOVER_OBJECT_MEMORY_USAGE DMV.
The output of the DMV is a table containing many rows that are very hard to read. The output structure is a parent/child hierarchy that starts with the instance name and ends with individual column information. Although the raw dataset is nearly impossible to read, one can
build a Power Pivot data model on top of this query, implementing the parent/child hierarchy structure and browsing the full memory map of the instance. Kasper De Jonge published
a workbook on his blog that does exactly this. It is available at http://www.powerpivotblog.nl/
what-is-using-all-that-memory-on-my-analysis-server-instance/.
Other useful DMVs to check the current state of the Tabular engine are DISCOVER_SESSIONS, DISCOVER_CONNECTIONS, and DISCOVER_COMMANDS. These DMVs provide information about active
sessions, connections, and executed commands. These views are used by an open source tool called
SSAS Activity Monitor, available at https://github.com/RichieBzzzt/SSASActivityMonitor/tree/master/
Download, that provides the same information (plus much more) in a more convenient way.
There are also DMVs that analyze the distribution of data in columns and tables, and the memory
required for compressed data. These are TMSCHEMA_COLUMN_STORAGES and DISCOVER_STORAGE_TABLE_COLUMNS. The former is the more recent one; the latter is there for compatibility with
older versions of the engine (compatibility level 1103 or lower).
Finally, a very useful DMV to analyze calculation dependency is DISCOVER_CALC_DEPENDENCY.
This DMV can be used to create a graph of dependencies between calculations in the data model,
including calculated columns, calculated tables, and measures. Figure 17-11 shows an excerpt of the
result of this DMV.
FIGURE 17-11 Partial result of the DISCOVER_CALC_DEPENDENCY DMV.
Understanding the use of relationships in VertiPaq
When a DAX query generates requests to the VertiPaq storage engine, the presence of relationships
in the data model allows a quicker transfer of the fi lter context from one table to another. The internal
implementation of a relationship in VertiPaq is worth knowing because relationships might affect the
performance of a query even though most of the calculation happens in the storage engine.

566 CHAPTER 17 The DAX engines
To understand how relationships work, we start from the analysis of a query that only involves one
table, Sales:

## Evaluate


## Row (


```dax
"Result", CALCULATE (
COUNTROWS ( Sales ),
Sales[Quantity] > 1
)
)
```

-- Result
-- 20016
A developer used to working with tables in relational databases might suppose that the engine
iterates the Sales table, tests the value of the Quantity column for each row of Sales, and increments
the returned value if the Quantity value is greater than 1. In fact, VertiPaq does it better: VertiPaq only
scans the Quantity column because it already provides the number of rows for the entire table. Therefore, a single column scan is enough to solve the entire query.
If we write a similar query using the column of another table as a fi lter, then scanning a single column is no longer enough to produce the result. For example, consider the following query that counts
the number of rows in Sales related to products of the Contoso brand:

## Evaluate


## Row (


```dax
"Result", CALCULATE (
COUNTROWS ( Sales ),
'Product'[Brand] = "Contoso"
)
)
```

-- Result
-- 37984
This time, we are using two different tables: Sales and Product. Solving this query requires a bit more
effort. Indeed, because the fi lter is on Product and the table to aggregate is Sales, it is not possible to
scan a single column.
If you are not used to columnar databases, you probably think that, to solve the query, the engine
should iterate the Sales table, follow the relationship with Product, and sum 1 if the product brand is
Contoso, 0 otherwise. This would be an algorithm like the following DAX code:

## Evaluate


## Row (

"Result", SUMX (
Sales,
IF ( RELATED ( 'Product'[Brand] ) = "Contoso", 1, 0 )
)

CHAPTER 17 The DAX engines 567
)
-- Result
-- 37984
Although this is a simple algorithm, it contains much more complexity than expected. Indeed, if we
carefully think about the columnar nature of VertiPaq, we realize that this query involves three different
columns:

> **Note:** Product[Brand] used to fi lter the Product table.

> **Note:** Product[ProductKey] used by the relationship between Product and Sales.

> **Note:** Sales[ProductKey] used on the Sales side of the relationship.
Iterating over Sales[ProductKey], searching the row number in Product scanning Product[ProductKey],
and fi nally gathering the brand in Product[Brand] would be extremely expensive. The process requires a
lot of random reads to memory, with negative consequences on performance. Therefore, VertiPaq uses
a completely different algorithm, optimized for columnar databases.
First, VertiPaq scans the Product[Brand] column and retrieves the row numbers of the Product table
where Product[Brand] is Contoso. As shown in Figure 17-12, VertiPaq scans the Brand dictionary (1),
retrieves the encoding of Contoso, and fi nally scans the segments (2) searching for the row numbers in
the product table where the dictionary ID equals 0 (corresponding to Contoso), returning the indexes
to the rows found (3).
Search «Contoso»
Output row numbers
Row
3
…
1 2
Product[Brand]
Row ID
1 2
2 0
3 0
4 1
… …
ID Brand
0 Contoso
1 Fabrikam
2 Proseware
3 Tailspin Toys
Dictionary
Data Rows
FIGURE 17-12 The output of a brand scan is the list of rows where Brand equals Contoso.

568 CHAPTER 17 The DAX engines
At this point, VertiPaq knows which rows in the Product table contain the given brand. The relationship between Product and Sales enables VertiPaq to translate the row numbers of Product in internal
data IDs for Sales[ProductKey]. VertiPaq performs a lookup of the selected row numbers to determine
the values of Sales[ProductKey] valid for those rows, as shown in Figure 17-13.
Look up row number Output IDs
ID
Row Num
3
…
Product[RowNumber]
Sales[ProductKey]
Relationship
Product
Row Num
Sales
[ProductKey]
1 1
2 5
3 8
4 7
5 6
6 100
7 111
8 87
9 54
… …
FIGURE 17-13 VertiPaq scans the product keys in the relationship to retrieve the IDs where brand equals Contoso.
The last step is to apply the fi lter on the Sales table. Since VertiPaq already has the list of values of
Sales[ProductKey], it is enough to scan the Sales[ProductKey] column to transform this list of values into
row numbers and fi nally count them. If, instead of computing a COUNTROWS, VertiPaq had to perform
the SUM of a column, then it would perform an additional step transforming row numbers into column
values to perform the last step.
The important takeaway is that the cost of a relationship depends on the cardinality of the column
that defi nes the relationship. Even though the previous query fi ltered only one brand, the cost of the
relationship was the number of products for that brand. The lower the cardinality of a relationship,
the better. When the cardinality of a relationship is above one million unique values, the end user can
experience slower performance. A performance degradation is already measurable when the relationship has 100,000 unique values. VertiPaq aggregations can mitigate the impact of high-cardinality relationships by pre-aggregating data at a different granularity, removing the cost of traversing expensive
relationships at query time. We briefl y discuss aggregations later in this chapter.
Introducing materialization
Now that we have provided a basic explanation of how VertiPaq stores data in memory, we can
describe what materialization is. Materialization is a step of the query execution that occurs when using
columnar databases. Understanding when and how it happens is of paramount importance.

CHAPTER 17 The DAX engines 569
The basic principle about materialization is that every time the formula engine sends a request to
the storage engine, the formula engine receives an uncompressed table that is generated dynamically
by the storage engine. This special temporary table is called a datacache. A datacache is always the
materialization of data that will be consumed by the formula engine, regardless of the storage engine
used. Both VertiPaq and DirectQuery generate datacaches.
A large materialization happens when a single storage engine query produces a large datacache.
The conditions for a DAX query to produce a large materialization depend on many factors; basically,
whenever the storage engine is not able to execute all the operations required by the DAX query, the
formula engine will do the work using a copy of the data owned by the storage engine. Be mindful that
the formula engine cannot access the raw data directly, whether VertiPaq or DirectQuery. To access the
raw data, the formula engine needs to ask the storage engine to retrieve the data and save it in a datacache. The amount and kind of materialization can be very different depending on the storage engine
used. In this book, we only describe how to reduce the materialization in VertiPaq. For DirectQuery
there could be differences between different data source drivers. Even so, the tools used to measure
the materialization produced by the storage engine are the same used for VertiPaq.
The next chapters describe how to measure the materialization produced by a DAX query using specifi c tools and metrics. In this section, we just introduce the concept of materialization and how it relates
to the result of a query. The cardinality of the result of every DAX query defi nes the optimal materialization. For example, the following query returns a single row, counting the number of rows in a table:

## Evaluate


## Row (

"Result", COUNTROWS ( Sales )
)
-- Result
-- 100231
The optimal materialization for the previous query is a datacache with only one row. This means that
the entire calculation is performed within the storage engine. The next query returns one row for each
year; therefore, the optimal materialization is three rows, one for each year with sales:

## Evaluate


## Summarizecolumns (

'Date'[Calendar Year],
"Sales Amount", [Sales Amount]
)
-- Calendar Year | Sales Amount
-----------------|---------------
-- CY 2007 | 11,309,946.12
-- CY 2008 | 9,927,582.99
-- CY 2009 | 9,353,814.87
Whenever the storage engine produces a single datacache with the same cardinality as the result
of the DAX query, that is called a late materialization. If the storage engine produces more datacaches
and/or the datacache produced has more rows than those displayed in the result, we have an early

570 CHAPTER 17 The DAX engines
materialization. With a late materialization the formula engine does not have to aggregate data,
whereas with an early materialization the formula engine must perform operations like joining and
grouping, which result in slower queries for the end users.
Predicting materialization is not easy without a deep knowledge of the VertiPaq engine. For example, the materialization of the following query is optimal because the entire calculation is executed
within the storage engine:

## Evaluate


```dax
VAR LargeOrders =
CALCULATETABLE (
DISTINCT ( Sales[Order Number] ),
Sales[Quantity] > 1
)
VAR Result =
ROW (
"Orders", COUNTROWS ( LargeOrders )
)
RETURN
Result
```

-- Orders
-- 8388
On the other hand, the next query creates a temporary table that corresponds to the number of
unique combinations between customers and dates related to sales with a quantity greater than one
(for a total of 6,290 combinations):

## Evaluate


```dax
VAR LargeSalesCustomerDates =
CALCULATETABLE (
SUMMARIZE ( Sales, Sales[CustomerKey], Sales[Order Date] ),
Sales[Quantity] > 1
)
VAR Result =
ROW (
"CustomerDates", COUNTROWS ( LargeSalesCustomerDates )
)
RETURN
Result
```

-- CustomerDates
-- 6290
The latter query has a materialization of 6,290 rows, even though there is only one row in the result.
The two queries are similar: a table is evaluated and then its rows are counted. The reason why the
former has an earlier materialization is because it involves a single column, whereas the calculation
requiring the combinations of two columns cannot be solved by the storage engine by just scanning
the two columns. In general, any operation involving a single column has higher chances of being
solved in the storage engine, but it would be a mistake to believe that involving multiple columns is

CHAPTER 17 The DAX engines 571
always an issue. For example, the following query has an optimal late materialization even though it
multiplies two columns from two tables, Sales and Product:

## Define


```dax
MEASURE Sales[Sales Amount] =
SUMX (
Sales,
Sales[Quantity] * RELATED ( 'Product'[Unit Price] )
)
EVALUATE
ROW ( "Sales Amount", [Sales Amount] )
```

-- Sales Amount
-- 33,690,148.51
In complex queries it is nearly impossible to obtain an optimal late materialization. Therefore, the
effort for optimizing a query is reducing the materialization, pushing most of the workload to the storage engine, if possible.
Introducing aggregations
A data model can have multiple tables related to the same original raw data. The purpose of this
redundancy is to offer alternative ways to the storage engine to retrieve the data faster. The tables used
to this purpose are called aggregations.
An aggregation is nothing but a pregrouped version of the original table. By pre-aggregating data, one
reduces the number of columns (hence, the number of rows) and replaces values with their aggregate.
As an example, consider the Sales table in Figure 17-14, which has one row for each date, product,
and customer.
Date Product Customer Quantity Amount
2018-09-01 AV010 C092 3 29.97
2018-09-01 AV022 C092 1 16.40
2018-09-01 AV010 C054 2 19.98
2018-09-01 FL892 C248 1 190.00
2018-09-01 GT400 C127 1 999.00
2018-09-02 AV010 C115 3 29.97
2018-09-02 FL580 C127 1 790.00
2018-09-02 AV022 C772 2 32.80
2018-09-02 KB723 C614 2 59.98
2018-09-02 FL580 C614 1 790.00

## … …… … …

Sales
FIGURE 17-14 The original Sales table has a high number of rows.

572 CHAPTER 17 The DAX engines
If a query requires the sum of Quantity or Amount by Date, the storage engine must evaluate and
aggregate all the rows with the same Date. In VertiPaq this operation is relatively quick, thanks to the
compression and the optimized algorithms that scan the memory. DirectQuery is usually much slower
than VertiPaq to perform the same operation. Anyway, VertiPaq also requires time to scan billions of
rows rather than millions of rows. Therefore, there could be an advantage in creating an alternate—
smaller—table to use in place of the original one.
Figure 17-15 shows the content of a Sales table aggregated by Date. In this case, there is only one
row for every date, and the Quantity and Amount columns store the sum of the values included in the
original rows, pre-aggregated by Date.
Date Quantity Amount
2018-09-01 8 1,255.35
2018-09-02 9 1,702.75
… …
Sales Agg Date
…
FIGURE 17-15 The Sales Agg Date table has one row for every date.
In an aggregated table, every column is either a “group by” or an aggregation of the original table.
If a request to the storage engine only needs columns that are present in an aggregation table, then
the engine uses the aggregation rather than the original source. The Sales Agg Date table shown in
Figure 17-15 can be mapped as an aggregation of Sales by specifying the role of each column:

> **Note:** Date: GroupBy Sales[Date]

> **Note:** Quantity: Sum Sales[Quantity]

> **Note:** Amount: Sum Sales[Amount]
The aggregation type must be specifi ed for every column that is not a “group by.” The aggregation
types available are Count, Min, Max, Sum, and count rows of the table. A column in an aggregation
table can only map native columns in the original table; it is not possible to specify an aggregation over
a calculated column.

Important Aggregations cannot be used to optimize the execution of complex calculations
in DAX. The only purpose of aggregations is to reduce the execution time of storage engine
queries. Aggregations can be useful for relatively small tables in DirectQuery, whereas
aggregations for VertiPaq should be considered only for tables with billions of rows.
A table in a Tabular model can have multiple aggregations with different priorities in case there are
multiple aggregations compatible with a specifi c storage engine request. Moreover, aggregations and
original tables can be stored with different storage engines. A common scenario is storing aggregations in VertiPaq to improve the performance of large tables accessed through DirectQuery. Nevertheless, it is also possible to create aggregations in the same storage engine used for the original table.

CHAPTER 17 The DAX engines 573
Note There could be limitations in storage engines available for aggregations and original
tables, depending on the version and the license of the product used. This section provides
general guidance on the concept of aggregations, which are one of the tools to optimize
performance of a DAX query as described in the following chapters.
Aggregations are powerful, but they require a lot of attention to detail. An incorrect defi nition of
aggregations produces incorrect or inconsistent results. It is a responsibility of the data modeler to
guarantee that a query executed in an aggregation produces the same result as an equivalent query
executed on the original table. Aggregations are an optimization tool and should be used only whenever strictly necessary. The presence of aggregations requires additional work to defi ne and maintain
the aggregation tables in the data model. One should therefore use them only after having checked
that a performance benefi t exists.
Choosing hardware for VertiPaq
Choosing the right hardware is critical for a solution based on a Tabular model using the VertiPaq storage engine. Spending more does not always mean having a better machine. This section describes how
to choose the right hardware for a Tabular model.
Since the introduction of Analysis Services 2012, we helped several companies adopt the new Tabular model in their solutions. A very common issue was that when going into production, performance
was slower than expected. Worse, sometimes it was slower than in the development environments.
Most of the times, the reason for that was incorrect hardware sizing, especially when the server was in
a virtualized environment. As we will explain, the problem is not the use of a virtual machine in itself.
Instead, the problem is more likely the technical specs of the underlying hardware. A very complete
and detailed hardware-sizing guide for Analysis Services Tabular is available in the whitepaper titled
“Hardware Sizing a Tabular Solution (SQL Server Analysis Services)” (http://msdn.microsoft.com/en-us/
library/jj874401.aspx). The goal of this section is to provide a quick guide to understand the issues
affecting many data centers when they host a Tabular solution. Users of Power Pivot or Power BI Desktop on a personal computer can skip the details about Non-Uniform Memory Access (NUMA) support,
but all the other considerations are equally true for choosing the right hardware.
Hardware choice as an option
The fi rst question is whether one can choose their hardware or not. The problem of using a virtual
machine for a Tabular solution is that often the hardware has already been selected and installed. One
can only infl uence the number of cores and the amount of RAM that are assigned to the server. Unfortunately, these parameters are not so relevant for performance. If there are limited choices available,
one should collect information about the CPU model and clock of the host server as soon as possible.
If this information is not accessible, ask for a small virtual machine running on the same host server
and run the Task Manager: The Performance tab shows the CPU model and the clock rate. With this

574 CHAPTER 17 The DAX engines
information, one can predict whether the performance will be worse than an average modern laptop.
Unfortunately, chances are that many developers will be in that position. If so, then they must sharpen
their political skills to convince the right people that running Tabular on that server is a bad idea. If the
host server is a good machine, then one still needs to avoid the pitfall of running a virtual machine on
different NUMA nodes (more on this later).
Set hardware priorities
If it is possible to infl uence the hardware selection, this is the order of priorities:
1. CPU Clock and Model: the faster, the better.
2. Memory Speed: the faster, the better.
3. Number of Cores: the higher, the better. Still, a few fast cores are way better than many
slow cores.
4. Memory Size.
Disk I/O performance is not on the list. Indeed, it is not important at query time although it could
have a role in improving the speed of a disaster recovery. There is only one condition (paging) where
disk I/O affects performance, and we discuss it later in this section. However, the RAM of the system
should be sized so that there will be no paging at all. Our reader should allocate the budget on CPU
and memory speed, memory size, and not waste money on disk I/O bandwidth. The following sections
include information to consider for such allocation.
CPU model
The most important factors that affect the speed of code running in VertiPaq are CPU clock and model.
Different CPU models might have a different performance at the same clock rate, so considering the
clock alone is not enough. The best practice is to run a benchmark measuring the different performance in queries that stress the formula engine. An example of such a query is the following:

## Define


```dax
VAR t1 =
SELECTCOLUMNS ( CALENDAR ( 1, 10000 ), "x", [Date] )
VAR t2 =
SELECTCOLUMNS ( CALENDAR ( 1, 10000 ), "y", [Date] )
VAR c =
CROSSJOIN ( t1, t2 )
VAR result =
COUNTROWS ( c )
EVALUATE
ROW ( "x", result )
```

This query can run in DAX Studio or SQL Server Management Studio connected to any Tabular
model; the execution is intentionally slow and does not produce any meaningful result. Using a query
of a typical workload for a specifi c data model is certainly better because performance might vary on

CHAPTER 17 The DAX engines 575
different hardware depending on the memory allocated to materialize intermediate results; the query
in the preceding code block has a minimal use of memory.
For example, this query runs in 9.5 seconds on an Intel i7-4770K 3.5 GHz, and in 14.4 seconds on
an Intel i7-6500U 2.5 GHz. These CPUs run a desktop workstation and a notebook, respectively. Do
not assume that a server will be faster. You should always evaluate hardware performance by running
the same test with the same version of the engine and looking at the results because they are often
surprising.
In general, Intel Xeon processors used on a server are E5 and E7 series, and it is common to fi nd
clock speed around 2–2.4 GHz even with a very high number of cores available. You should look for a
clock speed of 3 GHz or more. Another important factor is the L2 and L3 cache size: The larger, the better. This is especially important for large tables and relationships between tables based on columns that
have more than 1 million unique values.
The reason why CPU and cache are so important for VertiPaq is clarifi ed in Table 17-1, which compares the typical access time of data stored at different distances from the CPU. The column with
human metrics represents the same difference using metrics that are easier for humans to understand.
TABLE 17-1 Expanded versions of the tables
Access Access Time Human Metrics
1 CPU cycle 0.3 ns 1 s
L1 cache 0.9 ns 3 s
L2 cache 2.8 ns 9 s
L3 cache 12.9 ns 43 s
RAM access 120 ns 6 min
Solid-state disk I/O 50–150 μs s 2–6 days
Rotational disk I/O 1–10 ms 1–12 months
As shown here, the fastest storage in a PC is not the RAM; it is the core cache. It should be clear that
a large L2 cache is important, and the CPU speed plays a primary role in determining performance.
The same table also clarifi es why keeping data in RAM is so much better than accessing data in other,
slower storage devices.
Memory speed
The memory speed is an important factor for VertiPaq. Every operation made by the engine accesses
memory at a very high speed. When the RAM bandwidth is the bottleneck, performance counters
report CPU usage instead of I/O waits. Unfortunately, there are no performance counters that monitor
the time spent waiting for the RAM access. In Tabular, this amount of time can be relevant, and it is hard
to measure.

576 CHAPTER 17 The DAX engines
In general, you should use RAM that has at least 1,833 MHz; however, if the hardware platform
permits, you should select faster RAM—2,133 MHz or more.
Number of cores
VertiPaq splits execution on multiple threads only when the table involved has multiple segments. Each
segment contains 8 million rows by default (1 million on Power BI and Power Pivot). A CPU with eight
cores will not use all of them in a single query unless a table has at least 64 million rows, or 8 million
rows in Power BI and Power Pivot.
For these reasons, scalability over multiple cores is effective only for very large tables. Raising the
number of cores improves performance for a single query only when it hits a large table, 200 million
rows or more. In terms of scalability (number of concurrent users), a higher number of cores might not
improve performance if users access the same tables as they would contend access to shared RAM.
A better way to increase the number of concurrent users is to use more servers in a load-balancing
confi guration.
The best practice is to get the maximum number of cores available on a single socket, getting the
highest clock rate possible. Having two or more sockets on the same server is not good, even though
Analysis Services Tabular recognizes the NUMA architecture. NUMA requires a more expensive intersocket communication whenever a thread running on a socket accesses memory allocated by another
socket. You can fi nd more details about NUMA architecture in Hardware Sizing a Tabular Solution (SQL
Server Analysis Services) at http://msdn.microsoft.com/en-us/library/jj874401.aspx.
Memory size
The entire volume of data managed by VertiPaq must be stored in memory. Additional RAM is required
to execute process operations—unless there is a separate process server—and to execute queries.
Optimized queries usually do not have a high request for RAM, but a single query can materialize temporary tables that could be very large. Database tables have a high compression rate, whereas materialization of intermediate tables during a single query generates uncompressed data.
Having enough memory only guarantees that a query will end by returning a result, but increasing available RAM does not produce any performance improvement. Cache used by Tabular does
not increase just because there is more RAM available. However, a condition of low available memory
might negatively affect query performance if the server starts paging data. Developers should have
enough memory to store all the data of their database and to avoid materialization during query
execution. More memory than this is a waste of resources.
Disk I/O and paging
You should not allocate budget on storage I/O for Analysis Services Tabular. This is very different from
Multidimensional, where random I/O operation on disk occurs very frequently, especially in certain
measures. In Tabular, there are no direct storage I/O operations during a query. The only event when

CHAPTER 17 The DAX engines 577
this might happen is under low memory conditions. However, it is less expensive and more effective to
provide more RAM to a server than trying to improve performance by increasing storage I/O throughput when there is systematic paging caused by low memory availability.
Best practices in hardware selection
You should measure performance before choosing the hardware for SSAS Tabular. It is common to
observe a server running twice as slow as a development workstation, even if the server is very new.
This is because a server designed to be scalable—especially for virtual machines—does not usually
perform very well for activities made by a single thread. However, this type of workload is very common in VertiPaq. One will need time and numbers, doing a proper benchmark, to convince a company
that a “standard server” could be the weak point of their entire BI solution.
Conclusions
In this fi rst chapter about optimization we described the internal architecture of a Tabular engine, and
we provided the basic information about how data is stored in VertiPaq. As you will see in the following
chapters, this knowledge is of paramount importance to optimize your code.
These are the main topics you learned in the chapter:

> **Note:** There are two engines inside a Tabular server: the formula engine and storage engine.

> **Note:** The formula engine is the top-level query engine. It is very powerful but rather limited in terms
of speed because it is single-threaded.

> **Note:** There are two storage engines: VertiPaq and DirectQuery.

> **Note:** VertiPaq is an in-memory columnar database. It stores information on a column-by-column
basis, providing very quick access to single columns. Using multiple columns in a single DAX
formula might require materialization.

> **Note:** VertiPaq compresses columns to reduce the memory scan time. Optimizing a model means
optimizing the compression by reducing the cardinality of a column as much as possible.

> **Note:** Both VertiPaq and DirectQuery storage engines can coexist in the same model; this is called a
composite model. A single query can use only VertiPaq, only DirectQuery, or both, depending
on the storage model of the tables involved in the query.
Now that we have provided the basic knowledge about the internals of the engine, in the next
chapter we start learning a few techniques to optimize VertiPaq storage to reduce both the size of a
data model and its execution time.