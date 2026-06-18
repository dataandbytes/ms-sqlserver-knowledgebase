---
topic: Over-indexing — finding unused and duplicate indexes
keywords: [too many indexes, unused index, duplicate index, write penalty, index usage stats, drop index]
use_when: A table has many indexes, writes are slow, or you want to find indexes safe to drop.
---

# Over-Indexing

Every NCI is maintained on every INSERT, on every UPDATE touching its key/INCLUDE columns, and on every DELETE. OLTP tables become write-bound above 5–7 NCIs. Measure, do not guess.

**Find unused indexes** (usage stats reset on restart — wait at least one full business cycle before dropping):

```sql
SELECT OBJECT_NAME(i.object_id) AS TableName, i.name AS IndexName, i.type_desc,
       us.user_seeks, us.user_scans, us.user_lookups, us.user_updates, us.last_user_seek
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us
    ON us.object_id = i.object_id AND us.index_id = i.index_id AND us.database_id = DB_ID()
WHERE OBJECT_NAME(i.object_id) = 'Orders' AND i.index_id > 1
ORDER BY ISNULL(us.user_seeks + us.user_scans + us.user_lookups, 0) ASC;
```

An index with `user_updates` in the millions and `user_seeks = 0` is pure overhead — evaluate dropping it.

**Find duplicate indexes** (same leading column):

```sql
SELECT t.name AS TableName, i1.name AS Index1, i2.name AS Index2
FROM sys.indexes i1
JOIN sys.indexes i2 ON i1.object_id = i2.object_id AND i1.index_id < i2.index_id
JOIN sys.index_columns ic1 ON i1.object_id = ic1.object_id AND i1.index_id = ic1.index_id AND ic1.index_column_id = 1
JOIN sys.index_columns ic2 ON i2.object_id = ic2.object_id AND i2.index_id = ic2.index_id AND ic2.index_column_id = 1
JOIN sys.tables t ON i1.object_id = t.object_id
WHERE ic1.column_id = ic2.column_id AND i1.type > 0 AND i2.type > 0;
```
