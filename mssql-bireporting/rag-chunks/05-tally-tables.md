---
topic: Tally tables — stacking CTE pattern, GENERATE_SERIES (2022+), inline number and date sequences
keywords: [tally table, number table, stacking CTE, GENERATE_SERIES, inline sequence, date spine, ROW_NUMBER]
use_when: Generating a sequence of numbers or dates without a stored table, splitting strings, or generating a date spine for gap filling.
---

# Tally Tables

```sql
-- Stacking CTE: generates 2^n rows, then trims to needed count
-- MANDATORY: always include WHERE N <= @Count — without it materializes 65,536 rows
WITH
  N1  AS (SELECT 1 AS n UNION ALL SELECT 1),    --    2
  N2  AS (SELECT 1 AS n FROM N1 a, N1 b),       --    4
  N4  AS (SELECT 1 AS n FROM N2 a, N2 b),       --   16
  N8  AS (SELECT 1 AS n FROM N4 a, N4 b),       --  256
  N16 AS (SELECT 1 AS n FROM N8 a, N8 b),       -- 65536
  Nums AS (SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS N FROM N16)
SELECT N FROM Nums WHERE N <= @Count;

-- Date spine from stacking CTE
DECLARE @Start DATE = '2024-01-01', @End DATE = '2024-12-31';
DECLARE @Days INT = DATEDIFF(day, @Start, @End) + 1;

WITH
  N1 AS (SELECT 1 AS n UNION ALL SELECT 1),
  N2 AS (SELECT 1 AS n FROM N1 a, N1 b),
  N4 AS (SELECT 1 AS n FROM N2 a, N2 b),
  N8 AS (SELECT 1 AS n FROM N4 a, N4 b),
  Nums AS (SELECT ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS N FROM N8 a, N4 b)
SELECT DATEADD(day, N, @Start) AS D
FROM Nums WHERE N <= @Days - 1;

-- GENERATE_SERIES (SQL Server 2022+): simpler, same result
SELECT value AS N FROM GENERATE_SERIES(1, 100);
SELECT DATEADD(day, value, '2024-01-01') AS D
FROM GENERATE_SERIES(0, DATEDIFF(day, '2024-01-01', '2024-12-31'));
```

**Key rules:**
- Always include `WHERE N <= @Count` in the stacking CTE. Without it the full 65,536 rows are materialized before DISTINCT/JOIN filters apply.
- The stacking CTE is a cross-join chain: each level squares the previous. `N1 × N1 = 4`, `N2 × N2 = 16`, etc.
- `GENERATE_SERIES` (2022+) is cleaner and slightly more efficient — use it when available.
- For gap filling, join the tally/date spine LEFT JOIN to the fact table. Never store a tally table permanently — generate it inline.
