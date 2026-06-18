---
topic: Common SQL Server performance mistakes and their fixes
keywords: [common mistakes, anti-patterns, NOLOCK, key lookup, fill factor, parameter sniffing, large delete, spills]
use_when: You want a quick checklist of frequent performance anti-patterns and their corrections.
---

# Common Mistakes → Fixes

| Mistake | Fix |
|---|---|
| Non-SARGable WHERE clause | Never wrap the filtered column in a function — apply functions to parameters |
| Key Lookup on high-row-count query | Add needed SELECT columns to the NCI INCLUDE list |
| Using NOLOCK for "read performance" | Enable RCSI instead — consistent reads, no shared lock overhead |
| Fixing sniffing with a local variable copy | Use `OPTIMIZE FOR UNKNOWN` or statement-level `OPTION (RECOMPILE)` |
| Ignoring fill factor on random-key tables | Set 70–80% fill factor; rebuild on schedule |
| Blindly creating missing-index suggestions | Evaluate write penalty and overlap with existing indexes |
| Rebuilding all indexes regardless of fragmentation | Skip tables < 1,000 pages; skip indexes < 5% fragmented |
| Auto-update statistics never firing on large tables | Upgrade to compat 130+ for the dynamic threshold (fires ~20× sooner) |
| Table variable cardinality fixed at 1 row | Upgrade to compat 150 for deferred compilation |
| Hash/sort spills to tempdb | Fix statistics so the memory grant is sized correctly; row-mode MGF (2019+) auto-adjusts |
| NOLOCK as a deadlock fix | Deadlocks need lock ordering or RCSI — NOLOCK does not prevent them |
| Single large DELETE bloating the log | Chunk with `DELETE TOP (5000)` in a WHILE loop |
