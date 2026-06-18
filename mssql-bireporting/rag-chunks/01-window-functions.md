---
topic: Window functions — ROW_NUMBER, RANK, running totals, moving averages, LAG/LEAD, ROWS vs RANGE
keywords: [window function, OVER, PARTITION BY, ROW_NUMBER, RANK, DENSE_RANK, running total, moving average, LAG, LEAD, ROWS BETWEEN, RANGE BETWEEN, WINDOW clause]
use_when: Ranking rows, computing running totals, moving averages, year-over-year comparisons, or any calculation that needs values from other rows.
---

# Window Functions

```sql
-- ROW_NUMBER: greatest-N-per-group (latest order per customer)
SELECT CustomerID, OrderID, OrderDate, TotalAmount
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderDate DESC) AS rn
    FROM Sales.Orders
) r WHERE rn = 1;

-- Running total (cumulative sum)
SELECT OrderDate, TotalAmount,
       SUM(TotalAmount) OVER (ORDER BY OrderDate ROWS UNBOUNDED PRECEDING) AS RunningTotal
FROM Sales.Orders;

-- 7-day moving average
SELECT OrderDate, Revenue,
       AVG(Revenue) OVER (ORDER BY OrderDate ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS MA7
FROM Sales.DailyRevenue;

-- Year-over-year with LAG (offset by 12 months in a monthly series)
SELECT ReportMonth, Revenue,
       LAG(Revenue, 12) OVER (ORDER BY ReportMonth) AS PriorYearRevenue,
       Revenue - LAG(Revenue, 12) OVER (ORDER BY ReportMonth) AS YoY_Delta
FROM Sales.MonthlyRevenue;

-- Named WINDOW clause (SQL Server 2022+): define once, reference many times
SELECT OrderDate, Revenue,
       AVG(Revenue)  OVER w AS MA7,
       SUM(Revenue)  OVER w AS Rolling7Sum,
       COUNT(*)      OVER w AS DayCount
FROM Sales.DailyRevenue
WINDOW w AS (ORDER BY OrderDate ROWS BETWEEN 6 PRECEDING AND CURRENT ROW);
```

**ROWS vs RANGE:**
- `ROWS BETWEEN` uses physical row position — exact window size, deterministic.
- `RANGE BETWEEN` uses logical value — all rows with the same ORDER BY value are included in the frame. `RANGE UNBOUNDED PRECEDING` is the default when you write `OVER (ORDER BY col)` with no explicit frame — this can cause unexpected results on ties.

**Key rules:**
- Window functions cannot appear in WHERE or HAVING. Wrap in a subquery or CTE if you need to filter on the result.
- `PARTITION BY` resets the window calculation for each partition. Omit it only when you want a global window (e.g., grand total).
- `LAG` for training data — never use `LEAD` (it reads future rows, causing data leakage in ML features).
