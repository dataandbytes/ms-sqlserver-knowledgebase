---
topic: DBCC commands : CHECKDB, OPENTRAN, SHRINKFILE, SQLPERF, FREEPROCCACHE
keywords: [DBCC, CHECKDB, OPENTRAN, SHRINKFILE, SQLPERF, FREEPROCCACHE, DROPCLEANBUFFERS, consistency check, transaction, shrink]
use_when: Running database consistency checks, finding the oldest open transaction, shrinking a data or log file, or clearing cache for testing.
---

# DBCC Commands

```sql
-- DBCC CHECKDB: full database consistency check (page, row, index, allocation)
DBCC CHECKDB('YourDB') WITH NO_INFOMSGS, ALL_ERRORMSGS;
-- Use NO_INFOMSGS to suppress informational output
-- ALL_ERRORMSGS shows every error rather than the first 200

-- CHECKDB with repair (last resort : may cause data loss)
ALTER DATABASE YourDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
DBCC CHECKDB('YourDB', REPAIR_ALLOW_DATA_LOSS) WITH NO_INFOMSGS;
ALTER DATABASE YourDB SET MULTI_USER;

-- CHECKTABLE for a single table (fast targeted check)
DBCC CHECKTABLE('Sales.Orders') WITH NO_INFOMSGS;

-- Find oldest open transaction (blocks log truncation)
DBCC OPENTRAN('YourDB');

-- Shrink a data file (use sparingly : causes index fragmentation)
DBCC SHRINKFILE('YourDB_log', 1024);       -- target size 1024 MB
DBCC SHRINKDATABASE(YourDB, 10);            -- target 10% free space

-- Clear buffer pool (for testing cold-cache performance)
DBCC DROPCLEANBUFFERS;
DBCC FREEPROCCACHE;                         -- clear cached plans

-- Log space usage
DBCC SQLPERF(LOGSPACE);
```

**Key rules:**
- Run `DBCC CHECKDB` on a regular schedule (weekly minimum for critical databases). An untrusted database is a ticking clock.
- `DBCC SHRINKFILE` on data files causes severe index fragmentation. Always plan a REBUILD after shrinking. Log file shrinking does not cause fragmentation.
- `DBCC DROPCLEANBUFFERS` and `FREEPROCCACHE` are for development and testing only. Never run them on production.
- `DBCC OPENTRAN` returns nothing if there is no open transaction. If it returns a result, investigate immediately: that transaction blocks log truncation and VLF reuse.
- `REPAIR_ALLOW_DATA_LOSS` is a last resort. Always take a full backup before attempting repair. Prefer restoring from a clean backup over repair.
