---
topic: Performance diagnostics : top CPU queries, wait stats, blocking chains, spill detection
keywords: [DMV, dm_exec_query_stats, dm_os_wait_stats, blocking, dm_exec_requests, spills, top queries, wait statistics]
use_when: Diagnosing high CPU, identifying top queries, measuring waits, finding blocking sessions, or detecting memory spills.
---

# Performance Diagnostics (DMVs)

```sql
-- Top 10 queries by total CPU since last restart
SELECT TOP 10
    qs.total_worker_time / 1000                          AS total_cpu_ms,
    qs.total_worker_time / qs.execution_count / 1000     AS avg_cpu_ms,
    qs.total_logical_reads / qs.execution_count          AS avg_reads,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY total_cpu_ms DESC;

-- Wait statistics delta (snapshot → wait → diff)
SELECT wait_type, waiting_tasks_count, wait_time_ms
INTO #WaitSnap FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN ('SLEEP_TASK','WAITFOR','SQLTRACE_BUFFER_FLUSH',
    'CHECKPOINT_QUEUE','XE_DISPATCHER_WAIT','CXCONSUMER')
  AND wait_type NOT LIKE 'SLEEP_%';
-- ...wait 60+ seconds...
SELECT c.wait_type,
    c.wait_time_ms - b.wait_time_ms AS delta_ms,
    CAST(100.0 * (c.wait_time_ms - b.wait_time_ms)
        / NULLIF(SUM(c.wait_time_ms - b.wait_time_ms) OVER (), 0) AS DECIMAL(5,2)) AS pct
FROM sys.dm_os_wait_stats c JOIN #WaitSnap b ON b.wait_type = c.wait_type
WHERE c.wait_time_ms > b.wait_time_ms
ORDER BY delta_ms DESC;

-- Active blocking chains
SELECT r.session_id AS blocked, r.blocking_session_id AS blocker,
       r.wait_type, r.wait_time / 1000.0 AS wait_sec,
       SUBSTRING(st.text, (r.statement_start_offset/2)+1, 200) AS blocked_stmt
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0;

-- Queries with hash/sort spills (2016 SP1+)
SELECT TOP 10 qs.total_spills / qs.execution_count AS avg_spills,
       SUBSTRING(st.text, 1, 200) AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.total_spills > 0 ORDER BY avg_spills DESC;
```

**Common wait type diagnoses:**
| Wait type | Cause |
|---|---|
| `LCK_M_*` | Blocking / locking : find the head-of-chain blocker |
| `PAGEIOLATCH_SH` | Missing index or cold buffer cache |
| `CXPACKET` | Parallelism imbalance or MAXDOP too high |
| `RESOURCE_SEMAPHORE` | Memory grant starvation : fix statistics or reduce sort/hash |
| `WRITELOG` | Log I/O bottleneck : separate log to fast disk |
| `SOS_SCHEDULER_YIELD` | CPU saturation |
