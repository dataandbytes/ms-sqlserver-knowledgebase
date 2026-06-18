---
topic: Multi-level aggregation — ROLLUP, CUBE, GROUPING SETS, GROUPING(), GROUPING_ID()
keywords: [ROLLUP, CUBE, GROUPING SETS, GROUPING, GROUPING_ID, multi-level aggregate, subtotal, grand total, conditional aggregation]
use_when: Producing subtotals, grand totals, or multi-dimensional aggregations in a single query pass.
---

# Multi-Level Aggregation

```sql
-- ROLLUP: hierarchical subtotals (year → quarter → month → grand total)
SELECT
    YEAR(OrderDate)    AS Year,
    MONTH(OrderDate)   AS Month,
    SUM(TotalAmount)   AS Revenue,
    GROUPING(YEAR(OrderDate))  AS IsYearTotal,
    GROUPING(MONTH(OrderDate)) AS IsMonthTotal
FROM Sales.Orders
GROUP BY ROLLUP(YEAR(OrderDate), MONTH(OrderDate));

-- CUBE: all combinations of dimensions
SELECT Region, ProductCategory, SUM(Revenue) AS Revenue,
       GROUPING(Region) AS IsRegionAll, GROUPING(ProductCategory) AS IsCategoryAll
FROM Sales.Summary
GROUP BY CUBE(Region, ProductCategory)
ORDER BY GROUPING(Region), GROUPING(ProductCategory), Region, ProductCategory;

-- GROUPING SETS: specific combinations only (more control than CUBE)
SELECT Region, ProductCategory, SUM(Revenue) AS Revenue
FROM Sales.Summary
GROUP BY GROUPING SETS (
    (Region, ProductCategory),  -- detail
    (Region),                   -- region subtotal
    ()                          -- grand total
);

-- GROUPING_ID: bitmask identifying which columns are aggregated
SELECT Region, ProductCategory,
       GROUPING_ID(Region, ProductCategory) AS GroupLevel,
       SUM(Revenue) AS Revenue
FROM Sales.Summary
GROUP BY ROLLUP(Region, ProductCategory);
-- 0 = detail, 1 = region subtotal, 3 = grand total

-- Conditional aggregation (pivot-like without PIVOT syntax)
SELECT
    CustomerID,
    SUM(CASE WHEN Status = 'Completed' THEN TotalAmount ELSE 0 END) AS CompletedRevenue,
    SUM(CASE WHEN Status = 'Cancelled' THEN 1            ELSE 0 END) AS CancelledCount,
    SUM(CASE WHEN MONTH(OrderDate) = 1  THEN TotalAmount ELSE 0 END) AS JanRevenue
FROM Sales.Orders
GROUP BY CustomerID;
```

**Key rules:**
- `GROUPING(col) = 1` means that column is aggregated (it's a subtotal/grand total row). Use it to distinguish grand-total NULLs from data NULLs.
- `ROLLUP(a, b, c)` produces subtotals for: (a,b,c), (a,b), (a), and (). CUBE produces all 2^n combinations.
- HAVING filters after aggregation; WHERE filters before. You cannot reference a GROUPING() call in WHERE.
