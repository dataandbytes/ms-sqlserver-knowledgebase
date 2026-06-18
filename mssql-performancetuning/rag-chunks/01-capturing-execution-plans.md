---
topic: Capturing execution plans and reading STATISTICS IO/TIME
keywords: [execution plan, actual plan, estimated plan, statistics io, statistics time, logical reads, SSMS]
use_when: You need to capture a query's plan or measure its I/O and CPU cost.
---

# Capturing Execution Plans + STATISTICS IO/TIME

**Capture in SSMS:** `Ctrl+L` = estimated plan (no run); `Ctrl+M` = toggle actual plan mode then run; `Ctrl+Shift+Q` = live query statistics. Always use the **actual** plan when diagnosing — estimated plans hide actual row counts, granted memory, and spills.

Pull a plan from cache after running the query:

```sql
SELECT TOP 1 qp.query_plan,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.last_execution_time DESC;
```

**Measure cost** with STATISTICS IO/TIME (logical reads is the primary I/O metric):

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO
SELECT o.OrderID, o.TotalAmt FROM dbo.Orders o WHERE o.CustomerID = 42;
```

| Field | Meaning | What to look for |
|---|---|---|
| `logical reads` | Pages read from buffer pool (8 KB each) | Primary cost; high = scan or key lookups |
| `physical reads` | Pages fetched from disk | High = cold cache or missing index |
| `scan count > 1` | Table scanned multiple times | Inner table of Nested Loops — costly at scale |
| CPU time vs elapsed | — | CPU ≈ elapsed = CPU-bound; elapsed ≫ CPU = waiting (I/O, locks) |

Logical reads to MB: `logical_reads × 8 / 1024`. **Plan cost percentages are optimizer estimates, not measured time** — a 5% operator can dominate wall-clock; always validate with STATISTICS IO/TIME or actual row counts.
