# SQL Server Performance — Knowledge Base

> Single-file reference for diagnosing and optimizing SQL Server database performance:
> slow T-SQL queries, index tuning, execution plans, parameter sniffing, batch operations,
> transaction log bloat, locking/blocking, tempdb configuration, and wait statistics.

## Scope

**Use this for:** a query that used to be fast is now slow; reading execution plans; choosing
index strategies (clustered, covering, filtered); diagnosing locking, blocking, or deadlocks;
writing bulk INSERT/UPDATE/DELETE without bloating the log; fixing parameter sniffing; choosing
between temp tables and table variables; analyzing wait statistics.

**Not covered:** application schema design (table design, naming, access control), backup/restore
strategy, high availability configuration, replication setup, columnstore indexes.

## Contents

1. [Diagnostic Flowchart](#diagnostic-flowchart)
2. [Index Strategy](#index-strategy)
3. [Execution Plans](#execution-plans)
4. [Statistics Tuning](#statistics-tuning)
5. [Wait Statistics](#wait-statistics)
6. [Locking and Blocking](#locking-and-blocking)
7. [Batch Operations](#batch-operations)
8. [SARGability](#sargability)
9. [Parameter Sniffing](#parameter-sniffing)
10. [Temp Tables vs Table Variables](#temp-tables-vs-table-variables)
11. [Common Mistakes](#common-mistakes)
12. [Sources](#sources)

---

## Diagnostic Flowchart

When a query is slow, follow this sequence — each step narrows the cause before moving to the next.

**Step 1 — Get the actual execution plan**

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- Run the query with Ctrl+M (Include Actual Plan) in SSMS
```

Look for:
- **Key Lookup** — NCI seek followed by a clustered lookup per row. At 1,000+ rows this dominates. Fix: add missing SELECT columns to the NCI INCLUDE list.
- **Fat arrows** (thick connectors) between operators — estimated rows ≪ actual rows. Cardinality estimation failure. Fix: update statistics, check for parameter sniffing.
- **Yellow triangle on Hash Match** — memory grant too small, hash spilled to tempdb. Fix: update statistics so the grant sizes correctly.
- **Table Scan on a large table** — no clustered index or a non-SARGable predicate. Fix: add a clustered index or rewrite the predicate.
- **Index Spool** — SQL Server built a temporary index on the fly. Fix: create the permanent index.

**Step 2 — Check wait statistics**

```sql
-- Snapshot before the problem window
SELECT wait_type, wait_time_ms INTO #w FROM sys.dm_os_wait_stats;
-- ...wait through the slow period (60s minimum)...

-- Delta: what consumed wait time during the window
SELECT c.wait_type, c.wait_time_ms - b.wait_time_ms AS delta_ms,
       CAST(100.0 * (c.wait_time_ms - b.wait_time_ms)
            / NULLIF(SUM(c.wait_time_ms - b.wait_time_ms) OVER (), 0) AS DECIMAL(5,2)) AS pct
FROM sys.dm_os_wait_stats c JOIN #w b ON b.wait_type = c.wait_type
WHERE c.wait_time_ms > b.wait_time_ms
  AND c.wait_type NOT IN (
      'SLEEP_TASK','LAZYWRITER_SLEEP','WAITFOR','CXCONSUMER',
      'SQLTRACE_BUFFER_FLUSH','CHECKPOINT_QUEUE','XE_DISPATCHER_WAIT',
      'XE_TIMER_EVENT','BROKER_TO_FLUSH','BROKER_TASK_STOP','DISPATCHER_QUEUE_SEMAPHORE'
  )
  AND c.wait_type NOT LIKE 'SLEEP_%'
ORDER BY delta_ms DESC;
DROP TABLE #w;
```

| Dominant wait | Likely cause |
|---|---|
| `LCK_M_*` | Blocking/locking — see [Locking and Blocking](#locking-and-blocking) |
| `PAGEIOLATCH_SH` | Missing index or cold buffer pool |
| `CXPACKET` | Parallelism imbalance or MAXDOP too high |
| `RESOURCE_SEMAPHORE` | Memory grant starvation — hash/sort spills |
| `WRITELOG` | Transaction log I/O bottleneck |
| `SOS_SCHEDULER_YIELD` | CPU saturation |
| `PAGELATCH_EX` on `2:1:*` | TempDB allocation contention — add equally-sized tempdb files |

**Step 3 — Evaluate indexes**

```sql
-- Top missing index recommendations (resets on restart)
SELECT TOP 10
    d.statement                                                     AS [Table],
    d.equality_columns,
    d.inequality_columns,
    d.included_columns,
    ROUND(s.avg_total_user_cost * s.avg_user_impact
          * (s.user_seeks + s.user_scans), 0)                       AS estimated_improvement,
    s.user_seeks,
    s.last_user_seek
FROM sys.dm_db_missing_index_groups g
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
JOIN sys.dm_db_missing_index_details d     ON g.index_handle       = d.index_handle
WHERE d.database_id = DB_ID()
ORDER BY estimated_improvement DESC;
```

**Step 4 — Check statistics freshness**

```sql
-- Ascending key check: compare histogram max to actual max
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate') WITH HISTOGRAM;
SELECT MAX(OrderDate) AS actual_max FROM dbo.Orders;
-- If actual_max >> last RANGE_HI_KEY: ascending key problem. Fix: UPDATE STATISTICS WITH FULLSCAN

-- Modification counter: how stale are stats?
SELECT s.name AS stats_name, sp.last_updated, sp.modification_counter,
       CAST(100.0 * sp.modification_counter / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS pct_modified
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('dbo.Orders')
ORDER BY sp.modification_counter DESC;
```

**Step 5 — Diagnose parameter sniffing**

```sql
-- Queries with many distinct cached plans = sniffing instability
SELECT qsq.query_id, COUNT(DISTINCT qsp.plan_id) AS plan_count,
       qsqt.query_sql_text
FROM sys.query_store_query      qsq
JOIN sys.query_store_plan       qsp  ON qsp.query_id       = qsq.query_id
JOIN sys.query_store_query_text qsqt ON qsqt.query_text_id = qsq.query_text_id
GROUP BY qsq.query_id, qsqt.query_sql_text
HAVING COUNT(DISTINCT qsp.plan_id) > 3
ORDER BY plan_count DESC;
```

**Full server health check** — run this first on a new server you've never diagnosed:

```sql
-- 1. Top 10 queries by total CPU (since last restart)
SELECT TOP 10
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200)            AS stmt,
    qs.total_worker_time / 1000                                          AS total_cpu_ms,
    qs.total_worker_time / qs.execution_count / 1000                    AS avg_cpu_ms,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count                         AS avg_reads,
    DB_NAME(st.dbid)                                                     AS db_name
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY total_cpu_ms DESC;

-- 2. Tables with heaviest write load (index maintenance targets)
SELECT TOP 10 OBJECT_NAME(ios.object_id) AS TableName,
       SUM(ios.leaf_insert_count + ios.leaf_update_count + ios.leaf_delete_count) AS total_writes,
       SUM(ios.range_scan_count + ios.singleton_lookup_count)                      AS total_reads
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL) ios
WHERE OBJECT_NAME(ios.object_id) IS NOT NULL
GROUP BY ios.object_id ORDER BY total_writes DESC;

-- 3. Current blocking chains
SELECT r.session_id AS blocked, r.blocking_session_id AS blocker,
       r.wait_type, r.wait_time / 1000.0 AS wait_sec,
       SUBSTRING(st.text, (r.statement_start_offset/2)+1, 100) AS blocked_stmt
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0;
```

---

## Index Strategy

A table should have one well-chosen clustered index and targeted covering nonclustered indexes for secondary access patterns. Every index has a write penalty — keep OLTP tables at 3–5 nonclustered indexes total.

### Clustered Index Selection

The clustered index IS the table — leaf pages store actual rows in clustered key order. Every nonclustered index stores a copy of the clustered key as its row locator. A bad choice cascades to every other index.

Choose a key that is:
- **Narrow** — `UNIQUEIDENTIFIER` (16 bytes) vs `INT` (4 bytes) adds 12 bytes per NCI row. With 10 NCIs and 100M rows that is 12 GB of extra index storage.
- **Ever-increasing** — random inserts cause 50/50 page splits. `IDENTITY`, `SEQUENCE`, or `NEWSEQUENTIALID()` produce cheap 90/10 splits.
- **Unique** — SQL Server appends a 4-byte uniquifier to duplicate clustered key values, bloating the index.
- **Static** — updating the clustered key physically moves the row and cascades to every NCI.

```sql
-- GOOD: narrow, unique, ever-increasing
CREATE TABLE dbo.Orders (
    OrderID    INT           NOT NULL IDENTITY(1,1),
    CustomerID INT           NOT NULL,
    OrderDate  DATETIME2(0)  NOT NULL,
    TotalAmt   DECIMAL(12,2) NOT NULL,
    Status     TINYINT       NOT NULL DEFAULT 1,
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderID)
);

-- BAD: GUID clustered key bloats every NCI by 12 bytes per row
-- CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderGUID)
-- Fix: use NEWSEQUENTIALID() if a GUID is required as the clustered key
CREATE TABLE dbo.OrdersGuid (
    OrderGUID  UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID(),
    CustomerID INT              NOT NULL,
    OrderDate  DATETIME2(0)     NOT NULL,
    CONSTRAINT PK_OrdersGuid PRIMARY KEY CLUSTERED (OrderGUID)
);
```

**When the clustered key is not the primary key** — for tables with heavy range scans on a non-PK column:

```sql
-- EventLog: mostly accessed by date range, not EventID
CREATE TABLE dbo.EventLog (
    EventID    BIGINT        NOT NULL IDENTITY(1,1),
    OccurredAt DATETIME2(3)  NOT NULL,
    EventType  TINYINT       NOT NULL,
    Severity   TINYINT       NOT NULL,
    Message    NVARCHAR(500) NOT NULL,
    CONSTRAINT PK_EventLog PRIMARY KEY NONCLUSTERED (EventID)  -- PK but NOT the clustered key
);
CREATE CLUSTERED INDEX CIX_EventLog_OccurredAt ON dbo.EventLog (OccurredAt);
-- Range scans by date are now sequential reads; point lookups by EventID use the NCI

-- Useful query: verify row locator size impact on NCIs
SELECT i.name AS index_name, i.type_desc,
       SUM(a.used_pages) * 8        AS used_kb,
       SUM(a.data_pages) * 8        AS data_kb
FROM sys.indexes i
JOIN sys.partitions p ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.allocation_units a ON a.container_id = p.partition_id
WHERE i.object_id = OBJECT_ID('dbo.Orders')
GROUP BY i.name, i.type_desc ORDER BY used_kb DESC;
```

### Covering Indexes and INCLUDE Columns

A covering index satisfies a query entirely from the index — no Key Lookup. Each Key Lookup traverses the clustered B-tree (2–4 page reads) per row; at 1,000 rows that is thousands of extra logical reads, scaling linearly.

**Rule:** columns in the index *key* only if they appear in WHERE/JOIN/ORDER BY. Everything else in INCLUDE.

```sql
-- SCENARIO: CustomerID lookup that also returns OrderDate, Status, TotalAmt

-- BEFORE — causes Key Lookup (visible in execution plan)
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
    ON dbo.Orders (CustomerID);
-- STATISTICS IO output: logical reads 3,402 (key lookup dominates)

-- AFTER — covering index eliminates the Key Lookup entirely
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID_Covering
    ON dbo.Orders (CustomerID)
    INCLUDE (OrderDate, Status, TotalAmt);
-- STATISTICS IO output: logical reads 12 (index seek only)

-- SCENARIO: range query on date, filtered by status, sorted
-- Query: WHERE CustomerID = @cid AND Status = 1 AND OrderDate > @since ORDER BY OrderDate
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Status_Date
    ON dbo.Orders (CustomerID, Status, OrderDate)   -- equality, equality, range+sort
    INCLUDE (TotalAmt);                              -- SELECT-only column goes here

-- Verify the Key Lookup is gone: scan the plan cache
SELECT TOP 20 qs.total_logical_reads / qs.execution_count AS avg_reads,
       qs.execution_count,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%KeyLookup%'
ORDER BY avg_reads DESC;
```

### Filtered Indexes

A filtered index covers only rows matching a WHERE predicate — smaller, cheaper to maintain, more selective per byte.

```sql
-- SCENARIO 1: Queue table — only 'Pending' rows are ever searched
CREATE NONCLUSTERED INDEX IX_WorkQueue_Pending
    ON dbo.WorkQueue (ScheduledFor, QueuedAt)
    INCLUDE (WorkItemID, AttemptNum)
    WHERE Status = 'Pending';
-- Without filter: full-table NCI maintaining index on millions of completed rows
-- With filter:    < 1% of rows, dramatically faster seeks and cheaper maintenance

-- SCENARIO 2: Partial unique constraint — allow multiple NULLs, block duplicate values
CREATE UNIQUE NONCLUSTERED INDEX UX_Customer_ExternalID
    ON dbo.Customer (ExternalID)
    WHERE ExternalID IS NOT NULL;

-- SCENARIO 3: Sparse error column — only non-NULL errors need indexing
CREATE NONCLUSTERED INDEX IX_EventLog_ErrorCode
    ON dbo.EventLog (ErrorCode, OccurredAt)
    WHERE ErrorCode IS NOT NULL;

-- Check that a filtered index is actually being used
SELECT i.name, i.filter_definition,
       us.user_seeks, us.user_scans, us.user_updates, us.last_user_seek
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us
    ON us.object_id = i.object_id AND us.index_id = i.index_id AND us.database_id = DB_ID()
WHERE i.object_id = OBJECT_ID('dbo.WorkQueue')
  AND i.filter_definition IS NOT NULL;
```

### Over-Indexing

Every NCI is maintained on every INSERT, every UPDATE touching its key/INCLUDE columns, and every DELETE. OLTP tables become write-bound above 5–7 NCIs.

```sql
-- Write overhead per index on Orders table
SELECT OBJECT_NAME(ios.object_id) AS TableName, i.name AS IndexName,
       ios.leaf_insert_count, ios.leaf_update_count, ios.leaf_delete_count,
       ios.leaf_insert_count + ios.leaf_update_count + ios.leaf_delete_count AS total_writes,
       ios.range_scan_count + ios.singleton_lookup_count AS total_reads
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL) ios
JOIN sys.indexes i ON ios.object_id = i.object_id AND ios.index_id = i.index_id
WHERE OBJECT_NAME(ios.object_id) = 'Orders'
ORDER BY total_writes DESC;

-- Find unused indexes (usage stats reset on restart — wait one full business cycle)
SELECT OBJECT_NAME(i.object_id) AS TableName, i.name AS IndexName,
       ISNULL(us.user_seeks, 0)   AS seeks,
       ISNULL(us.user_scans, 0)   AS scans,
       ISNULL(us.user_lookups, 0) AS lookups,
       ISNULL(us.user_updates, 0) AS writes,
       us.last_user_seek
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us
    ON us.object_id = i.object_id AND us.index_id = i.index_id AND us.database_id = DB_ID()
WHERE i.type_desc = 'NONCLUSTERED'
  AND OBJECT_NAME(i.object_id) NOT LIKE 'sys%'
ORDER BY ISNULL(us.user_seeks + us.user_scans + us.user_lookups, 0) ASC,
         ISNULL(us.user_updates, 0) DESC;

-- Find duplicate indexes (same leading column)
SELECT t.name AS TableName, i1.name AS Index1, i2.name AS Index2
FROM sys.indexes i1
JOIN sys.indexes i2 ON i1.object_id = i2.object_id AND i1.index_id < i2.index_id
JOIN sys.index_columns ic1 ON i1.object_id = ic1.object_id AND i1.index_id = ic1.index_id AND ic1.index_column_id = 1
JOIN sys.index_columns ic2 ON i2.object_id = ic2.object_id AND i2.index_id = ic2.index_id AND ic2.index_column_id = 1
JOIN sys.tables t ON i1.object_id = t.object_id
WHERE ic1.column_id = ic2.column_id AND i1.type > 0 AND i2.type > 0;
```

### Fragmentation: Rebuild vs Reorganize

| Fragmentation | Page count | Action |
|---|---|---|
| < 5% | Any | Ignore |
| 5–30% | > 1,000 pages | `REORGANIZE` |
| > 30% | > 1,000 pages | `REBUILD` |
| Any | < 1,000 pages | Ignore — fix cost > fragmentation cost |

```sql
-- Check fragmentation across all indexes on all tables
SELECT OBJECT_NAME(ps.object_id) AS TableName, i.name AS IndexName, i.type_desc,
       ps.avg_fragmentation_in_percent, ps.page_count,
       CASE
           WHEN ps.page_count < 1000                     THEN 'Skip (small)'
           WHEN ps.avg_fragmentation_in_percent < 5      THEN 'Skip (low frag)'
           WHEN ps.avg_fragmentation_in_percent < 30     THEN 'REORGANIZE'
           ELSE                                               'REBUILD'
       END AS recommended_action
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ps
JOIN sys.indexes i ON ps.object_id = i.object_id AND ps.index_id = i.index_id
WHERE ps.index_id > 0
ORDER BY ps.avg_fragmentation_in_percent DESC;

-- REORGANIZE: online, compacts in place, does NOT update statistics
ALTER INDEX IX_Orders_CustomerID_Covering ON dbo.Orders REORGANIZE;
UPDATE STATISTICS dbo.Orders IX_Orders_CustomerID_Covering WITH FULLSCAN; -- run separately

-- REBUILD: full defragmentation + FULLSCAN statistics update
ALTER INDEX IX_Orders_CustomerID_Covering ON dbo.Orders REBUILD WITH (ONLINE = ON, FILLFACTOR = 85);

-- Rebuild all indexes on a table
ALTER INDEX ALL ON dbo.Orders REBUILD WITH (ONLINE = ON);
```

### Fill Factor

```sql
-- Fill factor applies only at rebuild time. Good starting values:
--   100 — read-only / archival tables
--    95 — monotonic IDENTITY keys (slight buffer for NCIs)
--    80 — random inserts into existing key range (natural keys)
--    75 — high-update heap or NCI with frequent mid-page inserts

ALTER INDEX IX_Orders_CustomerID_Covering ON dbo.Orders REBUILD WITH (FILLFACTOR = 80, ONLINE = ON);

-- Monitor page split activity (high = fill factor too tight for this workload)
SELECT OBJECT_NAME(ios.object_id) AS TableName, i.name AS IndexName,
       ios.leaf_allocation_count  AS page_splits,
       ios.leaf_update_count,
       ios.leaf_insert_count
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL) ios
JOIN sys.indexes i ON ios.object_id = i.object_id AND ios.index_id = i.index_id
WHERE ios.leaf_allocation_count > 1000
ORDER BY ios.leaf_allocation_count DESC;
```

---

## Execution Plans

### Capturing Plans

**In SSMS:** `Ctrl+L` = estimated (no run), `Ctrl+M` = toggle actual plan mode then run, `Ctrl+Shift+Q` = live query statistics. Always use the **actual** plan — estimated plans hide actual row counts, granted memory, and spills.

```sql
-- Pull the most recently executed plan from cache
SELECT TOP 1
    qp.query_plan,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    qs.total_worker_time   / qs.execution_count / 1000 AS avg_cpu_ms
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)    st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.last_execution_time DESC;

-- Find a specific stored procedure's plan
SELECT qp.query_plan, qs.execution_count,
       qs.total_worker_time / qs.execution_count / 1000 AS avg_cpu_ms
FROM sys.dm_exec_procedure_stats ps
CROSS APPLY sys.dm_exec_query_plan(ps.plan_handle) qp
CROSS APPLY sys.dm_exec_sql_text(ps.sql_handle)    st
WHERE OBJECT_NAME(ps.object_id) = 'uspGetCustomerOrders';
```

### SET STATISTICS IO and TIME

Always run alongside a slow query. Logical reads = the primary I/O cost metric.

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

-- Example: before adding a covering index
SELECT o.OrderID, o.OrderDate, o.TotalAmt, o.Status
FROM dbo.Orders o
WHERE o.CustomerID = 42
ORDER BY o.OrderDate DESC;
GO
-- Table 'Orders'. Scan count 1, logical reads 4821   <-- high due to key lookup
-- CPU time = 140 ms, elapsed time = 148 ms

-- After adding the covering index:
-- Table 'Orders'. Scan count 1, logical reads 8      <-- seek only
-- CPU time = 0 ms, elapsed time = 1 ms

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

| Field | What to look for |
|---|---|
| `logical reads` high | Scan or key lookups — missing or non-covering index |
| `physical reads` high | Cold cache or missing index pushing to disk |
| `scan count > 1` | Table is the inner loop of a Nested Loops join |
| CPU ≈ elapsed | CPU-bound (computation, sorts) |
| elapsed ≫ CPU | Waiting on I/O, locks, or network |

Logical reads to approximate MB: `logical_reads × 8 / 1024`.

### Key Operators

**Scan vs Seek** — a seek is O(log n); a scan is O(n).

| Operator | Status | Fix when bad |
|---|---|---|
| Clustered/Nonclustered Index Seek | Good | Check if a Key Lookup follows an NCI seek |
| Clustered Index Scan | Investigate | Non-SARGable predicate or deliberately large read |
| Table Scan | Always investigate | Missing clustered index |

**Key Lookup — before and after:**

```sql
-- BEFORE: IX_Orders_CustomerID lacks OrderDate, TotalAmt
-- Plan shows: NCI Seek → Key Lookup (per row) → SELECT
-- STATISTICS IO: logical reads 2,840

-- AFTER: covering index
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID_v2
    ON dbo.Orders (CustomerID) INCLUDE (OrderDate, TotalAmt, Status);
-- Plan shows: NCI Seek → SELECT (no Key Lookup)
-- STATISTICS IO: logical reads 6

-- Find all queries doing Key Lookups across the whole server
SELECT TOP 20 qs.total_logical_reads / qs.execution_count AS avg_reads,
       qs.execution_count,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt,
       DB_NAME(st.dbid) AS db_name
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)    st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%KeyLookup%'
ORDER BY avg_reads DESC;
```

**Join algorithms:**

| Algorithm | Best when | Memory grant |
|---|---|---|
| Nested Loops | Small outer, large inner with index seek | None |
| Hash Match | Large unsorted inputs, no useful index | Yes — can spill |
| Merge Join | Both inputs already sorted | None if sorted |

```sql
-- Force Nested Loops to compare against the optimizer's choice
SELECT o.OrderID, c.CustomerName
FROM dbo.Orders o
JOIN dbo.Customers c ON c.CustomerID = o.CustomerID
OPTION (LOOP JOIN);    -- or (HASH JOIN), (MERGE JOIN)

-- Detect queries with hash or sort spills (SQL Server 2016 SP1+)
SELECT TOP 20 qs.total_spills / qs.execution_count AS avg_spills,
       qs.execution_count,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.total_spills > 0
ORDER BY avg_spills DESC;
```

**Parallelism** — parallel setup (~50ms) is net-negative for queries completing under 100ms.

```sql
-- Force serial execution for a specific query
SELECT * FROM dbo.Orders WHERE CustomerID = 42 OPTION (MAXDOP 1);

-- Server-wide: check for queries where parallelism hurts more than it helps
SELECT TOP 20 qs.total_elapsed_time / qs.execution_count / 1000 AS avg_elapsed_ms,
       qs.total_worker_time  / qs.execution_count / 1000          AS avg_cpu_ms,
       -- If avg_cpu_ms >> avg_elapsed_ms, parallelism is likely helping
       -- If avg_elapsed_ms >> avg_cpu_ms, waiting on something else
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200)   AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.max_dop > 1   -- ran in parallel at least once
ORDER BY avg_elapsed_ms DESC;
```

### Warning Signs

| Warning (yellow triangle) | Meaning | Fix |
|---|---|---|
| Implicit conversion | Type mismatch forces column conversion; kills seeks | Match parameter types to column types |
| Missing index | Optimizer found a beneficial missing index | Evaluate and create if justified |
| No join predicate | Cartesian product — missing ON clause | Add the join condition |
| Memory grant warning | Spill likely from underestimated grant | Fix statistics |
| Residual I/O | Rows from storage > rows returned | Add a predicate to the index key |

```sql
-- Find implicit conversions killing index seeks (PlanAffectingConvert)
SELECT TOP 20 qs.total_logical_reads / qs.execution_count AS avg_reads,
       SUBSTRING(st.text, 1, 200) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)    st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%PlanAffectingConvert%'
ORDER BY avg_reads DESC;

-- Example that triggers PlanAffectingConvert — column is INT, param passed as VARCHAR
-- BAD:
DECLARE @cid VARCHAR(10) = '42';
SELECT * FROM dbo.Orders WHERE CustomerID = @cid; -- implicit INT→VARCHAR conversion, forces scan

-- GOOD: match the type
DECLARE @cid INT = 42;
SELECT * FROM dbo.Orders WHERE CustomerID = @cid; -- seeks normally
```

### Cardinality Estimation

CE predicts row counts per operator. Wrong estimates cascade into wrong join algorithms, wrong memory grants, and wrong parallelism decisions.

```sql
-- CE version follows compatibility level
SELECT name, compatibility_level FROM sys.databases WHERE name = DB_NAME();

-- Test with legacy CE (CE70) when a compat upgrade caused a regression
SELECT o.OrderID, o.TotalAmt
FROM dbo.Orders o WHERE o.OrderDate > '2025-01-01'
OPTION (USE HINT ('FORCE_LEGACY_CARDINALITY_ESTIMATION'));

-- Apply legacy CE database-wide while you diagnose regressions
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = ON;
-- Remove after queries are fixed
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = OFF;

-- Check what CE assumptions are actually in use for a query
-- In the XML plan, look for: CardinalityEstimationModelVersion="160"
SELECT CAST(qp.query_plan AS NVARCHAR(MAX)) AS plan_xml
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%uspGetCustomerOrders%'
  AND qp.query_plan IS NOT NULL;
```

| CE version | SQL Server | Compat |
|---|---|---|
| CE70 (legacy) | 2012 and earlier | ≤ 110 |
| CE120 | 2014 | 120 |
| CE130 | 2016 | 130 |
| CE140 | 2017 | 140 |
| CE150 | 2019 | 150 |
| CE160 | 2022 | 160 |

### Intelligent Query Processing (IQP)

```sql
-- Check which IQP features are active
SELECT name, compatibility_level FROM sys.databases WHERE name = DB_NAME();
SELECT name, value FROM sys.database_scoped_configurations
WHERE name IN ('DEFERRED_COMPILATION_TV','BATCH_MODE_ADAPTIVE_JOINS','MEMORY_GRANT_FEEDBACK');

-- Find plans where Memory Grant Feedback has adjusted a grant
SELECT qsp.plan_id, qsp.query_plan_hash, qsp.is_feedback_adjusted_grant,
       qsrs.avg_query_max_used_memory * 8 / 1024 AS avg_max_grant_mb
FROM sys.query_store_plan qsp
JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
WHERE qsp.is_feedback_adjusted_grant = 1;

-- Disable specific features when they cause problems
SELECT * FROM dbo.Orders OPTION (USE HINT ('DISABLE_BATCH_MODE_ADAPTIVE_JOINS'));
SELECT * FROM dbo.Orders OPTION (USE HINT ('DISABLE_QUERY_PLAN_FEEDBACK'));
ALTER DATABASE SCOPED CONFIGURATION SET DEFERRED_COMPILATION_TV = OFF;
```

| Feature | Min compat | What it does |
|---|---|---|
| Adaptive Joins | 140 | Chooses Nested Loops vs Hash Match at runtime |
| Batch Mode on Rowstore | 150 | Columnstore-style batch processing on rowstore |
| Table Variable Deferred Compilation | 150 | Defers compilation until populated (fixes 1-row estimate) |
| Row-Mode Memory Grant Feedback | 150 (2019) | Adjusts sort/hash grants on later executions |
| DOP Feedback | 160 | Lowers MAXDOP for queries that gain nothing from parallelism |
| CE Feedback | 160 | Corrects CE assumptions on repeated executions |
| PSPO | 160 | Multiple plan variants for skewed parameter distributions |

### Plan Cache Analysis

```sql
-- Most expensive queries by average logical reads
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count                     AS avg_reads,
    qs.execution_count,
    qs.total_worker_time  / qs.execution_count / 1000               AS avg_cpu_ms,
    qs.total_elapsed_time / qs.execution_count / 1000               AS avg_elapsed_ms,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS stmt,
    DB_NAME(st.dbid) AS db_name
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_reads DESC;

-- Single-use ad-hoc plans bloating cache
SELECT COUNT(*) AS single_use_plans,
       SUM(CAST(size_in_bytes AS BIGINT)) / 1024 / 1024 AS MB_wasted
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1 AND objtype = 'Adhoc';
-- Fix: use sp_executesql or enable forced parameterization
EXEC sp_executesql N'SELECT * FROM dbo.Orders WHERE CustomerID = @cid',
                   N'@cid INT', @cid = 42;  -- parameterized: reuses the plan

-- Inspect cache type breakdown
SELECT objtype, COUNT(*) AS plan_count,
       SUM(size_in_bytes) / 1024 / 1024 AS total_mb,
       AVG(usecounts) AS avg_use_count
FROM sys.dm_exec_cached_plans
GROUP BY objtype ORDER BY total_mb DESC;
```

### Query Store for Plan Regression

```sql
-- Enable Query Store with recommended settings
ALTER DATABASE YourDatabase SET QUERY_STORE = ON;
ALTER DATABASE YourDatabase SET QUERY_STORE (
    OPERATION_MODE            = READ_WRITE,
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    QUERY_CAPTURE_MODE        = AUTO,
    MAX_STORAGE_SIZE_MB       = 1024,
    SIZE_BASED_CLEANUP_MODE   = AUTO,
    WAIT_STATS_CAPTURE_MODE   = ON
);

-- Check state (ReadOnly = store is full)
SELECT actual_state_desc, current_storage_size_mb, max_storage_size_mb
FROM sys.database_query_store_options;

-- Find regressions: recent plan significantly slower than historical best
WITH BestPlan AS (
    SELECT qsq.query_id, qsp.plan_id, MIN(qsrs.avg_duration) AS best_us,
           ROW_NUMBER() OVER (PARTITION BY qsq.query_id ORDER BY MIN(qsrs.avg_duration)) AS rn
    FROM sys.query_store_query         qsq
    JOIN sys.query_store_plan          qsp  ON qsp.query_id = qsq.query_id
    JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
    GROUP BY qsq.query_id, qsp.plan_id
),
RecentPlan AS (
    SELECT qsq.query_id, qsp.plan_id, AVG(qsrs.avg_duration) AS recent_us
    FROM sys.query_store_query               qsq
    JOIN sys.query_store_plan                qsp  ON qsp.query_id = qsq.query_id
    JOIN sys.query_store_runtime_stats       qsrs ON qsrs.plan_id = qsp.plan_id
    JOIN sys.query_store_runtime_stats_interval qsrsi
         ON qsrsi.runtime_stats_interval_id = qsrs.runtime_stats_interval_id
    WHERE qsrsi.start_time >= DATEADD(HOUR, -4, GETUTCDATE())
    GROUP BY qsq.query_id, qsp.plan_id
)
SELECT r.query_id,
       b.plan_id AS best_plan_id, r.plan_id AS current_plan_id,
       b.best_us / 1000.0 AS best_ms, r.recent_us / 1000.0 AS current_ms,
       r.recent_us / NULLIF(b.best_us, 0.0) AS regression_ratio
FROM RecentPlan r
JOIN BestPlan   b ON b.query_id = r.query_id AND b.rn = 1
WHERE r.recent_us > b.best_us * 1.5
ORDER BY regression_ratio DESC;

-- Force a known-good plan
EXEC sys.sp_query_store_force_plan @query_id = 42, @plan_id = 7;

-- Monitor forced plans (fail if schema changes)
SELECT qsp.plan_id, qsp.force_failure_count, qsp.last_force_failure_reason_desc
FROM sys.query_store_plan qsp WHERE qsp.is_forced_plan = 1;

-- Top 10 queries by lock wait time (last 24 hours)
SELECT TOP 10 qsqt.query_sql_text,
       SUM(qsws.total_query_wait_time_ms) AS lock_wait_ms
FROM sys.query_store_query_text qsqt
JOIN sys.query_store_query      qsq  ON qsq.query_text_id          = qsqt.query_text_id
JOIN sys.query_store_plan       qsp  ON qsp.query_id               = qsq.query_id
JOIN sys.query_store_wait_stats qsws ON qsws.plan_id               = qsp.plan_id
JOIN sys.query_store_runtime_stats_interval qsrsi
     ON qsrsi.runtime_stats_interval_id = qsws.runtime_stats_interval_id
WHERE qsws.wait_category_desc = 'Lock'
  AND qsrsi.start_time >= DATEADD(HOUR, -24, GETUTCDATE())
GROUP BY qsqt.query_sql_text
ORDER BY lock_wait_ms DESC;
```

---

## Statistics Tuning

Statistics describe value distribution in columns. The optimizer uses them to estimate how many rows each predicate will return (cardinality). Wrong cardinality estimates → wrong join algorithms → wrong memory grants → wrong parallelism decisions → slow queries.

```sql
-- Verify auto-create and auto-update are enabled
SELECT name, is_auto_create_stats_on, is_auto_update_stats_on, is_auto_update_stats_async_on
FROM sys.databases WHERE name = DB_NAME();
-- is_auto_update_stats_async_on: OFF = synchronous (preferred for OLTP; fresh stats per plan)
--                                ON  = async (consistent latency; plans may use stale stats until update completes)
```

### Histogram Structure

The histogram covers only the **leading column** and has up to **200 steps**.

| Column | Meaning |
|---|---|
| `RANGE_HI_KEY` | Upper bound value of this step |
| `EQ_ROWS` | Estimated rows equal to `RANGE_HI_KEY` |
| `RANGE_ROWS` | Rows between previous and current `RANGE_HI_KEY` |
| `DISTINCT_RANGE_ROWS` | Distinct values in the range (excluding `RANGE_HI_KEY`) |
| `AVG_RANGE_ROWS` | `RANGE_ROWS / DISTINCT_RANGE_ROWS` — average rows per value in the range |

```sql
-- Read the full statistics object
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate');               -- header + density + histogram
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate') WITH HISTOGRAM;   -- histogram only
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate') WITH STAT_HEADER; -- updated, rows, rows_sampled

-- List all statistics on a table with freshness info
SELECT s.name AS stats_name, s.filter_definition,
       sp.last_updated, sp.rows, sp.rows_sampled,
       CAST(100.0 * sp.rows_sampled / NULLIF(sp.rows, 0) AS DECIMAL(5,1)) AS sample_pct,
       sp.modification_counter,
       CAST(100.0 * sp.modification_counter / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS pct_modified
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('dbo.Orders')
ORDER BY sp.last_updated ASC;  -- oldest first
```

### Auto-Update Thresholds

**Legacy (pre-compat 130):** 500 + 20% of rows. On a 10M-row table that is 2M modifications — stats stale for days.

**Dynamic (compat 130+):**
```
threshold = MIN(500 + 0.20 × n,  SQRT(1000 × n))
```

| Row count | Legacy | Dynamic | Effective |
|---|---|---|---|
| 100,000 | 20,500 | ~10,000 | **~10,000** |
| 1,000,000 | 200,500 | ~31,623 | **~31,623** |
| 10,000,000 | 2,000,500 | ~100,000 | **~100,000** |
| 100,000,000 | 20,000,500 | ~316,228 | **~316,228** |

~20× improvement at 10M rows. Upgrade to compat 130+ — no trace flags needed.

```sql
-- Check current compat level
SELECT name, compatibility_level FROM sys.databases WHERE name = DB_NAME();

-- Modification counter: which stats objects are stalest?
SELECT OBJECT_NAME(s.object_id) AS table_name, s.name AS stats_name,
       sp.last_updated, sp.modification_counter, sp.rows,
       CAST(100.0 * sp.modification_counter / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS pct_modified
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('dbo.Orders')
ORDER BY sp.modification_counter DESC;

-- Server-wide staleness report: all stats not updated in 24h with > 1% modifications
SELECT OBJECT_NAME(s.object_id) AS table_name, DB_NAME() AS db_name,
       s.name AS stats_name, sp.last_updated, sp.rows, sp.modification_counter,
       CAST(100.0 * sp.modification_counter / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS pct_modified
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE sp.modification_counter > sp.rows * 0.01
  AND sp.last_updated < DATEADD(HOUR, -24, GETDATE())
  AND OBJECT_NAME(s.object_id) NOT LIKE 'sys%'
ORDER BY pct_modified DESC;
```

### The Ascending Key Problem

The most common statistics failure: queries filtering a date/timestamp/IDENTITY column for **recent data** estimate ~1 row when millions exist.

**Root cause:** new rows land beyond the histogram's last `RANGE_HI_KEY`. The CE has no data there and falls back to a density fraction.

**Symptom in plan:** estimated rows = 1, actual rows = 50,000 on a date predicate → wrong join algorithm (Nested Loops instead of Hash Match) → undersized memory grant → sort/hash spill.

```sql
-- Diagnose: compare last histogram step to actual max
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate') WITH HISTOGRAM;
-- Note the last RANGE_HI_KEY value (e.g., 2024-11-15)

SELECT MAX(OrderDate) AS actual_max FROM dbo.Orders;
-- If actual_max = 2025-06-18 and RANGE_HI_KEY = 2024-11-15: 7-month gap → 1-row estimate

-- Fix 1: force a full statistics update immediately
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH FULLSCAN;

-- Fix 2: incremental statistics on partitioned tables (must be enabled at creation)
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate ON dbo.Orders (OrderDate)
    WITH (STATISTICS_INCREMENTAL = ON);
-- Then update only the new partition, not the whole table
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH RESAMPLE ON PARTITIONS (24, 25);

-- Fix 3: filtered statistics covering only recent data
CREATE STATISTICS stat_Orders_2025_Date ON dbo.Orders (OrderDate)
WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01';

-- Fix 4: persist the FULLSCAN sample rate so auto-update doesn't revert to sampling
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate
WITH FULLSCAN, PERSIST_SAMPLE_PERCENT = ON;
```

### UPDATE STATISTICS Options

| Option | Description | When |
|---|---|---|
| `FULLSCAN` | Read every row — most accurate | After bulk load, ascending key fix, initial setup |
| `SAMPLE n PERCENT` | Read n% of rows | Balance speed vs accuracy on huge tables |
| `RESAMPLE` | Use the same sample rate as the last update | Consistent maintenance jobs |
| `INCREMENTAL = ON` | Partition-level update | Partitioned tables with fast-growing newest partitions |
| `PERSIST_SAMPLE_PERCENT = ON` | Remember the FULLSCAN rate for auto-updates | 2016+: prevents auto-update reverting to default sampling |
| `NORECOMPUTE` | Disable auto-update for this object | Rarely — breaks automatic freshness |

```sql
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;                                         -- all stats on table
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH FULLSCAN;                     -- one stats object
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH SAMPLE 30 PERCENT;            -- faster but less accurate
UPDATE STATISTICS dbo.Orders IX_Orders_OrderDate WITH FULLSCAN, PERSIST_SAMPLE_PERCENT = ON;

EXEC sp_updatestats;  -- updates only changed stats, uses default sampling — does NOT fix ascending key
```

### Multi-Column and Filtered Statistics

```sql
-- SCENARIO: query has WHERE Status = 'Active' AND Region = 'West'
-- If Status and Region are correlated, the optimizer multiplies individual selectivities
-- and underestimates the result set → wrong join → possible spill

-- Fix: multi-column statistics capture the joint distribution
CREATE STATISTICS stat_Orders_Status_Region ON dbo.Orders (Status, Region);
-- Optimizer uses this when the query references Status alone, or Status + Region together

DBCC SHOW_STATISTICS ('dbo.Orders', 'stat_Orders_Status_Region') WITH DENSITY_VECTOR;
-- Low "All density" = high selectivity = useful statistic

-- Filtered statistics: heavy skew where 95% of rows are Status='Completed', 5% are 'Active'
CREATE STATISTICS stat_Orders_Active_Date ON dbo.Orders (OrderDate) WHERE Status = 'Active';
-- Used when query WHERE clause includes Status = 'Active' (or stronger predicate)

-- Check all statistics including filters
SELECT s.name, s.filter_definition, sp.last_updated, sp.rows, sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('dbo.Orders')
ORDER BY sp.last_updated DESC;
```

---

## Wait Statistics

Wait stats tell you the *category* of bottleneck before you look at individual queries. Start every performance investigation here.

### Baseline Methodology

```sql
-- Step 1: snapshot before the peak window (15–60 min)
SELECT wait_type, waiting_tasks_count, wait_time_ms, signal_wait_time_ms,
       GETDATE() AS snapshot_time
INTO #WaitBaseline
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','SLEEP_SYSTEMTASK','SLEEP_DBSTARTUP','SLEEP_DCOMSTARTUP',
    'SLEEP_MASTERMDREADY','SLEEP_MASTERUPGRADED','SLEEP_MSDBSTARTUP',
    'SLEEP_TEMPDBSTARTUP','SLEEP_MASTERSTART','WAITFOR','LAZYWRITER_SLEEP',
    'SQLTRACE_BUFFER_FLUSH','SQLTRACE_INCREMENTAL_FLUSH_SLEEP','CHECKPOINT_QUEUE',
    'BROKER_TO_FLUSH','BROKER_TASK_STOP','BROKER_EVENTHANDLER',
    'DISPATCHER_QUEUE_SEMAPHORE','FT_IFTS_SCHEDULER_IDLE_WAIT',
    'XE_DISPATCHER_WAIT','XE_TIMER_EVENT','CXCONSUMER',
    'SP_SERVER_DIAGNOSTICS_SLEEP','RESOURCE_QUEUE','SERVER_IDLE_CHECK',
    'REQUEST_FOR_DEADLOCK_SEARCH','LOGMGR_QUEUE','ONDEMAND_TASK_QUEUE',
    'KSOURCE_WAKEUP','SQLTRACE_WAIT_ENTRIES','DBMIRROR_EVENTS_QUEUE'
)
AND wait_type NOT LIKE 'SLEEP_%'
AND wait_type NOT LIKE 'HADR_%';

-- ... wait through the representative window ...

-- Step 2: delta ranks wait types by time consumed; top 2–3 point to the bottleneck
SELECT c.wait_type,
    c.waiting_tasks_count - b.waiting_tasks_count AS delta_tasks,
    c.wait_time_ms - b.wait_time_ms               AS delta_wait_ms,
    c.signal_wait_time_ms - b.signal_wait_time_ms AS delta_signal_ms,
    CAST(100.0 * (c.wait_time_ms - b.wait_time_ms)
        / NULLIF(SUM(c.wait_time_ms - b.wait_time_ms) OVER (), 0) AS DECIMAL(5,2)) AS pct_of_total
FROM sys.dm_os_wait_stats c
JOIN #WaitBaseline b ON b.wait_type = c.wait_type
WHERE c.wait_time_ms > b.wait_time_ms
ORDER BY delta_wait_ms DESC;
DROP TABLE #WaitBaseline;
```

**Signal wait interpretation:** if `delta_signal_ms / delta_wait_ms > 0.25`, the server is CPU-saturated — threads are ready to run but cannot get a CPU slot. If low, threads block on external resources (I/O, locks).

### Wait Type Reference

| Wait type | Meaning | Fix |
|---|---|---|
| `CXPACKET` | Thread waiting for slowest parallel sibling | Reduce MAXDOP; fix data-distribution skew; fix CE causing unnecessary parallelism |
| `CXCONSUMER` | Consumer waiting for producer (always paired with CXPACKET) | Reduce CXPACKET — CXCONSUMER follows |
| `LCK_M_S` | Shared lock wait | Reader blocked by writer; enable RCSI |
| `LCK_M_X` | Exclusive lock wait | Writer blocked by writer; shorten transactions |
| `LCK_M_U` | Update lock wait | Read-then-update pattern; use UPDLOCK |
| `PAGEIOLATCH_SH` | Data page fetched from disk | Missing index; working set > RAM; cold cache |
| `PAGELATCH_EX` (no IO) | In-memory latch contention | TempDB GAM/PFS contention — add equally-sized tempdb files |
| `WRITELOG` | Transaction log write latency | Dedicated NVMe SSD for log file |
| `SOS_SCHEDULER_YIELD` | Thread yielded CPU and re-queued | CPU saturation; reduce query cost |
| `RESOURCE_SEMAPHORE` | Waiting for memory grant | Fix statistics; check `max server memory` |
| `THREADPOOL` | Worker threads exhausted | Danger signal; increase max worker threads |
| `ASYNC_NETWORK_IO` | Server produced rows but client slow | Client buffering; reduce result set |

Key distinction: `PAGELATCH_*` (no IO) is in-memory contention — faster disks will not help. `PAGEIOLATCH_*` (with IO) is a disk-read problem.

### Live and Per-Query Wait Stats

```sql
-- Live active waits (not cumulative totals)
SELECT r.session_id, r.wait_type, r.wait_time / 1000.0 AS wait_sec,
       r.wait_resource,
       SUBSTRING(st.text, (r.statement_start_offset/2)+1, 200) AS stmt,
       r.blocking_session_id
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.wait_type IS NOT NULL AND r.session_id > 50
ORDER BY r.wait_time DESC;

-- Per-query wait breakdown via Query Store (2017+, requires WAIT_STATS_CAPTURE_MODE = ON)
SELECT qsws.wait_category_desc,
       SUM(qsws.total_query_wait_time_ms) AS total_wait_ms,
       AVG(qsws.avg_query_wait_time_ms)   AS avg_wait_ms,
       COUNT(*)                            AS intervals
FROM sys.query_store_wait_stats           qsws
JOIN sys.query_store_runtime_stats_interval qsrsi
    ON qsrsi.runtime_stats_interval_id = qsws.runtime_stats_interval_id
WHERE qsws.plan_id = @plan_id
  AND qsrsi.start_time >= DATEADD(HOUR, -24, GETUTCDATE())
GROUP BY qsws.wait_category_desc
ORDER BY total_wait_ms DESC;
```

### TempDB Configuration

`PAGELATCH_EX` waits on pages like `2:1:1` = GAM/SGAM/PFS contention. Fix: equally-sized tempdb files so proportional fill spreads allocations.

```sql
-- Check current tempdb file sizes (they must all match)
SELECT name, type_desc,
       size * 8 / 1024           AS size_mb,
       growth * 8 / 1024         AS growth_mb,
       is_percent_growth,
       physical_name
FROM tempdb.sys.database_files
ORDER BY type_desc, file_id;

-- Equalize sizes (run for each undersized file)
ALTER DATABASE tempdb MODIFY FILE (NAME = 'tempdev2', SIZE = 4096 MB, FILEGROWTH = 512 MB);
ALTER DATABASE tempdb MODIFY FILE (NAME = 'tempdev3', SIZE = 4096 MB, FILEGROWTH = 512 MB);
ALTER DATABASE tempdb MODIFY FILE (NAME = 'tempdev4', SIZE = 4096 MB, FILEGROWTH = 512 MB);
-- Note: Trace flags 1117 and 1118 are obsolete on SQL Server 2016+ — do not add them

-- Max server memory: RESOURCE_SEMAPHORE persists if too high (OS starvation → paging)
-- Reserve 10% or 4 GB (whichever is larger) for the OS
SELECT name, value_in_use FROM sys.configurations
WHERE name IN ('max server memory (MB)', 'min server memory (MB)');

-- Example: 32 GB server, reserve 4 GB for OS, give 28 GB to SQL Server
EXEC sp_configure 'max server memory (MB)', 28672;
RECONFIGURE;

-- Check how much memory SQL Server is currently using
SELECT physical_memory_in_use_kb / 1024 AS sql_memory_mb,
       page_fault_count
FROM sys.dm_os_process_memory;

-- Monitor RCSI / snapshot version store size (tempdb growth source)
SELECT DB_NAME(database_id) AS db_name,
       reserved_page_count * 8.0 / 1024 AS version_store_mb
FROM sys.dm_tran_version_store_space_usage
ORDER BY reserved_page_count DESC;
```

---

## Locking and Blocking

### Isolation Levels

| Level | Dirty read | Non-repeatable | Phantom | Notes |
|---|---|---|---|---|
| `READ UNCOMMITTED` | Yes | Yes | Yes | Never safe for financial data |
| `READ COMMITTED` | No | Yes | Yes | **Default** — S locks released after read |
| `REPEATABLE READ` | No | No | Yes | S locks held until commit; high contention |
| `SERIALIZABLE` | No | No | No | Key-range locks; severe blocking |
| `SNAPSHOT` | No | No | No | Row versioning from tempdb; txn-scoped view |
| `READ COMMITTED SNAPSHOT` | No | Yes | Yes | RCSI — row versioning, no S locks |

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;     -- session default
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;            -- requires database option

SELECT * FROM dbo.Orders WITH (NOLOCK);              -- READ UNCOMMITTED (see NOLOCK Dangers)
SELECT * FROM dbo.Orders WITH (HOLDLOCK);            -- SERIALIZABLE
```

### Read Committed Snapshot Isolation (RCSI)

The most impactful change for a read-heavy OLTP database: replaces shared locks on reads with row versioning, so readers and writers never block each other. No application changes needed — default READ COMMITTED reads automatically benefit.

```sql
-- Check current state
SELECT name, is_read_committed_snapshot_on FROM sys.databases WHERE name = DB_NAME();

-- Enable RCSI (briefly needs single-user access during the switch)
ALTER DATABASE YourDatabase SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;

-- Verify the old NOLOCK pattern is no longer needed
-- BEFORE (RCSI off): SELECT * FROM dbo.Orders WITH (NOLOCK) -- added to avoid blocking
-- AFTER (RCSI on):   SELECT * FROM dbo.Orders               -- consistent reads, no S locks

-- Monitor version store growth
SELECT DB_NAME(database_id) AS db_name, reserved_page_count * 8.0 / 1024 AS version_store_mb
FROM sys.dm_tran_version_store_space_usage ORDER BY reserved_page_count DESC;

-- Find long-running transactions preventing version store cleanup
SELECT TOP 5 s.session_id, s.login_name, s.program_name,
       t.elapsed_time_seconds, t.transaction_sequence_num
FROM sys.dm_tran_active_snapshot_database_transactions t
JOIN sys.dm_exec_sessions s ON t.session_id = s.session_id
ORDER BY t.elapsed_time_seconds DESC;
```

### Lock Modes

| Mode | Acquired by | Blocks |
|---|---|---|
| Shared (S) | SELECT (pessimistic) | X locks |
| Update (U) | UPDATE before modifying | U and X |
| Exclusive (X) | INSERT/UPDATE/DELETE | All others |
| Schema Stability (Sch-S) | SELECT compilation | Schema modifications only |
| Schema Modification (Sch-M) | DDL (ALTER TABLE, CREATE INDEX) | All locks |
| Bulk Update (BU) | BULK INSERT with TABLOCK | Other BU locks |
| Key-Range | SERIALIZABLE scans | Prevents phantom inserts in range |

Intent locks (IS, IX, SIX) at table/page level allow efficient compatibility checking. IX+IX is compatible — two writers on different rows in the same table proceed simultaneously.

### Lock Escalation

SQL Server escalates to a table lock when **one statement** acquires ≥ 5,000 locks on one object (checks fire every 1,250). The threshold is **per statement**, not per transaction. A table-level X lock blocks all readers and writers.

```sql
-- Check current escalation setting
SELECT name, lock_escalation_desc FROM sys.tables WHERE name = 'Orders';

-- Partition-first escalation (reduces contention on partitioned tables)
ALTER TABLE dbo.Orders SET (LOCK_ESCALATION = AUTO);

-- Detect lock escalation events via Extended Events
CREATE EVENT SESSION [LockEscalation] ON SERVER
ADD EVENT sqlserver.lock_escalation (
    ACTION (sqlserver.sql_text, sqlserver.session_id)
    WHERE sqlserver.database_name = N'YourDatabase'
)
ADD TARGET package0.ring_buffer;
GO
ALTER EVENT SESSION [LockEscalation] ON SERVER STATE = START;
```

### Lock Hints

```sql
-- READPAST + UPDLOCK: concurrent queue consumer pattern
-- Multiple workers can each claim a different row without blocking each other
DECLARE @ItemID INT;
BEGIN TRANSACTION;
    SELECT TOP (1) @ItemID = WorkItemID
    FROM dbo.WorkQueue WITH (READPAST, ROWLOCK, UPDLOCK)
    WHERE Status = 'Pending' AND ScheduledFor <= SYSDATETIME()
    ORDER BY QueuedAt;
    IF @ItemID IS NOT NULL
        UPDATE dbo.WorkQueue
        SET Status = 'Processing', StartedAt = SYSDATETIME()
        WHERE WorkItemID = @ItemID;
COMMIT;

-- UPDLOCK + HOLDLOCK: prevent phantom inserts in a check-then-insert pattern
-- Without these hints: two sessions can both see "no row" and both insert → duplicate
BEGIN TRANSACTION;
    IF NOT EXISTS (
        SELECT 1 FROM dbo.Customer WITH (UPDLOCK, HOLDLOCK) WHERE Email = @Email
    )
        INSERT INTO dbo.Customer (Email, CreatedAt) VALUES (@Email, SYSDATETIME());
COMMIT;

-- TABLOCK on bulk INSERT to enable minimal logging (SIMPLE/BULK_LOGGED recovery)
INSERT INTO dbo.StagingLoad WITH (TABLOCK)
SELECT * FROM dbo.SourceData WHERE LoadDate = CAST(GETDATE() AS DATE);
```

### Deadlocks

Two sessions each hold a lock the other needs. The monitor (every 5 sec) kills the cheapest-to-rollback session — the deadlock victim, error 1205.

```sql
-- Prevention 1: consistent lock ordering
-- BAD:  Session A locks Orders → Accounts; Session B locks Accounts → Orders → deadlock
-- GOOD: both sessions always lock Orders → then Accounts
BEGIN TRANSACTION;
    UPDATE dbo.Orders  SET Status = 'Processing' WHERE OrderID = @oid;
    UPDATE dbo.Accounts SET Balance = Balance - @amt WHERE AccountID = @acct;
COMMIT;

-- Prevention 2: UPDLOCK on read-then-update (prevents S→X conversion deadlock)
-- BAD: two sessions both acquire S lock, both try to promote to X → deadlock
SELECT Qty FROM dbo.Inventory WHERE ProductID = @pid;
UPDATE dbo.Inventory SET Qty = Qty - 1 WHERE ProductID = @pid;

-- GOOD: UPDLOCK prevents the second session from acquiring S at all
BEGIN TRANSACTION;
    SELECT Qty FROM dbo.Inventory WITH (UPDLOCK, ROWLOCK) WHERE ProductID = @pid;
    UPDATE dbo.Inventory SET Qty = Qty - 1 WHERE ProductID = @pid;
COMMIT;

-- Read recent deadlock graphs (built-in system_health XE session)
SELECT TOP 10
    xdr.value('@timestamp', 'datetime2') AS deadlock_time,
    xdr.query('.')                        AS deadlock_graph_xml
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets t
    JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
    WHERE s.name = 'system_health' AND t.target_name = 'ring_buffer'
) d
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS XEventData(xdr)
ORDER BY deadlock_time DESC;
```

Always code for error 1205 — catch, wait briefly (e.g. 500ms), and retry.

### Diagnosing Blocking

```sql
-- Current blocking chains with wait time and SQL text
SELECT r.session_id    AS blocked_session,
       r.blocking_session_id,
       r.wait_type,
       r.wait_time / 1000.0  AS wait_sec,
       r.wait_resource,
       SUBSTRING(st.text, (r.statement_start_offset/2)+1, 200) AS blocked_stmt,
       s.open_transaction_count,
       s.login_name,
       s.program_name
FROM sys.dm_exec_requests r
JOIN sys.dm_exec_sessions s ON s.session_id = r.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0
ORDER BY r.wait_time DESC;

-- What is the blocking session doing (or last ran)?
DECLARE @blocking_session_id INT = 55; -- fill in from query above
SELECT s.session_id, s.status, s.open_transaction_count,
       s.login_name, s.program_name, s.host_name,
       SUBSTRING(st.text, 1, 200) AS current_or_last_stmt
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_requests r ON r.session_id = s.session_id
OUTER APPLY sys.dm_exec_sql_text(
    ISNULL(r.sql_handle, s.last_request_sql_handle)) st
WHERE s.session_id = @blocking_session_id;

-- Head-of-chain blocker: find the root session causing cascading blocks
SELECT DISTINCT blocking_session_id AS head_blocker
FROM sys.dm_exec_requests
WHERE blocking_session_id > 0
  AND blocking_session_id NOT IN (
      SELECT session_id FROM sys.dm_exec_requests WHERE blocking_session_id > 0
  );

-- Set a lock timeout so apps fail fast rather than waiting indefinitely
SET LOCK_TIMEOUT 5000;   -- raises error 1222 after 5 seconds
```

### NOLOCK Dangers

`WITH (NOLOCK)` bypasses read consistency and can return rows twice (page split moves a row), skip rows, produce torn reads, or read rolled-back data. Replace with RCSI.

```sql
-- BAD: NOLOCK for "performance" produces inconsistent results
SELECT COUNT(*) FROM dbo.Orders WITH (NOLOCK);  -- may double-count rows mid-split

-- GOOD: enable RCSI once at the database level
ALTER DATABASE YourDatabase SET READ_COMMITTED_SNAPSHOT ON;
-- Then just read normally — no S locks, consistent results
SELECT COUNT(*) FROM dbo.Orders;

-- Only legitimate NOLOCK use: rough row estimates or read-only dashboard counters
-- where approximate correctness is explicitly documented and acceptable
SELECT COUNT(*) AS approx_row_count FROM dbo.Orders WITH (NOLOCK); -- labelled as approximate
```

---

## Batch Operations

### Chunked DML

Any DML affecting more than ~10,000 rows should be chunked. A single large DML reserves log space for the whole operation, may escalate to a table lock, and makes rollback slow if cancelled. Use `DELETE/UPDATE TOP (N)` in a WHILE loop — each iteration is its own autocommit.

```sql
-- Chunked DELETE: purge 90-day-old audit rows
DECLARE @RowsAffected INT = 1;
WHILE @RowsAffected > 0
BEGIN
    DELETE TOP (5000) FROM dbo.AuditLog
    WHERE LoggedAt < DATEADD(day, -90, SYSDATETIME());
    SET @RowsAffected = @@ROWCOUNT;
    -- Optional: yield to other workloads between iterations
    -- WAITFOR DELAY '00:00:00.100';
END;

-- Chunked UPDATE: backfill a new column on a large table
DECLARE @Rows INT = 1;
WHILE @Rows > 0
BEGIN
    UPDATE TOP (5000) dbo.Orders
    SET RegionCode = dbo.fn_GetRegionCode(ShipPostalCode)
    WHERE RegionCode IS NULL;
    SET @Rows = @@ROWCOUNT;
END;

-- Paired INSERT + DELETE with per-chunk atomicity
-- (without an explicit transaction a mid-loop restart can orphan rows)
DECLARE @Archived INT = 1;
WHILE @Archived > 0
BEGIN
    BEGIN TRANSACTION;
        INSERT INTO dbo.OrdersArchive (OrderID, CustomerID, OrderDate, TotalAmt)
        SELECT TOP (5000) OrderID, CustomerID, OrderDate, TotalAmt
        FROM dbo.Orders WHERE OrderDate < DATEADD(year, -7, SYSDATETIME());

        SET @Archived = @@ROWCOUNT;

        DELETE dbo.Orders WHERE OrderID IN (
            SELECT TOP (5000) OrderID FROM dbo.Orders
            WHERE OrderDate < DATEADD(year, -7, SYSDATETIME())
        );
    COMMIT;
END;
```

**Chunk size guidance:**
- Start at 5,000 — below the per-statement escalation threshold (≥ 5,000 locks).
- If the table has many NCIs, reduce proportionally: a table with 5 NCIs can hit 5,000 locks at ~1,000 rows (one lock per row per index).
- Monitor `WRITELOG` wait time; reduce chunk size if log I/O is the bottleneck.

### Minimal Logging

Under `SIMPLE` or `BULK_LOGGED` recovery, certain bulk INSERT operations log only extent allocations, not individual row changes — dramatically reducing log volume.

```sql
-- Check current recovery model
SELECT name, recovery_model_desc FROM sys.databases WHERE name = DB_NAME();

-- Minimal logging: empty staging table + SIMPLE recovery + TABLOCK
INSERT INTO dbo.StagingFact WITH (TABLOCK)
SELECT FactID, ProductID, SaleDate, Amount
FROM dbo.SourceFact WHERE SaleDate >= '2025-01-01';

-- SELECT INTO also qualifies (creates + inserts in one operation)
SELECT FactID, ProductID, SaleDate, Amount
INTO dbo.StagingFact2025
FROM dbo.SourceFact WHERE SaleDate >= '2025-01-01';

-- Temporarily switch to BULK_LOGGED for a bulk load
-- (accept: no point-in-time restore during the bulk window)
ALTER DATABASE YourDatabase SET RECOVERY BULK_LOGGED;
-- ... bulk operation ...
ALTER DATABASE YourDatabase SET RECOVERY FULL;
BACKUP LOG YourDatabase TO DISK = '\\backupserver\YourDatabase_post_bulk.trn';
-- DO NOT skip the log backup — without it you lose point-in-time restore after the bulk load
```

Conditions required for minimal logging on INSERT...SELECT:

| Condition | Required? |
|---|---|
| Recovery = SIMPLE or BULK_LOGGED | Yes — never applies under FULL |
| `TABLOCK` hint on target | Yes (for non-empty heap/clustered pre-2016) |
| Target has no NCIs | Yes (unless empty clustered index, SQL Server 2016+) |

### Partition Switching

Partition switching is a metadata-only operation — it completes in milliseconds regardless of data volume.

```sql
-- Archive old data out of a production table
-- Step 1: create an archive table with identical structure + constraints
CREATE TABLE dbo.OrdersArchive_2023 (
    OrderID    INT           NOT NULL,
    CustomerID INT           NOT NULL,
    OrderDate  DATETIME2(0)  NOT NULL,
    TotalAmt   DECIMAL(12,2) NOT NULL,
    Status     TINYINT       NOT NULL,
    CONSTRAINT PK_OrdersArchive_2023 PRIMARY KEY CLUSTERED (OrderID)
) ON [PRIMARY];

-- Step 2: switch the oldest partition out — milliseconds, no rows copied
ALTER TABLE dbo.Orders SWITCH PARTITION 1 TO dbo.OrdersArchive_2023;

-- Step 3: archive table now holds all 2023 data; truncate or drop it
TRUNCATE TABLE dbo.OrdersArchive_2023;

-- Verify partition elimination is working (Actual Partition Count < total in plan)
SELECT * FROM dbo.Orders WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01';
-- Check execution plan: look for "Actual Partition Count" on the scan operator
```

### Transaction Log Management

```sql
-- Check log space usage and recovery model
SELECT name, recovery_model_desc,
       log_size_mb  = size * 8.0 / 1024,
       log_used_mb  = FILEPROPERTY(name, 'SpaceUsed') * 8.0 / 1024
FROM sys.databases WHERE name = DB_NAME();

-- VLF count: ideal < 100, > 1,000 is problematic (slow recovery + log write overhead)
DBCC LOGINFO;
-- SQL Server 2016 SP2+: sys.dm_db_log_info (filterable, no DBCC output parsing)
SELECT COUNT(*) AS vlf_count FROM sys.dm_db_log_info(DB_ID());

-- Pre-size the log to avoid autogrowth (each autogrowth pauses all log writes)
ALTER DATABASE YourDatabase
MODIFY FILE (NAME = 'YourDatabase_log', SIZE = 10240 MB, FILEGROWTH = 512 MB);

-- TRUNCATE vs DELETE: same effect, dramatically different log volume
-- DELETE FROM dbo.StagingLoad;       -- one log record per row — slow, fully logged
-- TRUNCATE TABLE dbo.StagingLoad;    -- extent deallocations only — fast, still rollbackable
```

### BULK INSERT

```sql
BULK INSERT dbo.StagingLoad
FROM 'D:\data\load_file.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR   = '\n',
    FIRSTROW        = 2,          -- skip header row
    BATCHSIZE       = 10000,      -- commit every 10,000 rows; restartable at last committed batch
    TABLOCK,                      -- table-level lock; enables minimal logging
    MAXERRORS       = 0           -- fail on any error
);

-- Bulk loads do not auto-update statistics — always run this after
UPDATE STATISTICS dbo.StagingLoad WITH FULLSCAN;

-- Validate row count after load
SELECT COUNT(*) AS loaded_rows FROM dbo.StagingLoad;
```

---

## SARGability

A predicate is **SARGable** when the optimizer can use an index seek instead of scanning every row. Never wrap the filtered column in a function, expression, or implicit conversion — apply transforms to the parameter side.

| Non-SARGable (forces scan) | SARGable equivalent |
|---|---|
| `WHERE YEAR(OrderDate) = 2025` | `WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01'` |
| `WHERE MONTH(OrderDate) = 6` | `WHERE OrderDate >= '2025-06-01' AND OrderDate < '2025-07-01'` |
| `WHERE ISNULL(Region, 'N/A') = @val` | Ensure column is NOT NULL; `WHERE Region = @val` |
| `WHERE col LIKE '%search'` | `WHERE col LIKE 'search%'` (leading wildcard forces scan) |
| `WHERE DATEDIFF(day, col, GETDATE()) < 30` | `WHERE col > DATEADD(day, -30, GETDATE())` |
| `WHERE CAST(col AS VARCHAR) = '42'` | Fix parameter type to match the column |
| `WHERE LEFT(LastName, 3) = 'Smi'` | `WHERE LastName LIKE 'Smi%'` |
| `WHERE col + 0 = @Val` | `WHERE col = @Val` (no arithmetic on the column side) |
| `WHERE LOWER(Email) = @email` | Store/compare in a consistent case; or use a computed column + index |
| `WHERE SUBSTRING(AccountNum, 1, 3) = 'ACC'` | `WHERE AccountNum LIKE 'ACC%'` |

```sql
-- Real-world example: reporting query scanning 50M rows because of YEAR() wrapping
-- BEFORE:
SELECT SUM(TotalAmt) AS revenue
FROM dbo.Orders
WHERE YEAR(OrderDate) = 2025 AND MONTH(OrderDate) = 6;
-- Execution plan: Clustered Index Scan, logical reads 620,814

-- AFTER: SARGable predicate + covering index
SELECT SUM(TotalAmt) AS revenue
FROM dbo.Orders
WHERE OrderDate >= '2025-06-01' AND OrderDate < '2025-07-01';
-- Execution plan: Index Seek on IX_Orders_OrderDate, logical reads 412

-- Detect implicit conversions in the plan cache (PlanAffectingConvert warning)
SELECT TOP 20 qs.total_logical_reads / qs.execution_count AS avg_reads,
       SUBSTRING(st.text, 1, 200) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle)    st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%PlanAffectingConvert%'
ORDER BY avg_reads DESC;

-- Example of an implicit conversion: column is INT, parameter is VARCHAR
-- BEFORE (forces scan + convert on every row):
DECLARE @cid VARCHAR(10) = '42';
SELECT * FROM dbo.Orders WHERE CustomerID = @cid;

-- AFTER (matches column type → seek):
DECLARE @cid INT = 42;
SELECT * FROM dbo.Orders WHERE CustomerID = @cid;
```

---

## Parameter Sniffing

SQL Server compiles a stored procedure plan on first execution using the actual parameter values, caches it, and reuses it for all future calls. Beneficial ~90% of the time; it hurts when the data distribution is skewed and the sniffed values are atypical.

```sql
-- Detect sniffing instability: a query with many distinct cached plans
SELECT qsq.query_id, COUNT(DISTINCT qsp.plan_id) AS plan_count,
       qsqt.query_sql_text
FROM sys.query_store_query      qsq
JOIN sys.query_store_plan       qsp  ON qsp.query_id       = qsq.query_id
JOIN sys.query_store_query_text qsqt ON qsqt.query_text_id = qsq.query_text_id
GROUP BY qsq.query_id, qsqt.query_sql_text
HAVING COUNT(DISTINCT qsp.plan_id) > 3
ORDER BY plan_count DESC;

-- Inspect the two extreme plans: min and max duration
SELECT qsp.plan_id,
       MIN(qsrs.min_duration) / 1000.0 AS best_ms,
       MAX(qsrs.max_duration) / 1000.0 AS worst_ms,
       AVG(qsrs.avg_duration) / 1000.0 AS avg_ms,
       SUM(qsrs.count_executions)       AS total_executions
FROM sys.query_store_plan          qsp
JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
WHERE qsp.query_id = @query_id
GROUP BY qsp.plan_id
ORDER BY avg_ms DESC;
```

**Ranked fixes:**

**1. OPTIMIZE FOR UNKNOWN** — stable plan using average distribution statistics. Preferred when many values are possible and none dominates:

```sql
CREATE PROCEDURE dbo.GetOrders @CustomerID INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT OrderID, OrderDate, TotalAmt
    FROM dbo.Orders
    WHERE CustomerID = @CustomerID
    OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));
END;
```

**2. OPTION (RECOMPILE)** — fresh plan per execution, embedding the actual value. CPU compile cost on every call. Use for infrequent but highly variable procs:

```sql
-- Statement-level RECOMPILE (preferred over WITH RECOMPILE on the whole proc)
SELECT OrderID, OrderDate, TotalAmt
FROM dbo.Orders
WHERE CustomerID = @CustomerID
OPTION (RECOMPILE);
```

**3. Local variable copy — avoid.** Defeats sniffing entirely, including the 90% of cases where sniffing helps:

```sql
-- BAD: prevents sniffing even when it would benefit the common case
DECLARE @CID INT = @CustomerID;
SELECT OrderID FROM dbo.Orders WHERE CustomerID = @CID;

-- Use OPTIMIZE FOR UNKNOWN instead
SELECT OrderID FROM dbo.Orders WHERE CustomerID = @CustomerID
OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));
```

**4. Multiple specialized procedures** for highly skewed distributions (rare):

```sql
-- When 80% of calls are for the 5 VIP customers, rest is long tail
-- One proc optimized for high-volume (small result set → Nested Loops)
CREATE PROCEDURE dbo.GetOrders_VIP @CustomerID INT AS
    SELECT OrderID, OrderDate FROM dbo.Orders WHERE CustomerID = @CustomerID
    OPTION (OPTIMIZE FOR (@CustomerID = 1));  -- sniff a representative high-volume value

-- One proc for long-tail (large result set → Hash Match)
CREATE PROCEDURE dbo.GetOrders_General @CustomerID INT AS
    SELECT OrderID, OrderDate FROM dbo.Orders WHERE CustomerID = @CustomerID
    OPTION (RECOMPILE);
```

---

## Temp Tables vs Table Variables

| Dimension | Temp Table (`#t`) | Table Variable (`@t`) | CTE |
|---|---|---|---|
| Statistics | Yes — auto-created | No (1 row pre-2019; deferred 2019+) | None |
| Index support | All types | PRIMARY KEY, UNIQUE only | No |
| Parallelism | Yes | Yes | Yes |
| Recompile trigger | On schema/stats change | Rarely | No |
| Survives ROLLBACK | No — dropped | Yes — contents persist | No |
| Scope | Session + called procs | Current batch only | Current statement |
| TempDB caching | Yes — inside procedures | Same | No |

**Decision rules:**

- **Temp table** when rows exceed a few hundred, you need an index on intermediate results, or you need accurate cardinality for downstream joins.
- **Table variable** when the set is small (1–100 rows), you want to avoid recompile triggers, or the data must survive a ROLLBACK (e.g. an audit log inserted before work begins).
- **CTE** when the result is referenced exactly once — CTEs are not materialized; a CTE referenced twice scans the source twice.

```sql
-- SCENARIO: multi-step order summary report

-- WRONG: CTE referenced twice → source scanned twice
WITH RecentOrders AS (
    SELECT CustomerID, SUM(TotalAmt) AS total, COUNT(*) AS cnt
    FROM dbo.Orders WHERE OrderDate >= '2025-01-01' GROUP BY CustomerID
)
SELECT r.CustomerID, r.total, r.cnt, c.CustomerName
FROM RecentOrders r
JOIN dbo.Customers c ON c.CustomerID = r.CustomerID
WHERE r.total > 10000
UNION ALL
SELECT r.CustomerID, r.total, r.cnt, 'VIP' AS CustomerName
FROM RecentOrders r  -- second reference: full re-scan of Orders
WHERE r.cnt > 50;

-- CORRECT: temp table materializes the result once
SELECT CustomerID, SUM(TotalAmt) AS total, COUNT(*) AS cnt
INTO #RecentOrders
FROM dbo.Orders WHERE OrderDate >= '2025-01-01' GROUP BY CustomerID;

CREATE INDEX IX_RecentOrders_CustomerID ON #RecentOrders (CustomerID);  -- supports the joins

SELECT r.CustomerID, r.total, r.cnt, c.CustomerName
FROM #RecentOrders r JOIN dbo.Customers c ON c.CustomerID = r.CustomerID WHERE r.total > 10000
UNION ALL
SELECT r.CustomerID, r.total, r.cnt, 'VIP' FROM #RecentOrders r WHERE r.cnt > 50;

DROP TABLE #RecentOrders;

-- Table variable: correct use — audit log that must survive ROLLBACK
BEGIN TRANSACTION;
    DECLARE @AuditLog TABLE (EventType VARCHAR(50), EventTime DATETIME2, UserID INT);
    INSERT INTO @AuditLog VALUES ('OrderCreated', SYSDATETIME(), @UserID);

    -- Do the real work; if this fails and rolls back, @AuditLog still has the row
    INSERT INTO dbo.Orders (CustomerID, OrderDate, TotalAmt) VALUES (@cid, SYSDATETIME(), @amt);
ROLLBACK;
-- @AuditLog row survives here — write it to the audit table outside the transaction

-- Check deferred compilation status (SQL Server 2019+, compat 150+)
SELECT name, value FROM sys.database_scoped_configurations WHERE name = 'DEFERRED_COMPILATION_TV';
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Non-SARGable WHERE clause | Never wrap the filtered column in a function — apply functions to parameters |
| Key Lookup on high-row-count query | Add needed SELECT columns to the NCI INCLUDE list |
| Using NOLOCK for "read performance" | Enable RCSI instead — consistent reads, no shared lock overhead |
| Fixing sniffing with a local variable copy | Use `OPTIMIZE FOR UNKNOWN` or statement-level `OPTION (RECOMPILE)` |
| Ignoring fill factor on random-key tables | Set 70–80% fill factor; rebuild on schedule |
| Blindly creating missing index suggestions | Evaluate write penalty and overlap with existing indexes |
| Rebuilding all indexes regardless of fragmentation | Skip tables < 1,000 pages; skip indexes < 5% fragmented |
| Auto-update statistics never fires on large tables | Upgrade to compat 130+ for the dynamic threshold (fires ~20× sooner) |
| Table variable cardinality fixed at 1 row | Upgrade to compat 150 for deferred compilation |
| Hash/sort spills to tempdb | Fix statistics so the memory grant sizes correctly; row-mode MGF (2019+) auto-adjusts |
| NOLOCK as a deadlock fix | Deadlocks need lock ordering or RCSI — NOLOCK does not prevent them |
| Single large DELETE bloating the log | Chunk with `DELETE TOP (5000)` in a WHILE loop |
| YEAR(col) = 2025 in WHERE clause | Rewrite as `col >= '2025-01-01' AND col < '2026-01-01'` |
| Implicit type conversion on a filter column | Match parameter type exactly to column type |
| CTE referenced more than once | Materialize into a temp table so the source is scanned once |
| sp_updatestats to fix ascending key problem | Use `UPDATE STATISTICS ... WITH FULLSCAN` — sp_updatestats uses default sampling |
| tempdb trace flags 1117/1118 on SQL 2016+ | Remove them — this is the default behavior now; they are no-ops that cause confusion |

---

## Sources

Official Microsoft Learn documentation and canonical community references.

- [Execution Plan Overview](https://learn.microsoft.com/en-us/sql/relational-databases/performance/execution-plans)
- [sys.dm_exec_query_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-query-stats-transact-sql)
- [sys.dm_exec_cached_plans](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-cached-plans-transact-sql)
- [Monitor Performance by Using the Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [sp_query_store_force_plan](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-query-store-force-plan-transact-sql)
- [Cardinality Estimation (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/performance/cardinality-estimation-sql-server)
- [Intelligent Query Processing](https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing)
- [sys.query_store_wait_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-query-store-wait-stats-transact-sql)
- [Clustered and Nonclustered Indexes Described](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described)
- [Create Indexes with Included Columns](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-indexes-with-included-columns)
- [Create Filtered Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-filtered-indexes)
- [Reorganize and Rebuild Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes)
- [sys.dm_db_index_physical_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql)
- [sys.dm_db_missing_index_details](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-missing-index-details-transact-sql)
- [Specify Fill Factor for an Index](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/specify-fill-factor-for-an-index)
- [Statistics - SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/statistics/statistics)
- [DBCC SHOW_STATISTICS](https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-show-statistics-transact-sql)
- [UPDATE STATISTICS](https://learn.microsoft.com/en-us/sql/t-sql/statements/update-statistics-transact-sql)
- [sys.dm_db_stats_properties](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-stats-properties-transact-sql)
- [sys.dm_os_wait_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql)
- [Wait Statistics: Tell Me Where It Hurts (Paul Randal, SQLskills)](https://www.sqlskills.com/blogs/paul/wait-statistics-or-please-tell-me-where-it-hurts/)
- [sys.dm_exec_requests](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-requests-transact-sql)
- [Server Memory Configuration Options](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/server-memory-server-configuration-options)
- [Transaction Locking and Row Versioning Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide)
- [Use the system_health Session](https://learn.microsoft.com/en-us/sql/relational-databases/extended-events/use-the-system-health-session)
- [sys.dm_tran_version_store_space_usage](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-tran-version-store-space-usage)
- [sys.dm_tran_active_snapshot_database_transactions](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-tran-active-snapshot-database-transactions-transact-sql)
- [Partitioned Tables and Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes)
- [Prerequisites for Minimal Logging in Bulk Import](https://learn.microsoft.com/en-us/sql/relational-databases/import-export/prerequisites-for-minimal-logging-in-bulk-import)
- [BULK INSERT](https://learn.microsoft.com/en-us/sql/t-sql/statements/bulk-insert-transact-sql)
- [Transaction Log Architecture and Management Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide)
- [sys.dm_db_log_info](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-log-info-transact-sql)
- [SQL Server Index and Statistics Maintenance (Ola Hallengren)](https://ola.hallengren.com/sql-server-index-and-statistics-maintenance.html)
