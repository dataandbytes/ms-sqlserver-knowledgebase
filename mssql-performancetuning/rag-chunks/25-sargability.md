---
topic: SARGability — writing predicates that allow index seeks
keywords: [SARGable, index seek, function on column, implicit conversion, YEAR, DATEDIFF, LIKE wildcard, non-sargable]
use_when: A query forces a scan because of how the WHERE clause is written.
---

# SARGability

A predicate is **SARGable** when the optimizer can use an index seek instead of scanning every row. The rule: never wrap the filtered column in a function, expression, or implicit conversion — apply transformations to the parameter side instead.

| Non-SARGable (forces scan) | SARGable alternative |
|---|---|
| `WHERE YEAR(OrderDate) = 2025` | `WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01'` |
| `WHERE ISNULL(col, '') = @Val` | Make column NOT NULL; `WHERE col = @Val` |
| `WHERE col LIKE '%search'` | Trailing wildcard: `WHERE col LIKE 'search%'` |
| `WHERE DATEDIFF(day, col, GETDATE()) < 30` | `WHERE col > DATEADD(day, -30, GETDATE())` |
| `WHERE CAST(col AS VARCHAR) = '42'` | Fix parameter type to match the column |
| `WHERE LEFT(LastName, 3) = 'Smi'` | `WHERE LastName LIKE 'Smi%'` |
| `WHERE col + 0 = @Val` | `WHERE col = @Val` |

An implicit type conversion has the same effect as a function — it converts every row. Match parameter types exactly to column types. Look for `PlanAffectingConvert` warnings in the execution plan XML.
