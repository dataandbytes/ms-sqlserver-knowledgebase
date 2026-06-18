---
topic: Time-series patterns — YTD, cohort analysis, temporal table point-in-time joins
keywords: [YTD, year-to-date, running total, cohort analysis, CohortMonth, MonthOffset, temporal table, AS OF, historical price, fiscal year]
use_when: Computing YTD totals, building cohort retention tables, or joining to historical data at a specific point in time.
---

# Time-Series Patterns

```sql
-- YTD: running total reset at year boundary
SELECT OrderDate, Revenue,
       SUM(Revenue) OVER (
           PARTITION BY YEAR(OrderDate)
           ORDER BY OrderDate
           ROWS UNBOUNDED PRECEDING
       ) AS YTD_Revenue
FROM Sales.DailyRevenue;

-- Fiscal YTD (fiscal year starts July 1)
SELECT OrderDate, Revenue,
       CASE WHEN MONTH(OrderDate) >= 7
            THEN YEAR(OrderDate)
            ELSE YEAR(OrderDate) - 1
       END AS FiscalYear,
       SUM(Revenue) OVER (
           PARTITION BY CASE WHEN MONTH(OrderDate) >= 7
                             THEN YEAR(OrderDate)
                             ELSE YEAR(OrderDate) - 1 END
           ORDER BY OrderDate ROWS UNBOUNDED PRECEDING
       ) AS FiscalYTD
FROM Sales.DailyRevenue;

-- Cohort analysis: retention by month offset since first purchase
WITH FirstPurchase AS (
    SELECT CustomerID, DATETRUNC(month, MIN(OrderDate)) AS CohortMonth
    FROM Sales.Orders GROUP BY CustomerID
),
Activity AS (
    SELECT o.CustomerID,
           DATEDIFF(month, fp.CohortMonth, DATETRUNC(month, o.OrderDate)) AS MonthOffset
    FROM Sales.Orders o
    JOIN FirstPurchase fp ON fp.CustomerID = o.CustomerID
)
SELECT CohortMonth, MonthOffset, COUNT(DISTINCT CustomerID) AS ActiveCustomers
FROM FirstPurchase fp
JOIN Activity a ON a.CustomerID = fp.CustomerID
GROUP BY CohortMonth, MonthOffset
ORDER BY CohortMonth, MonthOffset;

-- Temporal table: point-in-time historical price join
SELECT o.OrderID, o.OrderDate, ol.ProductID, ol.Qty,
       ph.Price AS PriceAtOrderTime,
       ol.Qty * ph.Price AS LineTotal
FROM Sales.Orders o
JOIN Sales.OrderLines ol ON ol.OrderID = o.OrderID
JOIN Production.Product FOR SYSTEM_TIME AS OF o.OrderDate ph ON ph.ProductID = ol.ProductID;
```

**Key rules:**
- Cohort `MonthOffset = 0` is the acquisition month. `MonthOffset = 1` is the first retention month.
- Temporal table `FOR SYSTEM_TIME AS OF` times are UTC. If `OrderDate` is local time, convert with `AT TIME ZONE` first.
- For fiscal year calculations, use a Calendar table with a `FiscalYear` column instead of inline CASE expressions — it handles edge cases (fiscal year end on weekday, 53-week years) correctly.
