# DAX Patterns — 2nd Edition — Reference Index
**Authors:** Alberto Ferrari & Marco Russo  
**Publisher:** SQLBI Corp. | **Year:** 2020  
**ISBN:** 978-1-7353652-0-6 | **Website:** www.daxpatterns.com

---

## Pattern Selection Guide

| Your Need | Use This Pattern |
|-----------|-----------------|
| Standard Gregorian calendar time calcs | Ch 2 — Standard time-related |
| Month-level only, fiscal or 13-month | Ch 3 — Month-related |
| Week-based (4-4-5, ISO) calendar | Ch 4 — Week-related |
| Fully custom fiscal calendar | Ch 5 — Custom time-related |
| User-defined period comparison | Ch 6 — Comparing periods |
| Opening/closing balance, stock levels | Ch 7 — Semi-additive |
| Running total / cumulative sum | Ch 8 — Cumulative total |
| Dynamic slicers (TopN, scale, discount) | Ch 9 — Parameter table |
| Price range / static classification | Ch 10 — Static segmentation |
| Dynamic customer/product clustering | Ch 11 — Dynamic segmentation |
| ABC analysis (80/20 rule) | Ch 12 — ABC classification |
| New & returning customer analysis | Ch 13 — New & returning customers |
| Distinct count across multiple facts | Ch 14 — Related distinct count |
| Open orders, in-progress events | Ch 15 — Events in progress |
| Product/customer ranking (RANKX) | Ch 16 — Ranking |
| Navigation through hierarchies | Ch 17 — Hierarchies |
| Chart of accounts, parent-child | Ch 18 — Parent-child hierarchies |
| Same store sales comparison | Ch 19 — Like-for-like |
| Rating / status transition over time | Ch 20 — Transition matrix |
| Survey / multi-response analysis | Ch 21 — Survey |
| Product affinity / basket analysis | Ch 22 — Basket analysis |
| Multi-currency reporting | Ch 23 — Currency conversion |
| Budget vs actuals, variance | Ch 24 — Budget |

---

## Table of Contents

### Chapter 01: Time-related calculations — Introduction
**Sample:** https://sql.bi/dax-200  
**Topics:** Guide to choosing the right time pattern: Standard, Month, Week, or Custom

### Chapter 02: Standard time-related calculations
**Sample:** https://sql.bi/dax-201  
**Topics:** YTD, QTD, MTD, YoY, QoQ, MoM, MAT, moving averages using built-in DAX time intelligence functions

### Chapter 03: Month-related calculations
**Sample:** https://sql.bi/dax-202  
**Topics:** Month-granularity patterns without built-in functions; supports 13-month fiscal calendars

### Chapter 04: Week-related calculations
**Sample:** https://sql.bi/dax-203  
**Topics:** Week-based calendar (4-4-5 ISO), WTD, YoY, QoQ, WoW, filter-safe columns

### Chapter 05: Custom time-related calculations
**Sample:** https://sql.bi/dax-204  
**Topics:** Fully custom fiscal calendar patterns without any built-in time intelligence functions

### Chapter 06: Comparing different time periods
**Sample:** https://sql.bi/dax-205  
**Topics:** User-selected comparison periods with day-adjusted allocation factor

### Chapter 07: Semi-additive calculations
**Sample:** https://sql.bi/dax-206  
**Topics:** Opening/closing balances, first/last date, LASTNONBLANK, non-additive measures over time

### Chapter 08: Cumulative total (Running total)
**Sample:** https://sql.bi/dax-207  
**Topics:** Running total over date or any sortable column, cumulative sum pattern

### Chapter 09: Parameter table
**Sample:** https://sql.bi/dax-210  
**Topics:** Dynamic parameters via disconnected tables, SELECTEDVALUE, TopN, scale, dependent parameters

### Chapter 10: Static segmentation
**Sample:** https://sql.bi/dax-211  
**Topics:** Price range classification via calculated columns and relationships; large-table optimization

### Chapter 11: Dynamic segmentation
**Sample:** https://sql.bi/dax-212  
**Topics:** Customer/product clustering by dynamic measures; KEEPFILTERS, non-additive totals

### Chapter 12: ABC classification
**Sample:** https://sql.bi/dax-213  
**Topics:** Static, snapshot, and dynamic ABC; RANKX cumulative %, TREATAS for snapshot

### Chapter 13: New and returning customers
**Sample:** https://sql.bi/dax-218  
**Topics:** New, returning, lost, recovered customers; dynamic relative/absolute/by-category; snapshot version

### Chapter 14: Related distinct count
**Sample:** https://sql.bi/dax-208  
**Topics:** DISTINCTCOUNT across fact tables; INTERSECT, UNION, EXCEPT for set operations

### Chapter 15: Events in progress
**Sample:** https://sql.bi/dax-224  
**Topics:** Open orders ALL/EOP/AVG; snapshot table; TREATAS; USERELATIONSHIP optimization

