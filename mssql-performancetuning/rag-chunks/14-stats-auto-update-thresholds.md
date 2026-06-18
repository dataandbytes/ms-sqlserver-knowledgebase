---
topic: Statistics auto-update thresholds (legacy vs dynamic) and freshness monitoring
keywords: [auto update statistics, threshold, 20 percent, dynamic threshold, compat 130, modification_counter, trace flag 2371]
use_when: Auto-update statistics seems to never fire on a large table, or you are tuning stats freshness.
---

# Statistics Auto-Update Thresholds

SQL Server auto-updates statistics when a modification counter exceeds a threshold.

**Legacy** (pre-compat 130): empty table = any insert; < 500 rows = 500 mods; ≥ 500 rows = 500 + 20% of rows. On a 10M-row table that is 2M modifications before firing — stats can be stale for days.

**Dynamic** (compat 130+, SQL Server 2016+) — fires at whichever is smaller, no trace flags needed:

```
threshold = MIN(500 + 0.20 × n,  SQRT(1000 × n))
```

| Row count | Legacy | Dynamic | Effective |
|---|---|---|---|
| 100,000 | 20,500 | ~10,000 | ~10,000 |
| 1,000,000 | 200,500 | ~31,623 | ~31,623 |
| 10,000,000 | 2,000,500 | ~100,000 | ~100,000 |
| 100,000,000 | 20,000,500 | ~316,228 | ~316,228 |

~20× improvement at 10M rows. **Fix for "auto-update never fires on a big table": upgrade to compatibility level 130+.** (Trace flag 2371 forced this on older versions but is now obsolete.)

```sql
-- Check modification counters
SELECT OBJECT_NAME(s.object_id) AS table_name, s.name AS stats_name,
       sp.last_updated, sp.rows, sp.rows_sampled, sp.modification_counter,
       CAST(100.0 * sp.modification_counter / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS pct_modified
FROM sys.stats s CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('dbo.Orders') ORDER BY sp.modification_counter DESC;

-- Freshness monitor: not updated in 24h with > 1% modifications
SELECT OBJECT_NAME(s.object_id) AS table_name, s.name AS stats_name, sp.last_updated,
       sp.modification_counter, sp.rows
FROM sys.stats s CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE sp.modification_counter > sp.rows * 0.01
  AND sp.last_updated < DATEADD(HOUR, -24, GETDATE())
  AND OBJECT_NAME(s.object_id) NOT LIKE 'sys%'
ORDER BY sp.modification_counter DESC;
```
