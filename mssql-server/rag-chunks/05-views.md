---
topic: Views : standard, indexed (materialized), and WITH CHECK OPTION
keywords: [CREATE VIEW, indexed view, materialized view, SCHEMABINDING, COUNT_BIG, WITH CHECK OPTION, NOEXPAND]
use_when: Creating a view for read access, materializing an aggregate on disk as an indexed view, or preventing bad inserts through a view.
---

# Views

```sql
-- Standard view: join flatten + column subset
CREATE OR ALTER VIEW Sales.CustomerOrderSummary AS
SELECT c.CustomerID, c.Name, c.Email,
       COUNT(o.OrderID)    AS OrderCount,
       SUM(o.TotalAmount)  AS TotalSpend,
       MAX(o.OrderDate)    AS LastOrderDate
FROM Sales.Customers c
LEFT JOIN Sales.Orders o ON o.CustomerID = c.CustomerID
GROUP BY c.CustomerID, c.Name, c.Email;

-- Indexed (materialized) view : results stored on disk, auto-maintained
CREATE OR ALTER VIEW Sales.DailySalesSummary
WITH SCHEMABINDING AS
SELECT CAST(OrderDate AS DATE) AS OrderDay,
       COUNT_BIG(*) AS OrderCount,
       SUM(TotalAmount) AS Revenue
FROM Sales.Orders
GROUP BY CAST(OrderDate AS DATE);

CREATE UNIQUE CLUSTERED INDEX IX_DailySalesSummary
    ON Sales.DailySalesSummary (OrderDay);

-- WITH CHECK OPTION: rejects inserts/updates that would violate the view's WHERE
CREATE OR ALTER VIEW Sales.ActiveOrders AS
SELECT * FROM Sales.Orders WHERE Status <> 'Cancelled'
WITH CHECK OPTION;
```

**Indexed view requirements:**
- `WITH SCHEMABINDING` is mandatory.
- Aggregate views must include `COUNT_BIG(*)`.
- All table references must use two-part names (`schema.Table`).
- No outer joins, subqueries, DISTINCT, non-deterministic functions, or `SELECT *`.
- On Standard/Developer editions add `WITH (NOEXPAND)` hint to force use of the index; Enterprise auto-matches it.