### Chapter 16: Ranking
**Sample:** https://sql.bi/dax-229  
**Topics:** Static and dynamic RANKX; Top N by category; TOPN filter; visual-level vs measure filter

### Chapter 17: Hierarchies
**Sample:** https://sql.bi/dax-220  
**Topics:** ISINSCOPE level detection, % of parent node, hierarchy-driven calculations

### Chapter 18: Parent-child hierarchies
**Sample:** https://sql.bi/dax-221  
**Topics:** PATH/PATHITEM/PATHCONTAINS; flattened levels; chart of accounts; signed aggregation; security roles

### Chapter 19: Like-for-like comparison
**Sample:** https://sql.bi/dax-225  
**Topics:** Same store sales with snapshot; dynamic version without snapshot; SELECTEDVALUE filter

### Chapter 20: Transition matrix
**Sample:** https://sql.bi/dax-227  
**Topics:** Rating changes over time; static snapshot; dynamic transition matrix with TREATAS

### Chapter 21: Survey
**Sample:** https://sql.bi/dax-216  
**Topics:** Multi-response survey analysis; weighted scoring; matrix visualization

### Chapter 22: Basket analysis
**Sample:** https://sql.bi/dax-217  
**Topics:** Market basket/affinity analysis; product combinations; GENERATE, TOPN, cross-filter

### Chapter 23: Currency conversion
**Sample:** https://sql.bi/dax-219  
**Topics:** Multiple exchange rate types; triangulation; average/end-of-period rates; snapshot

### Chapter 24: Budget
**Sample:** https://sql.bi/dax-214  
**Topics:** Budget vs actuals; allocation from higher to lower granularity; variance analysis

---

## Common DAX Patterns Quick Reference

### Time Intelligence Naming Convention
All patterns use consistent naming for time shift measures:
| Suffix | Meaning | Example |
|--------|---------|---------|
| `YTD` | Year-to-date | `Sales YTD` |
| `QTD` | Quarter-to-date | `Sales QTD` |
| `MTD` | Month-to-date | `Sales MTD` |
| `PY` | Previous Year | `Sales PY` |
| `PQ` | Previous Quarter | `Sales PQ` |
| `PM` | Previous Month | `Sales PM` |
| `YOY` | Year-over-year | `Sales YOY` |
| `YOY%` | Year-over-year % | `Sales YOY %` |
| `PYTD` | Previous Year-to-date | `Sales PYTD` |
| `MAT` | Moving Annual Total | `Sales MAT` |
| `AVG 3M` | Moving average 3 months | `Sales AVG 3M` |
| `PYC` | Previous Year Complete | `Sales PYC` |

### Choosing a Time Intelligence Pattern

```
Is your calendar standard Gregorian?
├── YES → Do you need sub-month detail?
│         ├── YES → Chapter 2 (Standard time-related)
│         └── NO (month-level ok) → Chapter 3 (Month-related)
└── NO → Is it week-based (4-4-5 / ISO)?
          ├── YES → Chapter 4 (Week-related)
          └── NO (custom fiscal) → Chapter 5 (Custom time-related)
```

### Key Pattern Building Blocks
| Concept | Key Functions | Chapter |
|---------|--------------|---------|
| Date table requirements | `MARK AS DATE TABLE`, `DateWithSales` | Ch 2 |
| Future date hiding | `ShowValueForDates` measure | Ch 2,4,5 |
| Filter-safe columns | `ALLEXCEPT` vs `REMOVEFILTERS` | Ch 4,5 |
| Fair period comparison | `Date[DateWithSales]`, `DATEADD` | Ch 2–5 |
| Semi-additive (last value) | `LASTDATE`, `LASTNONBLANK` | Ch 7 |
| Running total | `CALCULATE` + date range filter | Ch 8 |
| Dynamic parameters | `SELECTEDVALUE` + disconnected table | Ch 9 |
| Static classification | Calculated column + relationship | Ch 10 |
| Dynamic classification | Measure + `KEEPFILTERS` | Ch 11 |
| ABC cumulative % | `RANKX` + `SUMX` cumulative | Ch 12 |
| Customer state tracking | Internal + external measure pattern | Ch 13 |
| Snapshot performance | Precomputed tables + `TREATAS` | Ch 12,13,15 |
| Hierarchy level detection | `ISINSCOPE` | Ch 17,18 |
| Parent-child traversal | `PATH`, `PATHITEM`, `PATHCONTAINS` | Ch 18 |
| Exchange rates | `USERELATIONSHIP` + inactive rel. | Ch 23 |

---

*Reference extracted from: DAX Patterns, 2nd Edition*  
*Alberto Ferrari & Marco Russo — SQLBI Corp., 2020*  
*All samples available at: www.daxpatterns.com*
