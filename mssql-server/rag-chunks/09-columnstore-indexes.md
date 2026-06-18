---
topic: Columnstore indexes : clustered and nonclustered, delta store, rowgroup health
keywords: [columnstore, clustered columnstore, nonclustered columnstore, CCI, NCCI, delta store, batch mode, rowgroup, segment elimination, compression]
use_when: Adding analytics performance to a DW or OLTP table, checking rowgroup health, or forcing delta store compression.
---

# Columnstore Indexes

```sql
-- Clustered columnstore: entire table in columnar format (DW/analytics)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales ON dbo.FactSales;

-- Nonclustered columnstore: analytical index on an OLTP rowstore table
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Analytics
    ON Sales.Orders (OrderDate, CustomerID, TotalAmount, Status);

-- Force delta store rows into compressed rowgroups (pre-load preparation)
ALTER INDEX CCI_FactSales ON dbo.FactSales
    REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);

-- Rowgroup health check
SELECT rg.state_desc, rg.total_rows, rg.deleted_rows,
       rg.size_in_bytes / 1024 / 1024 AS size_mb
FROM sys.column_store_row_groups rg
WHERE rg.object_id = OBJECT_ID('dbo.FactSales')
ORDER BY rg.state_desc, rg.total_rows DESC;
```

**How columnstore works:**
- Data is stored by column, not by row : reads only the columns referenced in the query.
- Typical compression ratio: 10:1 vs rowstore.
- Batch mode execution processes ~900 rows per CPU tick vs 1 for row mode : major speedup for aggregation queries.
- Segment elimination: the engine checks min/max per column segment and skips segments that can't match the WHERE predicate.
- Delta store: new rows accumulate in a B-tree delta store until 1,048,576 rows, then the engine compresses them into a new rowgroup. `REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON)` forces this manually.
- `OPEN` rowgroups in the delta store are not compressed : reads against them use row mode. Aim for all rowgroups in `COMPRESSED` state.
