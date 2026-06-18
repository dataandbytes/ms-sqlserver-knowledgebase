---
topic: Chunked DML for large INSERT/UPDATE/DELETE
keywords: [batch delete, chunked update, DELETE TOP, WHILE loop, lock escalation, transaction log, large delete]
use_when: You are writing a large DELETE/UPDATE that bloats the log or escalates to a table lock.
---

# Chunked DML

Any operation affecting more than ~10,000 rows should be chunked. A single large DML reserves log space for the entire operation, may escalate row locks to a table lock (blocking everyone), and blocks rollback for minutes if cancelled. Use `DELETE TOP (N)` / `UPDATE TOP (N)` in a WHILE loop — each iteration is its own autocommit.

```sql
DECLARE @RowsAffected INT = 1;
WHILE @RowsAffected > 0
BEGIN
    DELETE TOP (5000) FROM dbo.AuditLog WHERE LoggedAt < DATEADD(day, -90, SYSDATETIME());
    SET @RowsAffected = @@ROWCOUNT;
    -- WAITFOR DELAY '00:00:00.100';   -- optional: yield to other workloads
END;
```

**Chunk size:** start at 5,000 (below the per-statement escalation threshold). With many indexes, reduce proportionally — each modified row takes one lock per index, so 5 NCIs hits ~5,000 locks at ~1,000 rows. Monitor `WRITELOG` and reduce if log I/O is the bottleneck.

For atomicity-within-chunks (paired INSERT + DELETE), wrap each chunk in an explicit transaction so a mid-loop restart cannot orphan rows:

```sql
DECLARE @Rows INT = 1;
WHILE @Rows > 0
BEGIN
    BEGIN TRANSACTION;
        INSERT INTO dbo.OrderArchive (OrderID, CustomerID, TotalAmt, OrderDate)
        SELECT TOP (5000) OrderID, CustomerID, TotalAmt, OrderDate
        FROM dbo.Orders WHERE OrderDate < DATEADD(year, -7, SYSDATETIME());
        SET @Rows = @@ROWCOUNT;
        DELETE dbo.Orders WHERE OrderID IN (
            SELECT TOP (5000) OrderID FROM dbo.Orders WHERE OrderDate < DATEADD(year, -7, SYSDATETIME())
        );
    COMMIT;
END;
```
