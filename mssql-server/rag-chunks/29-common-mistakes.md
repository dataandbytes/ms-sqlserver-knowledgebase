---
topic: Common T-SQL mistakes and their fixes
keywords: [common mistakes, SARGable, implicit conversion, NOLOCK, scalar UDF, identity, window function, tally table, key lookup, fragmentation, stats, batch delete, MERGE, FOR JSON, dynamic SQL, temporal]
use_when: Quick reference for known T-SQL pitfalls when reviewing or debugging code.
---

# Common Mistakes

| Mistake | Fix |
|---|---|
| Non-SARGable WHERE (`YEAR(col) = 2024`) | Range predicate: `col >= '2024-01-01' AND col < '2025-01-01'` |
| Implicit type conversion in WHERE (column INT, param VARCHAR) | Match parameter type to column type exactly |
| `NOLOCK` everywhere "for performance" | Enable RCSI : consistent reads without shared lock overhead |
| CTE referenced twice (scans source twice) | Materialize into a `#temp` table |
| Scalar UDF in WHERE clause (prevents parallelism) | Replace with inline TVF + CROSS APPLY |
| `IDENTITY` for composite hierarchy child keys | Use max-plus-one functions scoped to parent key |
| Missing `PARTITION BY` in window functions on multi-entity data | Always `PARTITION BY entity_id` |
| Recursive CTE for number sequences | Use stacking CTE or `GENERATE_SERIES` (2022+) : 10-50x faster |
| Forgotten `WHERE N <= limit` in tally CTE | Always add the limit : without it materializes 65K rows |
| Key Lookup on high-row-count seek | Add SELECT columns to NCI `INCLUDE` list |
| Index fragmentation always triggering REBUILD | Skip indexes < 5% fragmented or < 1,000 pages |
| Stats staleness on large tables (compat < 130) | Upgrade to compat 130+ for dynamic auto-update threshold |
| Single large DELETE bloating the log | Chunk with `DELETE TOP (5000)` in a WHILE loop |
| MERGE without OUTPUT makes errors hard to diagnose | Add `OUTPUT $action, INSERTED.*, DELETED.*` |
| `FOR JSON AUTO` in production | Use `FOR JSON PATH` : AUTO breaks when aliases change |
| Dynamic SQL with string concatenation of user input | Always use `sp_executesql` with parameters |
| Temporal table queries with local time | `FOR SYSTEM_TIME` times are UTC : convert with `AT TIME ZONE` |
| Trace flags 1117/1118 on SQL Server 2016+ | Remove them : they are no-ops and add confusion |
