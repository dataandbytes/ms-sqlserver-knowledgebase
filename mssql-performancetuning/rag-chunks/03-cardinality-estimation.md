---
topic: Cardinality estimation, CE versions, and estimated vs actual rows
keywords: [cardinality estimation, CE, estimated rows, actual rows, compatibility level, legacy CE, regression after upgrade]
use_when: Estimated rows differ from actual rows, or a query regressed after a compatibility-level upgrade.
---

# Cardinality Estimation (CE)

CE predicts how many rows each operator returns. Wrong estimates cause wrong join algorithms, wrong memory grants, and wrong serial/parallel decisions.

**Symptom:** hover an operator in SSMS — Estimated vs Actual Number of Rows. A 10× difference warrants investigation; a 100× difference is usually the root cause.

**CE version follows compatibility level.** A regression right after a compat upgrade is often CE-related.

| SQL Server | Compat | CE version |
|---|---|---|
| 2012 and earlier | ≤ 110 | CE70 (legacy) |
| 2014 | 120 | CE120 |
| 2016 | 130 | CE130 |
| 2017 | 140 | CE140 |
| 2019 | 150 | CE150 |
| 2022 | 160 | CE160 |

```sql
-- Force legacy CE to test a regression
SELECT ... OPTION (USE HINT ('FORCE_LEGACY_CARDINALITY_ESTIMATION'));          -- one query
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = ON;    -- whole database
```

| Cause | Symptom | Fix |
|---|---|---|
| Stale statistics | Estimated ≪ actual | `UPDATE STATISTICS ... WITH FULLSCAN` |
| Ascending key | New dates past histogram max → 1-row estimate | FULLSCAN; incremental stats |
| Multi-predicate independence | Product of selectivities underestimates | Multi-column statistics |
| Parameter sniffing | Plan built for atypical first-run value | `OPTIMIZE FOR UNKNOWN` / `OPTION (RECOMPILE)` |
| Table variable (pre-2019) | Always estimates 1 row | Compat 150 deferred compilation; or temp table |
| CE version change after upgrade | Complex-query regression | Test `FORCE_LEGACY_CARDINALITY_ESTIMATION`, then fix the query |
