---
topic: SQL Server 2022 new features : IS DISTINCT FROM, GREATEST/LEAST, DATETRUNC, DATE_BUCKET, JSON_OBJECT, GENERATE_SERIES, S3 backup, Contained AG
keywords: [SQL Server 2022, IS DISTINCT FROM, GREATEST, LEAST, DATETRUNC, DATE_BUCKET, JSON_OBJECT, JSON_ARRAY, GENERATE_SERIES, S3 backup, Contained AG, APPROX_PERCENTILE_DISC]
use_when: Using any SQL Server 2022+ syntax or feature.
---

# SQL Server 2022 Features

```sql
-- IS [NOT] DISTINCT FROM: NULL-safe equality (no need for verbose OR IS NULL checks)
WHERE a.Status IS DISTINCT FROM b.Status;

-- GREATEST / LEAST across columns
SELECT ProductID,
       GREATEST(Q1, Q2, Q3, Q4) AS PeakQuarter,
       LEAST(Q1, Q2, Q3, Q4)    AS WeakestQuarter
FROM Sales.QuarterlySales;

-- DATETRUNC: truncate to a date part boundary
SELECT DATETRUNC(month, OrderDate) AS MonthStart FROM Sales.Orders;

-- DATE_BUCKET: arbitrary interval bucketing
SELECT DATE_BUCKET(hour, 4, EventTime) AS FourHourBucket FROM Sensors.Events;

-- JSON constructors
SELECT JSON_OBJECT('id': ProductID, 'name': Name, 'price': Price) AS json_row
FROM Production.Product;
SELECT JSON_ARRAY(1, 'two', NULL, GETDATE()) AS arr;

-- GENERATE_SERIES: inline number/date sequences (replaces stacking CTEs)
SELECT value AS N FROM GENERATE_SERIES(1, 100);
SELECT DATEADD(day, value, '2024-01-01') AS D FROM GENERATE_SERIES(0, 364);

-- Approximate percentiles (T-Digest : fast on large datasets)
SELECT APPROX_PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY Duration) OVER () AS p95
FROM Telemetry.RequestLog;

-- S3 backup
BACKUP DATABASE YourDB TO URL = 's3://bucket/YourDB.bak'
    WITH COMPRESSION, CHECKSUM, CREDENTIAL = 'S3BackupCred';

-- Contained Availability Group (AG-level logins)
ALTER AVAILABILITY GROUP [YourAG] SET (CONTAINED = ON);
```

**Migration note:** Features marked "2022+" in other knowledge base files require `ALTER DATABASE YourDB SET COMPATIBILITY_LEVEL = 160` (the SQL Server 2022 compat level) to be available.
