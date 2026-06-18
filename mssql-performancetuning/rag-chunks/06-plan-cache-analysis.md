---
topic: Plan cache analysis for expensive queries and ad-hoc bloat
keywords: [plan cache, dm_exec_query_stats, most expensive queries, single-use plans, ad-hoc, forced parameterization]
use_when: You want to find the most expensive queries server-wide or diagnose plan cache bloat.
---

# Plan Cache Analysis

Find the most expensive queries by average logical reads:

```sql
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    qs.execution_count,
    qs.total_worker_time / qs.execution_count / 1000 AS avg_cpu_ms,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS stmt,
    DB_NAME(st.dbid) AS db_name
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_reads DESC;
```

Diagnose ad-hoc plan bloat:

```sql
SELECT COUNT(*) AS single_use_plans,
       SUM(CAST(size_in_bytes AS BIGINT)) / 1024 / 1024 AS MB_wasted
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1 AND objtype = 'Adhoc';
```

A high count of single-use plans indicates unparameterized ad-hoc queries — enable forced parameterization or use `sp_executesql` with parameters. Note: `sys.dm_exec_query_stats` resets on restart and does not persist history — use Query Store for trend analysis over days/weeks.
