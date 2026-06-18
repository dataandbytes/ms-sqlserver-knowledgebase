---
topic: Error handling : TRY/CATCH, THROW, RAISERROR, XACT_ABORT
keywords: [TRY CATCH, THROW, RAISERROR, ERROR_NUMBER, ERROR_MESSAGE, XACT_ABORT, error handling, exception, rollback]
use_when: Wrapping T-SQL in error handling, re-throwing errors, or ensuring transactions roll back automatically on error.
---

# Error Handling

```sql
-- TRY/CATCH with transaction rollback and re-throw
BEGIN TRY
    BEGIN TRANSACTION;
        EXEC Sales.PlaceOrder @CustomerID = @cid, @Lines = @lines, @OrderID = @oid OUTPUT;
    COMMIT;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK;
    THROW;  -- re-throws original error number, severity, state, and message
END CATCH;

-- Error diagnostic functions (call inside CATCH block)
SELECT
    ERROR_NUMBER()    AS ErrNo,
    ERROR_SEVERITY()  AS Severity,
    ERROR_STATE()     AS State,
    ERROR_PROCEDURE() AS Proc,
    ERROR_LINE()      AS Line,
    ERROR_MESSAGE()   AS Msg;

-- THROW with custom message (2012+)
THROW 50001, 'Customer not found', 1;

-- RAISERROR (backward-compatible; requires message in sys.messages or inline string)
RAISERROR('Validation failed: %s', 16, 1, 'Price must be positive');

-- XACT_ABORT: auto-rollback + abort batch on any runtime error
SET XACT_ABORT ON;
```

**Key rules:**
- `THROW` (2012+) preserves the original error number and re-throws cleanly. Prefer it over `RAISERROR` for re-throwing.
- `RAISERROR` is needed when constructing formatted messages or using pre-registered message IDs (`sp_addmessage`).
- `SET XACT_ABORT ON` in procedures: if any statement errors, the entire transaction is rolled back automatically : no explicit `ROLLBACK` needed. Combine with TRY/CATCH for maximum safety.
- `@@TRANCOUNT > 0` check is essential : calling `ROLLBACK` when `@@TRANCOUNT = 0` raises its own error.
- Severity 11–18 = user errors (catchable). Severity 19–25 = fatal (terminates connection, some uncatchable).
