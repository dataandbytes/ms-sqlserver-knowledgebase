---
topic: Filtered indexes
keywords: [filtered index, partial index, WHERE predicate index, unique nullable, sparse column, OPTION RECOMPILE]
use_when: You want to index a subset of rows (active/pending rows, non-NULL values) or a partial unique constraint.
---

# Filtered Indexes

A filtered index covers only rows matching a WHERE predicate — smaller, cheaper to maintain, more selective per byte than a full-table NCI.

```sql
-- Index only active orders
CREATE NONCLUSTERED INDEX IX_Orders_Active
    ON dbo.Orders (CustomerID, OrderDate) INCLUDE (Status) WHERE Status = 1;

-- Partial unique constraint: multiple NULLs allowed, no duplicate non-NULL values
CREATE UNIQUE NONCLUSTERED INDEX UX_Orders_ExternalRef
    ON dbo.Orders (ExternalRef) WHERE ExternalRef IS NOT NULL;

-- Sparse error column: only non-NULL errors need indexing
CREATE NONCLUSTERED INDEX IX_EventLog_ErrorCode
    ON dbo.EventLog (ErrorCode) WHERE ErrorCode IS NOT NULL;
```

For the optimizer to use a filtered index:
- The query WHERE clause must include the filter predicate (or a logically stronger one).
- Parameterized queries often need `OPTION (RECOMPILE)` so the value is embedded — the optimizer cannot prove `WHERE Status = @status` always equals the filter at compile time.
- The filter must be a simple comparison, `IS NULL`, or `IS NOT NULL` (no functions).
