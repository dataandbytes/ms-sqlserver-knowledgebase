---
topic: Partition switching and transaction log management
keywords: [partition switching, SWITCH PARTITION, metadata only, partition elimination, transaction log, VLF, WRITELOG, TRUNCATE, autogrowth]
use_when: You are archiving large data fast, or managing transaction log size/latency.
---

# Partition Switching + Log Management

**Partition switching** is the fastest way to move large data in/out — metadata-only, milliseconds regardless of volume. Both tables must have identical structure, constraints, and filegroup placement.

```sql
CREATE TABLE dbo.OrdersArchive_2023 (
    OrderID INT NOT NULL, CustomerID INT NOT NULL,
    OrderDate DATETIME2 NOT NULL, TotalAmt DECIMAL(12,2) NOT NULL,
    CONSTRAINT PK_OrdersArchive_2023 PRIMARY KEY CLUSTERED (OrderID)
) ON [PRIMARY];
ALTER TABLE dbo.Orders SWITCH PARTITION 1 TO dbo.OrdersArchive_2023;  -- milliseconds, no rows copied
TRUNCATE TABLE dbo.OrdersArchive_2023;
```

**Partition elimination:** the optimizer excludes partitions that cannot match — verify "Actual Partition Count" < total on the scan operator in the actual plan.

**Transaction log:** a write-ahead sequential append; performance depends on log disk latency (single-threaded, no parallelism benefit). **WRITELOG** = log I/O is the bottleneck → faster (NVMe SSD) dedicated log disk, not chunking (chunking helps escalation and log space, not latency).

```sql
SELECT name, log_size_mb = size * 8 / 1024,
       log_used_mb = FILEPROPERTY(name, 'SpaceUsed') * 8 / 1024, recovery_model_desc
FROM sys.databases WHERE name = DB_NAME();
DBCC LOGINFO;  -- VLF count; ideal < 100, > 1,000 problematic. 2016 SP2+: sys.dm_db_log_info
```

Reduce log volume: BULK_LOGGED recovery; `TRUNCATE TABLE` instead of `DELETE FROM` (minimally logged, still rollbackable); chunk large DML. Pre-size the log to avoid autogrowth (each autogrowth pauses all log writes):

```sql
ALTER DATABASE YourDatabase MODIFY FILE (NAME = 'YourDatabase_log', SIZE = 10240 MB, FILEGROWTH = 512 MB);
```
