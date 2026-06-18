---
topic: Covering indexes, INCLUDE columns, and composite key ordering
keywords: [covering index, INCLUDE, key lookup, composite key, column order, equality range order by]
use_when: A query has a Key Lookup, or you are deciding column order for a multi-column index.
---

# Covering Indexes + Composite Key Ordering

A covering index satisfies a query entirely from the index — no Key Lookup back to the clustered index. Each Key Lookup traverses the clustered B-tree (2–4 page reads) per row; at 1,000 rows that is thousands of extra logical reads, scaling linearly.

**Rule:** put a column in the index *key* only if it appears in WHERE / JOIN / ORDER BY. Everything else goes in INCLUDE.

```sql
-- Before: Key Lookup for OrderDate, Status, TotalAmt
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID ON dbo.Orders (CustomerID);
-- After: covering, no Key Lookup
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID_Covering
    ON dbo.Orders (CustomerID) INCLUDE (OrderDate, Status, TotalAmt);
```

**INCLUDE limits:** key columns have a 900-byte / 16-column limit; INCLUDE columns are leaf-only, don't count toward the key limit, and hold most data types (total leaf row limit 1,700 bytes).

**Composite key ordering:** equality predicates first, range predicates last, ORDER BY columns after equality columns (free sort). A range column in the middle of the key blocks seeks on all subsequent columns.

```sql
-- WHERE CustomerID = @cid AND Status = @status AND OrderDate > @since
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Status_Date
    ON dbo.Orders (CustomerID, Status, OrderDate) INCLUDE (TotalAmt);
```

Detect Key Lookups in the plan cache:

```sql
SELECT TOP 20 qs.total_logical_reads / qs.execution_count AS avg_reads, qs.execution_count,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE CAST(qp.query_plan AS NVARCHAR(MAX)) LIKE '%KeyLookup%'
ORDER BY avg_reads DESC;
```
