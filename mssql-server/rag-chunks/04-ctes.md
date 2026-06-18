---
topic: CTEs : non-recursive and recursive common table expressions
keywords: [CTE, WITH, recursive CTE, MAXRECURSION, hierarchy, org chart, BOM, anchor, union all]
use_when: Structuring multi-step queries, traversing hierarchies (org chart, category tree, BOM), or breaking up complex joins.
---

# CTEs

```sql
-- Non-recursive: readable multi-step query
WITH ActiveProducts AS (
    SELECT ProductID, Name, Price, CategoryID
    FROM Production.Product WHERE IsActive = 1
),
CategoryTotals AS (
    SELECT p.CategoryID, c.Name AS Category,
           COUNT(*) AS ProductCount, SUM(p.Price) AS TotalValue
    FROM ActiveProducts p
    JOIN Production.Category c ON c.CategoryID = p.CategoryID
    GROUP BY p.CategoryID, c.Name
)
SELECT * FROM CategoryTotals WHERE ProductCount > 5 ORDER BY TotalValue DESC;

-- Recursive CTE: hierarchy traversal
WITH OrgHierarchy AS (
    -- Anchor: root nodes
    SELECT EmployeeID, Name, ManagerID, 1 AS Level,
           CAST(Name AS NVARCHAR(500)) AS Path
    FROM HR.Employee WHERE ManagerID IS NULL

    UNION ALL

    -- Recursive member: children of rows already in the CTE
    SELECT e.EmployeeID, e.Name, e.ManagerID, h.Level + 1,
           CAST(h.Path + ' > ' + e.Name AS NVARCHAR(500))
    FROM HR.Employee e
    JOIN OrgHierarchy h ON h.EmployeeID = e.ManagerID
)
SELECT * FROM OrgHierarchy ORDER BY Path
OPTION (MAXRECURSION 100);  -- default 100; 0 = unlimited (use carefully)
```

**Key rules:**
- A CTE referenced more than once in the same query is evaluated twice (no materialization). If that is expensive, load into a `#temp` table instead.
- The recursive member must reference the CTE exactly once (in the FROM clause), must use `UNION ALL` (not `UNION`), and cannot use aggregates, GROUP BY, DISTINCT, TOP, or outer joins against the CTE.
- `MAXRECURSION 0` removes the limit but risks infinite loops on circular data : add a cycle-detection column (e.g., `CAST(EmployeeID AS VARCHAR) + ','` path check) for untrusted data.
