---
topic: Choosing a clustered index key
keywords: [clustered index, primary key, narrow ever-increasing unique static, GUID, IDENTITY, page split, uniquifier]
use_when: You are designing a table's clustered key or evaluating an existing one (e.g. a GUID clustered key).
---

# Clustered Index Selection

The clustered index IS the table — leaf pages store the actual rows in clustered key order, and every nonclustered index stores a copy of the clustered key as its row locator. A bad choice cascades to every other index. Choose a key that is:

- **Narrow** — copied into every NCI leaf row. `UNIQUEIDENTIFIER` (16 bytes) vs `INT` (4) adds 12 bytes per NCI row; with 10 NCIs and 100M rows that is ~12 GB of extra index storage.
- **Ever-increasing** — random inserts cause expensive 50/50 page splits. Monotonic keys (`IDENTITY`, `SEQUENCE`, `NEWSEQUENTIALID()`) produce cheap 90/10 splits.
- **Unique** — SQL Server silently appends a 4-byte uniquifier to duplicates, bloating the index and every NCI lookup.
- **Static** — updating the clustered key physically moves the row and cascades the row-locator update to every NCI.

```sql
-- Good: narrow, unique, ever-increasing
CREATE TABLE dbo.Orders (
    OrderID INT NOT NULL IDENTITY(1,1),
    CustomerID INT NOT NULL, OrderDate DATETIME2 NOT NULL, TotalAmt DECIMAL(12,2) NOT NULL,
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderID)
);
```

**When the clustered key is not the PK:** for heavy range scans on a non-PK column, declare the PK NONCLUSTERED and cluster on the scan column:

```sql
CREATE TABLE dbo.EventLog (
    EventID BIGINT NOT NULL IDENTITY(1,1), OccurredAt DATETIME2 NOT NULL,
    EventType TINYINT NOT NULL, Payload NVARCHAR(MAX) NULL,
    CONSTRAINT PK_EventLog PRIMARY KEY NONCLUSTERED (EventID)
);
CREATE CLUSTERED INDEX CIX_EventLog_OccurredAt ON dbo.EventLog (OccurredAt);
```
