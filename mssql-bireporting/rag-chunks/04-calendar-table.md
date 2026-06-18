---
topic: Calendar table — design, seeding, business day calculations, gap filling
keywords: [calendar table, date dimension, DateKey, FiscalYear, IsWeekend, IsHoliday, business days, working days, GENERATE_SERIES, gap filling, LEFT JOIN]
use_when: Creating or querying a calendar table, calculating business days between dates, or filling gaps in a date-based time series.
---

# Calendar Table

```sql
-- Calendar table design
CREATE TABLE dbo.Calendar (
    DateKey        DATE         NOT NULL PRIMARY KEY,
    DayOfWeek      TINYINT      NOT NULL,  -- 1=Monday through 7=Sunday (ISO)
    WeekNo         TINYINT      NOT NULL,
    MonthNo        TINYINT      NOT NULL,
    QuarterNo      TINYINT      NOT NULL,
    Year           SMALLINT     NOT NULL,
    FiscalYear     SMALLINT     NOT NULL,  -- July-start fiscal year
    IsWeekend      BIT          NOT NULL,
    IsHoliday      BIT          NOT NULL DEFAULT 0,
    MonthStart     DATE         NOT NULL,
    QuarterStart   DATE         NOT NULL,
    FiscalYearStart DATE        NOT NULL
);

-- Seed via stacking CTE (no GENERATE_SERIES needed pre-2022)
WITH N1  AS (SELECT 1 AS n UNION ALL SELECT 1),
     N2  AS (SELECT 1 AS n FROM N1 a, N1 b),
     N4  AS (SELECT 1 AS n FROM N2 a, N2 b),
     N8  AS (SELECT 1 AS n FROM N4 a, N4 b),
     Nums AS (SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n FROM N8 a, N4 b)
INSERT INTO dbo.Calendar (DateKey, DayOfWeek, WeekNo, MonthNo, QuarterNo, Year,
                          FiscalYear, IsWeekend, MonthStart, QuarterStart, FiscalYearStart)
SELECT
    d,
    -- ISO weekday: Mon=1 ... Sun=7
    ((DATEPART(weekday, d) + @@DATEFIRST - 2) % 7) + 1 AS DayOfWeek,
    DATEPART(iso_week, d),
    MONTH(d), DATEPART(quarter, d), YEAR(d),
    CASE WHEN MONTH(d) >= 7 THEN YEAR(d) ELSE YEAR(d)-1 END,
    CASE WHEN DATEPART(weekday,d) IN (1,7) THEN 1 ELSE 0 END,
    DATEADD(month, DATEDIFF(month,0,d), 0),
    DATEADD(quarter, DATEDIFF(quarter,0,d), 0),
    DATEFROMPARTS(CASE WHEN MONTH(d)>=7 THEN YEAR(d) ELSE YEAR(d)-1 END, 7, 1)
FROM (SELECT DATEADD(day, n, '2000-01-01') AS d FROM Nums WHERE n <= 36524) dates;

-- Business days between two dates
SELECT COUNT(*) AS BusinessDays
FROM dbo.Calendar
WHERE DateKey BETWEEN @Start AND @End
  AND IsWeekend = 0 AND IsHoliday = 0;

-- Gap filling: show all dates even when no orders
SELECT c.DateKey, ISNULL(SUM(o.TotalAmount), 0) AS Revenue
FROM dbo.Calendar c
LEFT JOIN Sales.Orders o ON CAST(o.OrderDate AS DATE) = c.DateKey
    AND o.OrderDate >= @Start AND o.OrderDate < @End  -- SARGable
WHERE c.DateKey BETWEEN @Start AND @End
GROUP BY c.DateKey
ORDER BY c.DateKey;
```
