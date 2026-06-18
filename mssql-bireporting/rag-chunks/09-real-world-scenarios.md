---
topic: Real-world BI reports — executive dashboard, SaaS MRR bridge, retention cohort, RFM, Pareto, anomaly detection, reorder report
keywords: [executive dashboard, KPI, MoM, YoY, YTD, MRR, revenue bridge, expansion, churn, retention cohort, RFM segmentation, NTILE, Pareto, ABC analysis, 80/20, anomaly detection, z-score, inventory reorder, days of cover, real world report]
use_when: You need a complete, business-framed report that combines multiple BI techniques rather than a single isolated pattern.
---

# Real-World BI Scenarios

Complete reports combining aggregation, window functions, and conditional logic. Schema:
`Sales.Orders` (OrderID, CustomerID, OrderDate, TotalAmount, Status, Region, ProductID),
`Sales.OrderLines` (OrderID, ProductID, ProductName, Qty, UnitPrice, Cost),
`Sales.Customers` (CustomerID, SignupDate), `Production.Products`, `dbo.Calendar`.

## Executive sales dashboard (every KPI in one query)

```sql
WITH Monthly AS (
    SELECT DATEFROMPARTS(YEAR(OrderDate), MONTH(OrderDate), 1) AS MonthStart,
           SUM(TotalAmount) AS Revenue, COUNT(*) AS Orders,
           COUNT(DISTINCT CustomerID) AS ActiveCustomers
    FROM Sales.Orders
    WHERE Status = 'Completed' AND OrderDate >= '2023-01-01' AND OrderDate < '2025-01-01'
    GROUP BY DATEFROMPARTS(YEAR(OrderDate), MONTH(OrderDate), 1)
)
SELECT MonthStart, Revenue, Orders, Revenue / NULLIF(Orders,0) AS AvgOrderValue,
    (Revenue - LAG(Revenue) OVER (ORDER BY MonthStart)) * 100.0
        / NULLIF(LAG(Revenue) OVER (ORDER BY MonthStart),0) AS MoM_Pct,
    (Revenue - LAG(Revenue,12) OVER (ORDER BY MonthStart)) * 100.0
        / NULLIF(LAG(Revenue,12) OVER (ORDER BY MonthStart),0) AS YoY_Pct,
    SUM(Revenue) OVER (PARTITION BY YEAR(MonthStart) ORDER BY MonthStart ROWS UNBOUNDED PRECEDING) AS YTD_Revenue,
    AVG(Revenue) OVER (ORDER BY MonthStart ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS MA_3Month
FROM Monthly ORDER BY MonthStart;
```

## SaaS MRR movement (revenue bridge: new / expansion / contraction / churn)

```sql
WITH WithPrev AS (
    SELECT CustomerID, MonthStart, MRR,
           LAG(MRR) OVER (PARTITION BY CustomerID ORDER BY MonthStart) AS PrevMRR
    FROM Billing.Subscriptions
)
SELECT MonthStart,
    SUM(CASE WHEN PrevMRR IS NULL              THEN MRR           ELSE 0 END) AS NewMRR,
    SUM(CASE WHEN PrevMRR > 0 AND MRR > PrevMRR THEN MRR - PrevMRR ELSE 0 END) AS ExpansionMRR,
    SUM(CASE WHEN MRR > 0 AND MRR < PrevMRR     THEN MRR - PrevMRR ELSE 0 END) AS ContractionMRR,
    SUM(CASE WHEN PrevMRR > 0 AND MRR = 0       THEN -PrevMRR      ELSE 0 END) AS ChurnedMRR,
    SUM(MRR) AS EndingMRR
FROM WithPrev GROUP BY MonthStart ORDER BY MonthStart;
```

## RFM segmentation (NTILE quintiles → named segments)

```sql
DECLARE @AsOf DATE = '2024-12-31';
WITH RFM AS (
    SELECT CustomerID,
           DATEDIFF(day, MAX(OrderDate), @AsOf) AS Recency,
           COUNT(*) AS Frequency, SUM(TotalAmount) AS Monetary
    FROM Sales.Orders WHERE Status = 'Completed' AND OrderDate <= @AsOf
    GROUP BY CustomerID
),
Scored AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY Recency DESC)  AS R,   -- DESC: recent buyers score 5
        NTILE(5) OVER (ORDER BY Frequency ASC) AS F,
        NTILE(5) OVER (ORDER BY Monetary ASC)  AS M
    FROM RFM
)
SELECT CustomerID, R, F, M, CONCAT(R,F,M) AS RFM_Cell,
    CASE WHEN R>=4 AND F>=4 AND M>=4 THEN 'Champions'
         WHEN R>=4 AND F>=3          THEN 'Loyal'
         WHEN R<=2 AND F>=4          THEN 'At Risk (high value)'
         ELSE 'Hibernating' END AS Segment
FROM Scored ORDER BY M DESC, F DESC;
```

## Pareto / ABC analysis (80/20 cumulative contribution)

```sql
WITH ProductRevenue AS (
    SELECT ol.ProductID, SUM(ol.Qty * ol.UnitPrice) AS Revenue
    FROM Sales.OrderLines ol GROUP BY ol.ProductID
),
Ranked AS (
    SELECT *,
        SUM(Revenue) OVER (ORDER BY Revenue DESC ROWS UNBOUNDED PRECEDING) AS CumRevenue,
        SUM(Revenue) OVER () AS TotalRevenue
    FROM ProductRevenue
)
SELECT ProductID, Revenue,
    CAST(100.0*CumRevenue/TotalRevenue AS DECIMAL(5,2)) AS CumulativePct,
    CASE WHEN 100.0*CumRevenue/TotalRevenue <= 80 THEN 'A'
         WHEN 100.0*CumRevenue/TotalRevenue <= 95 THEN 'B' ELSE 'C' END AS ABC_Class
FROM Ranked ORDER BY Revenue DESC;
```

## Daily revenue anomaly detection (z-score vs trailing 28-day baseline)

```sql
WITH Daily AS (
    SELECT C.FullDate, ISNULL(SUM(O.TotalAmount),0) AS Revenue
    FROM dbo.Calendar C
    LEFT JOIN Sales.Orders O ON O.OrderDate >= C.FullDate
        AND O.OrderDate < DATEADD(day,1,C.FullDate) AND O.Status = 'Completed'
    WHERE C.FullDate >= '2024-01-01' AND C.FullDate < '2025-01-01'
    GROUP BY C.FullDate
),
Baseline AS (
    SELECT FullDate, Revenue,
        AVG(Revenue)   OVER (ORDER BY FullDate ROWS BETWEEN 28 PRECEDING AND 1 PRECEDING) AS TrailingAvg,
        STDEV(Revenue) OVER (ORDER BY FullDate ROWS BETWEEN 28 PRECEDING AND 1 PRECEDING) AS TrailingStdev
    FROM Daily
)
SELECT FullDate, Revenue,
    CAST((Revenue - TrailingAvg)/NULLIF(TrailingStdev,0) AS DECIMAL(6,2)) AS ZScore,
    CASE WHEN ABS((Revenue-TrailingAvg)/NULLIF(TrailingStdev,0)) > 2 THEN 'ANOMALY' ELSE 'normal' END AS Flag
FROM Baseline WHERE TrailingStdev IS NOT NULL ORDER BY FullDate;
```

**Note:** the baseline window ends at `1 PRECEDING` so the current day is excluded from its own
baseline. Skip the first 28 warm-up days where `TrailingStdev IS NULL`.
