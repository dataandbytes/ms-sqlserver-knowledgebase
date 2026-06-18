---
topic: Gaps and islands — ROW_NUMBER difference technique, streaks, session analysis
keywords: [gaps and islands, ROW_NUMBER, difference technique, consecutive, streak, session, island, gap, LEAD, status change]
use_when: Finding consecutive sequences, identifying gaps in a series, detecting session boundaries, or tracking status change streaks.
---

# Gaps and Islands

## The ROW_NUMBER Difference Technique

Consecutive rows of the same value produce a constant difference when you subtract a global ROW_NUMBER from a within-group ROW_NUMBER:

```sql
-- Customer purchase streaks (consecutive months with purchases)
WITH MonthlyPurchases AS (
    SELECT CustomerID,
           DATETRUNC(month, OrderDate) AS PurchaseMonth
    FROM Sales.Orders
    GROUP BY CustomerID, DATETRUNC(month, OrderDate)
),
Numbered AS (
    SELECT CustomerID, PurchaseMonth,
           -- global rank
           ROW_NUMBER() OVER (ORDER BY CustomerID, PurchaseMonth) AS rn_global,
           -- rank within each customer
           ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY PurchaseMonth) AS rn_customer
    FROM MonthlyPurchases
)
-- rn_global - rn_customer is constant for consecutive months per customer
SELECT CustomerID,
       MIN(PurchaseMonth) AS StreakStart,
       MAX(PurchaseMonth) AS StreakEnd,
       COUNT(*) AS StreakLength
FROM Numbered
GROUP BY CustomerID, rn_global - rn_customer
HAVING COUNT(*) >= 3  -- streaks of 3+ months only
ORDER BY CustomerID, StreakStart;
```

## Gap Detection with LEAD

```sql
-- Find gaps in order sequence within customer
SELECT CustomerID, OrderNo,
       LEAD(OrderNo) OVER (PARTITION BY CustomerID ORDER BY OrderNo) AS NextOrderNo,
       LEAD(OrderNo) OVER (PARTITION BY CustomerID ORDER BY OrderNo) - OrderNo - 1 AS GapSize
FROM Sales.Order
-- Window functions cannot appear in WHERE — wrap in CTE:
```

```sql
WITH Gaps AS (
    SELECT CustomerID, OrderNo,
           LEAD(OrderNo) OVER (PARTITION BY CustomerID ORDER BY OrderNo) - OrderNo - 1 AS GapSize
    FROM Sales.Order
)
SELECT * FROM Gaps WHERE GapSize > 0;
```

## Session Analysis (30-minute timeout)

```sql
WITH Events AS (
    SELECT UserID, EventTime,
           LAG(EventTime) OVER (PARTITION BY UserID ORDER BY EventTime) AS PrevEventTime
    FROM WebLogs.Events
),
Sessions AS (
    SELECT UserID, EventTime,
           SUM(CASE WHEN DATEDIFF(minute, PrevEventTime, EventTime) > 30
                    OR PrevEventTime IS NULL THEN 1 ELSE 0 END)
               OVER (PARTITION BY UserID ORDER BY EventTime) AS SessionNo
    FROM Events
)
SELECT UserID, SessionNo,
       MIN(EventTime) AS SessionStart,
       MAX(EventTime) AS SessionEnd,
       COUNT(*) AS EventCount
FROM Sessions
GROUP BY UserID, SessionNo;
```
