---
topic: Isolation levels and Read Committed Snapshot Isolation (RCSI)
keywords: [isolation level, READ COMMITTED, SNAPSHOT, RCSI, row versioning, version store, dirty read, phantom, reader writer blocking]
use_when: You are choosing an isolation level, or considering RCSI to eliminate reader/writer blocking.
---

# Isolation Levels + RCSI

| Level | Dirty read | Non-repeatable | Phantom | Implementation |
|---|---|---|---|---|
| READ UNCOMMITTED | Yes | Yes | Yes | No shared locks (never safe for financial data) |
| READ COMMITTED | No | Yes | Yes | S locks released after read (**default**) |
| REPEATABLE READ | No | No | Yes | S locks held until commit |
| SERIALIZABLE | No | No | No | Key-range locks (severe blocking) |
| SNAPSHOT | No | No | No | Row versioning from tempdb (txn-scoped view) |
| READ COMMITTED SNAPSHOT | No | Yes | Yes | Row versioning, no S locks (RCSI — db option) |

Choosing: mixed OLTP readers+writers → RCSI; long reports alongside OLTP writes → SNAPSHOT; financial conflict prevention → pessimistic (`UPDLOCK + SERIALIZABLE`) or SNAPSHOT with conflict handling; Azure SQL Database → RCSI always on.

**RCSI** is the most impactful change for a read-heavy OLTP database: it replaces shared locks on reads with row versioning, so readers and writers never block each other (writers still block writers). No app changes — default READ COMMITTED reads automatically benefit.

```sql
SELECT name, is_read_committed_snapshot_on FROM sys.databases WHERE name = DB_NAME();
ALTER DATABASE YourDatabase SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;
```

**Cost:** the tempdb version store grows during peak writes (14-byte header + before-image per updated row). A long-running snapshot transaction prevents version cleanup — find and address it:

```sql
SELECT DB_NAME(database_id) AS db_name, reserved_page_count * 8.0 / 1024 AS version_store_mb
FROM sys.dm_tran_version_store_space_usage ORDER BY reserved_page_count DESC;

SELECT TOP 5 s.session_id, s.login_name, t.elapsed_time_seconds, t.transaction_sequence_num
FROM sys.dm_tran_active_snapshot_database_transactions t
JOIN sys.dm_exec_sessions s ON t.session_id = s.session_id
ORDER BY t.elapsed_time_seconds DESC;
```
