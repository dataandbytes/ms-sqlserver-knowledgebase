---
topic: Pivot and unpivot — PIVOT operator, dynamic PIVOT, CROSS APPLY VALUES unpivot
keywords: [PIVOT, UNPIVOT, dynamic PIVOT, CROSS APPLY VALUES, STRING_AGG, QUOTENAME, sp_executesql, unpivot, normalize]
use_when: Rotating rows to columns (pivot), rotating columns to rows (unpivot), or building a dynamic pivot with data-driven column names.
---

# Pivot and Unpivot

```sql
-- Static PIVOT: known columns at query time
SELECT ProductName, [2022], [2023], [2024]
FROM (
    SELECT p.Name AS ProductName, YEAR(o.OrderDate) AS OrderYear, ol.TotalAmount
    FROM Sales.OrderLines ol
    JOIN Sales.Orders o ON o.OrderID = ol.OrderID
    JOIN Production.Product p ON p.ProductID = ol.ProductID
) Src
PIVOT (SUM(TotalAmount) FOR OrderYear IN ([2022],[2023],[2024])) Pvt;

-- Dynamic PIVOT: column names come from data
DECLARE @Cols NVARCHAR(MAX), @SQL NVARCHAR(MAX);
SELECT @Cols = STRING_AGG(QUOTENAME(CAST(OrderYear AS VARCHAR(4))), ',')
               WITHIN GROUP (ORDER BY OrderYear)
FROM (SELECT DISTINCT YEAR(OrderDate) AS OrderYear FROM Sales.Orders) y;

SET @SQL = N'
SELECT ProductName, ' + @Cols + N'
FROM (
    SELECT p.Name AS ProductName, YEAR(o.OrderDate) AS yr, ol.TotalAmount
    FROM Sales.OrderLines ol
    JOIN Sales.Orders o ON o.OrderID = ol.OrderID
    JOIN Production.Product p ON p.ProductID = ol.ProductID
) Src
PIVOT (SUM(TotalAmount) FOR yr IN (' + @Cols + N')) Pvt;';
EXEC sp_executesql @SQL;

-- CROSS APPLY VALUES unpivot (preserves NULLs; UNPIVOT operator drops them)
SELECT ProductID, Quarter, Revenue
FROM Sales.QuarterlySales
CROSS APPLY (VALUES
    ('Q1', Q1_Revenue),
    ('Q2', Q2_Revenue),
    ('Q3', Q3_Revenue),
    ('Q4', Q4_Revenue)
) AS u(Quarter, Revenue)
WHERE Revenue IS NOT NULL;  -- optional: filter nulls explicitly
```

**CROSS APPLY VALUES vs UNPIVOT:**
- `UNPIVOT` drops rows where the unpivoted value is NULL — it cannot be changed.
- `CROSS APPLY VALUES` keeps NULL rows unless you explicitly filter them. Prefer it for control.
