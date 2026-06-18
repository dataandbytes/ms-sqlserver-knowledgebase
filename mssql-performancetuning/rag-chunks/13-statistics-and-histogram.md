---
topic: How statistics drive plans, histogram structure, and DBCC SHOW_STATISTICS
keywords: [statistics, histogram, RANGE_HI_KEY, EQ_ROWS, AVG_RANGE_ROWS, DBCC SHOW_STATISTICS, density, 200 steps]
use_when: You are reading a statistics histogram or want to know how the optimizer uses statistics.
---

# Statistics & Histogram

Statistics describe value distribution and drive cardinality estimates, which in turn drive join algorithm selection, memory grant sizing, index selection, and parallelism. Wrong estimates → wrong plans. A query fast in dev (small data, fresh stats) can be catastrophically slow in production.

Created automatically on index creation, automatically on query predicates (`AUTO_CREATE_STATISTICS` ON by default), or manually via `CREATE/UPDATE STATISTICS`. On OLTP prefer synchronous auto-update (`AUTO_UPDATE_STATS_ASYNC = OFF`) so each plan compiles with fresh stats.

**Histogram** covers only the leading column and has up to **200 steps**:

| Column | Meaning |
|---|---|
| `RANGE_HI_KEY` | Upper bound value of this step |
| `EQ_ROWS` | Estimated rows equal to `RANGE_HI_KEY` |
| `RANGE_ROWS` | Estimated rows between previous and current `RANGE_HI_KEY` |
| `DISTINCT_RANGE_ROWS` | Distinct values in the range (excluding `RANGE_HI_KEY`) |
| `AVG_RANGE_ROWS` | `RANGE_ROWS / DISTINCT_RANGE_ROWS` |

`col = @val` matching a `RANGE_HI_KEY` → `EQ_ROWS`; falling between steps → `AVG_RANGE_ROWS`; beyond the max `RANGE_HI_KEY` (ascending key) → density fraction, often 1 row. At 200 steps over millions of distinct values, each step spans a wide range and `AVG_RANGE_ROWS` is wrong for skew within it.

```sql
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate');                  -- full
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate') WITH HISTOGRAM;   -- histogram only
DBCC SHOW_STATISTICS ('dbo.Orders', 'IX_Orders_OrderDate') WITH STAT_HEADER; -- header only
```

Header fields to check: `Updated` (age), `Rows` (at last update vs current), `Rows Sampled` (low = limited accuracy on skew), `Steps` (200 = fully packed), `Filter Expression`.
