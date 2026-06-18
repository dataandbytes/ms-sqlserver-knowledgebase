---
topic: Relational queues — READPAST + UPDLOCK + ROWLOCK for concurrent queue consumers
keywords: [relational queue, READPAST, UPDLOCK, ROWLOCK, queue consumer, concurrent, job queue, notification, dequeue]
use_when: Implementing a queue backed by a SQL Server table where multiple consumers dequeue rows concurrently without blocking each other.
---

# Relational Queues

```sql
-- Queue table
CREATE TABLE Messaging.NotificationJob (
    JobNo       INT           NOT NULL IDENTITY(1,1),
    Payload     NVARCHAR(MAX) NOT NULL,
    Status      VARCHAR(20)   NOT NULL DEFAULT 'Pending',
    CreatedAt   dbo.Timestamp NOT NULL DEFAULT SYSDATETIME(),
    LockedAt    DATETIME2     NULL,
    CompletedAt DATETIME2     NULL,
    CONSTRAINT PK_NotificationJob PRIMARY KEY (JobNo),
    CONSTRAINT CK_NotificationJob_Status CHECK (Status IN ('Pending','Locked','Done','Failed'))
);
CREATE INDEX IX_NotificationJob_Pending ON Messaging.NotificationJob (Status, CreatedAt)
    WHERE Status = 'Pending';

-- Dequeue: claim the next available job (concurrent-safe)
CREATE OR ALTER FUNCTION Messaging.Next_NotificationJob_ut ()
RETURNS TABLE AS RETURN (
    SELECT TOP 1 JobNo, Payload
    FROM Messaging.NotificationJob WITH (READPAST, ROWLOCK, UPDLOCK)
    WHERE Status = 'Pending'
    ORDER BY CreatedAt
);

-- Mark as done (called after successful processing)
CREATE OR ALTER PROCEDURE Messaging.Modify_NotificationJob_Done_utx
    @JobNo INT
AS BEGIN
    SET NOCOUNT ON;
    UPDATE Messaging.NotificationJob
    SET Status = 'Done', CompletedAt = SYSDATETIME()
    WHERE JobNo = @JobNo AND Status = 'Locked';
END;
```

**How the lock hints work together:**
- `UPDLOCK`: takes an update lock instead of a shared lock on the row being read. Prevents two consumers from reading the same row simultaneously.
- `READPAST`: skips rows that are already locked. Consumer 2 gets the next unlocked row instead of waiting.
- `ROWLOCK`: requests row-level locking (not page). Essential — without it, SQL Server may escalate to a page lock and block all consumers.

**Consumer pattern:**
```sql
BEGIN TRANSACTION;
    DECLARE @JobNo INT, @Payload NVARCHAR(MAX);
    SELECT @JobNo = JobNo, @Payload = Payload FROM Messaging.Next_NotificationJob_ut();
    IF @JobNo IS NULL BEGIN ROLLBACK; RETURN; END

    UPDATE Messaging.NotificationJob SET Status = 'Locked', LockedAt = SYSDATETIME()
    WHERE JobNo = @JobNo;
COMMIT;
-- ... process @Payload ...
EXEC Messaging.Modify_NotificationJob_Done_utx @JobNo;
```
