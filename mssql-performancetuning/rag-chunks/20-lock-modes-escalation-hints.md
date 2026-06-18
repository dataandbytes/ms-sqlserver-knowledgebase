---
topic: Lock modes, lock escalation, and lock hints
keywords: [lock modes, shared exclusive update, intent locks, lock escalation, 5000 locks, LOCK_ESCALATION, NOLOCK, UPDLOCK, READPAST, HOLDLOCK, TABLOCK]
use_when: You are reasoning about lock compatibility, escalation thresholds, or which lock hint to use.
---

# Lock Modes, Escalation, and Hints

**Lock modes:** Shared (S, blocks X), Update (U, blocks U/X), Exclusive (X, blocks all), Schema Stability (Sch-S), Schema Modification (Sch-M, blocks all — DDL), Bulk Update (BU), Key-Range (SERIALIZABLE). Intent locks (IS/IX/SIX) are taken at table/page level before row locks; IX+IX is compatible, so two writers on different rows in the same table proceed simultaneously.

**Lock escalation:** SQL Server escalates to a single table lock when **one statement** acquires ≥ 5,000 locks on one object, or lock memory exceeds ~24% of the buffer pool. The threshold is **per statement, not per transaction**; checks fire every 1,250 locks. A table X lock blocks all readers and writers. Fix: chunk large DML to stay under 5,000 locks per statement.

```sql
SELECT name, lock_escalation_desc FROM sys.tables WHERE name = 'Orders';
ALTER TABLE dbo.Orders SET (LOCK_ESCALATION = AUTO);  -- partition-first on partitioned tables
```

**Lock hints** (use deliberately, not defensively):

| Hint | Effect | When |
|---|---|---|
| `NOLOCK` | Read uncommitted | Rarely — see NOLOCK dangers |
| `UPDLOCK` | U lock instead of S on read | Prevent S→X conversion deadlocks |
| `HOLDLOCK` | SERIALIZABLE key-range locks | Prevent phantoms in check-then-insert |
| `READPAST` | Skip locked rows | Queue consumers claim different rows |
| `ROWLOCK` | Force row granularity | Hint; may escalate anyway |
| `TABLOCK` | Table-level shared lock | Bulk ops needing minimal logging |
| `TABLOCKX` | Table-level exclusive | Rare; no concurrent access |

```sql
-- Concurrent queue consumer
DECLARE @ItemID INT;
BEGIN TRANSACTION;
    SELECT TOP (1) @ItemID = WorkItemID FROM dbo.WorkQueue WITH (READPAST, ROWLOCK, UPDLOCK)
    WHERE Status = 'Pending' AND ScheduledFor <= SYSDATETIME() ORDER BY QueuedAt;
    IF @ItemID IS NOT NULL
        UPDATE dbo.WorkQueue SET Status = 'Processing' WHERE WorkItemID = @ItemID;
COMMIT;

-- Prevent phantom inserts (check-then-insert)
BEGIN TRANSACTION;
    IF NOT EXISTS (SELECT 1 FROM dbo.Customer WITH (UPDLOCK, HOLDLOCK) WHERE Email = @Email)
        INSERT INTO dbo.Customer (Email, ...) VALUES (@Email, ...);
COMMIT;
```
