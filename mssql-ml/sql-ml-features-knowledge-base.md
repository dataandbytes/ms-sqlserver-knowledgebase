# SQL for Machine Learning — Knowledge Base

> T-SQL patterns for building ML-ready feature tables from SQL Server: numeric transforms,
> categorical encoding, temporal features, rolling windows, RFM scoring, NULL imputation,
> sampling, train/test splitting, and data leakage prevention.

## Scope

**Use this for:** building a feature table or training dataset from relational data; computing
rolling aggregations, lag features, and RFM scores in T-SQL; encoding categorical variables;
imputing NULLs; splitting data into train/test/validation partitions; exporting to pandas or
Parquet; preventing temporal and target leakage in feature queries.

**Not for:** BI dashboards, summary reports, schema design, query performance tuning, or running
R/Python inside SQL Server (SQL Server ML Services).

## Contents

1. [Feature Table Build Workflow](#feature-table-build-workflow)
2. [Numeric Features](#numeric-features)
3. [Categorical Encoding](#categorical-encoding)
4. [Temporal Features](#temporal-features)
5. [RFM Pattern](#rfm-pattern)
6. [Rolling Window Features](#rolling-window-features)
7. [Lag Features](#lag-features)
8. [NULL Imputation](#null-imputation)
9. [Sampling and Train/Test Splitting](#sampling-and-traintest-splitting)
10. [Data Leakage Prevention](#data-leakage-prevention)
11. [Export to Pandas](#export-to-pandas)
12. [Quick Reference: SQL Pattern to ML Concept](#quick-reference-sql-pattern-to-ml-concept)
13. [Common Mistakes](#common-mistakes)

---

## Feature Table Build Workflow

Follow these seven steps in order. Each step has a validation checkpoint.

```sql
-- Step 1: set a fixed snapshot date — never use GETDATE() in feature logic
DECLARE @SnapshotDate DATE = '2024-06-01';

-- Step 2: build the base entity table — one row per entity, PK only
SELECT DISTINCT CustomerID
INTO #Base
FROM Sales.Customers
WHERE SignupDate < @SnapshotDate;
-- Validate: SELECT COUNT(*), COUNT(DISTINCT CustomerID) FROM #Base  → counts must match (no dupes)

-- Step 3: LEFT JOIN feature sets to #Base with temporal bounds
-- Every join is bounded by AND EventDate < @SnapshotDate
SELECT
    b.CustomerID,
    ISNULL(f.OrderCount, 0)                        AS OrderCount,
    ISNULL(f.TotalRevenue, 0)                      AS TotalRevenue,
    ISNULL(DATEDIFF(day, f.LastOrder, @SnapshotDate), 999) AS DaysSinceLastOrder
INTO #Features
FROM #Base b
LEFT JOIN (
    SELECT CustomerID, COUNT(*) AS OrderCount,
           SUM(TotalAmount) AS TotalRevenue, MAX(OrderDate) AS LastOrder
    FROM Sales.Orders WHERE OrderDate < @SnapshotDate
    GROUP BY CustomerID
) f ON f.CustomerID = b.CustomerID;
-- Validate: SELECT COUNT(*) FROM #Features should equal COUNT(*) FROM #Base

-- Step 4: impute NULLs — no model input should be NULL
-- Already done with ISNULL above; verify:
-- SELECT SUM(CASE WHEN col IS NULL THEN 1 ELSE 0 END) FROM #Features  → 0 for every column

-- Step 5: encode categoricals — all model inputs must be numeric
-- (see Categorical Encoding section)

-- Step 6: split train/test
-- (see Sampling and Train/Test Splitting section)

-- Step 7: export
-- (see Export to Pandas section)
```

---

## Numeric Features

### Raw values, ratios, and safe division

```sql
SELECT
    CustomerID,
    TotalOrderValue,
    NumOrders,
    -- Safe division with NULLIF avoids divide-by-zero
    TotalOrderValue / NULLIF(NumOrders, 0)      AS AvgOrderValue,
    -- Month-over-month delta
    CurrentMonthRevenue - PriorMonthRevenue     AS RevenueDelta,
    (CurrentMonthRevenue - PriorMonthRevenue)
        / NULLIF(PriorMonthRevenue, 0) * 100.0  AS RevenuePctChange
FROM CustomerMonthly;
```

### Log transform (right-skewed distributions)

```sql
-- LOG is the natural log in T-SQL. +1 handles zero values; omit only when zeros are impossible.
SELECT
    CustomerID,
    LOG(TotalOrderValue + 1)  AS LogRevenue,
    LOG(NumOrders + 1)        AS LogOrderCount,
    LOG(TotalOrderValue + 1, 10) AS Log10Revenue  -- base-10 (SQL Server 2012+)
FROM CustomerSummary;
```

### Z-score normalization

Standardizes to mean=0, stdev=1. Useful for scale-sensitive models (logistic regression, SVM,
neural networks).

```sql
-- IMPORTANT: compute mean and stdev from training rows only (see Data Leakage section)
SELECT
    CustomerID,
    TotalOrderValue,
    -- Global z-score
    (TotalOrderValue - AVG(TotalOrderValue) OVER ())
        / NULLIF(STDEV(TotalOrderValue) OVER (), 0)             AS Revenue_ZScore,
    -- Group-level z-score (within Category)
    (TotalOrderValue - AVG(TotalOrderValue) OVER (PARTITION BY Category))
        / NULLIF(STDEV(TotalOrderValue) OVER (PARTITION BY Category), 0)
                                                                AS Revenue_ZScore_ByCategory
FROM CustomerSummary;
```

### Min-max scaling

Rescales to [0, 1]. Preferred for models expecting bounded inputs.

```sql
SELECT
    CustomerID,
    (TotalOrderValue - MIN(TotalOrderValue) OVER ())
        / NULLIF(MAX(TotalOrderValue) OVER () - MIN(TotalOrderValue) OVER (), 0)
                                                AS Revenue_MinMax
FROM CustomerSummary;
-- Compute MIN/MAX from training rows only — see Data Leakage Prevention
```

### Quantile bucketing

```sql
SELECT
    CustomerID, TotalOrderValue,
    NTILE(10) OVER (ORDER BY TotalOrderValue) AS RevenueDecile,  -- 1=lowest, 10=highest
    NTILE(5)  OVER (ORDER BY TotalOrderValue) AS RevenueQuintile
FROM CustomerSummary;
```

---

## Categorical Encoding

### One-hot encoding (CASE expressions)

Best for low-cardinality categoricals (< ~20 distinct values). Creates one binary column per category.

```sql
SELECT
    ProductID, Category,
    CASE WHEN Category = 'Electronics' THEN 1 ELSE 0 END AS Cat_Electronics,
    CASE WHEN Category = 'Apparel'     THEN 1 ELSE 0 END AS Cat_Apparel,
    CASE WHEN Category = 'Home'        THEN 1 ELSE 0 END AS Cat_Home,
    CASE WHEN Category = 'Sports'      THEN 1 ELSE 0 END AS Cat_Sports,
    -- NULLs in Category produce 0 in all columns; add an explicit NULL indicator:
    CASE WHEN Category IS NULL         THEN 1 ELSE 0 END AS Cat_Missing
FROM Production.Products;
```

### Ordinal encoding (DENSE_RANK)

Integer rank per category. Works for tree-based models (XGBoost, LightGBM).

```sql
SELECT
    OrderID, Region,
    DENSE_RANK() OVER (ORDER BY Region) AS RegionOrdinal  -- 1, 1, 2, 3 (ties same, no skip)
FROM Sales.Orders;
-- Warning: ordinal values change when new categories appear. Store the mapping in a reference
-- table if you need to reproduce the same encoding at inference time.
```

### Frequency encoding

Replaces each category with its relative frequency. Good for high-cardinality columns.

```sql
-- Step 1: compute frequency from TRAINING rows only
SELECT Region, COUNT(*) * 1.0 / SUM(COUNT(*)) OVER () AS Freq
INTO #RegionFreq
FROM Sales.Orders WHERE SplitLabel = 'train'
GROUP BY Region;

-- Step 2: join to all rows
SELECT o.*, COALESCE(f.Freq, 0) AS Region_Freq
FROM Sales.Orders o
LEFT JOIN #RegionFreq f ON f.Region = o.Region;
-- Computing frequency on all rows before splitting leaks test-set distribution into training
```

---

## Temporal Features

**Rule: all temporal features must reference a fixed `@SnapshotDate`. Never use `GETDATE()`
inside feature logic.**

```sql
DECLARE @SnapshotDate DATE = '2024-06-01';

-- Recency: how long ago was the entity's last event?
SELECT
    CustomerID,
    MAX(OrderDate)                                  AS LastOrderDate,
    DATEDIFF(day, MAX(OrderDate), @SnapshotDate)    AS DaysSinceLastOrder,
    DATEDIFF(month, MAX(OrderDate), @SnapshotDate)  AS MonthsSinceLastOrder
FROM Sales.Orders
WHERE OrderDate < @SnapshotDate   -- never look past the snapshot date
GROUP BY CustomerID;

-- Frequency: how many times did the entity act in a window?
SELECT
    CustomerID,
    COUNT(*)                                                          AS TotalOrders,
    COUNT(CASE WHEN OrderDate >= DATEADD(day, -30, @SnapshotDate)
                AND OrderDate <  @SnapshotDate THEN 1 END)           AS Orders_Last30Days,
    COUNT(CASE WHEN OrderDate >= DATEADD(day, -90, @SnapshotDate)
                AND OrderDate <  @SnapshotDate THEN 1 END)           AS Orders_Last90Days
FROM Sales.Orders
WHERE OrderDate < @SnapshotDate
GROUP BY CustomerID;

-- Tenure: how long has the entity been active?
SELECT
    CustomerID,
    MIN(OrderDate)                                              AS FirstOrderDate,
    DATEDIFF(day, MIN(OrderDate), @SnapshotDate)                AS CustomerTenureDays,
    DATEDIFF(day, MIN(OrderDate), MAX(OrderDate))               AS ActiveSpanDays
FROM Sales.Orders
WHERE OrderDate < @SnapshotDate
GROUP BY CustomerID;

-- Calendar features
SELECT
    OrderID,
    DATEPART(weekday, OrderDate)    AS DayOfWeek,
    DATEPART(month, OrderDate)      AS MonthOfYear,
    DATEPART(quarter, OrderDate)    AS Quarter,
    DATEPART(hour, OrderTime)       AS HourOfDay,
    CASE WHEN DATEPART(weekday, OrderDate) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend
FROM Sales.Orders;
-- Note: DATEPART(weekday) is affected by @@DATEFIRST.
-- Use DATENAME(weekday, date) for a stable string, or set @@DATEFIRST 1 for ISO (Monday=1).
```

---

## RFM Pattern

RFM (Recency, Frequency, Monetary) is the standard customer feature set for churn and LTV models.

```sql
DECLARE @SnapshotDate DATE = '2024-06-01';

WITH RFM AS (
    SELECT
        CustomerID,
        DATEDIFF(day, MAX(OrderDate), @SnapshotDate)    AS Recency,     -- lower = more recent
        COUNT(*)                                        AS Frequency,
        SUM(OrderAmount)                                AS Monetary
    FROM Sales.Orders
    WHERE OrderDate < @SnapshotDate
    GROUP BY CustomerID
)
SELECT
    CustomerID, Recency, Frequency, Monetary,
    -- Quintile scores: 5 = best, 1 = worst
    -- Recency: ORDER BY DESC so fewer days (more recent) = higher score
    NTILE(5) OVER (ORDER BY Recency     DESC) AS R_Score,
    NTILE(5) OVER (ORDER BY Frequency   DESC) AS F_Score,
    NTILE(5) OVER (ORDER BY Monetary    DESC) AS M_Score,
    -- Combined score
    NTILE(5) OVER (ORDER BY Recency DESC)
    + NTILE(5) OVER (ORDER BY Frequency DESC)
    + NTILE(5) OVER (ORDER BY Monetary  DESC)       AS RFM_Total,
    -- Segment string (for interpretability only — not a model input)
    CAST(NTILE(5) OVER (ORDER BY Recency DESC) AS VARCHAR)
    + CAST(NTILE(5) OVER (ORDER BY Frequency DESC) AS VARCHAR)
    + CAST(NTILE(5) OVER (ORDER BY Monetary  DESC) AS VARCHAR) AS RFM_Segment
FROM RFM;
-- NTILE(5) distributes into 5 buckets. If row count is not divisible by 5,
-- the first (count % 5) buckets get one extra row.
```

---

## Rolling Window Features

Always use `ROWS BETWEEN`, not `RANGE BETWEEN`. `RANGE` includes all rows with the same
ORDER BY value — for timestamp data with duplicates, this silently inflates counts.

```sql
SELECT
    CustomerID, OrderDate, OrderAmount,
    -- 7-day rolling sum (current row + 6 before it)
    SUM(OrderAmount) OVER (
        PARTITION BY CustomerID ORDER BY OrderDate
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    )                                               AS Revenue_7Day,
    -- 30-day rolling count
    COUNT(*) OVER (
        PARTITION BY CustomerID ORDER BY OrderDate
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    )                                               AS OrderCount_30Day,
    -- 30-day rolling volatility (standard deviation of amount)
    STDEV(OrderAmount) OVER (
        PARTITION BY CustomerID ORDER BY OrderDate
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    )                                               AS AmountVolatility_30Day,
    -- First and last value in the partition
    FIRST_VALUE(ProductCategory) OVER (
        PARTITION BY CustomerID ORDER BY OrderDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                               AS FirstCategory
FROM Sales.Orders
WHERE OrderDate < @SnapshotDate;
-- LAST_VALUE requires ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
-- otherwise it returns the current row's value (default frame gotcha)

-- SQL Server 2022+: WINDOW clause to avoid repeating partition/order
SELECT
    CustomerID, OrderDate,
    SUM(OrderAmount) OVER w AS Revenue_7Day,
    COUNT(*)         OVER w AS Count_7Day,
    AVG(OrderAmount) OVER w AS Avg_7Day
FROM Sales.Orders
WINDOW w AS (
    PARTITION BY CustomerID ORDER BY OrderDate
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
);
```

---

## Lag Features

Lag features capture the value at a prior time step. `LAG` is safe — it looks backward.
Never use `LEAD` for training features; it looks into the future and leaks the answer.

```sql
SELECT
    CustomerID, OrderDate, OrderAmount,
    -- Previous order amount
    LAG(OrderAmount, 1) OVER (PARTITION BY CustomerID ORDER BY OrderDate) AS PrevOrderAmt,
    -- Two orders ago
    LAG(OrderAmount, 2) OVER (PARTITION BY CustomerID ORDER BY OrderDate) AS OrderAmt_2Lag,
    -- Delta from previous order
    OrderAmount - LAG(OrderAmount, 1) OVER (PARTITION BY CustomerID ORDER BY OrderDate)
                                                                          AS AmountDelta,
    -- Days since previous order
    DATEDIFF(day,
        LAG(OrderDate, 1) OVER (PARTITION BY CustomerID ORDER BY OrderDate),
        OrderDate)                                                         AS DaysSincePrevOrder
FROM Sales.Orders
WHERE OrderDate < @SnapshotDate;

-- Multi-horizon lags for demand forecasting
SELECT
    ProductID, SalesDate, UnitsSold,
    LAG(UnitsSold, 1)  OVER (PARTITION BY ProductID ORDER BY SalesDate) AS Lag1,
    LAG(UnitsSold, 7)  OVER (PARTITION BY ProductID ORDER BY SalesDate) AS Lag7,
    LAG(UnitsSold, 14) OVER (PARTITION BY ProductID ORDER BY SalesDate) AS Lag14,
    LAG(UnitsSold, 28) OVER (PARTITION BY ProductID ORDER BY SalesDate) AS Lag28
FROM DailySales
WHERE SalesDate < @SnapshotDate;
```

---

## NULL Imputation

Impute on training rows only, then apply to all rows. Imputing on the full dataset before
splitting leaks test-set statistics into training features.

```sql
-- Mean imputation (for roughly symmetric distributions)
-- Step 1: compute mean from training rows
SELECT AVG(CAST(OrderAmount AS FLOAT)) AS TrainMean
INTO #MeanStats
FROM CustomerFeatures WHERE SplitLabel = 'train';

-- Step 2: apply to all rows
SELECT f.CustomerID,
       COALESCE(f.OrderAmount, s.TrainMean) AS OrderAmount_Imputed
FROM CustomerFeatures f CROSS JOIN #MeanStats s;

-- Median imputation (for skewed distributions, robust to outliers)
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY OrderAmount) OVER () AS MedianAmount
FROM CustomerFeatures WHERE SplitLabel = 'train';

-- Mode imputation (for categoricals)
SELECT TOP 1 Category, COUNT(*) AS cnt
FROM CustomerFeatures WHERE SplitLabel = 'train' AND Category IS NOT NULL
GROUP BY Category ORDER BY cnt DESC;

-- Forward fill (sensor/time-series: carry the last known value forward)
-- Requires PARTITION BY to avoid bleeding across entities
SELECT
    SensorID, ReadingTime, Temperature,
    LAST_VALUE(Temperature IGNORE NULLS) OVER (  -- SQL Server 2022+
        PARTITION BY SensorID
        ORDER BY ReadingTime
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS Temperature_FwdFill
FROM Sensors.Readings;

-- Pre-2022 forward fill: find the most recent non-NULL value per row
SELECT
    SensorID, ReadingTime,
    COALESCE(Temperature,
        (SELECT TOP 1 Temperature FROM Sensors.Readings r2
         WHERE r2.SensorID = r.SensorID AND r2.ReadingTime < r.ReadingTime
           AND r2.Temperature IS NOT NULL
         ORDER BY r2.ReadingTime DESC)
    ) AS Temperature_FwdFill
FROM Sensors.Readings r;

-- NULL indicator column (preserves missingness as a signal)
SELECT
    CustomerID,
    COALESCE(LastLoginDate, @SnapshotDate)         AS LastLoginDate_Imputed,
    CASE WHEN LastLoginDate IS NULL THEN 1 ELSE 0 END AS LastLoginDate_IsMissing
FROM Customers;
```

---

## Sampling and Train/Test Splitting

### HASHBYTES deterministic split

```sql
-- HASHBYTES('SHA2_256', key) produces a deterministic, well-distributed hash.
-- ABS(CAST(... AS BIGINT)) % 10 assigns each entity to bucket 0-9 reproducibly.
-- The same customer lands in the same bucket every run, regardless of table state.

SELECT
    CustomerID,
    CASE
        WHEN (CAST(CAST(HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
                        AS BINARY(8)) AS BIGINT) % 10 + 10) % 10 < 8 THEN 'train'
        ELSE 'test'
    END AS SplitLabel
FROM Sales.Customers;
-- (% 10 + 10) % 10 handles negative modulo without ABS (avoids BIGINT min overflow)

-- Three-way split: 70% train / 15% validation / 15% test
SELECT
    CustomerID,
    CASE
        WHEN ((CAST(CAST(HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
                         AS BINARY(8)) AS BIGINT) % 20) + 20) % 20 < 14 THEN 'train'
        WHEN ((CAST(CAST(HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
                         AS BINARY(8)) AS BIGINT) % 20) + 20) % 20 < 17 THEN 'validation'
        ELSE 'test'
    END AS SplitLabel
FROM Sales.Customers;
-- Gotcha: HASHBYTES on NULL returns NULL → those customers land in no split. Filter them first.
```

### Time-based split (required for temporal data)

```sql
DECLARE @TrainEnd   DATE = '2023-12-31';
DECLARE @TestStart  DATE = '2024-01-01';

-- Random splits are wrong for temporal data: they allow the model to train on future events
-- relative to some test rows, inflating evaluation metrics.
WITH Labeled AS (
    SELECT CustomerID, OrderDate,
           CASE WHEN OrderDate <= @TrainEnd  THEN 'train'
                WHEN OrderDate >= @TestStart THEN 'test'
           END AS SplitLabel
    FROM Sales.Orders
    WHERE OrderDate <= @TestStart
)
SELECT * FROM Labeled WHERE SplitLabel IS NOT NULL;
-- Rows between TrainEnd and TestStart form a gap buffer — excluding them is optional but safer
```

### K-fold cross-validation

```sql
-- Deterministic fold assignment shuffled by hash
SELECT
    CustomerID,
    NTILE(5) OVER (
        ORDER BY HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
    ) AS FoldID   -- Folds 1-5; use fold i as validation, rest as training
FROM Sales.Customers;
-- Standard k-fold is invalid for time-series data — use walk-forward validation instead
```

### Stratified sampling (preserve class distribution)

```sql
WITH Stratified AS (
    SELECT CustomerID, Label,
           ROW_NUMBER() OVER (
               PARTITION BY Label
               ORDER BY HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
           ) AS RowWithinClass,
           COUNT(*) OVER (PARTITION BY Label) AS ClassSize
    FROM CustomerFeatures
)
SELECT CustomerID, Label,
       CASE WHEN RowWithinClass <= CAST(ClassSize * 0.8 AS INT) THEN 'train' ELSE 'test' END AS Split
FROM Stratified;
-- Each label class contributes exactly 80% to train, 20% to test
-- → class distribution is preserved in both partitions
```

### Downsampling majority class (imbalanced classification)

```sql
DECLARE @KeepRatio FLOAT = 0.1;  -- keep 10% of majority class

-- Keep all minority class rows
SELECT CustomerID, Label FROM CustomerFeatures WHERE Label = 1

UNION ALL

-- Keep a deterministic 10% subset of majority class rows
SELECT CustomerID, Label
FROM CustomerFeatures
WHERE Label = 0
  AND ((CAST(CAST(HASHBYTES('SHA2_256', CAST(CustomerID AS NVARCHAR(20)))
                  AS BINARY(8)) AS BIGINT) % 100) + 100) % 100
      < CAST(@KeepRatio * 100 AS INT);
-- Upsample the minority class only in the training set — never before splitting
```

---

## Data Leakage Prevention

Leakage is the #1 cause of models that appear to work in evaluation but fail in production.

### Temporal leakage

Feature reflects events that had not yet occurred at prediction time.

```sql
-- WRONG: uses GETDATE() — recency changes every time the query runs
SELECT CustomerID, DATEDIFF(day, MAX(OrderDate), GETDATE()) AS DaysSinceLastOrder
FROM Sales.Orders GROUP BY CustomerID;

-- WRONG: no upper date bound — future orders count toward recency
SELECT CustomerID, MAX(OrderDate) AS LastOrderDate FROM Sales.Orders GROUP BY CustomerID;

-- CORRECT: fixed snapshot date, strict upper bound
DECLARE @SnapshotDate DATE = '2023-06-01';
SELECT CustomerID, DATEDIFF(day, MAX(OrderDate), @SnapshotDate) AS DaysSinceLastOrder
FROM Sales.Orders
WHERE OrderDate < @SnapshotDate    -- never include events after the snapshot date
GROUP BY CustomerID;

-- WRONG: LEAD reads the next event (future data)
SELECT CustomerID, OrderDate,
       LEAD(OrderDate, 1) OVER (PARTITION BY CustomerID ORDER BY OrderDate) AS NextOrderDate
FROM Sales.Orders;

-- CORRECT: LAG only — it looks backward
LAG(OrderDate, 1) OVER (PARTITION BY CustomerID ORDER BY OrderDate) AS PrevOrderDate

-- Use temporal tables for point-in-time feature lookups
SELECT p.ProductID, p.Price
FROM dbo.ProductPricing FOR SYSTEM_TIME AS OF @SnapshotDate AS p;
-- Without FOR SYSTEM_TIME AS OF, you get today's price, not the price at prediction time
```

### Target leakage

Feature is derived from or encodes the outcome variable.

```sql
-- WRONG: CancellationDate is non-NULL only for churned customers
-- Including it as a feature tells the model the answer
SELECT CustomerID, CancellationDate,
       CASE WHEN CancellationDate IS NULL THEN 0 ELSE 1 END AS Churned  -- this is the label
FROM Customers;

-- WRONG: counts all returns including the current order being labeled
SELECT o.OrderID,
       COUNT(r.ReturnID) OVER (PARTITION BY o.CustomerID) AS TotalReturns
FROM Orders o LEFT JOIN Returns r ON r.OrderID = o.OrderID;

-- CORRECT: count prior returns only — strictly before this order's date
SELECT o.OrderID,
       COUNT(r.ReturnID) AS PriorReturns
FROM Orders o
LEFT JOIN Returns r
    ON r.CustomerID = o.CustomerID AND r.ReturnDate < o.OrderDate  -- strictly before
GROUP BY o.OrderID;
```

### Train/test leakage

Statistics computed on all data before the split contaminate test features.

```sql
-- WRONG: AVG and STDEV include test-set rows
SELECT CustomerID, Revenue,
       (Revenue - AVG(Revenue) OVER ()) / NULLIF(STDEV(Revenue) OVER (), 0) AS ZScore
FROM FeatureTable;

-- CORRECT: compute statistics from training rows only, apply to all rows
SELECT AVG(CAST(Revenue AS FLOAT)) AS Mu, STDEV(Revenue) AS Sigma
INTO #TrainStats
FROM FeatureTable WHERE Split = 'train';

SELECT f.CustomerID, f.Revenue,
       (f.Revenue - s.Mu) / NULLIF(s.Sigma, 0) AS Revenue_ZScore
FROM FeatureTable f CROSS JOIN #TrainStats s;
```

### Leakage quick-reference

| SQL pattern | What leaks | Fix |
|---|---|---|
| `GETDATE()` in feature logic | Feature changes at query time | Replace with `@SnapshotDate` |
| `LEAD(val) OVER (...)` as feature | Next event = future data | Use only `LAG` |
| `JOIN ON e.Date >= o.Date` without upper bound | All future events join | Add `AND e.Date < @SnapshotDate` |
| `AVG(col) OVER ()` before split | Test-set values shift the mean | Compute on `WHERE Split = 'train'` only |
| `COUNT(*) OVER (PARTITION BY label_col)` | Label information in feature value | Partition by non-label columns |
| `ROW_NUMBER() OVER (ORDER BY label_col)` | Top rows = positive class | Order by date or entity key |
| Global z-score before split | Test statistics in train features | Compute separately on train rows |
| Forward fill without `PARTITION BY entity` | Last value bleeds across entities | Always `PARTITION BY entity_id` |

---

## Export to Pandas

```python
import pyodbc
import pandas as pd

conn_str = (
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=YourServer;DATABASE=YourDB;Trusted_Connection=yes;"
)
conn = pyodbc.connect(conn_str)

# Load a feature table
df = pd.read_sql("SELECT * FROM dbo.CustomerFeatureStore WHERE SnapshotDate = '2024-06-01'", conn)

# For large tables: chunk to avoid memory issues
chunks = pd.read_sql(
    "SELECT * FROM dbo.CustomerFeatureStore WHERE SnapshotDate = '2024-06-01'",
    conn, chunksize=50000
)
df = pd.concat(chunks, ignore_index=True)
```

```sql
-- SQL side: export to CSV with BCP (runs on the server, fast for large tables)
-- bcp "SELECT * FROM YourDB.dbo.CustomerFeatureStore WHERE SnapshotDate='2024-06-01'"
--     queryout features.csv -c -t, -r\n -S YourServer -d YourDB -T
```

---

## Quick Reference: SQL Pattern to ML Concept

| ML concept | T-SQL pattern |
|---|---|
| Recency | `DATEDIFF(day, MAX(EventDate), @SnapshotDate)` |
| 30-day order count | `COUNT(*) OVER (PARTITION BY entity ORDER BY dt ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)` |
| Rolling 7-day revenue | `SUM(Amount) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` |
| Lag feature (t-1) | `LAG(Amount, 1) OVER (PARTITION BY entity ORDER BY dt)` |
| One-hot encode | `CASE WHEN Category = 'A' THEN 1 ELSE 0 END AS Cat_A` |
| Ordinal encode | `DENSE_RANK() OVER (ORDER BY Category)` |
| Frequency encode | `COUNT(*) OVER (PARTITION BY Category) * 1.0 / COUNT(*) OVER ()` (training rows only) |
| Log transform | `LOG(Amount + 1)` |
| Quantile bucket | `NTILE(10) OVER (ORDER BY Score)` |
| Z-score | `(col - AVG(col) OVER ()) / NULLIF(STDEV(col) OVER (), 0)` (training rows only) |
| Min-max scale | `(col - MIN(col) OVER ()) / NULLIF(MAX(col) OVER () - MIN(col) OVER (), 0)` |
| Row hash for split | `((CAST(CAST(HASHBYTES('SHA2_256', CAST(ID AS NVARCHAR(20))) AS BINARY(8)) AS BIGINT) % 10) + 10) % 10` |
| Mean imputation | `COALESCE(col, AVG(col) OVER ())` (from training rows only) |
| Median imputation | `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY col) OVER ()` |
| NULL indicator | `CASE WHEN col IS NULL THEN 1 ELSE 0 END AS col_missing` |
| Date truncation | `DATETRUNC(month, EventDate)` (2022+) or `DATEFROMPARTS(YEAR(d), MONTH(d), 1)` |
| Rolling volatility | `STDEV(col) OVER (PARTITION BY entity ORDER BY dt ROWS BETWEEN 29 PRECEDING AND CURRENT ROW)` |

---

## Common Mistakes

| Mistake | What goes wrong | Fix |
|---|---|---|
| `GETDATE()` in feature queries | Feature changes every time the query runs | Replace with a fixed `@SnapshotDate` variable |
| Missing `WHERE EventDate < @SnapshotDate` | Future events included in features | Add strict upper bound to every event join |
| `LEAD(val)` as a training feature | Future event value leaks the prediction target | Use only `LAG` for training features |
| `AVG(col) OVER ()` before split | Test-set statistics contaminate training features | Compute from `WHERE Split = 'train'` only |
| Rolling window with `RANGE` instead of `ROWS` | Tied timestamps inflate window counts silently | Always use `ROWS BETWEEN n PRECEDING AND CURRENT ROW` |
| `AVG(col)` when NULLs mean zero | AVG divides by non-NULL count — mean is inflated | Use `AVG(COALESCE(col, 0))` |
| `LAST_VALUE` without explicit frame | Default frame ends at current row, not partition end | Use `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` |
| Forward fill without `PARTITION BY entity` | Last value bleeds across entity boundaries | Always `PARTITION BY entity_id` in forward-fill window |
| Random split on time-series data | Test set contains past rows relative to training rows | Use a cutoff-date split |
| `HASHBYTES(algo, NULL)` on entity key | NULL keys land in no split partition | Filter or handle NULL keys before splitting |
| `UNPIVOT` for one-hot encoding | Silently drops NULL values | Use `CROSS APPLY VALUES` to preserve NULLs as zeros |
| Including a consequence of the label as a feature | Model learns the answer directly | Exclude any column derived from the outcome variable |
| Computing global stats before train/test split | Test distribution shapes training statistics | Assign splits first, compute stats from training rows only |
