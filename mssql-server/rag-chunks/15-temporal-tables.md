---
topic: Temporal tables : system-versioned, point-in-time queries, retention policy
keywords: [temporal table, system-versioned, FOR SYSTEM_TIME, AS OF, BETWEEN, history table, ValidFrom, ValidTo, retention policy]
use_when: Creating a table that tracks row history automatically, querying data as it was at a past point in time, or setting history retention.
---

# Temporal Tables

```sql
-- Create system-versioned temporal table
CREATE TABLE HR.Employee (
    EmployeeID   INT           NOT NULL PRIMARY KEY,
    Name         NVARCHAR(200) NOT NULL,
    DepartmentID INT           NOT NULL,
    Salary       DECIMAL(12,2) NOT NULL,
    ValidFrom    DATETIME2     GENERATED ALWAYS AS ROW START NOT NULL,
    ValidTo      DATETIME2     GENERATED ALWAYS AS ROW END   NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = HR.Employee_History));

-- Point-in-time query : what did the row look like at this exact moment?
SELECT EmployeeID, Name, Salary
FROM HR.Employee
FOR SYSTEM_TIME AS OF '2023-06-01T00:00:00';   -- times are UTC

-- All versions in a date range
SELECT EmployeeID, Name, Salary, ValidFrom, ValidTo
FROM HR.Employee
FOR SYSTEM_TIME BETWEEN '2022-01-01' AND '2024-01-01'
WHERE EmployeeID = 42
ORDER BY ValidFrom;

-- Complete history including current row
SELECT * FROM HR.Employee FOR SYSTEM_TIME ALL
WHERE EmployeeID = 42 ORDER BY ValidFrom;

-- Auto-purge history older than 2 years
ALTER TABLE HR.Employee SET (SYSTEM_VERSIONING = ON (
    HISTORY_RETENTION_PERIOD = 2 YEARS
));
```

**Key rules:**
- `ValidFrom` and `ValidTo` are maintained by the engine : you cannot set them directly.
- All `FOR SYSTEM_TIME` timestamps are **UTC**. Convert with `AT TIME ZONE` if your application stores local time.
- The history table has no clustered index by default : add one on `(EmployeeID, ValidFrom)` for efficient range queries.
- Retention policy requires the database compatibility level ≥ 130 and `TEMPORAL_HISTORY_RETENTION = ON` at the database level.
- `UPDATE` on a temporal table writes the old row to the history table and updates the current row. `DELETE` moves the row to history and removes it from the current table.
