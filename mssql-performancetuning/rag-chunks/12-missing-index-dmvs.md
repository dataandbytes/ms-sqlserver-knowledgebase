---
topic: Missing index DMVs and their limitations
keywords: [missing index, dm_db_missing_index, index recommendation, equality columns, included columns]
use_when: You want index recommendations from the optimizer, or are evaluating a missing-index suggestion.
---

# Missing Index DMVs

The optimizer records missing-index recommendations during compilation. These DMVs **reset on restart** — collect before reboots.

```sql
SELECT TOP 20
    d.statement AS [Table], d.equality_columns, d.inequality_columns, d.included_columns,
    ROUND(s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans), 0) AS estimated_improvement,
    s.user_seeks, s.user_scans, s.last_user_seek
FROM sys.dm_db_missing_index_groups g
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
JOIN sys.dm_db_missing_index_details d ON g.index_handle = d.index_handle
WHERE d.database_id = DB_ID()
ORDER BY estimated_improvement DESC;
```

**Do not blindly create every suggestion.** The DMVs ignore: overlap with existing indexes of a different column order, write overhead on INSERT/UPDATE/DELETE, impact on other queries, and selectivity (a suggestion on a low-selectivity column may not help). Always validate against the actual execution plan before creating.
