---
topic: The ascending key problem and statistics update options
keywords: [ascending key, stale statistics, recent data, 1 row estimate, UPDATE STATISTICS, FULLSCAN, incremental statistics, filtered statistics, multi-column statistics]
use_when: Recent-date queries estimate 1 row, or you need to choose UPDATE STATISTICS options.
---

# Ascending Key Problem + Statistics Fixes

**Symptom:** a date/timestamp/IDENTITY predicate for recent data estimates ~1 row when thousands exist, causing the wrong join (Nested Loops instead of Hash Match) and an undersized memory grant (spills). **Cause:** between stat updates, new rows land beyond the histogram's max `RANGE_HI_KEY`; the CE has no data there and falls back to a density fraction.

**Diagnose:** compare the last histogram step to the actual max — `actual_max >> max RANGE_HI_KEY` confirms it.

**Fixes, in order of preference:**

```sql
-- 1. More frequent updates on rapidly-growing tables
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH FULLSCAN;

-- 2. Incremental stats (2014+) for partitioned tables — MUST be enabled at creation
--    (WITH INCREMENTAL = ON on an existing object raises error 9111)
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate ON dbo.Orders (OrderDate)
    WITH (STATISTICS_INCREMENTAL = ON);
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH RESAMPLE ON PARTITIONS (12, 13);

-- 3. Filtered statistics for recent data (maintain as "recent" moves)
CREATE STATISTICS stat_Orders_Recent_Date ON dbo.Orders (OrderDate) WHERE OrderDate >= '2024-01-01';
```

4. Compat level 130+ dynamic threshold — fires before the gap grows large.

**UPDATE STATISTICS options:**

| Option | When |
|---|---|
| `FULLSCAN` | After bulk load, ascending-key fix, initial setup — most accurate |
| `SAMPLE n PERCENT` | Balance speed vs accuracy on very large tables |
| `RESAMPLE` | Reuse last sample rate in maintenance jobs |
| `PERSIST_SAMPLE_PERCENT = ON` | 2016+: keep the FULLSCAN rate so auto-update doesn't revert to sampling |
| `NORECOMPUTE` | Rarely — disables auto-update for that object |

```sql
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;
EXEC sp_updatestats;  -- only changed stats, default sampling — may NOT fix ascending key
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH FULLSCAN, PERSIST_SAMPLE_PERCENT = ON;
```

**Multi-column statistics** fix correlated-predicate underestimates (`WHERE Status = 'Active' AND Region = 'West'` where the optimizer otherwise multiplies independent selectivities):

```sql
CREATE STATISTICS stat_Orders_Status_Region ON dbo.Orders (Status, Region);
```
