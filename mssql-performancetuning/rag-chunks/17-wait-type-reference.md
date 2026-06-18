---
topic: Wait type reference and resource correlation
keywords: [CXPACKET, PAGEIOLATCH, PAGELATCH, WRITELOG, RESOURCE_SEMAPHORE, SOS_SCHEDULER_YIELD, THREADPOOL, ASYNC_NETWORK_IO, LCK_M]
use_when: You have a dominant wait type and need to know its cause and fix.
---

# Wait Type Reference

| Wait type | Meaning | Fix |
|---|---|---|
| `CXPACKET` | Thread waiting for slowest parallel sibling | Reduce MAXDOP; fix skewed distribution / bad CE. Alone it is normal parallel cost |
| `CXCONSUMER` | Consumer waiting for producer (paired with CXPACKET) | Reduce CXPACKET |
| `LCK_M_S` | Shared lock wait | Reader blocked by writer; consider RCSI |
| `LCK_M_X` | Exclusive lock wait | Writer blocked by writer; review transaction length |
| `LCK_M_U` | Update lock wait | Read-then-update; UPDLOCK to prevent S→X deadlocks |
| `PAGEIOLATCH_SH` / `_EX` | Data page fetched from disk into buffer pool | Missing index; working set > RAM; cold cache; I/O throughput |
| `PAGELATCH_EX` (no IO) | In-memory latch contention | tempdb GAM/PFS/SGAM contention — add equally-sized tempdb files |
| `WRITELOG` | Transaction log write latency | Faster/dedicated log disk (NVMe SSD); separate log from data |
| `SOS_SCHEDULER_YIELD` | Thread yielded CPU and re-queued | CPU saturation; reduce query cost |
| `RESOURCE_SEMAPHORE` | Waiting for a memory grant (sort/hash) | Fix statistics; row-mode MGF (2019+); check `max server memory` |
| `RESOURCE_SEMAPHORE_QUERY_COMPILE` | Too many concurrent compilations | Reduce OPTION(RECOMPILE) overuse / ad-hoc churn |
| `THREADPOOL` | Worker threads exhausted | Danger signal; raise max worker threads or reduce concurrency |
| `ASYNC_NETWORK_IO` | Server produced rows, client slow to consume | Client buffering; reduce result set; check bandwidth |
| `OLEDB` | Linked server waiting for remote | Use OPENQUERY; ETL to local tables |

Key distinction: `PAGELATCH_*` (no IO) is in-memory contention — faster disks will not help. `PAGEIOLATCH_*` (with IO) is a disk read problem.

```sql
-- Live active waits (not cumulative)
SELECT r.session_id, r.wait_type, r.wait_time / 1000.0 AS wait_sec, r.wait_resource,
       SUBSTRING(st.text, (r.statement_start_offset/2)+1, 200) AS stmt
FROM sys.dm_exec_requests r CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.wait_type IS NOT NULL AND r.session_id > 50 ORDER BY r.wait_time DESC;
```
