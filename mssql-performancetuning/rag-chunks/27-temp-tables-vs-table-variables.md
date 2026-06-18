---
topic: Temp tables vs table variables vs CTEs
keywords: [temp table, table variable, CTE, statistics, deferred compilation, rollback, intermediate results, #t @t]
use_when: You are choosing between a temp table, a table variable, or a CTE for an intermediate result.
---

# Temp Tables vs Table Variables vs CTEs

| Dimension | Temp Table (`#t`) | Table Variable (`@t`) | CTE |
|---|---|---|---|
| Statistics | Yes — auto-created | No (1 row pre-2019; deferred 2019+) | None |
| Index support | All types | PRIMARY KEY, UNIQUE only | No |
| Recompile trigger | On schema/stats change | Rarely | No |
| Survives ROLLBACK | No — dropped | Yes — contents persist | No |
| Scope | Session + called procs | Current batch only | Current statement |

**Decision rules:**
- **Temp table** when rows exceed a few hundred, you need an index on the intermediate result, or you need accurate cardinality for downstream joins.
- **Table variable** when the set is small (1–100 rows), you want to avoid recompile triggers, or the data must survive a ROLLBACK (e.g. an audit log inserted before work begins).
- **CTE** when the result is referenced exactly once — CTEs are not materialized; each reference re-executes the subquery (a CTE referenced twice scans the source twice).

**SQL Server 2019 deferred compilation** (compat 150+): the optimizer defers compiling table-variable statements until after population, using the actual row count — largely eliminating the old "1 row" estimate. Temp tables still win for column-level statistics and indexes on non-key columns.

```sql
SELECT name, value FROM sys.database_scoped_configurations WHERE name = 'DEFERRED_COMPILATION_TV';
```
