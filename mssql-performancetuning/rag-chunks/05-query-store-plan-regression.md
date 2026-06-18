---
topic: Query Store for plan regression detection and plan forcing
keywords: [query store, plan regression, used to be fast, force plan, plan_id, sp_query_store_force_plan]
use_when: A query that used to be fast regressed, or you want to pin a known-good plan.
---

# Query Store: Plan Regression

Query Store persists plan and performance history across restarts — the primary tool for "this query used to be fast." Not enabled by default on 2016–2019 on-prem.

```sql
ALTER DATABASE YourDatabase SET QUERY_STORE = ON;
ALTER DATABASE YourDatabase SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE, DATA_FLUSH_INTERVAL_SECONDS = 900,
    QUERY_CAPTURE_MODE = AUTO, MAX_STORAGE_SIZE_MB = 1024,
    SIZE_BASED_CLEANUP_MODE = AUTO, WAIT_STATS_CAPTURE_MODE = ON
);
-- actual_state_desc = ReadOnly means the store is full
SELECT actual_state_desc, current_storage_size_mb, max_storage_size_mb
FROM sys.database_query_store_options;
```

Find regressions (current plan slower than historical best):

```sql
WITH BestPlan AS (
    SELECT qsq.query_id, qsp.plan_id, MIN(qsrs.avg_duration) AS best_us,
           ROW_NUMBER() OVER (PARTITION BY qsq.query_id ORDER BY MIN(qsrs.avg_duration)) AS rn
    FROM sys.query_store_query qsq
    JOIN sys.query_store_plan qsp ON qsp.query_id = qsq.query_id
    JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
    GROUP BY qsq.query_id, qsp.plan_id
),
RecentPlan AS (
    SELECT qsq.query_id, qsp.plan_id, AVG(qsrs.avg_duration) AS recent_us
    FROM sys.query_store_query qsq
    JOIN sys.query_store_plan qsp ON qsp.query_id = qsq.query_id
    JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
    JOIN sys.query_store_runtime_stats_interval qsrsi
         ON qsrsi.runtime_stats_interval_id = qsrs.runtime_stats_interval_id
    WHERE qsrsi.start_time >= DATEADD(HOUR, -4, GETUTCDATE())
    GROUP BY qsq.query_id, qsp.plan_id
)
SELECT r.query_id, b.plan_id AS best_plan_id, r.plan_id AS current_plan_id,
       r.recent_us * 1.0 / NULLIF(b.best_us, 0) AS regression_ratio
FROM RecentPlan r JOIN BestPlan b ON b.query_id = r.query_id AND b.rn = 1
WHERE r.recent_us > b.best_us * 1.5
ORDER BY regression_ratio DESC;
```

Force a known-good plan (it fails if the underlying schema changes):

```sql
EXEC sys.sp_query_store_force_plan @query_id = 42, @plan_id = 7;
SELECT qsp.plan_id, qsp.force_failure_count, qsp.last_force_failure_reason_desc
FROM sys.query_store_plan qsp WHERE qsp.is_forced_plan = 1;
```
