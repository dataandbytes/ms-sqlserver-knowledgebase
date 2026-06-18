---
topic: NULL imputation — mean, median, mode, forward fill, NULL indicator column
keywords: [NULL imputation, mean imputation, median, PERCENTILE_CONT, mode, forward fill, LAST_VALUE, IGNORE NULLS, NULL indicator, missing data]
use_when: Handling NULL values in feature columns before ML model training.
---

# NULL Imputation

```sql
-- Mean imputation (two-step: compute on train rows, apply to all)
SELECT AVG(CAST(Age AS FLOAT)) AS MeanAge
INTO #MeanStats
FROM #Base WHERE Split = 'train' AND Age IS NOT NULL;

UPDATE b SET Age = (SELECT MeanAge FROM #MeanStats)
FROM #Base b WHERE b.Age IS NULL;

-- Median imputation (PERCENTILE_CONT returns interpolated median)
UPDATE b SET Age = (
    SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY Age)
    FROM #Base WHERE Split = 'train' AND Age IS NOT NULL
)
FROM #Base b WHERE b.Age IS NULL;

-- Mode imputation (most frequent value)
UPDATE b SET Category = (
    SELECT TOP 1 Category FROM #Base
    WHERE Split = 'train' AND Category IS NOT NULL
    GROUP BY Category ORDER BY COUNT(*) DESC
)
FROM #Base b WHERE b.Category IS NULL;

-- Forward fill (carry last non-NULL value forward in time series)
-- SQL Server 2022+: LAST_VALUE with IGNORE NULLS
SELECT EntityID, EventDate,
       LAST_VALUE(Score IGNORE NULLS) OVER (
           PARTITION BY EntityID ORDER BY EventDate
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS Score_Filled
FROM #Base;

-- Pre-2022 forward fill (correlated subquery)
SELECT b.EntityID, b.EventDate,
       (SELECT TOP 1 Score FROM #Base b2
        WHERE b2.EntityID = b.EntityID AND b2.EventDate <= b.EventDate
          AND b2.Score IS NOT NULL
        ORDER BY b2.EventDate DESC) AS Score_Filled
FROM #Base b;

-- NULL indicator column (keep original + add flag)
ALTER TABLE #Base ADD Age_WasNull BIT NOT NULL DEFAULT 0;
UPDATE #Base SET Age_WasNull = 1 WHERE Age IS NULL;
-- Then apply mean imputation to Age column
```

**Key rules:**
- Always compute imputation statistics (mean, median, mode) on train rows only — computing on the full dataset leaks test distribution into training.
- NULL indicator columns (`Age_WasNull`) tell the model that missingness itself may be informative. Use when NULLs are not random.
- Forward fill is appropriate for time-series sensor data where a NULL means "no change." It is inappropriate for one-row-per-entity feature tables.
