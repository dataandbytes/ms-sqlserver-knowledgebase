---
topic: Index fragmentation, rebuild vs reorganize, and fill factor
keywords: [fragmentation, rebuild, reorganize, fill factor, page split, ALTER INDEX, maintenance]
use_when: You are running index maintenance, or deciding rebuild vs reorganize and what fill factor to set.
---

# Fragmentation, Rebuild vs Reorganize, Fill Factor

| Fragmentation | Page count | Action |
|---|---|---|
| < 5% | Any | Ignore |
| 5–30% | > 1,000 pages | `REORGANIZE` |
| > 30% | > 1,000 pages | `REBUILD` |
| Any | < 1,000 pages | Ignore — fix cost > fragmentation cost |

```sql
SELECT OBJECT_NAME(ps.object_id) AS TableName, i.name AS IndexName,
       ps.avg_fragmentation_in_percent, ps.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ps
JOIN sys.indexes i ON ps.object_id = i.object_id AND ps.index_id = i.index_id
WHERE ps.page_count > 128 ORDER BY ps.avg_fragmentation_in_percent DESC;

ALTER INDEX IX_Orders_CustomerID ON dbo.Orders REORGANIZE;             -- online, in place
ALTER INDEX IX_Orders_CustomerID ON dbo.Orders REBUILD WITH (ONLINE = ON);  -- full defrag + FULLSCAN stats
```

**REBUILD updates statistics** (FULLSCAN-equivalent); **REORGANIZE does not** — run `UPDATE STATISTICS` separately afterward if stats are stale.

**Fill factor** sets leaf-page fullness at build/rebuild time only (pages fill naturally afterward):

| Scenario | Fill factor |
|---|---|
| Read-only / archival | 100 |
| Monotonic clustered key (IDENTITY) | 100 clustered, 90–95 NCIs |
| Random inserts into existing key range (natural keys) | 70–80 |
| High-update heap or NCI with mid-page inserts | 70–75 |

```sql
ALTER INDEX IX_Orders_CustomerID ON dbo.Orders REBUILD WITH (FILLFACTOR = 80, ONLINE = ON);
```
