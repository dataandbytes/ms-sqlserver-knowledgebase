---
topic: TempDB file configuration and max server memory
keywords: [tempdb, PAGELATCH, GAM SGAM PFS, tempdb files, proportional fill, trace flag 1117 1118, max server memory]
use_when: You see PAGELATCH_EX contention on tempdb, or are configuring tempdb files / max server memory.
---

# TempDB Configuration + Max Server Memory

**Symptom:** `PAGELATCH_EX` waits on pages like `2:1:1` = contention on tempdb allocation bitmaps (GAM/SGAM/PFS). **Fix:** equally-sized tempdb data files so SQL Server's proportional-fill algorithm spreads allocations across files.

- 2016+ setup recommends 1 data file per logical core, capped at 8.
- All data files identical in initial size and autogrowth.
- Trace flags 1117 (uniform autogrowth) and 1118 (uniform extent allocation) are **obsolete** as of 2016 — this is the default for tempdb now. Do not add them.

```sql
SELECT name, type_desc, size * 8 / 1024 AS size_mb, growth * 8 / 1024 AS growth_mb, is_percent_growth
FROM tempdb.sys.database_files ORDER BY type_desc, file_id;
ALTER DATABASE tempdb MODIFY FILE (NAME = 'tempdev2', SIZE = 4096 MB, FILEGROWTH = 512 MB);
```

**Max server memory:** `RESOURCE_SEMAPHORE` waits can persist if `max server memory` is set too high, starving the OS and causing Windows to page SQL Server memory. Reserve 10% or 4 GB (whichever is larger) for the OS.

```sql
SELECT name, value_in_use FROM sys.configurations
WHERE name IN ('max server memory (MB)', 'min server memory (MB)');
EXEC sp_configure 'max server memory (MB)', 28672;  -- e.g. 28 GB on a 32 GB server
RECONFIGURE;
```
