---
topic: TempDB : file sizing, version store, long-running transactions
keywords: [tempdb, data files, version store, dm_tran_version_store_space_usage, proportional fill]
use_when: Sizing or checking TempDB files, diagnosing version store bloat from long-running transactions.
---

# TempDB

```sql
-- Check file sizes (all data files must be equal for proportional fill)
SELECT name, type_desc, size * 8 / 1024 AS size_mb, growth * 8 / 1024 AS growth_mb
FROM tempdb.sys.database_files ORDER BY type_desc, file_id;

-- Equalize undersized data files
ALTER DATABASE tempdb MODIFY FILE (NAME = 'tempdev2', SIZE = 4096 MB, FILEGROWTH = 512 MB);

-- Version store size (RCSI / snapshot isolation)
SELECT DB_NAME(database_id) AS db_name,
       reserved_page_count * 8.0 / 1024 AS version_store_mb
FROM sys.dm_tran_version_store_space_usage ORDER BY reserved_page_count DESC;

-- Long-running transactions blocking version store cleanup
SELECT s.session_id, s.login_name, t.elapsed_time_seconds, t.transaction_sequence_num
FROM sys.dm_tran_active_snapshot_database_transactions t
JOIN sys.dm_exec_sessions s ON s.session_id = t.session_id
ORDER BY t.elapsed_time_seconds DESC;
```

**TempDB sizing rule:** One data file per logical CPU core, up to 8. All data files must be equal size. SQL Server 2016+ handles uniform extent allocation automatically. Remove trace flags 1117 and 1118 if present (they are no-ops and can cause confusion).
