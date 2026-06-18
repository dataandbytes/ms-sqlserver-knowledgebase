---
topic: Diagnostic flowchart for a slow SQL Server query
keywords: [slow query, where to start, diagnose, used to be fast, troubleshoot, methodology]
use_when: A SQL Server query is slow and you do not yet know the cause; you need a starting sequence.
---

# Slow Query: Diagnostic Sequence

When a SQL Server query is slow, work these five steps in order. Each step narrows the cause before the next.

**Step 1 — Get the actual execution plan.** Run with `SET STATISTICS IO ON; SET STATISTICS TIME ON;` and capture the actual plan (Ctrl+M in SSMS). Look for: Key Lookup (add covering INCLUDE columns), fat arrows / estimated rows ≪ actual rows (update statistics, check parameter sniffing), yellow triangle on Hash Match (memory grant too small, hash spilled — fix statistics), Table Scan on a large table (add clustered index or rewrite a non-SARGable predicate), Index Spool (create the permanent index).

**Step 2 — Check wait statistics.** Snapshot `sys.dm_os_wait_stats`, wait through the slow window, then diff. Dominant wait → cause: `LCK_M_*` = blocking/locking; `PAGEIOLATCH_SH` = missing index or cold cache; `CXPACKET` = parallelism imbalance / MAXDOP too high; `RESOURCE_SEMAPHORE` = memory grant starvation; `WRITELOG` = log I/O bottleneck; `SOS_SCHEDULER_YIELD` = CPU saturation.

**Step 3 — Evaluate indexes** via `sys.dm_db_missing_index_*` DMVs (they reset on restart, so act promptly), but validate every suggestion against the actual plan first.

**Step 4 — Check statistics freshness.** Compare the histogram's last `RANGE_HI_KEY` to the actual `MAX()` (ascending key problem) and check `modification_counter` via `sys.dm_db_stats_properties`.

**Step 5 — Diagnose parameter sniffing.** In Query Store, a single query with many distinct cached plan_ids signals sniffing instability.

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- run the query with Ctrl+M (Include Actual Plan) in SSMS
```
