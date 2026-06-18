---
topic: Date bucketing — DATETRUNC, DATE_BUCKET, DATEADD/DATEDIFF patterns, SARGable date filters
keywords: [date bucketing, DATETRUNC, DATE_BUCKET, DATEADD, DATEDIFF, SARGable, date filter, time bucket, 15-minute bucket, week start]
use_when: Truncating dates to a period boundary, grouping into fixed-width time buckets, or writing SARGable date range filters.
---

# Date Bucketing

```sql
-- DATETRUNC (SQL Server 2022+): clean truncation to any part
SELECT DATETRUNC(month,  OrderDate) AS MonthStart  FROM Sales.Orders;
SELECT DATETRUNC(week,   EventTime) AS WeekStart   FROM Logs.Events;
SELECT DATETRUNC(quarter,SaleDate)  AS QtrStart    FROM Sales.Summary;

-- DATE_BUCKET (SQL Server 2022+): arbitrary interval bucketing
SELECT DATE_BUCKET(minute, 15, ReadingTime) AS Bucket15Min
FROM Sensors.Readings;   -- rounds down to nearest 15-min boundary

-- Pre-2022: DATEADD/DATEDIFF pattern
-- Month start
SELECT DATEADD(month, DATEDIFF(month, 0, OrderDate), 0) AS MonthStart FROM Sales.Orders;
-- Week start (Monday)
SELECT DATEADD(week,  DATEDIFF(week,  0, OrderDate), 0) AS WeekStart  FROM Sales.Orders;
-- Quarter start
SELECT DATEADD(quarter, DATEDIFF(quarter, 0, OrderDate), 0) AS QtrStart FROM Sales.Orders;

-- SARGable date filters (use range predicates, never wrap the column in a function)
-- WRONG: WHERE YEAR(OrderDate) = 2024 AND MONTH(OrderDate) = 3
-- RIGHT:
WHERE OrderDate >= '2024-03-01' AND OrderDate < '2024-04-01'
-- WRONG: WHERE CONVERT(DATE, OrderDate) = '2024-03-15'
-- RIGHT:
WHERE OrderDate >= '2024-03-15' AND OrderDate < '2024-03-16'
```

**Key rules:**
- Any function applied to the column side of a WHERE predicate (`YEAR(col)`, `CONVERT(DATE, col)`, `CAST(col AS DATE)`) makes the predicate non-SARGable — the optimizer cannot use a range scan on the index.
- `DATETRUNC` and `DATE_BUCKET` are for SELECT lists and GROUP BY, not WHERE — use range predicates in WHERE.
- `DATEDIFF(month, 0, date)` counts months since 1900-01-01, then `DATEADD` adds that count back — the result is always midnight on the first of the month, regardless of time component.
