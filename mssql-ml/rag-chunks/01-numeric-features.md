---
topic: Numeric feature transforms — log, z-score, min-max, NTILE bucketing
keywords: [log transform, z-score, min-max scaling, NTILE, normalization, standardization, numeric features, safe division, NULLIF]
use_when: Applying log transform, z-score standardization, min-max normalization, or bucketing a numeric column into deciles/quintiles.
---

# Numeric Features

```sql
DECLARE @SnapshotDate DATE = '2024-06-01';

SELECT CustomerID,
    -- Safe division (NULLIF prevents divide-by-zero)
    TotalRevenue / NULLIF(OrderCount, 0) AS AvgOrderValue,

    -- Log(x+1) transform: compresses right-skewed distributions
    LOG(TotalRevenue + 1) AS LogRevenue,

    -- Z-score: (x - mean) / stdev — computed over training rows only (see leakage note)
    (TotalRevenue - AVG(TotalRevenue) OVER ()) / NULLIF(STDEV(TotalRevenue) OVER (), 0) AS ZScore_Revenue,

    -- Min-max scaling: (x - min) / (max - min) → [0, 1]
    (TotalRevenue - MIN(TotalRevenue) OVER ()) /
        NULLIF(MAX(TotalRevenue) OVER () - MIN(TotalRevenue) OVER (), 0) AS MinMax_Revenue,

    -- NTILE: bucket into N equal-sized groups (deciles = 10, quintiles = 5)
    NTILE(10) OVER (ORDER BY TotalRevenue) AS RevDecile,
    NTILE(5)  OVER (ORDER BY TotalRevenue) AS RevQuintile

FROM #Base;
```

**Key rules:**
- `NULLIF(denominator, 0)` returns NULL instead of raising a divide-by-zero error. Handle the resulting NULLs in the imputation step.
- `LOG()` in T-SQL is natural log (base e). For log base 10 use `LOG10()`. `LOG(0)` is an error — always add `+ 1`.
- Z-score and min-max computed with `OVER ()` use all rows in `#Base`. If `#Base` contains both train and test rows, this causes train-test leakage. Compute statistics on train rows only, then apply to all rows.
- NTILE bucketing is affected by the ORDER BY — `NTILE(10) OVER (ORDER BY Revenue)` puts the lowest earners in bucket 1 and highest in bucket 10.
