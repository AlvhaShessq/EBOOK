# The Definitive Guide to DAX — Reference Index
**Authors:** Marco Russo & Alberto Ferrari  
**Edition:** 2nd Edition (2019)  
**Platform:** Microsoft Power BI, SQL Server Analysis Services, Excel  
**Purpose:** Enterprise DAX Reference for Power BI Development

---

## Quick Navigation by Topic

| Topic | Chapter |
|-------|---------|
| DAX Fundamentals & Language Overview | Ch 1–2 |
| Table Functions (FILTER, ALL, VALUES) | Ch 3 |
| Evaluation Contexts (Row & Filter) | Ch 4 |
| CALCULATE & CALCULATETABLE | Ch 5 |
| Variables (VAR/RETURN) | Ch 6 |
| Iterators (SUMX, RANKX, etc.) | Ch 7 |
| Time Intelligence | Ch 8 |
| Calculation Groups | Ch 9 |
| Advanced Filter Context | Ch 10 |
| Hierarchies | Ch 11 |
| Advanced Table Functions | Ch 12–13 |
| Advanced DAX Concepts (lineage, TREATAS) | Ch 14 |
| Advanced Relationships | Ch 15 |
| Advanced Calculations & Patterns | Ch 16 |
| DAX Engine Internals (FE/SE/VertiPaq) | Ch 17 |
| VertiPaq Optimization | Ch 18 |
| Query Plan Analysis | Ch 19 |
| DAX Performance Optimization | Ch 20 |

---

## Table of Contents

### Chapter 01: What is DAX?
**File:** `ch01_what_is_dax?.md`  
**Topics:** Introduction to DAX language, comparison with Excel/SQL/MDX

### Chapter 02: Introducing DAX
**File:** `ch02_introducing_dax.md`  
**Topics:** Data types, operators, calculated columns, measures, variables, error handling, aggregators

### Chapter 03: Using Basic Table Functions
**File:** `ch03_using_basic_table_functions.md`  
**Topics:** FILTER, ALL, ALLEXCEPT, VALUES, DISTINCT, ALLSELECTED, EVALUATE

### Chapter 04: Understanding Evaluation Contexts
**File:** `ch04_understanding_evaluation_contexts.md`  
**Topics:** Filter context, row context, EARLIER, context interactions, relationships

### Chapter 05: Understanding CALCULATE and CALCULATETABLE
**File:** `ch05_understanding_calculate_and.md`  
**Topics:** CALCULATE, CALCULATETABLE, KEEPFILTERS, context transition, circular dependencies, modifiers

### Chapter 06: Variables
**File:** `ch06_variables.md`  
**Topics:** VAR/RETURN syntax, readability, performance, scoping rules

### Chapter 07: Working with Iterators and CALCULATE
**File:** `ch07_working_with_iterators_and_with.md`  
**Topics:** SUMX, AVERAGEX, RANKX, MAXX, MINX, iterator + CALCULATE combinations

### Chapter 08: Time Intelligence Calculations
**File:** `ch08_time_intelligence_calculations.md`  
**Topics:** YTD, QTD, MTD, same period last year, rolling averages, DATEADD, PARALLELPERIOD

### Chapter 09: Calculation Groups
**File:** `ch09_calculation_groups.md`  
**Topics:** Calculation groups, calculation items, precedence, SELECTEDMEASURE

### Chapter 10: Working with the Filter Context
**File:** `ch10_working_with_the_fi_lter_context.md`  
**Topics:** Advanced filter context manipulation, ALLSELECTED internals, shadow filter contexts

### Chapter 11: Handling Hierarchies
**File:** `ch11_handling_hierarchies.md`  
**Topics:** Parent-child hierarchies, PATHITEM, PATHCONTAINS, ragged hierarchies

### Chapter 12: Working with Tables
**File:** `ch12_working_with_tables.md`  
**Topics:** SUMMARIZE, ADDCOLUMNS, CROSSJOIN, UNION, INTERSECT, EXCEPT, GENERATE

### Chapter 13: Authoring Queries
**File:** `ch13_authoring_queries.md`  
**Topics:** EVALUATE, ORDER BY, START AT, DEFINE, DAX Studio queries

### Chapter 14: Advanced DAX Concepts
**File:** `ch14_advanced_dax_concepts.md`  
**Topics:** ALLSELECTED deep dive, expanded tables, lineage, TREATAS

### Chapter 15: Advanced Relationships
**File:** `ch15_advanced_relationships.md`  
**Topics:** Bidirectional, many-to-many, weak relationships, USERELATIONSHIP, virtual relationships

### Chapter 16: Advanced Calculations in DAX
**File:** `ch16_advanced_calculations_in_dax.md`  
**Topics:** ABC classification, dynamic segmentation, cohort analysis, basket analysis

