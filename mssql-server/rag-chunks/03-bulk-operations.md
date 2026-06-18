---
topic: Bulk operations : BULK INSERT, OPENROWSET BULK, BCP, minimal logging
keywords: [BULK INSERT, OPENROWSET BULK, BCP, bulk copy, minimal logging, TABLOCK, batch size, format file]
use_when: Loading large datasets efficiently, choosing between BULK INSERT and BCP, or enabling minimal logging for bulk imports.
---

# Bulk Operations

```sql
-- BULK INSERT from a CSV file
BULK INSERT Sales.Orders
FROM '\\fileserver\data\orders_2025.csv'
WITH (
    FORMAT       = 'CSV',
    FIRSTROW     = 2,               -- skip header
    FIELDTERMINATOR = ',',
    ROWTERMINATOR   = '\n',
    BATCHSIZE    = 10000,
    TABLOCK,                         -- enables minimal logging
    CODEPAGE     = '65001'           -- UTF-8
);

-- OPENROWSET BULK with format file (complex delimited files)
SELECT *
FROM OPENROWSET BULK '\\fileserver\data\orders.xml',
    FORMATFILE = '\\fileserver\format\orders.fmt'
) AS ord;

-- OPENROWSET BULK as a single column (raw text files)
SELECT BulkColumn
FROM OPENROWSET(BULK '\\fileserver\logs\error.log', SINGLE_CLOB) AS err;

-- BCP via T-SQL xp_cmdshell (when you need BCP control from within SQL Server)
-- This requires xp_cmdshell enabled:
EXEC xp_cmdshell 'bcp Sales.Orders OUT D:\export\orders.dat -T -c -t,';
```

**Minimal logging prerequisites:**
- Database must be in `SIMPLE` or `BULK_LOGGED` recovery model.
- `TABLOCK` hint must be specified on the target table.
- Table must be empty or have no nonclustered indexes (or use `TABLOCK` with a heap).
- If any condition is missing, the operation falls back to full logging.

**Key rules:**
- `BATCHSIZE` controls how many rows are committed per batch. Increase for larger loads (10,000-100,000 is typical). A single batch means one huge transaction.
- Break very large imports (100M+ rows) into multiple `BULK INSERT` calls with `BATCHSIZE` to keep the log manageable.
- For SQL Server 2019+, use `FORMAT = 'CSV'` for native CSV parsing. Older versions need a format file or `FIELDTERMINATOR` only.
- Query performance drops during bulk loads. Schedule them during maintenance windows or use resource governor.
