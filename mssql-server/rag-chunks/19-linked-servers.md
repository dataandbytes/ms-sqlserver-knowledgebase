---
topic: Linked servers : create, query, OPENQUERY, OPENROWSET, security
keywords: [linked server, OPENQUERY, OPENROWSET, sp_addlinkedserver, sp_addlinkedsrvlogin, four-part name, distributed query, pass-through]
use_when: Querying data across SQL Server instances or from OLE DB sources (Oracle, Excel, CSV), setting up a linked server connection.
---

# Linked Servers

```sql
-- Create a linked server to another SQL Server instance
EXEC sp_addlinkedserver
    @server     = N'SQLPROD',
    @srvproduct = N'SQL Server',
    @provider   = N'SQLNCLI',
    @datasrc    = N'prodsql.internal\PROD01';

-- Set login mapping (use current login's credentials)
EXEC sp_addlinkedsrvlogin
    @rmtsrvname  = N'SQLPROD',
    @useself     = N'True';

-- Query via four-part name (slowest : pulls full table then filters locally)
SELECT * FROM SQLPROD.YourDB.dbo.Orders WHERE OrderDate >= '2025-01-01';

-- Query via OPENQUERY (faster : pushes query to remote server)
SELECT * FROM OPENQUERY(SQLPROD, '
    SELECT OrderID, OrderDate, TotalAmount
    FROM YourDB.dbo.Orders
    WHERE OrderDate >= ''2025-01-01''
');

-- OPENROWSET for ad-hoc connections (no linked server needed)
SELECT *
FROM OPENROWSET(
    'SQLNCLI',
    'Server=prodsql\PROD01;Trusted_Connection=yes;',
    'SELECT OrderID, OrderDate FROM YourDB.dbo.Orders'
);

-- OPENROWSET with Excel or CSV
SELECT * FROM OPENROWSET(
    'Microsoft.ACE.OLEDB.12.0',
    'Excel 12.0;Database=D:\reports\inventory.xlsx;',
    'SELECT * FROM [Sheet1$]'
);
```

**Key rules:**
- Always prefer `OPENQUERY` over four-part names for filtered queries. Four-part names fetch the entire table and apply WHERE locally, causing massive unnecessary network traffic.
- Four-part names are acceptable only when querying small reference tables (under 1,000 rows) or when the WHERE clause is on the join key that SQL Server can push back.
- Linked servers use the `SQLNCLI` provider by default. For SQL Server 2022+, use `MSOLEDBSQL` instead (the current OLE DB provider).
- Cross-server transactions use distributed transactions (MSDTC) by default. Set `@rmtdataaccess = 0` or use `OPENQUERY` in a separate connection to avoid DTC overhead.
- Monitor `sys.dm_exec_requests` for `OLEDB` wait types : they indicate linked server query bottlenecks.
