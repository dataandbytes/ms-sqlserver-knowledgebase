---
topic: Deadlocks and blocking diagnosis, including NOLOCK dangers
keywords: [deadlock, error 1205, deadlock victim, lock ordering, blocking chain, blocking_session_id, system_health, lock timeout, NOLOCK]
use_when: You are diagnosing deadlocks (error 1205) or a blocking chain, or considering NOLOCK.
---

# Deadlocks, Blocking, and NOLOCK

**Deadlock** = two sessions each hold a lock the other needs. The monitor runs every 5 seconds, detects the cycle, and kills the cheapest-to-roll-back session (the victim, error 1205). Prevention: (1) consistent lock ordering across all transactions; (2) `UPDLOCK` on reads to prevent S→X conversion deadlocks; (3) RCSI (readers stop taking S locks); (4) shorter transactions. **Always code for error 1205** — catch, wait briefly, retry.

```sql
-- Read deadlock graphs from the built-in system_health session
SELECT TOP 10 xdr.value('@timestamp', 'datetime2') AS deadlock_time, xdr.query('.') AS deadlock_graph_xml
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets t
    JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
    WHERE s.name = 'system_health' AND t.target_name = 'ring_buffer'
) AS d
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS XEventData(xdr)
ORDER BY deadlock_time DESC;
```

**Blocking** differs — a single chain where one session waits indefinitely for another to release a lock.

```sql
SELECT r.session_id AS blocked_session, r.blocking_session_id, r.wait_type,
       r.wait_time / 1000.0 AS wait_sec, r.wait_resource,
       SUBSTRING(st.text, (r.statement_start_offset/2)+1, 200) AS blocked_stmt,
       s.open_transaction_count, s.login_name, s.program_name
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON s.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0 ORDER BY r.wait_time DESC;
```

Common causes: forgotten `COMMIT`; app-side row-by-row processing inside a transaction; implicit transactions left open. Set `SET LOCK_TIMEOUT 5000;` so apps fail predictably rather than waiting forever.

**NOLOCK dangers:** `WITH (NOLOCK)` / READ UNCOMMITTED bypasses read consistency and can return rows twice, skip rows, produce torn reads, and read uncommitted (rolled-back) data. The performance benefit is overstated. Replace with RCSI for lock-free, consistent reads. NOLOCK as a deadlock fix does not work — deadlocks need lock ordering or RCSI. Only legitimate use: rough row-count estimates or dashboards where approximate correctness is explicitly acceptable.
