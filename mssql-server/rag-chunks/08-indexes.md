---
topic: Nonclustered indexes : covering, filtered, rebuild vs reorganize, missing index DMVs, unused index detection
keywords: [nonclustered index, covering index, INCLUDE, filtered index, REBUILD, REORGANIZE, fragmentation, missing index, unused index, fill factor]
use_when: Creating an index to fix a Key Lookup or scan, choosing between REBUILD and REORGANIZE, finding missing or unused indexes.
---

# Indexes

```sql
-- Covering nonclustered: key columns satisfy seek; INCLUDE eliminates Key Lookup
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate
    ON Sales.Orders (CustomerID, OrderDate)
    INCLUDE (TotalAmount, Status);

-- Filtered index: covers only rows matching the filter (smaller, faster)
CREATE NONCLUSTERED INDEX IX_Orders_Pending
    ON Sales.Orders (ScheduledAt, CustomerID)
    INCLUDE (OrderID)
    WHERE Status = 'Pending';

-- Rebuild vs reorganize
ALTER INDEX IX_Orders_CustomerDate ON Sales.Orders
    REBUILD WITH (ONLINE = ON, FILLFACTOR = 85);
ALTER INDEX IX_Orders_CustomerDate ON Sales.Orders REORGANIZE;
```

**Fragmentation thresholds:**
| Fragmentation | Page count | Action |
|---|---|---|
| < 5% | any | Skip |
| 5–30% | > 1,000 | REORGANIZE |
| > 30% | > 1,000 | REBUILD |
| any | < 1,000 | Skip |

REORGANIZE does NOT update statistics : run `UPDATE STATISTICS` separately after.

```sql
-- Missing index candidates (reset on restart : wait one business cycle before acting)
SELECT TOP 10
    d.statement AS [Table],
    d.equality_columns, d.inequality_columns, d.included_columns,
    ROUND(s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans), 0) AS Score
FROM sys.dm_db_missing_index_groups g
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
JOIN sys.dm_db_missing_index_details d ON g.index_handle = d.index_handle
WHERE d.database_id = DB_ID()
ORDER BY Score DESC;

-- Unused indexes (high write cost, zero reads since last restart)
SELECT OBJECT_NAME(i.object_id) AS TableName, i.name,
       ISNULL(us.user_seeks + us.user_scans + us.user_lookups, 0) AS total_reads,
       ISNULL(us.user_updates, 0) AS writes
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us
    ON us.object_id = i.object_id AND us.index_id = i.index_id
    AND us.database_id = DB_ID()
WHERE i.type_desc = 'NONCLUSTERED'
ORDER BY total_reads ASC, writes DESC;
```
