---
topic: Data leakage prevention — temporal, target, and train/test leakage
keywords: [data leakage, temporal leakage, target leakage, train test leakage, @SnapshotDate, GETDATE, LEAD, CancellationDate, z-score on test, leakage prevention]
use_when: Auditing a feature table for data leakage, or understanding the three types of leakage in SQL-based ML pipelines.
---

# Data Leakage Prevention

## Type 1: Temporal Leakage (future data in features)

```sql
-- WRONG: GETDATE() makes features depend on when the query runs
SELECT CustomerID, DATEDIFF(day, LastOrderDate, GETDATE()) AS DaysSinceOrder ...

-- RIGHT: use a fixed snapshot date
DECLARE @SnapshotDate DATE = '2024-06-01';
SELECT CustomerID, DATEDIFF(day, LastOrderDate, @SnapshotDate) AS DaysSinceOrder ...

-- WRONG: LEAD reads the next row (future event)
LAG_LEAD_Revenue AS LEAD(TotalAmount) OVER (PARTITION BY CustomerID ORDER BY OrderDate)

-- RIGHT: LAG reads the previous row (past event)
Prev_Revenue AS LAG(TotalAmount) OVER (PARTITION BY CustomerID ORDER BY OrderDate)

-- WRONG: temporal table uses current time implicitly
SELECT Price FROM Production.Product FOR SYSTEM_TIME AS OF GETDATE()

-- RIGHT: fixed snapshot
SELECT Price FROM Production.Product FOR SYSTEM_TIME AS OF @SnapshotDate
```

## Type 2: Target Leakage (label-correlated features)

```sql
-- WRONG: CancellationDate is NULL only for completed orders — it IS the label
SELECT CustomerID,
       CancellationDate,       -- leaks the outcome
       DATEDIFF(day, OrderDate, CancellationDate) AS DaysToCancel  -- leaks the outcome
FROM Sales.Orders;

-- WRONG: TotalReturns includes returns from the order being predicted
SUM(r.ReturnAmount) AS TotalReturns  -- includes future returns on current order

-- RIGHT: strict date bound — only returns before the snapshot date on prior orders
SUM(CASE WHEN r.ReturnDate < @SnapshotDate
          AND r.OrderID <> @CurrentOrderID THEN r.ReturnAmount ELSE 0 END)
```

## Type 3: Train/Test Leakage (test data influences train statistics)

```sql
-- WRONG: z-score computed over entire #Base (train + test)
(Revenue - AVG(Revenue) OVER ()) / STDEV(Revenue) OVER () AS ZScore

-- RIGHT: compute mean/stdev on train rows only, apply to all rows
SELECT AVG(Revenue) AS MeanRev, STDEV(Revenue) AS StdevRev
INTO #TrainStats FROM #Base WHERE Split = 'train';

SELECT b.CustomerID,
    (b.Revenue - s.MeanRev) / NULLIF(s.StdevRev, 0) AS ZScore_Revenue
FROM #Base b CROSS JOIN #TrainStats s;
```

**Quick reference — common leaky patterns:**

| Leaky pattern | Fix |
|---|---|
| `GETDATE()` in feature query | `@SnapshotDate` fixed date |
| `LEAD()` window function | Replace with `LAG()` |
| Columns that directly encode the label | Exclude from feature set |
| Aggregates that include the target event | Add strict date bound `< @SnapshotDate` |
| Z-score / min-max over all rows | Compute on train split, apply to all |
| `NEWID()` for random split | HASHBYTES for deterministic split |
