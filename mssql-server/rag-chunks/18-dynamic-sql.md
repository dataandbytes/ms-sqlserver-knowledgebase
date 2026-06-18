---
topic: Dynamic SQL : sp_executesql, parameterized queries, dynamic WHERE, dynamic PIVOT
keywords: [dynamic SQL, sp_executesql, SQL injection, QUOTENAME, STRING_AGG, dynamic WHERE, dynamic PIVOT, parameterized]
use_when: Building a query where column names, table names, or predicate structure are not known at compile time. Always use sp_executesql with parameters : never string concatenation of user input.
---

# Dynamic SQL

```sql
-- Always use sp_executesql with typed parameters (never concatenate user input)
DECLARE @SQL   NVARCHAR(MAX);
DECLARE @Param NVARCHAR(500);
DECLARE @CID   INT  = 42;
DECLARE @Since DATE = '2024-01-01';

SET @SQL = N'SELECT OrderID, OrderDate, TotalAmount
             FROM Sales.Orders
             WHERE CustomerID = @CustomerID AND OrderDate >= @Since';
SET @Param = N'@CustomerID INT, @Since DATE';
EXEC sp_executesql @SQL, @Param, @CustomerID = @CID, @Since = @Since;

-- Dynamic WHERE: build the predicate string, pass values as parameters
DECLARE @Status VARCHAR(20) = NULL;
SET @SQL = N'SELECT * FROM Sales.Orders WHERE 1=1';
IF @CID    IS NOT NULL SET @SQL += N' AND CustomerID = @CustomerID';
IF @Status IS NOT NULL SET @SQL += N' AND Status = @Status';
SET @SQL += N' ORDER BY OrderDate DESC';
EXEC sp_executesql @SQL,
     N'@CustomerID INT, @Status VARCHAR(20)',
     @CustomerID = @CID, @Status = @Status;

-- Dynamic PIVOT with QUOTENAME (injection-safe column names from data)
DECLARE @Cols NVARCHAR(MAX);
SELECT @Cols = STRING_AGG(QUOTENAME(CAST(OrderYear AS VARCHAR(4))), ',')
               WITHIN GROUP (ORDER BY OrderYear)
FROM (SELECT DISTINCT YEAR(OrderDate) AS OrderYear FROM Sales.Orders) y;

SET @SQL = N'
SELECT ProductName, ' + @Cols + N'
FROM (
    SELECT p.Name AS ProductName, YEAR(o.OrderDate) AS OrderYear, ol.TotalAmount
    FROM Sales.OrderLines ol
    JOIN Sales.Orders o ON o.OrderID = ol.OrderID
    JOIN Production.Product p ON p.ProductID = ol.ProductID
) Src
PIVOT (SUM(TotalAmount) FOR OrderYear IN (' + @Cols + N')) Pvt;';
EXEC sp_executesql @SQL;
```

**Key rules:**
- `QUOTENAME()` wraps identifiers in `[brackets]` and escapes embedded brackets : always use it for dynamic column/table names.
- `STRING_AGG` replaces the `FOR XML PATH` pattern for building comma-separated lists (2017+).
- Dynamic SQL executes in a new scope : local variables from the calling batch are not visible unless passed as parameters.
- Table and column names cannot be parameterized (only values can). Validate them against `sys.columns` or an allowlist before concatenating.
