---
topic: User-defined functions : inline TVF vs scalar UDF, UDF inlining
keywords: [inline TVF, scalar UDF, CREATE FUNCTION, RETURNS TABLE, UDF inlining, is_inlineable, CROSS APPLY, performance]
use_when: Creating a reusable SQL function, replacing a scalar UDF with an inline TVF for performance, or checking UDF inlining eligibility.
---

# User-Defined Functions

```sql
-- Inline TVF: treated as a parameterized view : can be indexed, used in WHERE, joined
CREATE OR ALTER FUNCTION Sales.fn_GetCustomerOrders (
    @CustomerID INT, @Since DATE
)
RETURNS TABLE AS RETURN (
    SELECT OrderID, OrderDate, TotalAmount, Status
    FROM Sales.Orders
    WHERE CustomerID = @CustomerID AND OrderDate >= @Since
);
-- Usage: SELECT * FROM Sales.fn_GetCustomerOrders(42, '2024-01-01')
-- Or via CROSS APPLY for per-row invocation:
-- SELECT c.Name, o.* FROM Customers c CROSS APPLY Sales.fn_GetCustomerOrders(c.CustomerID, '2024-01-01') o

-- Scalar UDF (avoid in WHERE/JOIN : prevents parallelism, executes row-by-row pre-2019)
CREATE OR ALTER FUNCTION dbo.fn_FormatCurrency (@Amount DECIMAL(18,2), @Currency CHAR(3))
RETURNS NVARCHAR(30) AS BEGIN
    RETURN @Currency + ' ' + FORMAT(@Amount, 'N2');
END;
```

**Inline TVF vs Scalar UDF:**
- Inline TVF is expanded inline by the optimizer : the engine can push predicates into it, use indexes, and parallelize.
- Scalar UDF in a WHERE clause or JOIN condition forces a row-by-row cursor-like execution and disables parallelism (pre-SQL Server 2019).
- SQL Server 2019+ has scalar UDF inlining: automatically rewrites eligible scalar UDFs as inline expressions. Check eligibility:

```sql
SELECT name, is_inlineable
FROM sys.sql_modules
WHERE OBJECT_NAME(object_id) = 'YourFunctionName';
```

- If `is_inlineable = 0`, replace the scalar UDF with an inline TVF + `CROSS APPLY` to restore performance.
