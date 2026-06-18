---
topic: Temporal features — recency, frequency windows, tenure, calendar features, @SnapshotDate
keywords: [temporal features, recency, frequency, tenure, 30-day window, 90-day window, @SnapshotDate, DayOfWeek, @@DATEFIRST, calendar features]
use_when: Engineering time-based features (recency, frequency, tenure, day-of-week, month) for ML models.
---

# Temporal Features

All temporal features must reference a fixed `@SnapshotDate`, never `GETDATE()`.

```sql
DECLARE @SnapshotDate DATE = '2024-06-01';

SELECT CustomerID,
    -- Recency: days since last purchase
    DATEDIFF(day, MAX(OrderDate), @SnapshotDate) AS DaysSinceLastOrder,

    -- Frequency windows: orders in last 30 and 90 days
    SUM(CASE WHEN OrderDate >= DATEADD(day, -30, @SnapshotDate) THEN 1 ELSE 0 END) AS Orders_30d,
    SUM(CASE WHEN OrderDate >= DATEADD(day, -90, @SnapshotDate) THEN 1 ELSE 0 END) AS Orders_90d,

    -- Tenure: days since first purchase
    DATEDIFF(day, MIN(OrderDate), @SnapshotDate) AS TenureDays,

    -- Revenue windows
    SUM(CASE WHEN OrderDate >= DATEADD(day, -30, @SnapshotDate) THEN TotalAmount ELSE 0 END) AS Revenue_30d,
    SUM(CASE WHEN OrderDate >= DATEADD(day, -90, @SnapshotDate) THEN TotalAmount ELSE 0 END) AS Revenue_90d

FROM Sales.Orders
WHERE OrderDate < @SnapshotDate
GROUP BY CustomerID;

-- Calendar features (from the order date itself)
SELECT OrderID,
    DATEPART(weekday, OrderDate)                  AS DayOfWeek_Raw,     -- affected by @@DATEFIRST
    ((DATEPART(weekday,OrderDate)+@@DATEFIRST-2)%7)+1 AS DayOfWeek_ISO,  -- 1=Mon, 7=Sun always
    MONTH(OrderDate)                              AS OrderMonth,
    DATEPART(quarter, OrderDate)                  AS OrderQuarter,
    CASE WHEN DATEPART(weekday,OrderDate) IN (1,7) THEN 1 ELSE 0 END AS IsWeekend
FROM Sales.Orders;
```

**Key rules:**
- Always use the ISO weekday formula `((DATEPART(weekday,d) + @@DATEFIRST - 2) % 7) + 1` — `DATEPART(weekday)` is offset by `@@DATEFIRST` (server setting, may differ between environments).
- Frequency window features (`Orders_30d`, `Orders_90d`) compare `OrderDate >= DATEADD(day, -N, @SnapshotDate)` — this is inclusive of the window start date. Verify whether you want `>` or `>=` per business definition.
- Do not include orders on or after `@SnapshotDate` — they are future data relative to the feature point.
