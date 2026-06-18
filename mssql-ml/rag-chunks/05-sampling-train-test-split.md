---
topic: Sampling and train/test splitting — HASHBYTES deterministic split, stratified sampling, time-based split
keywords: [train test split, HASHBYTES, deterministic sampling, stratified sampling, time-based split, k-fold, downsampling, oversampling, class imbalance]
use_when: Splitting a feature table into train/validation/test sets, performing stratified sampling, or splitting time-series data correctly.
---

# Sampling and Train/Test Splitting

```sql
-- HASHBYTES deterministic split (70% train, 15% val, 15% test)
-- HASHBYTES produces a consistent hash per CustomerID — same row always lands in same split
ALTER TABLE #Base ADD Split VARCHAR(10);

UPDATE #Base SET Split =
    CASE
        WHEN ABS(CAST(CAST(HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
                     AS VARBINARY(8)) AS BIGINT)) % 10 < 7 THEN 'train'
        WHEN ABS(CAST(CAST(HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
                     AS VARBINARY(8)) AS BIGINT)) % 10 < 8 THEN 'val'
        ELSE 'test'
    END;
-- Note: (% 10 + 10) % 10 avoids ABS overflow on BIGINT MIN_VALUE if needed

-- Verify distribution
SELECT Split, COUNT(*) AS N, CAST(100.0*COUNT(*)/SUM(COUNT(*)) OVER () AS DECIMAL(5,1)) AS Pct
FROM #Base GROUP BY Split;

-- Time-based split (correct for time-series — never shuffle past/future)
DECLARE @TrainEnd   DATE = '2024-03-31';
DECLARE @GapEnd     DATE = '2024-04-30';  -- gap buffer prevents leakage
DECLARE @TestStart  DATE = '2024-05-01';

UPDATE #Base SET Split =
    CASE
        WHEN SnapshotDate <= @TrainEnd  THEN 'train'
        WHEN SnapshotDate >= @TestStart THEN 'test'
        ELSE 'gap'                            -- exclude gap rows from training
    END;

-- Stratified sampling: preserve class distribution
WITH Stratified AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY Label ORDER BY NEWID()) AS rn,
              COUNT(*) OVER (PARTITION BY Label) AS ClassCount
    FROM #Base WHERE Split = 'train'
)
SELECT * FROM Stratified WHERE rn <= @SampleSize / COUNT(DISTINCT Label);

-- Downsample majority class (binary classification imbalance)
WITH Majority AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))) AS rn
    FROM #Base WHERE Label = 0 AND Split = 'train'
)
SELECT * FROM #Base WHERE Label = 1 AND Split = 'train'
UNION ALL
SELECT * FROM Majority WHERE rn <= (SELECT COUNT(*) FROM #Base WHERE Label = 1 AND Split = 'train') * 3;
```

**Key rules:**
- HASHBYTES-based split: `(% 10 + 10) % 10` handles the rare `BIGINT MIN_VALUE` edge case where `ABS()` overflows. Safe to use either form.
- Time-based split: never shuffle temporal data. Always split by time with a gap buffer to prevent feature leakage across the boundary.
- Never use `NEWID()` for reproducible splits — it changes every query. Use HASHBYTES for determinism.
