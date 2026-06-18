---
topic: Query Store : enable, find regressed queries, force plans, automatic plan correction
keywords: [Query Store, plan regression, force plan, sp_query_store_force_plan, automatic tuning, FORCE_LAST_GOOD_PLAN, query_store_query, query_store_plan]
use_when: Enabling Query Store, finding queries whose plans regressed, forcing a known-good plan, or enabling automatic plan correction.
---

# Query Store

```sql
-- Enable with recommended settings
ALTER DATABASE YourDB SET QUERY_STORE = ON;
ALTER DATABASE YourDB SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = AUTO,
    MAX_STORAGE_SIZE_MB = 1024,
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    WAIT_STATS_CAPTURE_MODE = ON
);

-- Find regressed queries (recent avg > 1.5x historical best)
WITH BestPlan AS (
    SELECT qsq.query_id, qsp.plan_id,
           MIN(qsrs.avg_duration) AS best_us,
           ROW_NUMBER() OVER (PARTITION BY qsq.query_id ORDER BY MIN(qsrs.avg_duration)) AS rn
    FROM sys.query_store_query qsq
    JOIN sys.query_store_plan  qsp  ON qsp.query_id = qsq.query_id
    JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
    GROUP BY qsq.query_id, qsp.plan_id
),
Recent AS (
    SELECT qsp.query_id, qsp.plan_id, AVG(qsrs.avg_duration) AS recent_us
    FROM sys.query_store_plan qsp
    JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
    JOIN sys.query_store_runtime_stats_interval qsrsi
         ON qsrsi.runtime_stats_interval_id = qsrs.runtime_stats_interval_id
    WHERE qsrsi.start_time >= DATEADD(HOUR, -4, GETUTCDATE())
    GROUP BY qsp.query_id, qsp.plan_id
)
SELECT r.query_id, b.plan_id AS best_plan, r.plan_id AS current_plan,
       b.best_us/1000.0 AS best_ms, r.recent_us/1000.0 AS current_ms,
       r.recent_us / NULLIF(b.best_us, 0.0) AS regression_ratio
FROM Recent r JOIN BestPlan b ON b.query_id = r.query_id AND b.rn = 1
WHERE r.recent_us > b.best_us * 1.5
ORDER BY regression_ratio DESC;

-- Force a known-good plan
EXEC sys.sp_query_store_force_plan @query_id = 42, @plan_id = 7;

-- Automatic plan correction (SQL Server 2017+)
ALTER DATABASE YourDB SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON);
```

**Key rules:**
- Query Store stores both plan and runtime stats : unlike DMVs, it survives server restarts.
- `QUERY_CAPTURE_MODE = AUTO` ignores infrequent and trivial queries : reduces storage noise.
- Forced plans are not permanent : if the plan becomes invalid (schema change, stats update), the engine falls back and logs a warning.
- `FORCE_LAST_GOOD_PLAN` automatically forces a plan when it detects regression. Review forced plans periodically : they can mask root cause issues.
