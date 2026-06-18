---
topic: Stored procedures : TVP parameters, EXECUTE AS, parameter sniffing fixes
keywords: [stored procedure, CREATE PROCEDURE, TVP, table-valued parameter, EXECUTE AS, OPTION RECOMPILE, OPTIMIZE FOR UNKNOWN, parameter sniffing, OUTPUT parameter]
use_when: Creating a stored procedure, passing a set of rows as input via TVP, or fixing parameter sniffing on a variable-selectivity query.
---

# Stored Procedures

```sql
-- TVP: pass a set of rows as a single parameter
CREATE TYPE Sales.OrderLineType AS TABLE (
    ProductID INT NOT NULL, Qty INT NOT NULL, UnitPrice DECIMAL(10,2) NOT NULL
);

CREATE OR ALTER PROCEDURE Sales.PlaceOrder
    @CustomerID INT,
    @Lines      Sales.OrderLineType READONLY,  -- TVP must be READONLY
    @OrderID    INT OUTPUT
AS BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
            INSERT INTO Sales.Orders (CustomerID, OrderDate)
            VALUES (@CustomerID, SYSDATETIME());
            SET @OrderID = SCOPE_IDENTITY();

            INSERT INTO Sales.OrderLines (OrderID, ProductID, Qty, UnitPrice)
            SELECT @OrderID, ProductID, Qty, UnitPrice FROM @Lines;
        COMMIT;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        THROW;
    END CATCH;
END;

-- EXECUTE AS: run with a fixed security context
CREATE OR ALTER PROCEDURE Reporting.GetAllOrders
    WITH EXECUTE AS 'ReportingUser'
AS SELECT * FROM Sales.Orders;

-- Parameter sniffing fix (query-level, not whole procedure)
SELECT OrderID, OrderDate FROM Sales.Orders
WHERE CustomerID = @CustomerID
OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));  -- use average distribution instead of sniffed value
-- Alternative for highly variable, infrequent procs:
-- OPTION (RECOMPILE) : new plan every call, higher CPU cost
```

**Key rules:**
- Always `SET NOCOUNT ON` : eliminates the "N rows affected" message that breaks some ORMs.
- `SCOPE_IDENTITY()` returns the last identity value inserted in the current scope. Never use `@@IDENTITY` (affected by triggers).
- TVP parameters must be declared `READONLY` : they cannot be modified inside the procedure.
- Choose `OPTIMIZE FOR UNKNOWN` for moderately variable parameters; `RECOMPILE` only when the query is called infrequently and the plan cost varies dramatically.
