---
topic: DQL : SELECT, pagination, CROSS APPLY, OUTER APPLY, set operators, PIVOT
keywords: [SELECT, OFFSET FETCH, pagination, CROSS APPLY, OUTER APPLY, INTERSECT, EXCEPT, PIVOT, IS DISTINCT FROM]
use_when: Writing SELECT queries with pagination, correlated subqueries as inline tables, set operations, or column pivoting.
---

# DQL: SELECT and Query Syntax

```sql
-- Pagination (page 3, 25 rows per page)
SELECT ProductID, Name, Price FROM Production.Product
WHERE IsActive = 1 ORDER BY Name
OFFSET 50 ROWS FETCH NEXT 25 ROWS ONLY;

-- CROSS APPLY: correlated inline table (excludes rows with no match)
SELECT c.CustomerID, c.Name, recent.LastOrderDate, recent.OrderCount
FROM Sales.Customers c
CROSS APPLY (
    SELECT MAX(OrderDate) AS LastOrderDate, COUNT(*) AS OrderCount
    FROM Sales.Orders o WHERE o.CustomerID = c.CustomerID
) recent;
-- Use OUTER APPLY to keep customers with no orders (NULLs returned for recent.*)

-- Set operators
SELECT ProductID FROM Production.Product WHERE CategoryID = 1
INTERSECT
SELECT ProductID FROM Sales.OrderLines WHERE Qty > 100;

SELECT ProductID FROM Production.Product
EXCEPT
SELECT ProductID FROM Sales.Discontinued;

-- PIVOT: rows to columns
SELECT ProductName, [2022], [2023], [2024]
FROM (
    SELECT p.Name AS ProductName, YEAR(o.OrderDate) AS OrderYear, ol.TotalAmount
    FROM Sales.OrderLines ol
    JOIN Sales.Orders o  ON o.OrderID = ol.OrderID
    JOIN Production.Product p ON p.ProductID = ol.ProductID
) Src
PIVOT (SUM(TotalAmount) FOR OrderYear IN ([2022],[2023],[2024])) Pvt;

-- NULL-safe comparison (SQL Server 2022+)
WHERE a.Status IS DISTINCT FROM b.Status;
-- Pre-2022: requires verbose OR (a IS NULL AND b IS NOT NULL) OR (a IS NOT NULL AND b IS NULL) OR a <> b
```

**Key rules:**
- `CROSS APPLY` with a correlated subquery is equivalent to an INNER JOIN to the subquery result. `OUTER APPLY` = LEFT JOIN.
- `OFFSET FETCH` requires `ORDER BY` : the order clause is mandatory, not optional.
- PIVOT column list must be static (literal values). For dynamic lists, use dynamic SQL with `STRING_AGG` + `QUOTENAME`.
