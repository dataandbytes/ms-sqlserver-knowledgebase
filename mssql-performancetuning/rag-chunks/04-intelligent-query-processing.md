---
topic: Intelligent Query Processing (IQP) features and compatibility levels
keywords: [IQP, adaptive joins, memory grant feedback, MGHF, batch mode, DOP feedback, CE feedback, PSPO, deferred compilation]
use_when: You are using or disabling IQP features, or wondering why a plan self-corrects between runs.
---

# Intelligent Query Processing (IQP)

Optimizer features (SQL Server 2017–2022) that learn from execution and self-correct. Most need a specific compatibility level and Query Store.

| Feature | Min compat | What it does |
|---|---|---|
| Adaptive Joins | 140 | Chooses Nested Loops vs Hash Match at runtime |
| Batch Mode on Rowstore | 150 | Columnstore-style batch processing on rowstore |
| Table Variable Deferred Compilation | 150 | Defers compilation until populated (fixes 1-row estimate) |
| Row-Mode Memory Grant Feedback (MGHF) | 150 (2019) | Adjusts sort/hash grants on later executions |
| DOP Feedback | 160 | Lowers MAXDOP for queries that don't benefit |
| CE Feedback | 160 | Corrects CE assumptions on repeated executions |
| Parameter-Sensitive Plan Optimization (PSPO) | 160 | Multiple plan variants for skewed parameter distributions |

If spills persist despite MGHF, the statistics are stale — fix the statistics rather than relying on feedback. An Adaptive Join operator chooses its algorithm at runtime; if it appears and the query is slow, check whether the row-count threshold is crossed inconsistently (unstable statistics).

```sql
SELECT name, compatibility_level FROM sys.databases WHERE name = DB_NAME();
SELECT name, value FROM sys.database_scoped_configurations WHERE name = 'DEFERRED_COMPILATION_TV';

-- Disable a feature that causes problems
SELECT ... OPTION (USE HINT ('DISABLE_BATCH_MODE_ADAPTIVE_JOINS'));
SELECT ... OPTION (USE HINT ('DISABLE_QUERY_PLAN_FEEDBACK'));
ALTER DATABASE SCOPED CONFIGURATION SET DEFERRED_COMPILATION_TV = OFF;
```
