---
topic: Feature table build workflow — 7-step pattern, #Base table, snapshot date
keywords: [feature table, ML features, #Base, @SnapshotDate, feature engineering, workflow, training data, entity table]
use_when: Building a feature table for ML training from SQL Server data.
---

# Feature Table Build Workflow

The standard 7-step pattern for building an ML feature table:

```sql
DECLARE @SnapshotDate DATE = '2024-06-01';  -- fixed reference date (NEVER GETDATE())

-- Step 1: Entity base (all entities in scope)
SELECT DISTINCT CustomerID INTO #Base
FROM Sales.Orders
WHERE OrderDate < @SnapshotDate;

-- Step 2: Add features in groups (one join per feature block)
-- Monetary features
ALTER TABLE #Base ADD
    TotalRevenue     DECIMAL(18,2),
    AvgOrderValue    DECIMAL(18,2),
    OrderCount       INT;

UPDATE b SET
    TotalRevenue  = f.TotalRevenue,
    AvgOrderValue = f.AvgOrderValue,
    OrderCount    = f.OrderCount
FROM #Base b
JOIN (
    SELECT CustomerID,
           SUM(TotalAmount) AS TotalRevenue,
           AVG(TotalAmount) AS AvgOrderValue,
           COUNT(*)         AS OrderCount
    FROM Sales.Orders
    WHERE OrderDate < @SnapshotDate
    GROUP BY CustomerID
) f ON f.CustomerID = b.CustomerID;

-- Step 3: Validate completeness
SELECT COUNT(*) AS TotalRows,
       SUM(CASE WHEN TotalRevenue IS NULL THEN 1 ELSE 0 END) AS NullRevenue
FROM #Base;

-- Step 4-7: Add remaining feature blocks (temporal, categorical, lag, etc.)
-- then impute NULLs, assign train/test split, export

-- Final select for export
SELECT b.CustomerID,
       b.TotalRevenue, b.AvgOrderValue, b.OrderCount,
       -- ... other features ...
       b.Split
FROM #Base b;
```

**Key rules:**
- `@SnapshotDate` must be a fixed date — never `GETDATE()`. Using `GETDATE()` makes the feature table non-reproducible and causes temporal leakage.
- `#Base` anchors the entity set. All features join back to `#Base`, never to each other.
- Validate after each feature block: check row count hasn't changed and NULLs are expected.
- Add features in groups: monetary → temporal → categorical → lag → rolling windows → imputation → split assignment.
