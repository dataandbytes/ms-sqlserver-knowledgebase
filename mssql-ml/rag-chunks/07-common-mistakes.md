---
topic: Common ML feature engineering mistakes and their fixes
keywords: [common mistakes, data leakage, GETDATE, LEAD, z-score, NTILE, NULLIF, RFM segment, train test split, NEWID, forward fill]
use_when: Quick reference for known ML feature engineering pitfalls when reviewing feature table code.
---

# Common ML Feature Engineering Mistakes

| Mistake | Fix |
|---|---|
| `GETDATE()` instead of `@SnapshotDate` | Fix `@SnapshotDate` at query time; rebuild for each snapshot date |
| `LEAD()` in lag features | Use `LAG()` — LEAD reads future rows and causes temporal leakage |
| Z-score computed over entire dataset | Compute mean/stdev on train rows only; apply via CROSS JOIN |
| Target-correlated column included as feature | Audit every column: does it encode what happened after the snapshot? Remove if yes |
| `NEWID()` for train/test split | Use HASHBYTES — NEWID changes every run, splitting is not reproducible |
| `NTILE` without `ORDER BY` producing random buckets | Always specify `ORDER BY` inside `NTILE(N) OVER (ORDER BY col)` |
| `NULLIF` omitted on division | Add `NULLIF(denominator, 0)` — divide-by-zero raises a runtime error |
| RFM_Segment string used as model input | Use R_Score, F_Score, M_Score as separate numeric columns |
| Forward fill applied to non-time-series features | Forward fill is only valid when rows are ordered by time for the same entity |
| Frequency window uses `>` instead of `>=` on date | Verify window boundary: `OrderDate >= DATEADD(day, -30, @SnapshotDate)` includes day -30 |
| Aggregates over all orders including post-snapshot | Add `WHERE OrderDate < @SnapshotDate` to all aggregation CTEs |
| NULL imputation computed on full dataset | Always filter `WHERE Split = 'train'` when computing imputation statistics |
| One-hot encoding mapping computed globally | Map categories from train set only; unseen test categories map to 0 vector |
