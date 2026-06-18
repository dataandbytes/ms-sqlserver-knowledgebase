# SQL BI and Reporting — Knowledge Base

> T-SQL patterns for business intelligence, analytics, and reporting: multi-level aggregation,
> time-series queries, gap filling, pivot/unpivot, date bucketing, cohort analysis, window
> functions, and data export from SQL Server.

## Scope

**Use this for:** summary reports with subtotals; time-series bucketing and gap filling;
year-over-year and period-over-period comparisons; pivot/unpivot transformations; gaps-and-islands
problems (consecutive sequences, session analysis); exporting results as JSON, XML, or CSV;
writing queries optimized for Power BI DirectQuery or SSRS.

**Not for:** index tuning, execution plans, schema design, or ETL pipeline architecture.

## Contents

1. [Multi-Level Aggregation](#multi-level-aggregation)
2. [Window Functions](#window-functions)
3. [Date Bucketing](#date-bucketing)
4. [Time-Series Patterns](#time-series-patterns)
5. [Calendar Tables](#calendar-tables)
6. [Gap Filling](#gap-filling)
7. [Tally Tables](#tally-tables)
8. [Pivot and Unpivot](#pivot-and-unpivot)
9. [Gaps and Islands](#gaps-and-islands)
10. [Data Export](#data-export)
11. [Real-World Scenarios](#real-world-scenarios)
12. [Common Mistakes](#common-mistakes)

---

## Multi-Level Aggregation

Three operators add subtotals to GROUP BY results in a single table scan:

| Operator | Grouping sets produced | Use when |
|---|---|---|
| `ROLLUP(A, B, C)` | (A,B,C), (A,B), (A), () | Columns form a hierarchy |
| `CUBE(A, B)` | All 2^n combinations | Need every cross-dimensional cut |
| `GROUPING SETS(...)` | Exactly what you list | Need precise combinations, not a formula |

```sql
-- ROLLUP: year > quarter > month hierarchy with grand total
SELECT
    YEAR(OrderDate)                AS OrderYear,
    DATEPART(quarter, OrderDate)   AS OrderQtr,
    MONTH(OrderDate)               AS OrderMonth,
    SUM(TotalAmount)               AS Revenue,
    COUNT(*)                       AS OrderCount
FROM Sales.Orders
WHERE OrderDate >= '2023-01-01' AND OrderDate < '2025-01-01'
GROUP BY ROLLUP(
    YEAR(OrderDate),
    DATEPART(quarter, OrderDate),
    MONTH(OrderDate)
)
ORDER BY OrderYear, OrderQtr, OrderMonth;
-- Produces: detail rows, quarterly subtotals, yearly subtotals, grand total

-- CUBE: every combination of Region + Category
SELECT
    Region,
    ProductCategory,
    SUM(Revenue)  AS TotalRevenue,
    COUNT(*)      AS Orders
FROM Sales.Orders
GROUP BY CUBE(Region, ProductCategory)
ORDER BY GROUPING(Region), GROUPING(ProductCategory), Region, ProductCategory;

-- GROUPING SETS: specify exactly the cuts you want (not a hierarchy, not all combinations)
SELECT
    YEAR(OrderDate)  AS OrderYear,
    ProductCategory,
    SUM(TotalAmount) AS Revenue
FROM Sales.Orders
GROUP BY GROUPING SETS (
    (YEAR(OrderDate), ProductCategory),  -- detail
    (YEAR(OrderDate)),                   -- year subtotal
    (ProductCategory),                   -- category subtotal
    ()                                   -- grand total
);
```

**Distinguishing subtotal NULLs from real NULLs:**

```sql
-- GROUPING(col) returns 1 for a subtotal placeholder, 0 for a real value
SELECT
    CASE WHEN GROUPING(Region) = 1 THEN '(All Regions)' ELSE Region END AS Region,
    CASE WHEN GROUPING(ProductCategory) = 1 THEN '(All Categories)' ELSE ProductCategory END AS Category,
    SUM(Revenue) AS TotalRevenue
FROM Sales.Orders
GROUP BY ROLLUP(Region, ProductCategory);

-- GROUPING_ID bitmask: useful for ordering and filtering to a specific subtotal level
SELECT
    Region, ProductCategory, SUM(Revenue) AS Revenue,
    GROUPING_ID(Region, ProductCategory) AS Level
FROM Sales.Orders
GROUP BY CUBE(Region, ProductCategory)
ORDER BY Level, Region, ProductCategory;
-- Level 0 = detail, 1 = category subtotal, 2 = region subtotal, 3 = grand total
```

**Conditional aggregation — computed in one pass, no subqueries:**

```sql
SELECT
    Region,
    COUNT(*)                                               AS TotalOrders,
    SUM(TotalAmount)                                       AS TotalRevenue,
    SUM(CASE WHEN Status = 'Completed' THEN TotalAmount ELSE 0 END) AS CompletedRevenue,
    SUM(CASE WHEN Status = 'Cancelled' THEN 1 ELSE 0 END)          AS CancelledCount,
    -- Monthly breakdown as columns (alternative to PIVOT)
    SUM(CASE WHEN MONTH(OrderDate) = 1 THEN TotalAmount END) AS Jan,
    SUM(CASE WHEN MONTH(OrderDate) = 2 THEN TotalAmount END) AS Feb,
    SUM(CASE WHEN MONTH(OrderDate) = 3 THEN TotalAmount END) AS Mar
FROM Sales.Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
GROUP BY Region;
```

**HAVING vs WHERE:** WHERE filters rows before grouping (cheaper). HAVING filters groups
after aggregation. Only use HAVING when the filter depends on an aggregate result.

```sql
-- GOOD: non-aggregate filter in WHERE, aggregate filter in HAVING
SELECT Region, SUM(TotalAmount) AS Revenue
FROM Sales.Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'  -- filters rows first
GROUP BY Region
HAVING SUM(TotalAmount) > 100000;                             -- filters groups second
```

---

## Window Functions

### Ranking and row numbering

```sql
-- ROW_NUMBER: unique sequential number per partition (no ties)
-- RANK: ties get same number, next rank skips (1, 1, 3)
-- DENSE_RANK: ties get same number, no skip (1, 1, 2)

-- Most recent order per customer (greatest-N-per-group)
WITH Ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderDate DESC) AS RN
    FROM Sales.Orders
)
SELECT OrderID, CustomerID, OrderDate, TotalAmount
FROM Ranked WHERE RN = 1;

-- Top 3 products by revenue per region
WITH Ranked AS (
    SELECT Region, ProductName, SUM(Revenue) AS TotalRevenue,
        DENSE_RANK() OVER (PARTITION BY Region ORDER BY SUM(Revenue) DESC) AS RN
    FROM Sales.OrderLines
    GROUP BY Region, ProductName
)
SELECT Region, ProductName, TotalRevenue FROM Ranked WHERE RN <= 3;
```

### Running totals and cumulative sums

Always specify `ROWS` explicitly. The default frame (`RANGE UNBOUNDED PRECEDING`) handles
ties differently and is slower — it reads the same frame-end for every tied row.

```sql
SELECT
    OrderDate,
    TotalAmount,
    SUM(TotalAmount) OVER (
        ORDER BY OrderDate
        ROWS UNBOUNDED PRECEDING        -- running total: all rows from start to here
    ) AS RunningTotal,
    SUM(TotalAmount) OVER (
        PARTITION BY CustomerID
        ORDER BY OrderDate
        ROWS UNBOUNDED PRECEDING        -- per-customer running total
    ) AS CustomerRunningTotal
FROM Sales.Orders
ORDER BY OrderDate;
```

### Moving averages

```sql
SELECT
    OrderDate,
    DailyRevenue,
    AVG(DailyRevenue) OVER (
        ORDER BY OrderDate
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW    -- 7-day moving average
    ) AS MA_7Day,
    AVG(DailyRevenue) OVER (
        ORDER BY OrderDate
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW   -- 30-day moving average
    ) AS MA_30Day,
    AVG(DailyRevenue) OVER (
        ORDER BY OrderDate
        ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING    -- centered 7-day (trend use only)
    ) AS CenteredMA_7Day
FROM DailyRevenueSummary
ORDER BY OrderDate;
```

### LAG and LEAD for period comparisons

```sql
-- Year-over-year monthly comparison
WITH MonthlyRevenue AS (
    SELECT
        YEAR(OrderDate)  AS OrderYear,
        MONTH(OrderDate) AS OrderMonth,
        SUM(TotalAmount) AS Revenue
    FROM Sales.Orders
    GROUP BY YEAR(OrderDate), MONTH(OrderDate)
)
SELECT
    OrderYear, OrderMonth, Revenue,
    LAG(Revenue, 12) OVER (ORDER BY OrderYear, OrderMonth) AS PriorYearRevenue,
    Revenue - LAG(Revenue, 12) OVER (ORDER BY OrderYear, OrderMonth) AS YoYDelta,
    CASE WHEN LAG(Revenue, 12) OVER (ORDER BY OrderYear, OrderMonth) > 0
         THEN (Revenue - LAG(Revenue, 12) OVER (ORDER BY OrderYear, OrderMonth))
              * 100.0 / LAG(Revenue, 12) OVER (ORDER BY OrderYear, OrderMonth)
    END AS YoYPctChange
FROM MonthlyRevenue
ORDER BY OrderYear, OrderMonth;

-- LAG offset 12 on monthly data = same month prior year
-- If gaps exist (some months have no rows), use a self-join instead of LAG
```

### Named WINDOW clause (SQL Server 2022+)

Reuse the same partition/order without repeating it:

```sql
SELECT
    OrderID,
    ROW_NUMBER()     OVER Win AS RowNum,
    SUM(TotalAmount) OVER Win AS RunningTotal,
    LAG(TotalAmount) OVER Win AS PrevAmount
FROM Sales.Orders
WINDOW Win AS (PARTITION BY CustomerID ORDER BY OrderDate ROWS UNBOUNDED PRECEDING);
```

---

## Date Bucketing

### SQL Server 2022+: DATETRUNC and DATE_BUCKET

```sql
-- DATETRUNC: snap to natural boundary
SELECT
    DATETRUNC(hour,    EventTime) AS HourBucket,
    DATETRUNC(day,     EventTime) AS DayBucket,
    DATETRUNC(month,   EventTime) AS MonthBucket,
    DATETRUNC(quarter, EventTime) AS QuarterBucket,
    DATETRUNC(year,    EventTime) AS YearBucket
FROM Sensors.Events;

-- DATE_BUCKET: arbitrary-width intervals with optional origin
SELECT
    DATE_BUCKET(minute, 15, ReadingTime)                AS FifteenMinBucket,
    DATE_BUCKET(week,   1,  OrderDate, '2024-01-01')    AS WeekStart  -- Monday-aligned
FROM Sales.Orders;
```

### Pre-2022: DATEADD/DATEDIFF pattern

```sql
-- Truncate to hour
DATEADD(hour,   DATEDIFF(hour,   0, EventTime), 0)

-- Truncate to day
CAST(EventTime AS DATE)

-- Truncate to month start
DATEFROMPARTS(YEAR(EventTime), MONTH(EventTime), 1)

-- 15-minute buckets
DATEADD(minute, DATEDIFF(minute, 0, EventTime) / 15 * 15, 0)

-- Practical: sensor readings in 15-minute buckets
SELECT
    DATEADD(minute, DATEDIFF(minute, 0, ReadingTime) / 15 * 15, 0) AS Bucket,
    AVG(Temperature) AS AvgTemp,
    MIN(Temperature) AS MinTemp,
    MAX(Temperature) AS MaxTemp,
    COUNT(*)         AS Readings
FROM Sensors.Readings
WHERE ReadingTime >= '2024-06-01' AND ReadingTime < '2024-06-02'
GROUP BY DATEADD(minute, DATEDIFF(minute, 0, ReadingTime) / 15 * 15, 0)
ORDER BY Bucket;
```

**SARGable date filters** — never wrap the indexed column:

```sql
-- BAD: forces index scan
WHERE YEAR(OrderDate) = 2024
WHERE CAST(OrderDate AS DATE) = '2024-06-15'

-- GOOD: allows index seek
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
WHERE OrderDate >= '2024-06-15'  AND OrderDate < '2024-06-16'
```

---

## Time-Series Patterns

### Year-to-date and fiscal-year totals

```sql
-- YTD revenue by month
WITH MonthlyRevenue AS (
    SELECT DATEFROMPARTS(YEAR(OrderDate), MONTH(OrderDate), 1) AS MonthStart,
           SUM(TotalAmount) AS Revenue
    FROM Sales.Orders WHERE YEAR(OrderDate) = 2024
    GROUP BY YEAR(OrderDate), MONTH(OrderDate)
)
SELECT
    MonthStart,
    Revenue,
    SUM(Revenue) OVER (ORDER BY MonthStart ROWS UNBOUNDED PRECEDING) AS YTDRevenue
FROM MonthlyRevenue
ORDER BY MonthStart;

-- Fiscal YTD (fiscal year starts July 1)
-- Per-fiscal-year running total restarts at each July
SUM(Revenue) OVER (
    PARTITION BY CASE WHEN MONTH(MonthStart) >= 7 THEN YEAR(MonthStart) + 1
                      ELSE YEAR(MonthStart) END          -- fiscal year key
    ORDER BY MonthStart
    ROWS UNBOUNDED PRECEDING
) AS FiscalYTD
```

### Cohort analysis

Group customers by their first-purchase month, then count active customers per subsequent month:

```sql
WITH FirstPurchase AS (
    SELECT CustomerID,
           DATEFROMPARTS(YEAR(MIN(OrderDate)), MONTH(MIN(OrderDate)), 1) AS CohortMonth
    FROM Sales.Orders
    GROUP BY CustomerID
),
Activity AS (
    SELECT f.CustomerID, f.CohortMonth,
           DATEDIFF(month, f.CohortMonth,
               DATEFROMPARTS(YEAR(o.OrderDate), MONTH(o.OrderDate), 1)) AS MonthOffset
    FROM FirstPurchase f
    JOIN Sales.Orders o ON o.CustomerID = f.CustomerID
)
SELECT
    CohortMonth,
    MonthOffset,
    COUNT(DISTINCT CustomerID) AS ActiveCustomers
FROM Activity
GROUP BY CohortMonth, MonthOffset
ORDER BY CohortMonth, MonthOffset;
-- Pivot MonthOffset 0..N into columns for a retention matrix
```

### Temporal table point-in-time queries

```sql
-- What did the product catalog look like at a specific time?
SELECT ProductID, ProductName, ListPrice
FROM Production.Products
FOR SYSTEM_TIME AS OF '2024-06-15T12:00:00'
ORDER BY ProductID;

-- Price at time of each order (historical join)
SELECT o.OrderID, o.OrderDate, p.ProductName, p.ListPrice AS PriceAtOrderTime
FROM Sales.Orders o
CROSS APPLY (
    SELECT ProductName, ListPrice
    FROM Production.Products
    FOR SYSTEM_TIME AS OF o.OrderDate
    WHERE ProductID = o.ProductID
) p;
-- FOR SYSTEM_TIME times are always UTC. Convert local times with AT TIME ZONE before use.
```

---

## Calendar Tables

A permanent calendar table provides date attributes (day name, fiscal period, holiday flags)
and acts as the backbone for gap filling.

```sql
CREATE TABLE dbo.Calendar (
    DateKey       INT        NOT NULL PRIMARY KEY,  -- YYYYMMDD integer key
    FullDate      DATE       NOT NULL UNIQUE,
    Year          SMALLINT   NOT NULL,
    Quarter       TINYINT    NOT NULL,
    Month         TINYINT    NOT NULL,
    MonthName     VARCHAR(10) NOT NULL,
    DayOfWeek     TINYINT    NOT NULL,              -- 1=Monday, 7=Sunday
    DayName       VARCHAR(10) NOT NULL,
    IsWeekend     BIT        NOT NULL,
    IsHoliday     BIT        NOT NULL DEFAULT 0,
    FiscalYear    SMALLINT   NOT NULL,
    FiscalQuarter TINYINT    NOT NULL
);

-- Seed with tally CTE (2020-01-01 through 2030-12-31)
WITH
    L0 AS (SELECT 1 c UNION ALL SELECT 1),
    L1 AS (SELECT 1 c FROM L0 CROSS JOIN L0 B),
    L2 AS (SELECT 1 c FROM L1 CROSS JOIN L1 B),
    L3 AS (SELECT 1 c FROM L2 CROSS JOIN L2 B),
    L4 AS (SELECT 1 c FROM L3 CROSS JOIN L3 B),
    Nums AS (SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS N FROM L4),
    Dates AS (SELECT DATEADD(day, N, '2020-01-01') AS D
              FROM Nums WHERE N <= DATEDIFF(day, '2020-01-01', '2030-12-31'))
INSERT INTO dbo.Calendar (DateKey, FullDate, Year, Quarter, Month, MonthName,
                          DayOfWeek, DayName, IsWeekend, FiscalYear, FiscalQuarter)
SELECT
    YEAR(D)*10000 + MONTH(D)*100 + DAY(D),
    D, YEAR(D), DATEPART(quarter, D), MONTH(D), DATENAME(month, D),
    (DATEPART(weekday, D) + @@DATEFIRST + 5) % 7 + 1,  -- Monday=1
    DATENAME(weekday, D),
    CASE WHEN (DATEPART(weekday, D) + @@DATEFIRST + 5) % 7 + 1 >= 6 THEN 1 ELSE 0 END,
    CASE WHEN MONTH(D) >= 7 THEN YEAR(D) + 1 ELSE YEAR(D) END,     -- fiscal year: July start
    CASE WHEN MONTH(D) >= 7 THEN (MONTH(D)-7)/3 + 1 ELSE (MONTH(D)+5)/3 + 1 END
FROM Dates;

-- Business days between two dates
SELECT COUNT(*) AS BusinessDays
FROM dbo.Calendar
WHERE FullDate >= '2024-01-15' AND FullDate < '2024-02-15'
  AND IsWeekend = 0 AND IsHoliday = 0;
```

---

## Gap Filling

LEFT JOIN from a complete date spine to sparse data. Every date appears even when no rows exist for it.

```sql
-- Daily revenue with zero-fill for missing days
SELECT
    C.FullDate,
    ISNULL(SUM(O.TotalAmount), 0) AS Revenue,
    COUNT(O.OrderID)              AS OrderCount
FROM dbo.Calendar C
LEFT JOIN Sales.Orders O
    ON O.OrderDate >= C.FullDate
   AND O.OrderDate <  DATEADD(day, 1, C.FullDate)  -- range join keeps OrderDate SARGable
WHERE C.FullDate >= '2024-01-01' AND C.FullDate < '2025-01-01'
GROUP BY C.FullDate
ORDER BY C.FullDate;

-- Without a permanent calendar table, generate the spine inline (see Tally Tables)
```

---

## Tally Tables

Generate number sequences with zero I/O using the stacking CTE pattern:

```sql
WITH
    L0 AS (SELECT 1 c UNION ALL SELECT 1),
    L1 AS (SELECT 1 c FROM L0 CROSS JOIN L0 B),
    L2 AS (SELECT 1 c FROM L1 CROSS JOIN L1 B),
    L3 AS (SELECT 1 c FROM L2 CROSS JOIN L2 B),
    L4 AS (SELECT 1 c FROM L3 CROSS JOIN L3 B),
    Nums AS (SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS N FROM L4)
SELECT N FROM Nums
WHERE N <= @Count;  -- REQUIRED: without this limit, all 65,536 rows are materialized

-- SQL Server 2022+: simpler syntax
SELECT value AS N FROM GENERATE_SERIES(1, @Count);

-- Date spine: generate one row per day for a range
WITH DateSpine AS (
    SELECT DATEADD(day, N - 1, @StartDate) AS D
    FROM (
        SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS N
        FROM (VALUES(1),(1),(1),(1),(1),(1),(1),(1)) A(x)   -- 8^2 = 64 rows, extend as needed
        CROSS JOIN (VALUES(1),(1),(1),(1),(1),(1),(1),(1)) B(x)
    ) Nums
    WHERE N <= DATEDIFF(day, @StartDate, @EndDate) + 1
)
SELECT D AS SpineDate FROM DateSpine;
```

**Never use recursive CTEs for number generation** — they process row by row and are 10-50x
slower than the stacking CTE approach.

---

## Pivot and Unpivot

### Static PIVOT (known column values)

```sql
SELECT ProductName, [Q1], [Q2], [Q3], [Q4]
FROM (
    SELECT ProductName,
           'Q' + CAST(DATEPART(quarter, OrderDate) AS VARCHAR) AS Qtr,
           TotalAmount
    FROM Sales.Orders o
    JOIN Production.Products p ON p.ProductID = o.ProductID
) AS Src
PIVOT (
    SUM(TotalAmount)
    FOR Qtr IN ([Q1], [Q2], [Q3], [Q4])
) AS Pvt
ORDER BY ProductName;
```

### Dynamic PIVOT (column values unknown at design time)

```sql
DECLARE @Cols NVARCHAR(MAX);
DECLARE @SQL  NVARCHAR(MAX);

SELECT @Cols = STRING_AGG(QUOTENAME(Qtr), ',') WITHIN GROUP (ORDER BY Qtr)
FROM (SELECT DISTINCT 'Q' + CAST(DATEPART(quarter, OrderDate) AS VARCHAR) AS Qtr
      FROM Sales.Orders) q;

SET @SQL = N'
SELECT ProductName, ' + @Cols + N'
FROM (
    SELECT p.ProductName,
           ''Q'' + CAST(DATEPART(quarter, o.OrderDate) AS VARCHAR) AS Qtr,
           o.TotalAmount
    FROM Sales.Orders o
    JOIN Production.Products p ON p.ProductID = o.ProductID
) Src
PIVOT (SUM(TotalAmount) FOR Qtr IN (' + @Cols + N')) Pvt;';

-- ALWAYS inspect the generated SQL before executing
PRINT @SQL;
EXEC sp_executesql @SQL;  -- QUOTENAME prevents SQL injection on column names
```

### Unpivot with CROSS APPLY VALUES (preserves NULLs)

```sql
-- UNPIVOT operator silently drops NULL values — never use it in production
-- CROSS APPLY VALUES preserves NULLs as explicit rows

SELECT s.ProductID, q.Quarter, q.Revenue
FROM Sales.QuarterlySummary s
CROSS APPLY (
    VALUES
        ('Q1', s.Q1),
        ('Q2', s.Q2),
        ('Q3', s.Q3),
        ('Q4', s.Q4)
) AS q(Quarter, Revenue);
-- NULLs appear as rows with NULL Revenue, not silently dropped
```

---

## Gaps and Islands

### Island detection: ROW_NUMBER difference technique

Within a consecutive sequence, value - row_number is constant. When the sequence breaks,
the difference changes. Group by the difference to find each island.

```sql
-- Find consecutive date ranges with sensor readings
WITH Grouped AS (
    SELECT
        SensorID,
        ReadingDate,
        DATEADD(day,
            -ROW_NUMBER() OVER (PARTITION BY SensorID ORDER BY ReadingDate),
            ReadingDate
        ) AS GroupKey
    FROM Sensors.DailyReadings
)
SELECT
    SensorID,
    MIN(ReadingDate) AS IslandStart,
    MAX(ReadingDate) AS IslandEnd,
    COUNT(*)         AS ConsecutiveDays
FROM Grouped
GROUP BY SensorID, GroupKey
ORDER BY SensorID, IslandStart;

-- How it works (trace for SensorID=1):
-- Date       RowNum  Date - RowNum   Island
-- 2024-01-01   1     2023-12-31      Island 1
-- 2024-01-02   2     2023-12-31      Island 1
-- 2024-01-03   3     2023-12-31      Island 1
-- 2024-01-05   4     2024-01-01      Island 2  (gap on Jan 4 shifts GroupKey)
-- 2024-01-06   5     2024-01-01      Island 2
```

### Customer purchase streaks

```sql
-- Find consecutive months each customer placed at least one order
WITH MonthlyActivity AS (
    SELECT DISTINCT CustomerID,
           DATEFROMPARTS(YEAR(OrderDate), MONTH(OrderDate), 1) AS ActivityMonth
    FROM Sales.Orders
),
Grouped AS (
    SELECT CustomerID, ActivityMonth,
           DATEADD(month,
               -ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY ActivityMonth),
               ActivityMonth) AS GroupKey
    FROM MonthlyActivity
)
SELECT
    CustomerID,
    MIN(ActivityMonth) AS StreakStart,
    MAX(ActivityMonth) AS StreakEnd,
    COUNT(*)           AS ConsecutiveMonths
FROM Grouped
GROUP BY CustomerID, GroupKey
HAVING COUNT(*) >= 3         -- only streaks of 3+ consecutive months
ORDER BY ConsecutiveMonths DESC;
```

### Gap detection with LEAD

```sql
-- Find gaps in daily order history
WITH OrderDates AS (
    SELECT DISTINCT CAST(OrderDate AS DATE) AS OrderDay
    FROM Sales.Orders
    WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
),
WithNext AS (
    SELECT OrderDay,
           LEAD(OrderDay) OVER (ORDER BY OrderDay) AS NextDay
    FROM OrderDates
)
SELECT
    OrderDay   AS LastDayBeforeGap,
    NextDay    AS FirstDayAfterGap,
    DATEDIFF(day, OrderDay, NextDay) - 1 AS MissingDays
FROM WithNext
WHERE DATEDIFF(day, OrderDay, NextDay) > 1
ORDER BY OrderDay;
-- Window functions cannot appear in WHERE — always wrap in a CTE first
```

### Session analysis

```sql
-- Group web page views into sessions (30-minute inactivity = new session)
WITH EventBoundaries AS (
    SELECT
        UserID, EventTime,
        CASE WHEN DATEDIFF(minute,
                    LAG(EventTime) OVER (PARTITION BY UserID ORDER BY EventTime),
                    EventTime) > 30
                OR LAG(EventTime) OVER (PARTITION BY UserID ORDER BY EventTime) IS NULL
             THEN 1 ELSE 0 END AS IsNewSession
    FROM WebAnalytics.PageViews
),
SessionIDs AS (
    SELECT UserID, EventTime,
           SUM(IsNewSession) OVER (
               PARTITION BY UserID ORDER BY EventTime
               ROWS UNBOUNDED PRECEDING
           ) AS SessionNo
    FROM EventBoundaries
)
SELECT UserID, SessionNo,
       MIN(EventTime)                                    AS SessionStart,
       MAX(EventTime)                                    AS SessionEnd,
       DATEDIFF(minute, MIN(EventTime), MAX(EventTime))  AS DurationMinutes,
       COUNT(*)                                          AS PageViews
FROM SessionIDs
GROUP BY UserID, SessionNo
ORDER BY UserID, SessionNo;
```

### Status change tracking (SLA reporting)

```sql
-- Time spent in each status per order
WITH StatusPeriods AS (
    SELECT
        OrderID, Status, ChangedAt AS PeriodStart,
        LEAD(ChangedAt) OVER (PARTITION BY OrderID ORDER BY ChangedAt) AS PeriodEnd
    FROM Sales.OrderStatusHistory
)
SELECT
    OrderID, Status, PeriodStart,
    ISNULL(PeriodEnd, SYSUTCDATETIME()) AS PeriodEnd,
    DATEDIFF(minute, PeriodStart,
        ISNULL(PeriodEnd, SYSUTCDATETIME())) AS MinutesInStatus
FROM StatusPeriods
ORDER BY OrderID, PeriodStart;
```

---

## Data Export

### FOR JSON PATH

```sql
-- Flat JSON with dot-notation nesting (no subqueries needed)
SELECT
    o.OrderID       AS "id",
    o.OrderDate     AS "date",
    c.CustomerName  AS "customer.name",
    c.Email         AS "customer.email",
    o.TotalAmount   AS "total"
FROM Sales.Orders o
JOIN Sales.Customers c ON c.CustomerID = o.CustomerID
FOR JSON PATH;
-- Result: [{"id":1001,"date":"2024-06-15","customer":{"name":"Acme","email":"..."},"total":299.99}]

-- Nested array: orders with line items
SELECT
    o.OrderID      AS "orderId",
    o.OrderDate    AS "date",
    (SELECT ol.ProductName AS "product", ol.Qty AS "qty", ol.Price AS "price"
     FROM Sales.OrderLines ol WHERE ol.OrderID = o.OrderID
     FOR JSON PATH) AS "lines"
FROM Sales.Orders o
FOR JSON PATH;

-- Use FOR JSON PATH (not AUTO) in production. AUTO changes shape when aliases change.
```

### STRING_AGG for CSV columns

```sql
-- Comma-separated list per group
SELECT CustomerID,
       STRING_AGG(ProductName, ', ') WITHIN GROUP (ORDER BY ProductName) AS Products
FROM Sales.OrderLines
GROUP BY CustomerID;
```

### BCP export from command line

```bash
bcp "SELECT OrderID, CustomerID, OrderDate, TotalAmount FROM SalesDB.Sales.Orders" \
    queryout orders_export.csv \
    -c -t, -r\n \
    -S YourServer -d SalesDB -T
# -c = character mode (UTF-8), -t = column delimiter, -r = row delimiter, -T = trusted connection
```

---

## Real-World Scenarios

Each scenario is a complete report that combines several techniques from the sections above. They
use a consistent schema: `Sales.Orders` (OrderID, CustomerID, OrderDate, TotalAmount, Status,
Region, ProductID), `Sales.OrderLines` (OrderID, ProductID, ProductName, Qty, UnitPrice, Cost),
`Sales.Customers` (CustomerID, CustomerName, Email, SignupDate), `Production.Products` (ProductID,
ProductName, Category, ListPrice), and `dbo.Calendar`.

### 1. Executive sales dashboard (one query, every KPI)

The single query a leadership deck pulls from monthly: revenue, MoM growth, YoY growth, YTD, and a
3-month trailing average — all per month.

```sql
WITH Monthly AS (
    SELECT
        DATEFROMPARTS(YEAR(OrderDate), MONTH(OrderDate), 1) AS MonthStart,
        SUM(TotalAmount)           AS Revenue,
        COUNT(*)                   AS Orders,
        COUNT(DISTINCT CustomerID) AS ActiveCustomers
    FROM Sales.Orders
    WHERE Status = 'Completed'
      AND OrderDate >= '2023-01-01' AND OrderDate < '2025-01-01'
    GROUP BY DATEFROMPARTS(YEAR(OrderDate), MONTH(OrderDate), 1)
)
SELECT
    MonthStart,
    Revenue,
    Orders,
    ActiveCustomers,
    Revenue / NULLIF(Orders, 0)                                       AS AvgOrderValue,
    -- Month-over-month growth %
    (Revenue - LAG(Revenue) OVER (ORDER BY MonthStart))
        * 100.0 / NULLIF(LAG(Revenue) OVER (ORDER BY MonthStart), 0)  AS MoM_Pct,
    -- Year-over-year growth % (offset 12 months — assumes no missing months)
    (Revenue - LAG(Revenue, 12) OVER (ORDER BY MonthStart))
        * 100.0 / NULLIF(LAG(Revenue, 12) OVER (ORDER BY MonthStart), 0) AS YoY_Pct,
    -- Year-to-date revenue, resets each calendar year
    SUM(Revenue) OVER (
        PARTITION BY YEAR(MonthStart)
        ORDER BY MonthStart ROWS UNBOUNDED PRECEDING
    ) AS YTD_Revenue,
    -- 3-month trailing average smooths seasonality
    AVG(Revenue) OVER (ORDER BY MonthStart ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS MA_3Month
FROM Monthly
ORDER BY MonthStart;
```

Combines: monthly bucketing, `LAG` for MoM/YoY, partitioned running total for YTD, moving average.

### 2. SaaS MRR movement (revenue bridge)

Decompose month-to-month recurring revenue into new, expansion, contraction, and churned MRR. This
is the canonical "waterfall" SaaS finance report. Assumes a `Billing.Subscriptions` snapshot table
(CustomerID, MonthStart, MRR) with one row per active customer per month.

```sql
WITH WithPrev AS (
    SELECT
        CustomerID,
        MonthStart,
        MRR,
        LAG(MRR) OVER (PARTITION BY CustomerID ORDER BY MonthStart) AS PrevMRR
    FROM Billing.Subscriptions
)
SELECT
    MonthStart,
    SUM(CASE WHEN PrevMRR IS NULL              THEN MRR              ELSE 0 END) AS NewMRR,
    SUM(CASE WHEN PrevMRR > 0 AND MRR > PrevMRR THEN MRR - PrevMRR    ELSE 0 END) AS ExpansionMRR,
    SUM(CASE WHEN MRR > 0 AND MRR < PrevMRR     THEN MRR - PrevMRR    ELSE 0 END) AS ContractionMRR,
    -- Churn: a customer present last month with no row this month is handled by the gap-fill join;
    -- here we capture downgrades to zero within the snapshot
    SUM(CASE WHEN PrevMRR > 0 AND MRR = 0       THEN -PrevMRR         ELSE 0 END) AS ChurnedMRR,
    SUM(MRR)                                                                     AS EndingMRR,
    SUM(MRR) - SUM(ISNULL(PrevMRR, 0))                                           AS NetNewMRR
FROM WithPrev
GROUP BY MonthStart
ORDER BY MonthStart;
```

Combines: `LAG` per customer, conditional aggregation to classify each movement type. The four
movement columns plus prior-month total reconcile exactly to `EndingMRR`.

### 3. Customer retention cohort matrix

Group customers by signup month, then show what percent are still active N months later — pivoted
into a triangular retention grid. This is the heatmap product teams live by.

```sql
WITH Cohort AS (
    SELECT
        c.CustomerID,
        DATEFROMPARTS(YEAR(c.SignupDate), MONTH(c.SignupDate), 1) AS CohortMonth
    FROM Sales.Customers c
),
Activity AS (
    SELECT DISTINCT
        ch.CohortMonth,
        ch.CustomerID,
        DATEDIFF(month, ch.CohortMonth,
            DATEFROMPARTS(YEAR(o.OrderDate), MONTH(o.OrderDate), 1)) AS MonthOffset
    FROM Cohort ch
    JOIN Sales.Orders o ON o.CustomerID = ch.CustomerID
),
CohortSize AS (
    SELECT CohortMonth, COUNT(DISTINCT CustomerID) AS Customers
    FROM Cohort GROUP BY CohortMonth
)
SELECT
    a.CohortMonth,
    cs.Customers AS CohortSize,
    CAST(100.0 * COUNT(DISTINCT CASE WHEN MonthOffset = 0 THEN a.CustomerID END) / cs.Customers AS DECIMAL(5,1)) AS M0,
    CAST(100.0 * COUNT(DISTINCT CASE WHEN MonthOffset = 1 THEN a.CustomerID END) / cs.Customers AS DECIMAL(5,1)) AS M1,
    CAST(100.0 * COUNT(DISTINCT CASE WHEN MonthOffset = 2 THEN a.CustomerID END) / cs.Customers AS DECIMAL(5,1)) AS M2,
    CAST(100.0 * COUNT(DISTINCT CASE WHEN MonthOffset = 3 THEN a.CustomerID END) / cs.Customers AS DECIMAL(5,1)) AS M3,
    CAST(100.0 * COUNT(DISTINCT CASE WHEN MonthOffset = 6 THEN a.CustomerID END) / cs.Customers AS DECIMAL(5,1)) AS M6
FROM Activity a
JOIN CohortSize cs ON cs.CohortMonth = a.CohortMonth
GROUP BY a.CohortMonth, cs.Customers
ORDER BY a.CohortMonth;
```

Combines: cohort analysis, conditional aggregation as a manual pivot, percentage-of-cohort math.

### 4. RFM segmentation for marketing

Score every customer on Recency, Frequency, and Monetary value using quintiles, then map the
combined score to a named segment. Drives email targeting and win-back campaigns.

```sql
DECLARE @AsOf DATE = '2024-12-31';

WITH RFM AS (
    SELECT
        CustomerID,
        DATEDIFF(day, MAX(OrderDate), @AsOf) AS Recency,   -- lower is better
        COUNT(*)                             AS Frequency,
        SUM(TotalAmount)                     AS Monetary
    FROM Sales.Orders
    WHERE Status = 'Completed' AND OrderDate <= @AsOf
    GROUP BY CustomerID
),
Scored AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY Recency DESC)   AS R,  -- DESC so recent buyers score 5
        NTILE(5) OVER (ORDER BY Frequency ASC)  AS F,
        NTILE(5) OVER (ORDER BY Monetary ASC)   AS M
    FROM RFM
)
SELECT
    CustomerID, Recency, Frequency, Monetary,
    R, F, M,
    CONCAT(R, F, M) AS RFM_Cell,
    CASE
        WHEN R >= 4 AND F >= 4 AND M >= 4 THEN 'Champions'
        WHEN R >= 4 AND F >= 3            THEN 'Loyal'
        WHEN R >= 4 AND F <= 2            THEN 'New / Promising'
        WHEN R = 3                        THEN 'Needs Attention'
        WHEN R <= 2 AND F >= 4            THEN 'At Risk (high value)'
        WHEN R <= 2 AND M >= 4            THEN 'Cannot Lose Them'
        ELSE 'Hibernating'
    END AS Segment
FROM Scored
ORDER BY M DESC, F DESC, R DESC;
```

Combines: aggregation to RFM metrics, `NTILE` quintile scoring (note the `DESC` on Recency), CASE
segmentation. Quintile cutoffs adapt automatically as the customer base shifts.

### 5. Pareto (ABC) analysis — the 80/20 cut

Rank products by revenue and find the cumulative contribution, then classify into A/B/C tiers.
Inventory and category managers use this to focus on the vital few SKUs.

```sql
WITH ProductRevenue AS (
    SELECT
        ol.ProductID,
        p.ProductName,
        SUM(ol.Qty * ol.UnitPrice) AS Revenue
    FROM Sales.OrderLines ol
    JOIN Production.Products p ON p.ProductID = ol.ProductID
    GROUP BY ol.ProductID, p.ProductName
),
Ranked AS (
    SELECT *,
        SUM(Revenue) OVER (ORDER BY Revenue DESC ROWS UNBOUNDED PRECEDING) AS CumRevenue,
        SUM(Revenue) OVER ()                                               AS TotalRevenue
    FROM ProductRevenue
)
SELECT
    ProductID, ProductName, Revenue,
    CAST(100.0 * Revenue    / TotalRevenue AS DECIMAL(5,2)) AS PctOfTotal,
    CAST(100.0 * CumRevenue / TotalRevenue AS DECIMAL(5,2)) AS CumulativePct,
    CASE
        WHEN 100.0 * CumRevenue / TotalRevenue <= 80 THEN 'A'   -- top 80% of revenue
        WHEN 100.0 * CumRevenue / TotalRevenue <= 95 THEN 'B'   -- next 15%
        ELSE 'C'                                                -- last 5%
    END AS ABC_Class
FROM Ranked
ORDER BY Revenue DESC;
```

Combines: running total via window frame, grand total via empty `OVER ()`, threshold classification.

### 6. Daily revenue anomaly detection

Flag days where revenue deviates more than 2 standard deviations from a trailing 28-day baseline.
Powers "something looks wrong" alerts without a separate stats tool.

```sql
WITH Daily AS (
    SELECT C.FullDate,
           ISNULL(SUM(O.TotalAmount), 0) AS Revenue
    FROM dbo.Calendar C
    LEFT JOIN Sales.Orders O
        ON O.OrderDate >= C.FullDate
       AND O.OrderDate <  DATEADD(day, 1, C.FullDate)
       AND O.Status = 'Completed'
    WHERE C.FullDate >= '2024-01-01' AND C.FullDate < '2025-01-01'
    GROUP BY C.FullDate
),
Baseline AS (
    SELECT
        FullDate, Revenue,
        AVG(Revenue)   OVER (ORDER BY FullDate ROWS BETWEEN 28 PRECEDING AND 1 PRECEDING) AS TrailingAvg,
        STDEV(Revenue) OVER (ORDER BY FullDate ROWS BETWEEN 28 PRECEDING AND 1 PRECEDING) AS TrailingStdev
    FROM Daily
)
SELECT
    FullDate, Revenue,
    CAST(TrailingAvg AS DECIMAL(12,2))   AS TrailingAvg,
    CAST(TrailingStdev AS DECIMAL(12,2)) AS TrailingStdev,
    CAST((Revenue - TrailingAvg) / NULLIF(TrailingStdev, 0) AS DECIMAL(6,2)) AS ZScore,
    CASE
        WHEN ABS((Revenue - TrailingAvg) / NULLIF(TrailingStdev, 0)) > 2 THEN 'ANOMALY'
        ELSE 'normal'
    END AS Flag
FROM Baseline
WHERE TrailingStdev IS NOT NULL          -- skip the first 28 warm-up days
ORDER BY FullDate;
```

Combines: gap-fill from the calendar table, trailing window that ends at `1 PRECEDING` (excludes the
current day from its own baseline), z-score math, threshold flagging.

### 7. Inventory reorder report

Combine recent demand velocity with current stock to decide what to reorder. Assumes
`Inventory.Stock` (ProductID, QtyOnHand, LeadTimeDays, ReorderPoint).

```sql
DECLARE @AsOf DATE = '2024-12-31';

WITH Demand AS (
    SELECT
        ol.ProductID,
        SUM(ol.Qty) * 1.0 / 90 AS DailyDemand     -- avg units/day over last 90 days
    FROM Sales.OrderLines ol
    JOIN Sales.Orders o ON o.OrderID = ol.OrderID
    WHERE o.OrderDate >= DATEADD(day, -90, @AsOf) AND o.OrderDate < @AsOf
    GROUP BY ol.ProductID
)
SELECT
    p.ProductID, p.ProductName, p.Category,
    s.QtyOnHand,
    s.LeadTimeDays,
    CAST(d.DailyDemand AS DECIMAL(10,2))                       AS DailyDemand,
    CAST(d.DailyDemand * s.LeadTimeDays AS DECIMAL(10,2))      AS LeadTimeDemand,
    CAST(s.QtyOnHand / NULLIF(d.DailyDemand, 0) AS DECIMAL(10,1)) AS DaysOfCover,
    CASE
        WHEN s.QtyOnHand <= d.DailyDemand * s.LeadTimeDays THEN 'REORDER NOW'
        WHEN s.QtyOnHand <= s.ReorderPoint                 THEN 'Approaching reorder point'
        ELSE 'OK'
    END AS Action
FROM Production.Products p
JOIN Inventory.Stock s ON s.ProductID = p.ProductID
LEFT JOIN Demand     d ON d.ProductID = p.ProductID
ORDER BY DaysOfCover ASC;
```

Combines: windowed demand aggregation, safe division for days-of-cover, lead-time demand vs stock
comparison for the reorder decision.

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Default window frame with ORDER BY (`RANGE`, not `ROWS`) | Always specify `ROWS BETWEEN ...` explicitly |
| Moving average with `RANGE` (includes ties) | Use `ROWS BETWEEN N PRECEDING AND CURRENT ROW` |
| `YEAR(col) = 2024` in WHERE | Range predicate: `col >= '2024-01-01' AND col < '2025-01-01'` |
| `CAST(col AS DATE)` in JOIN condition | Use range join: `ON col >= date AND col < DATEADD(day, 1, date)` |
| Tally CTE without `WHERE N <= limit` | Always add the limit — without it generates all 65K rows |
| `COUNT(column)` when you want total rows | `COUNT(*)` includes NULLs; `COUNT(col)` excludes them |
| `AVG(col)` when NULLs mean zero | `AVG(COALESCE(col, 0))` includes NULLs as zero |
| `UNPIVOT` operator in production | Use `CROSS APPLY VALUES` — UNPIVOT silently drops NULLs |
| `NOT IN` with nullable subquery | Use `NOT EXISTS` — NOT IN returns nothing if subquery has NULLs |
| Recursive CTE for number generation | Use the stacking CTE — set-based and 10-50x faster |
| `FOR JSON AUTO` in production | Use `FOR JSON PATH` — AUTO changes shape when aliases change |
| `HAVING` for non-aggregate filters | Move to `WHERE` — it filters before grouping |
| `FORMAT` for date truncation | FORMAT is CLR-backed (10-50x slower). Use `DATETRUNC` (2022+) or `DATEADD/DATEDIFF` |
| `LAST_VALUE` without explicit frame | Default frame ends at `CURRENT ROW`. Use `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` |
| `LAG` with fixed offset on sparse data | If months are missing, LAG(val, 12) reaches the wrong year. Use a self-join |
| Filtering on window function result in WHERE | Wrap in a CTE first — window functions cannot appear in WHERE |
| GROUPING SETS NULL vs real NULL | Use `GROUPING(col)` to distinguish subtotal placeholders from real NULLs |
