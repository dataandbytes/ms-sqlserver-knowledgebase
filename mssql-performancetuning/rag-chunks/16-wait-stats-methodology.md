---
topic: Wait statistics baseline methodology and benign waits to exclude
keywords: [wait stats, sys.dm_os_wait_stats, delta, baseline, signal wait, benign waits, snapshot]
use_when: You are running a wait-stats analysis and need the correct delta methodology.
---

# Wait Statistics Methodology

Wait stats identify what resources threads wait for — start every performance investigation here, before looking at individual queries. When a thread can't progress it enters a wait queue recorded in `sys.dm_os_wait_stats`: `waiting_tasks_count`, `wait_time_ms` (incl. signal), `signal_wait_time_ms` (time on the runnable queue waiting for CPU). High signal-wait fraction = CPU saturation; low = blocked on external resources (I/O, locks, network).

The DMV accumulates since restart. **Always compare a delta between two snapshots** over a representative window (15–60 min of peak) — never act on raw cumulative totals. The top 2–3 wait types almost always point to the bottleneck.

```sql
SELECT wait_type, waiting_tasks_count, wait_time_ms, signal_wait_time_ms, GETDATE() AS snapshot_time
INTO #WaitBaseline
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','SLEEP_SYSTEMTASK','LAZYWRITER_SLEEP','SQLTRACE_BUFFER_FLUSH',
    'CHECKPOINT_QUEUE','BROKER_TO_FLUSH','BROKER_TASK_STOP','WAITFOR',
    'DISPATCHER_QUEUE_SEMAPHORE','XE_DISPATCHER_WAIT','XE_TIMER_EVENT','CXCONSUMER'
);
-- ... wait through the representative window ...
SELECT c.wait_type,
    c.wait_time_ms - b.wait_time_ms AS delta_wait_ms,
    c.signal_wait_time_ms - b.signal_wait_time_ms AS delta_signal_ms,
    CAST(100.0 * (c.wait_time_ms - b.wait_time_ms)
        / NULLIF(SUM(c.wait_time_ms - b.wait_time_ms) OVER (), 0) AS DECIMAL(5,2)) AS pct_of_total
FROM sys.dm_os_wait_stats c JOIN #WaitBaseline b ON b.wait_type = c.wait_type
WHERE c.wait_time_ms > b.wait_time_ms
ORDER BY delta_wait_ms DESC;
DROP TABLE #WaitBaseline;
```

Benign waits to exclude include all `SLEEP_*`, `WAITFOR`, `LAZYWRITER_SLEEP`, `XE_*`, `BROKER_*`, `CXCONSUMER`, `CHECKPOINT_QUEUE`, `DISPATCHER_QUEUE_SEMAPHORE`, `REQUEST_FOR_DEADLOCK_SEARCH`, and `HADR_*` (unless diagnosing Availability Groups).
