---
topic: Transactions, isolation levels, lock hints, RCSI, deadlock detection
keywords: [BEGIN TRANSACTION, COMMIT, ROLLBACK, SAVEPOINT, isolation level, RCSI, READ COMMITTED SNAPSHOT, NOLOCK, UPDLOCK, READPAST, TABLOCK, deadlock, blocking]
use_when: Writing multi-statement transactions, choosing an isolation level, applying lock hints, enabling RCSI, or detecting deadlock graphs.
---

# Transactions and Locking

```sql
-- Transaction with savepoint (partial rollback)
BEGIN TRANSACTION OrderProcessing;
SAVE TRANSACTION PreInventoryCheck;

    UPDATE Inventory.Stock SET Qty = Qty - @Qty WHERE ProductID = @ProductID;

    IF (SELECT Qty FROM Inventory.Stock WHERE ProductID = @ProductID) < 0 BEGIN
        ROLLBACK TRANSACTION PreInventoryCheck;  -- undo only to savepoint
        ROLLBACK TRANSACTION OrderProcessing;
        RETURN;
    END

COMMIT TRANSACTION OrderProcessing;

-- Enable RCSI (biggest OLTP concurrency improvement : eliminates shared read locks)
ALTER DATABASE YourDB SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;

-- Isolation levels
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- default
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;          -- row versioning
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;      -- key-range locks, no phantoms

-- Lock hints
SELECT * FROM Sales.Orders WITH (NOLOCK);              -- READ UNCOMMITTED (dirty reads)
SELECT * FROM Sales.Orders WITH (UPDLOCK, ROWLOCK);    -- prevent S→X deadlock pattern
SELECT * FROM Jobs         WITH (READPAST, ROWLOCK, UPDLOCK);  -- skip locked rows (queue)

-- Deadlock graph from system_health XE session
SELECT xdr.value('@timestamp','datetime2') AS deadlock_time, xdr.query('.') AS graph
FROM (SELECT CAST(target_data AS XML) AS td
      FROM sys.dm_xe_session_targets t
      JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
      WHERE s.name = 'system_health' AND t.target_name = 'ring_buffer') d
CROSS APPLY td.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS XEventData(xdr)
ORDER BY deadlock_time DESC;
```

**Key rules:**
- `NOLOCK` allows dirty reads and phantom rows : use only for approximate reporting where stale data is acceptable. Never on financial or inventory data.
- `UPDLOCK` taken at read time converts to an exclusive lock at write time : prevents the classic check-then-insert deadlock.
- `READPAST` skips rows locked by other transactions : the correct pattern for queue consumers.
- RCSI makes `READ COMMITTED` use row versions instead of shared locks. It eliminates most reader-writer blocking with no application changes.