### Chapter 17: The DAX Engines
**File:** `ch17_the_dax_engines.md`  
**Topics:** Formula Engine (FE), Storage Engine (SE), VertiPaq, DirectQuery, cache

### Chapter 18: Optimizing VertiPaq
**File:** `ch18_optimizing_vertipaq.md`  
**Topics:** Column cardinality, compression, relationships, VertiPaq Analyzer, model design

### Chapter 19: Analyzing DAX Query Plans
**File:** `ch19_analyzing_dax_query_plans.md`  
**Topics:** Logical/physical query plans, DAX Studio, Server Timings, Vertipaq cache analysis

### Chapter 20: Optimizing DAX
**File:** `ch20_optimizing_dax.md`  
**Topics:** Performance patterns, avoiding iterators, SUMX vs SUM, query optimization techniques

---

## Key DAX Concepts Quick Reference

### Evaluation Context Types
| Context | Created By | Scope |
|---------|-----------|-------|
| **Filter Context** | Report filters, slicers, CALCULATE | Filters rows visible to a measure |
| **Row Context** | Calculated columns, iterators (SUMX, etc.) | Current row being iterated |
| **Context Transition** | CALCULATE inside a row context | Converts row context → filter context |

### Most Critical Functions
| Function | Category | Key Behavior |
|----------|----------|-------------|
| `CALCULATE()` | Context Modifier | #1 most important DAX function — modifies filter context |
| `FILTER()` | Table | Returns filtered table — use carefully (row-by-row iteration) |
| `ALL()` | Context Removal | Removes filters from column(s) or entire table |
| `ALLEXCEPT()` | Context Removal | Removes all filters except specified columns |
| `ALLSELECTED()` | Context Partial | Respects user selections, removes inner filters |
| `VALUES()` | Table | Unique values including blank row from invalid relationships |
| `DISTINCT()` | Table | Unique values excluding blank row |
| `SUMX()` | Iterator | Row-by-row sum over expression — use sparingly for performance |
| `RELATED()` | Relationship | Access column from related table in row context |
| `USERELATIONSHIP()` | Relationship | Activate inactive relationship in CALCULATE |
| `TREATAS()` | Relationship | Apply filter as if from a different column (virtual relationship) |
| `SELECTEDVALUE()` | Single Value | Returns value if single value selected, else alternate result |
| `KEEPFILTERS()` | CALCULATE Modifier | Intersects (AND) instead of overrides filters |
| `REMOVEFILTERS()` | CALCULATE Modifier | Removes filters (cleaner alias for ALL in CALCULATE) |

### Time Intelligence Quick Reference
| Pattern | Function | Notes |
|---------|----------|-------|
| Year-to-Date | `TOTALYTD()` or `CALCULATE([M], DATESYTD(Date[Date]))` | Requires marked Date table |
| Quarter-to-Date | `TOTALQTD()` or `DATESQTD()` | |
| Month-to-Date | `TOTALMTD()` or `DATESMTD()` | |
| Same Period Last Year | `CALCULATE([M], SAMEPERIODLASTYEAR(Date[Date]))` | |
| Prior Month | `CALCULATE([M], DATEADD(Date[Date], -1, MONTH))` | |
| Prior Year | `CALCULATE([M], DATEADD(Date[Date], -1, YEAR))` | |
| Rolling 12M | `CALCULATE([M], DATESINPERIOD(Date[Date], LASTDATE(Date[Date]), -12, MONTH))` | |
| YoY % Growth | `DIVIDE([Sales] - [Sales LY], [Sales LY])` | |

### Performance Rules (Ch 17–20)
1. **Prefer SUM over SUMX** when no row-level expression is needed
2. **Use Variables (VAR)** to avoid repeated measure evaluation
3. **Avoid bidirectional relationships** unless absolutely required
4. **Reduce column cardinality** to improve VertiPaq compression
5. **Use REMOVEFILTERS instead of ALL** in CALCULATE for clarity
6. **Avoid iterator nesting** — SUMX inside SUMX is expensive
7. **Leverage the Storage Engine** — FE is single-threaded, SE is multithreaded
8. **DirectQuery models** require different optimization strategy than Import mode
9. **DIVIDE() over /** — always handles division by zero gracefully
10. **Inactive relationships** — use USERELATIONSHIP or TREATAS for role-playing dimensions

### Star Schema Best Practices (Power BI)
- Fact tables: measures + foreign keys only
- Dimension tables: descriptive attributes + surrogate key
- Date table: must be marked as Date Table, continuous dates, no gaps
- Avoid snowflake schemas in semantic layer (flatten in ETL)
- One-to-many relationships from Dimension → Fact
- Avoid many-to-many unless using bridge tables or calculation groups

---

*Reference extracted from: The Definitive Guide to DAX, 2nd Edition*  
*Marco Russo & Alberto Ferrari — Microsoft Press, 2019*
