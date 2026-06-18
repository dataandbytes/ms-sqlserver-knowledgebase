---
topic: Execution plan operators (scan/seek, key lookup, joins, sort, spool, parallelism)
keywords: [scan, seek, key lookup, nested loops, hash match, merge join, sort, spool, CXPACKET, spill]
use_when: You are interpreting operators in an execution plan or deciding which is the bottleneck.
---

# Key Execution Plan Operators

**Scan vs Seek** — a seek traverses the B-tree to a key range (O(log n)); a scan reads all leaf pages (O(n)).

| Operator | Good or bad? | Fix when bad |
|---|---|---|
| Clustered/Nonclustered Index Seek | Good | Check for a Key Lookup following an NCI seek |
| Clustered Index Scan | Investigate | Non-SARGable predicate or intentional large read |
| Table Scan | Always investigate | Missing clustered index |
| Nonclustered Index Scan | Investigate | Non-selective predicate or index intersection |

**Key Lookup** — the NCI satisfied the seek but lacked SELECT columns; SQL Server navigates the clustered B-tree (2–4 page reads) per row. At 1,000+ rows this dominates. Fix: covering index with INCLUDE columns. The optimizer switches NCI seek+lookup → clustered scan once ~0.5–2% of the table is expected.

**Joins:**

| Algorithm | Best when | Memory grant |
|---|---|---|
| Nested Loops | Small outer, large inner with index seek | None |
| Hash Match | Large unsorted inputs, no useful index | Yes — can spill |
| Merge Join | Both inputs already sorted | None if sorted |

Wrong join algorithm is a classic symptom of cardinality estimation failure — underestimated rows push the optimizer to Nested Loops where Hash Match would win. **Hash Match spills to tempdb** (yellow triangle) when the grant is too small; fix the statistics, not the join.

**Sort** is blocking (consumes all input first) and needs a memory grant — spills if too small. **Index Spool** = a temp index was built because a permanent one is missing (create it). **Table Spool** = intermediate results cached in tempdb (correlated subqueries, Halloween problem) — rewrite with a CTE or temp table.

**Parallelism** — setup (~50ms) makes it net-negative under 100ms; force serial with `OPTION (MAXDOP 1)`. **CXPACKET** = threads waiting for the slowest sibling (skewed work distribution).

Plans read **right-to-left, top-to-bottom**; arrow thickness scales with estimated row count.
