# MSSQL Server : Knowledge Base

> Comprehensive T-SQL reference covering syntax, index design, transactions, security, temporal
> tables, JSON/XML, performance diagnostics, Query Store, backup/restore, high availability,
> and modern SQL Server features through SQL Server 2025.

## Scope

**Use this for:** writing and debugging T-SQL; reading execution plans; index design; diagnosing
locking, blocking, and wait statistics; stored procedures; transactions; security (RLS, TDE,
Always Encrypted); temporal tables; JSON and XML; bulk operations; backup and restore; HA
configuration; SQL Server 2022/2025 features.

**Not for:** PostgreSQL, MySQL, or Oracle : all syntax and behavior is SQL Server / T-SQL specific.

## Contents

1. [DDL: Tables, Schemas, Constraints](#ddl-tables-schemas-constraints)
2. [DQL: SELECT and Query Syntax](#dql-select-and-query-syntax)
3. [DML: INSERT, UPDATE, DELETE, MERGE](#dml-insert-update-delete-merge)
4. [Bulk Operations](#bulk-operations)
5. [CTEs](#ctes)
6. [Views](#views)
7. [Stored Procedures](#stored-procedures)
8. [User-Defined Functions](#user-defined-functions)
9. [Indexes](#indexes)
10. [Columnstore Indexes](#columnstore-indexes)
11. [Partitioning](#partitioning)
12. [Transactions and Locking](#transactions-and-locking)
13. [Error Handling](#error-handling)
14. [DBCC Commands](#dbcc-commands)
15. [Security](#security)
16. [Temporal Tables](#temporal-tables)
17. [Change Data Capture (CDC)](#change-data-capture-cdc)
18. [JSON and XML](#json-and-xml)
19. [Dynamic SQL](#dynamic-sql)
20. [Linked Servers](#linked-servers)
21. [Performance Diagnostics](#performance-diagnostics)
22. [Query Store](#query-store)
23. [Extended Events](#extended-events)
24. [TempDB](#tempdb)
25. [Backup and Restore](#backup-and-restore)
26. [High Availability](#high-availability)
27. [SQL Agent](#sql-agent)
28. [SQL Server 2022 Features](#sql-server-2022-features)
29. [SQL Server 2025 Features](#sql-server-2025-features)
30. [Common Mistakes](#common-mistakes)

---

## DDL: Tables, Schemas, Constraints

```sql
-- Create table with common constraint types
CREATE TABLE Production.Product (
    ProductID   INT           NOT NULL IDENTITY(1,1),
    SKU         VARCHAR(50)   NOT NULL,
    Name        NVARCHAR(200) NOT NULL,
    Price       DECIMAL(10,2) NOT NULL CHECK (Price > 0),
    CategoryID  INT           NOT NULL,
    IsActive    BIT           NOT NULL DEFAULT 1,
    CreatedAt   DATETIME2(0)  NOT NULL DEFAULT SYSDATETIME(),
    Weight      DECIMAL(8,3)  NULL,
    CONSTRAINT PK_Product      PRIMARY KEY CLUSTERED (ProductID),
    CONSTRAINT UQ_Product_SKU  UNIQUE (SKU),
    CONSTRAINT FK_Product_Category FOREIGN KEY (CategoryID)
        REFERENCES Production.Category(CategoryID)
);

-- Add column idempotently
IF NOT EXISTS (SELECT 1 FROM sys.columns
               WHERE object_id = OBJECT_ID('Production.Product')
               AND name = 'Barcode')
    ALTER TABLE Production.Product ADD Barcode VARCHAR(50) NULL;

-- Computed column (persisted = stored on disk, can be indexed)
ALTER TABLE Production.Product ADD
    FullLabel AS (SKU + ' - ' + Name) PERSISTED;

-- Sequence (alternative to IDENTITY : can be shared across tables)
CREATE SEQUENCE dbo.InvoiceSeq START WITH 100000 INCREMENT BY 1 MINVALUE 100000 CYCLE = OFF;
SELECT NEXT VALUE FOR dbo.InvoiceSeq AS NextInvoiceNo;

-- Schema isolation
CREATE SCHEMA Reporting AUTHORIZATION dbo;
CREATE TABLE Reporting.DailySummary (ReportDate DATE PRIMARY KEY, Revenue DECIMAL(18,2));
```

---

## DQL: SELECT and Query Syntax

```sql
-- Pagination with OFFSET FETCH
SELECT ProductID, Name, Price
FROM Production.Product
WHERE IsActive = 1
ORDER BY Name
OFFSET 50 ROWS FETCH NEXT 25 ROWS ONLY;  -- page 3 of 25 items per page

-- CROSS APPLY: invoke a TVF per row or use a correlated subquery
SELECT c.CustomerID, c.Name, recent.LastOrderDate, recent.OrderCount
FROM Sales.Customers c
CROSS APPLY (
    SELECT MAX(OrderDate) AS LastOrderDate, COUNT(*) AS OrderCount
    FROM Sales.Orders o WHERE o.CustomerID = c.CustomerID
) recent;
-- OUTER APPLY includes customers with no orders (NULLs for recent.*)

-- Set operators
SELECT ProductID FROM Production.Product WHERE CategoryID = 1
INTERSECT
SELECT ProductID FROM Sales.OrderLines WHERE Qty > 100;

SELECT ProductID FROM Production.Product
EXCEPT
SELECT ProductID FROM Sales.Discontinued;

-- PIVOT: rows to columns
SELECT ProductName, [2022], [2023], [2024]
FROM (
    SELECT p.Name AS ProductName, YEAR(o.OrderDate) AS OrderYear, ol.TotalAmount
    FROM Sales.OrderLines ol
    JOIN Sales.Orders o ON o.OrderID = ol.OrderID
    JOIN Production.Product p ON p.ProductID = ol.ProductID
) Src
PIVOT (SUM(TotalAmount) FOR OrderYear IN ([2022],[2023],[2024])) Pvt;

-- IS DISTINCT FROM (SQL Server 2022+) : NULL-safe comparison
WHERE a.Status IS DISTINCT FROM b.Status;   -- true even when both are NULL
-- Pre-2022: (a.Status <> b.Status OR (a.Status IS NULL AND b.Status IS NOT NULL) OR ...)
```

---

## DML: INSERT, UPDATE, DELETE, MERGE

```sql
-- INSERT with OUTPUT clause : capture inserted rows
INSERT INTO Sales.Orders (CustomerID, OrderDate, TotalAmount)
OUTPUT INSERTED.OrderID, INSERTED.CustomerID, GETDATE() INTO Sales.OrderAudit
VALUES (42, SYSDATETIME(), 150.00);

-- UPDATE with OUTPUT (shows before and after)
UPDATE Production.Product
SET Price = Price * 1.10
OUTPUT DELETED.ProductID, DELETED.Price AS OldPrice, INSERTED.Price AS NewPrice
WHERE CategoryID = 3 AND IsActive = 1;

-- DELETE with OUTPUT (archive before delete)
DELETE TOP (1000) FROM Sales.OldOrders
OUTPUT DELETED.* INTO Sales.ArchivedOrders
WHERE OrderDate < '2020-01-01';

-- MERGE (upsert): update if matched, insert if not, delete if not in source
MERGE Production.Product AS tgt
USING (SELECT * FROM Staging.ProductImport) AS src
    ON tgt.SKU = src.SKU
WHEN MATCHED AND tgt.Price <> src.Price THEN
    UPDATE SET tgt.Price = src.Price, tgt.Name = src.Name
WHEN NOT MATCHED BY TARGET THEN
    INSERT (SKU, Name, Price, CategoryID)
    VALUES (src.SKU, src.Name, src.Price, src.CategoryID)
WHEN NOT MATCHED BY SOURCE AND tgt.IsActive = 1 THEN
    UPDATE SET tgt.IsActive = 0
OUTPUT $action, INSERTED.ProductID, INSERTED.SKU;
-- OUTPUT INSERTED.* after MERGE shows the final state; DELETED.* shows the pre-merge state
```

---

## Bulk Operations

```sql
-- BULK INSERT from a CSV file
BULK INSERT Sales.Orders
FROM '\\fileserver\data\orders_2025.csv'
WITH (
    FORMAT       = 'CSV',
    FIRSTROW     = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR   = '\n',
    BATCHSIZE    = 10000,
    TABLOCK,
    CODEPAGE     = '65001'
);

-- OPENROWSET BULK (ad-hoc file access)
SELECT BulkColumn FROM OPENROWSET(BULK '\\fileserver\logs\error.log', SINGLE_CLOB) AS err;
```

**Minimal logging requires:** SIMPLE or BULK_LOGGED recovery + TABLOCK hint + empty or heap table. If any condition is missing, full logging applies.

---

## CTEs

```sql
-- Non-recursive: readability and reuse within one query
WITH ActiveProducts AS (
    SELECT ProductID, Name, Price, CategoryID
    FROM Production.Product WHERE IsActive = 1
),
CategoryTotals AS (
    SELECT p.CategoryID, c.Name AS Category, COUNT(*) AS ProductCount, SUM(p.Price) AS TotalValue
    FROM ActiveProducts p
    JOIN Production.Category c ON c.CategoryID = p.CategoryID
    GROUP BY p.CategoryID, c.Name
)
SELECT * FROM CategoryTotals WHERE ProductCount > 5 ORDER BY TotalValue DESC;

-- Recursive CTE: hierarchy traversal (org chart, category tree, BOM)
WITH OrgHierarchy AS (
    -- Anchor: top-level employees (no manager)
    SELECT EmployeeID, Name, ManagerID, 1 AS Level, CAST(Name AS NVARCHAR(500)) AS Path
    FROM HR.Employee WHERE ManagerID IS NULL

    UNION ALL

    -- Recursive member: employees whose manager is already in the CTE
    SELECT e.EmployeeID, e.Name, e.ManagerID, h.Level + 1,
           CAST(h.Path + ' > ' + e.Name AS NVARCHAR(500))
    FROM HR.Employee e
    JOIN OrgHierarchy h ON h.EmployeeID = e.ManagerID
)
SELECT * FROM OrgHierarchy ORDER BY Path
OPTION (MAXRECURSION 100);  -- default 100; set to 0 for unlimited (use with caution)
```

---

## Views

```sql
-- Standard view: join flatten, column subset
CREATE OR ALTER VIEW Sales.CustomerOrderSummary AS
SELECT c.CustomerID, c.Name, c.Email,
       COUNT(o.OrderID)    AS OrderCount,
       SUM(o.TotalAmount)  AS TotalSpend,
       MAX(o.OrderDate)    AS LastOrderDate
FROM Sales.Customers c
LEFT JOIN Sales.Orders o ON o.CustomerID = c.CustomerID
GROUP BY c.CustomerID, c.Name, c.Email;

-- Indexed view: materialized on disk, updated automatically, NOEXPAND hint for non-Enterprise
CREATE OR ALTER VIEW Sales.DailySalesSummary
WITH SCHEMABINDING AS
SELECT CAST(OrderDate AS DATE) AS OrderDay, COUNT_BIG(*) AS OrderCount, SUM(TotalAmount) AS Revenue
FROM Sales.Orders
GROUP BY CAST(OrderDate AS DATE);

CREATE UNIQUE CLUSTERED INDEX IX_DailySalesSummary ON Sales.DailySalesSummary (OrderDay);
-- Indexed view requirements: SCHEMABINDING, COUNT_BIG(*) for aggregate views,
-- two-part names, no outer joins, no subqueries, no DISTINCT, no non-deterministic functions

-- WITH CHECK OPTION: prevents inserts/updates through the view that would violate the filter
CREATE OR ALTER VIEW Sales.ActiveOrders AS
SELECT * FROM Sales.Orders WHERE Status <> 'Cancelled'
WITH CHECK OPTION;
```

---

## Stored Procedures

```sql
-- Procedure with table-valued parameter (TVP) : passes a set of rows as input
CREATE TYPE Sales.OrderLineType AS TABLE (
    ProductID INT NOT NULL, Qty INT NOT NULL, UnitPrice DECIMAL(10,2) NOT NULL
);

CREATE OR ALTER PROCEDURE Sales.PlaceOrder
    @CustomerID INT,
    @Lines      Sales.OrderLineType READONLY,   -- TVP must be READONLY
    @OrderID    INT OUTPUT
AS BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;
            INSERT INTO Sales.Orders (CustomerID, OrderDate)
            VALUES (@CustomerID, SYSDATETIME());
            SET @OrderID = SCOPE_IDENTITY();

            INSERT INTO Sales.OrderLines (OrderID, ProductID, Qty, UnitPrice)
            SELECT @OrderID, ProductID, Qty, UnitPrice FROM @Lines;

        COMMIT;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        THROW;
    END CATCH;
END;

-- EXECUTE AS: run with specific security context
CREATE OR ALTER PROCEDURE Reporting.GetAllOrders
    WITH EXECUTE AS 'ReportingUser'   -- impersonate a specific user
AS
    SELECT * FROM Sales.Orders;

-- Parameter sniffing fix (at statement level, not whole procedure)
SELECT OrderID, OrderDate FROM Sales.Orders
WHERE CustomerID = @CustomerID
OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));  -- plan based on average distribution
-- Alternative for infrequently-called but highly variable procs:
-- OPTION (RECOMPILE) : fresh plan per execution, CPU cost on every call
```

---

## User-Defined Functions

```sql
-- Inline TVF (treated as a parameterized view : can be indexed, joined)
CREATE OR ALTER FUNCTION Sales.fn_GetCustomerOrders (
    @CustomerID INT, @Since DATE
)
RETURNS TABLE AS RETURN (
    SELECT OrderID, OrderDate, TotalAmount, Status
    FROM Sales.Orders
    WHERE CustomerID = @CustomerID AND OrderDate >= @Since
);
-- Usage: SELECT * FROM Sales.fn_GetCustomerOrders(42, '2024-01-01')

-- Scalar UDF (avoid in WHERE clauses : prevents parallelism, runs row-by-row)
-- Prefer inline TVF with CROSS APPLY for performance-critical paths
CREATE OR ALTER FUNCTION dbo.fn_FormatCurrency (@Amount DECIMAL(18,2), @Currency CHAR(3))
RETURNS NVARCHAR(30) AS BEGIN
    RETURN @Currency + ' ' + FORMAT(@Amount, 'N2');
END;

-- Inline TVF replaces scalar UDF for better performance (SQL Server 2019+: UDF inlining)
-- Inlining is automatic when the UDF meets the eligibility criteria; check:
SELECT name, is_inlineable FROM sys.sql_modules WHERE OBJECT_NAME(object_id) = 'YourFunction';
```

---

## Indexes

```sql
-- Covering nonclustered index: key columns in seek predicates, INCLUDE for SELECT columns
CREATE NONCLUSTERED INDEX IX_Orders_CustomerDate
    ON Sales.Orders (CustomerID, OrderDate)
    INCLUDE (TotalAmount, Status);
-- Eliminates Key Lookup for queries: WHERE CustomerID = ? ORDER BY OrderDate

-- Filtered index: only indexes rows matching the filter : smaller, faster
CREATE NONCLUSTERED INDEX IX_Orders_Pending
    ON Sales.Orders (ScheduledAt, CustomerID)
    INCLUDE (OrderID)
    WHERE Status = 'Pending';

-- Rebuild vs reorganize
ALTER INDEX IX_Orders_CustomerDate ON Sales.Orders REBUILD WITH (ONLINE = ON, FILLFACTOR = 85);
ALTER INDEX IX_Orders_CustomerDate ON Sales.Orders REORGANIZE;
-- Reorganize: compacts in-place, online, does NOT update statistics → UPDATE STATISTICS after
-- Rebuild: full defragmentation + FULLSCAN statistics update; ONLINE = ON on Enterprise

-- Fragmentation decision thresholds
SELECT i.name, ps.avg_fragmentation_in_percent, ps.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ps
JOIN sys.indexes i ON i.object_id = ps.object_id AND i.index_id = ps.index_id
ORDER BY ps.avg_fragmentation_in_percent DESC;
-- < 5%: skip | 5-30% & >1000 pages: REORGANIZE | >30% & >1000 pages: REBUILD | <1000 pages: skip

-- Missing index DMVs (reset on restart; wait one full business cycle before acting)
SELECT TOP 10
    d.statement AS [Table],
    d.equality_columns, d.inequality_columns, d.included_columns,
    ROUND(s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans), 0) AS Score
FROM sys.dm_db_missing_index_groups g
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
JOIN sys.dm_db_missing_index_details d ON g.index_handle = d.index_handle
WHERE d.database_id = DB_ID()
ORDER BY Score DESC;

-- Unused indexes (high write cost, zero reads)
SELECT OBJECT_NAME(i.object_id) AS TableName, i.name,
       ISNULL(us.user_seeks + us.user_scans + us.user_lookups, 0) AS total_reads,
       ISNULL(us.user_updates, 0) AS writes
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats us
    ON us.object_id = i.object_id AND us.index_id = i.index_id AND us.database_id = DB_ID()
WHERE i.type_desc = 'NONCLUSTERED'
ORDER BY total_reads ASC, writes DESC;
```

---

## Columnstore Indexes

```sql
-- Clustered columnstore: entire table in columnar format (DW/analytics tables)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales ON dbo.FactSales;

-- Nonclustered columnstore: analytical index on an OLTP table (2012+)
-- Allows both row-mode and batch-mode execution on the same table
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Analytics
    ON Sales.Orders (OrderDate, CustomerID, TotalAmount, Status);

-- Columnstore benefits:
-- 1. Column-level compression (10:1 typical)
-- 2. Batch mode execution (processes 900 rows per CPU tick vs 1)
-- 3. Segment elimination: only loads row groups matching WHERE predicates

-- Delta store: new rows accumulate in a B-tree delta store until 1M rows, then compressed
-- Force delta store flush (use for testing or pre-load preparation):
ALTER INDEX CCI_FactSales ON dbo.FactSales REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);

-- Check rowgroup health
SELECT rg.object_id, rg.state_desc, rg.total_rows, rg.deleted_rows,
       rg.size_in_bytes / 1024 / 1024 AS size_mb
FROM sys.column_store_row_groups rg
WHERE rg.object_id = OBJECT_ID('dbo.FactSales')
ORDER BY rg.state_desc, rg.total_rows DESC;
```

---

## Partitioning

```sql
-- Partition function: defines the boundaries
CREATE PARTITION FUNCTION PF_Orders_Monthly (DATE)
AS RANGE RIGHT FOR VALUES (
    '2023-01-01','2023-02-01','2023-03-01','2023-04-01','2023-05-01',
    '2023-06-01','2023-07-01','2023-08-01','2023-09-01','2023-10-01',
    '2023-11-01','2023-12-01','2024-01-01'
);

-- Partition scheme: maps partitions to filegroups
CREATE PARTITION SCHEME PS_Orders_Monthly
    AS PARTITION PF_Orders_Monthly ALL TO ([PRIMARY]);

-- Partitioned table
CREATE TABLE Sales.Orders (
    OrderID   INT           NOT NULL,
    OrderDate DATE          NOT NULL,
    ...
) ON PS_Orders_Monthly (OrderDate);

-- Switch old partition to archive (metadata-only, milliseconds)
ALTER TABLE Sales.Orders SWITCH PARTITION 1 TO Sales.OrdersArchive_2022_12;

-- Add new partition for next month (sliding window)
ALTER PARTITION FUNCTION PF_Orders_Monthly() SPLIT RANGE ('2024-02-01');
ALTER PARTITION SCHEME PS_Orders_Monthly NEXT USED [PRIMARY];
```

---

## Transactions and Locking

```sql
-- Explicit transaction with savepoint
BEGIN TRANSACTION OrderProcessing;
SAVE TRANSACTION PreInventoryCheck;

    UPDATE Inventory.Stock SET Qty = Qty - @Qty WHERE ProductID = @ProductID;

    IF (SELECT Qty FROM Inventory.Stock WHERE ProductID = @ProductID) < 0 BEGIN
        ROLLBACK TRANSACTION PreInventoryCheck;  -- partial rollback to savepoint only
        RAISERROR('Insufficient stock', 16, 1);
        ROLLBACK TRANSACTION OrderProcessing;
        RETURN;
    END

COMMIT TRANSACTION OrderProcessing;

-- Isolation levels
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;     -- default
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;            -- row versioning, requires db option
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;        -- key-range locks, prevents phantoms

-- Enable RCSI (eliminates shared locks on reads : biggest impact for OLTP)
ALTER DATABASE YourDB SET READ_COMMITTED_SNAPSHOT ON WITH ROLLBACK IMMEDIATE;

-- Lock hints for specific operations
SELECT * FROM Sales.Orders WITH (NOLOCK);            -- READ UNCOMMITTED (use sparingly)
SELECT * FROM Sales.Orders WITH (UPDLOCK, ROWLOCK);  -- update lock on row (prevents S→X deadlock)
SELECT * FROM Sales.Orders WITH (TABLOCK);           -- table-level lock (bulk operations)
SELECT * FROM Jobs WITH (READPAST, ROWLOCK, UPDLOCK); -- skip locked rows (queue consumer)

-- Deadlock graph from system_health Extended Events session
SELECT xdr.value('@timestamp','datetime2') AS deadlock_time, xdr.query('.') AS graph
FROM (SELECT CAST(target_data AS XML) AS td
      FROM sys.dm_xe_session_targets t
      JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
      WHERE s.name = 'system_health' AND t.target_name = 'ring_buffer') d
CROSS APPLY td.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS XEventData(xdr)
ORDER BY deadlock_time DESC;
```

---

## Error Handling

```sql
-- TRY/CATCH: catches runtime errors
BEGIN TRY
    BEGIN TRANSACTION;
        EXEC Sales.PlaceOrder @CustomerID = @cid, @Lines = @lines, @OrderID = @oid OUTPUT;
    COMMIT;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK;
    -- Re-throw preserves original error number, state, and message
    THROW;
    -- Or RAISERROR for backward compatibility:
    -- RAISERROR(ERROR_MESSAGE(), ERROR_SEVERITY(), ERROR_STATE());
END CATCH;

-- Error diagnostic functions
SELECT ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE(),
       ERROR_PROCEDURE(), ERROR_LINE(), ERROR_MESSAGE();

-- THROW with custom message (2012+)
THROW 50001, 'Customer not found', 1;

-- RAISERROR with message ID (requires sp_addmessage)
RAISERROR(50001, 16, 1, 'Customer lookup');

-- XACT_ABORT: automatically rolls back and aborts batch on error
SET XACT_ABORT ON;  -- recommended for transaction safety in procedures
```

---

## DBCC Commands

```sql
-- Full database consistency check
DBCC CHECKDB('YourDB') WITH NO_INFOMSGS, ALL_ERRORMSGS;

-- Find oldest open transaction (blocks log truncation)
DBCC OPENTRAN('YourDB');

-- Shrink a data file (use sparingly: causes index fragmentation)
DBCC SHRINKFILE('YourDB_log', 1024);

-- Log space usage
DBCC SQLPERF(LOGSPACE);
```

**Key rules:** Run CHECKDB weekly minimum. Shrink data files only when necessary followed by REBUILD. REPAIR_ALLOW_DATA_LOSS is a last resort : prefer restore over repair.

---

## Security

### Principals and permissions

```sql
-- Server login + database user
CREATE LOGIN AppUser WITH PASSWORD = 'S3cur3P@ss!';
CREATE USER  AppUser FOR LOGIN AppUser;

-- Role-based permissions (never grant directly to users)
CREATE ROLE ReportReader;
GRANT SELECT ON SCHEMA::Reporting TO ReportReader;
ALTER ROLE ReportReader ADD MEMBER AppUser;

-- Deny overrides Grant
DENY  SELECT ON Sales.Orders TO AppUser;
REVOKE SELECT ON Sales.Orders FROM AppUser;  -- REVOKE removes an explicit grant or deny
```

### Row-Level Security

```sql
-- RLS: filter predicate restricts rows seen by non-admin sessions
CREATE FUNCTION Security.fn_TenantFilter (@TenantID INT)
RETURNS TABLE WITH SCHEMABINDING AS RETURN (
    SELECT 1 AS allow
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT)
       OR IS_ROLEMEMBER('db_owner') = 1
);

CREATE SECURITY POLICY Sales.TenantPolicy
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID) ON Sales.Orders,
ADD BLOCK  PREDICATE Security.fn_TenantFilter(TenantID) ON Sales.Orders AFTER INSERT
WITH (STATE = ON);

-- Set context at connection time (application sets this per session)
EXEC sys.sp_set_session_context N'TenantID', 7;
```

### Dynamic Data Masking

```sql
-- Mask PII : underlying data is unchanged, exposed value is masked per user privilege
ALTER TABLE Sales.Customers ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
ALTER TABLE Sales.Customers ALTER COLUMN Phone ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)');
ALTER TABLE Sales.Customers ALTER COLUMN CreditCard ADD MASKED WITH (FUNCTION = 'default()');

-- UNMASK permission reveals actual values
GRANT UNMASK TO PrivilegedReportUser;
```

### Transparent Data Encryption

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongKeyPass1!';
CREATE CERTIFICATE TDECert WITH SUBJECT = 'TDE Certificate';
CREATE DATABASE ENCRYPTION KEY WITH ALGORITHM = AES_256 ENCRYPTION BY SERVER CERTIFICATE TDECert;
ALTER DATABASE YourDB SET ENCRYPTION ON;

SELECT name, encryption_state_desc, percent_complete FROM sys.dm_database_encryption_keys;
```

---

## Temporal Tables

```sql
-- Create system-versioned temporal table
CREATE TABLE HR.Employee (
    EmployeeID   INT           NOT NULL PRIMARY KEY,
    Name         NVARCHAR(200) NOT NULL,
    DepartmentID INT           NOT NULL,
    Salary       DECIMAL(12,2) NOT NULL,
    ValidFrom    DATETIME2     GENERATED ALWAYS AS ROW START NOT NULL,
    ValidTo      DATETIME2     GENERATED ALWAYS AS ROW END   NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = HR.Employee_History));

-- Point-in-time queries : transparent, works like a regular SELECT
SELECT EmployeeID, Name, Salary
FROM HR.Employee
FOR SYSTEM_TIME AS OF '2023-06-01T00:00:00';   -- UTC

-- Range: all versions of a row between two times
SELECT EmployeeID, Name, Salary, ValidFrom, ValidTo
FROM HR.Employee
FOR SYSTEM_TIME BETWEEN '2022-01-01' AND '2024-01-01'
WHERE EmployeeID = 42
ORDER BY ValidFrom;

-- ALL: complete history including current
SELECT * FROM HR.Employee FOR SYSTEM_TIME ALL WHERE EmployeeID = 42 ORDER BY ValidFrom;

-- Retention policy: auto-purge history older than 2 years
ALTER TABLE HR.Employee SET (SYSTEM_VERSIONING = ON (
    HISTORY_RETENTION_PERIOD = 2 YEARS
));
```

---

## Change Data Capture (CDC)

```sql
-- Enable CDC at database and table level
EXEC sys.sp_cdc_enable_db;
EXEC sys.sp_cdc_enable_table @source_schema = 'Sales', @source_name = 'Orders', @role_name = 'CDCReader';

-- Get all changes between two LSNs
DECLARE @from_lsn BINARY(10), @to_lsn BINARY(10);
SET @from_lsn = sys.fn_cdc_get_min_lsn('Sales_Orders');
SET @to_lsn   = sys.fn_cdc_get_max_lsn();
SELECT * FROM cdc.fn_cdc_get_all_changes_Sales_Orders(@from_lsn, @to_lsn, N'all');

-- Net changes (one row per source row, final operation)
SELECT * FROM cdc.fn_cdc_get_net_changes_Sales_Orders(@from_lsn, @to_lsn, N'all');
```

CDC tracks INSERT, UPDATE, DELETE via the transaction log. Available in Enterprise, Developer, and Standard (2016 SP1+).

---

## JSON and XML

```sql
-- FOR JSON PATH: controlled JSON shape
SELECT
    o.OrderID   AS "id",
    o.OrderDate AS "date",
    c.Name      AS "customer.name",
    c.Email     AS "customer.email"
FROM Sales.Orders o JOIN Sales.Customers c ON c.CustomerID = o.CustomerID
FOR JSON PATH, ROOT('orders');

-- JSON_OBJECT and JSON_ARRAY (SQL Server 2022+)
SELECT JSON_OBJECT('id': ProductID, 'name': Name, 'price': Price) AS json_row
FROM Production.Product WHERE IsActive = 1;

-- Parse JSON input
DECLARE @json NVARCHAR(MAX) = '[{"id":1,"name":"Widget"},{"id":2,"name":"Gadget"}]';
SELECT j.id, j.name
FROM OPENJSON(@json) WITH (id INT '$.id', name NVARCHAR(100) '$.name') j;

-- Store JSON in a column + index a property
ALTER TABLE Logs.Events ADD Payload NVARCHAR(MAX);
ALTER TABLE Logs.Events ADD EventType AS JSON_VALUE(Payload, '$.type') PERSISTED;
CREATE INDEX IX_Events_Type ON Logs.Events (EventType);

-- FOR XML PATH (use only when downstream requires XML)
SELECT c.Name AS "customer/@name", o.OrderID AS "order/@id", o.TotalAmount AS "order"
FROM Sales.Orders o JOIN Sales.Customers c ON c.CustomerID = o.CustomerID
FOR XML PATH('row'), ROOT('orders');
```

---

## Dynamic SQL

```sql
-- Always use sp_executesql with parameters (never string concatenation with user input)
DECLARE @SQL   NVARCHAR(MAX);
DECLARE @Param NVARCHAR(500);
DECLARE @CID   INT = 42;
DECLARE @Since DATE = '2024-01-01';

SET @SQL = N'
SELECT OrderID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE CustomerID = @CustomerID AND OrderDate >= @Since';

SET @Param = N'@CustomerID INT, @Since DATE';

EXEC sp_executesql @SQL, @Param, @CustomerID = @CID, @Since = @Since;

-- Dynamic WHERE clause (variable predicates)
SET @SQL = N'SELECT * FROM Sales.Orders WHERE 1=1';
IF @CustomerID IS NOT NULL SET @SQL += N' AND CustomerID = @CustomerID';
IF @Status     IS NOT NULL SET @SQL += N' AND Status = @Status';
SET @SQL += N' ORDER BY OrderDate DESC';
EXEC sp_executesql @SQL, N'@CustomerID INT, @Status VARCHAR(20)', @CustomerID, @Status;

-- Dynamic PIVOT with QUOTENAME (injection-safe column names)
DECLARE @Cols NVARCHAR(MAX);
SELECT @Cols = STRING_AGG(QUOTENAME(CAST(OrderYear AS VARCHAR(4))), ',')
               WITHIN GROUP (ORDER BY OrderYear)
FROM (SELECT DISTINCT YEAR(OrderDate) AS OrderYear FROM Sales.Orders) y;

SET @SQL = N'SELECT ProductName, ' + @Cols + N'
FROM (...) AS Src PIVOT (SUM(Revenue) FOR OrderYear IN (' + @Cols + N')) Pvt;';
EXEC sp_executesql @SQL;
```

---

## Linked Servers

```sql
-- Create a linked server and query via OPENQUERY (faster than four-part names)
EXEC sp_addlinkedserver @server = N'SQLPROD', @srvproduct = N'SQL Server', @datasrc = N'prodsql\PROD01';
EXEC sp_addlinkedsrvlogin @rmtsrvname = N'SQLPROD', @useself = N'True';

SELECT * FROM OPENQUERY(SQLPROD, 'SELECT OrderID, OrderDate FROM YourDB.dbo.Orders WHERE OrderDate >= ''2025-01-01''');
-- Four-part name (SELECT * FROM SQLPROD.YourDB.dbo.Orders) pulls the entire table and filters locally : avoid for filtered queries
```

Always prefer OPENQUERY over four-part names for filtered distributed queries. Use `MSOLEDBSQL` provider for SQL Server 2022+.

---

## Performance Diagnostics

```sql
-- Top 10 queries by total CPU since last restart
SELECT TOP 10
    qs.total_worker_time / 1000              AS total_cpu_ms,
    qs.total_worker_time / qs.execution_count / 1000 AS avg_cpu_ms,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS stmt,
    DB_NAME(st.dbid) AS db_name
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY total_cpu_ms DESC;

-- Wait statistics delta (snapshot-and-diff methodology)
SELECT wait_type, waiting_tasks_count, wait_time_ms
INTO #WaitSnap FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN ('SLEEP_TASK','WAITFOR','LAZYWRITER_SLEEP','SQLTRACE_BUFFER_FLUSH',
    'CHECKPOINT_QUEUE','XE_DISPATCHER_WAIT','CXCONSUMER','DISPATCHER_QUEUE_SEMAPHORE')
  AND wait_type NOT LIKE 'SLEEP_%';

-- ...wait 60+ seconds...

SELECT c.wait_type,
    c.wait_time_ms - b.wait_time_ms AS delta_ms,
    CAST(100.0 * (c.wait_time_ms - b.wait_time_ms)
        / NULLIF(SUM(c.wait_time_ms - b.wait_time_ms) OVER (), 0) AS DECIMAL(5,2)) AS pct
FROM sys.dm_os_wait_stats c JOIN #WaitSnap b ON b.wait_type = c.wait_type
WHERE c.wait_time_ms > b.wait_time_ms
ORDER BY delta_ms DESC;

-- Active blocking chains
SELECT r.session_id AS blocked, r.blocking_session_id AS blocker,
       r.wait_type, r.wait_time / 1000.0 AS wait_sec,
       SUBSTRING(st.text, (r.statement_start_offset/2)+1, 200) AS blocked_stmt
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
WHERE r.blocking_session_id > 0;

-- Queries with hash/sort spills (SQL Server 2016 SP1+)
SELECT TOP 10 qs.total_spills / qs.execution_count AS avg_spills,
       SUBSTRING(st.text, 1, 200) AS stmt
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.total_spills > 0 ORDER BY avg_spills DESC;
```

---

## Query Store

```sql
-- Enable with recommended settings
ALTER DATABASE YourDB SET QUERY_STORE = ON;
ALTER DATABASE YourDB SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = AUTO,
    MAX_STORAGE_SIZE_MB = 1024,
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    WAIT_STATS_CAPTURE_MODE = ON
);

-- Top regressed queries (recent avg > 1.5x historical best)
WITH BestPlan AS (
    SELECT qsq.query_id, qsp.plan_id, MIN(qsrs.avg_duration) AS best_us,
           ROW_NUMBER() OVER (PARTITION BY qsq.query_id ORDER BY MIN(qsrs.avg_duration)) AS rn
    FROM sys.query_store_query qsq
    JOIN sys.query_store_plan  qsp  ON qsp.query_id = qsq.query_id
    JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
    GROUP BY qsq.query_id, qsp.plan_id
),
Recent AS (
    SELECT qsp.query_id, qsp.plan_id, AVG(qsrs.avg_duration) AS recent_us
    FROM sys.query_store_plan qsp
    JOIN sys.query_store_runtime_stats qsrs ON qsrs.plan_id = qsp.plan_id
    JOIN sys.query_store_runtime_stats_interval qsrsi
         ON qsrsi.runtime_stats_interval_id = qsrs.runtime_stats_interval_id
    WHERE qsrsi.start_time >= DATEADD(HOUR, -4, GETUTCDATE())
    GROUP BY qsp.query_id, qsp.plan_id
)
SELECT r.query_id, b.plan_id AS best_plan, r.plan_id AS current_plan,
       b.best_us/1000.0 AS best_ms, r.recent_us/1000.0 AS current_ms,
       r.recent_us / NULLIF(b.best_us, 0.0) AS regression_ratio
FROM Recent r JOIN BestPlan b ON b.query_id = r.query_id AND b.rn = 1
WHERE r.recent_us > b.best_us * 1.5
ORDER BY regression_ratio DESC;

-- Force a known-good plan
EXEC sys.sp_query_store_force_plan @query_id = 42, @plan_id = 7;

-- Automatic plan correction (SQL Server 2017+)
ALTER DATABASE YourDB SET AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON);
```

---

## Extended Events

```sql
-- Capture deadlocks and long-running queries
CREATE EVENT SESSION [PerfTriage] ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    ACTION (sqlserver.sql_text, sqlserver.session_id, sqlserver.database_name)
    WHERE duration > 5000000   -- > 5 seconds (microseconds)
),
ADD EVENT sqlserver.xml_deadlock_report (
    ACTION (sqlserver.sql_text, sqlserver.session_id)
)
ADD TARGET package0.ring_buffer (SET max_memory = 51200),
ADD TARGET package0.event_file  (SET filename = 'D:\XELogs\PerfTriage.xel', max_file_size = 100);
GO

ALTER EVENT SESSION [PerfTriage] ON SERVER STATE = START;

-- Read from ring buffer
SELECT xdr.value('@timestamp','datetime2') AS event_time,
       xdr.value('(data[@name="duration"]/value)[1]', 'BIGINT') / 1000 AS duration_ms,
       xdr.value('(action[@name="sql_text"]/value)[1]', 'NVARCHAR(MAX)') AS sql_text
FROM (SELECT CAST(target_data AS XML) AS td
      FROM sys.dm_xe_session_targets t
      JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
      WHERE s.name = 'PerfTriage' AND t.target_name = 'ring_buffer') d
CROSS APPLY td.nodes('//RingBufferTarget/event[@name="sql_statement_completed"]') AS e(xdr)
ORDER BY event_time DESC;

ALTER EVENT SESSION [PerfTriage] ON SERVER STATE = STOP;
DROP EVENT SESSION [PerfTriage] ON SERVER;
```

---

## TempDB

```sql
-- Check tempdb file sizes (all data files must be equal for proportional fill)
SELECT name, type_desc, size * 8 / 1024 AS size_mb, growth * 8 / 1024 AS growth_mb
FROM tempdb.sys.database_files ORDER BY type_desc, file_id;

-- Equalize undersized files
ALTER DATABASE tempdb MODIFY FILE (NAME = 'tempdev2', SIZE = 4096 MB, FILEGROWTH = 512 MB);

-- TF 1117 and 1118 are obsolete on 2016+ : remove them if present
-- 2016+ enables equal proportional fill and uniform extent allocation by default

-- Version store size (RCSI / snapshot isolation source)
SELECT DB_NAME(database_id) AS db_name,
       reserved_page_count * 8.0 / 1024 AS version_store_mb
FROM sys.dm_tran_version_store_space_usage ORDER BY reserved_page_count DESC;

-- Long-running transactions blocking version store cleanup
SELECT s.session_id, s.login_name, t.elapsed_time_seconds, t.transaction_sequence_num
FROM sys.dm_tran_active_snapshot_database_transactions t
JOIN sys.dm_exec_sessions s ON s.session_id = t.session_id
ORDER BY t.elapsed_time_seconds DESC;
```

---

## Backup and Restore

```sql
-- Full backup
BACKUP DATABASE YourDB
    TO DISK = '\\backupserver\backups\YourDB_full.bak'
    WITH COMPRESSION, CHECKSUM, STATS = 10, INIT;

-- Differential backup (since last full)
BACKUP DATABASE YourDB TO DISK = '\\backupserver\backups\YourDB_diff.bak'
    WITH DIFFERENTIAL, COMPRESSION, CHECKSUM;

-- Transaction log backup (required for point-in-time restore under FULL recovery)
BACKUP LOG YourDB TO DISK = '\\backupserver\backups\YourDB_log_20240615_1400.trn'
    WITH COMPRESSION, CHECKSUM;

-- Point-in-time restore
RESTORE DATABASE YourDB_Test FROM DISK = '\\backupserver\backups\YourDB_full.bak'
    WITH NORECOVERY, MOVE 'YourDB'     TO 'D:\data\YourDB_Test.mdf',
                    MOVE 'YourDB_log'  TO 'D:\log\YourDB_Test.ldf',
                    REPLACE;

RESTORE LOG YourDB_Test FROM DISK = '\\backupserver\backups\YourDB_log_20240615_1400.trn'
    WITH NORECOVERY;

-- Final log backup with STOPATMARK or STOPAT for precise point-in-time
RESTORE LOG YourDB_Test FROM DISK = '\\backupserver\backups\YourDB_log_20240615_1500.trn'
    WITH RECOVERY, STOPAT = '2024-06-15T14:42:00';
-- Database is online after the final RESTORE ... WITH RECOVERY

-- S3 backup (SQL Server 2022+)
BACKUP DATABASE YourDB TO URL = 's3://my-backup-bucket/YourDB_full.bak'
    WITH COMPRESSION, CHECKSUM, CREDENTIAL = 'S3BackupCred';

-- Verify backup integrity
RESTORE VERIFYONLY FROM DISK = '\\backupserver\backups\YourDB_full.bak' WITH CHECKSUM;
```

---

## High Availability

```sql
-- Always On Availability Group
CREATE AVAILABILITY GROUP [YourAG]
WITH (AUTOMATED_BACKUP_PREFERENCE = SECONDARY,
      DB_FAILOVER = ON,
      DTC_SUPPORT = PER_DB)
FOR DATABASE YourDB
REPLICA ON
    N'Primary\SQL1' WITH (ENDPOINT_URL = 'TCP://primary.internal:5022',
                          AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
                          FAILOVER_MODE = AUTOMATIC,
                          SEEDING_MODE = AUTOMATIC),
    N'Secondary\SQL2' WITH (ENDPOINT_URL = 'TCP://secondary.internal:5022',
                            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
                            FAILOVER_MODE = AUTOMATIC,
                            READABLE_SECONDARY = YES,
                            SEEDING_MODE = AUTOMATIC);

-- Monitor AG health
SELECT ag.name, ars.role_desc, ar.replica_server_name,
       drs.synchronization_state_desc, drs.synchronization_health_desc,
       drs.log_send_queue_size, drs.redo_queue_size
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ar.group_id = ag.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ars.replica_id = ar.replica_id
JOIN sys.dm_hadr_database_replica_states drs ON drs.replica_id = ar.replica_id;

-- Direct reads to a readable secondary
SELECT * FROM Sales.Orders WITH (NOLOCK);  -- readable secondary allows only snapshot-level reads
-- Set application intent in the connection string:
-- ApplicationIntent=ReadOnly : AG listener routes to readable secondary automatically

-- Contained AG (SQL Server 2022+): AG-level logins and jobs, no dependency on master
ALTER AVAILABILITY GROUP [YourAG] SET (CONTAINED = ON);
CREATE USER ContainedUser FOR LOGIN ContainedLogin
IN AVAILABILITY GROUP [YourAG];
```

---

## SQL Agent

```sql
-- Create a job step
EXEC msdb.dbo.sp_add_job @job_name = N'Daily Index Rebuild', @enabled = 1;
EXEC msdb.dbo.sp_add_jobstep @job_name = N'Daily Index Rebuild', @step_name = N'Rebuild',
    @subsystem = N'TSQL', @command = N'ALTER INDEX ALL ON Sales.Orders REBUILD;', @database_name = N'YourDB';
EXEC msdb.dbo.sp_add_schedule @schedule_name = N'Nightly 2AM', @freq_type = 4, @freq_interval = 1, @active_start_time = 020000;
EXEC msdb.dbo.sp_attach_schedule @job_name = N'Daily Index Rebuild', @schedule_name = N'Nightly 2AM';
```

Monitor jobs via `msdb.dbo.sysjobactivity` (running) and `msdb.dbo.sysjobhistory` (history). Enable Database Mail before using email notifications.

---

## SQL Server 2022 Features

```sql
-- IS [NOT] DISTINCT FROM: NULL-safe equality
SELECT a.val, b.val FROM A JOIN B ON A.key = B.key
WHERE a.val IS DISTINCT FROM b.val;  -- true when values differ, including NULL vs non-NULL

-- GREATEST / LEAST across multiple columns
SELECT ProductID, GREATEST(Q1, Q2, Q3, Q4) AS PeakQuarter,
                  LEAST(Q1, Q2, Q3, Q4)    AS WeakestQuarter
FROM Sales.QuarterlySales;

-- DATETRUNC: clean date truncation
SELECT DATETRUNC(month, OrderDate) AS MonthStart FROM Sales.Orders;

-- DATE_BUCKET: arbitrary interval bucketing
SELECT DATE_BUCKET(hour, 4, EventTime) AS FourHourBucket FROM Sensors.Events;

-- JSON_OBJECT and JSON_ARRAY constructors
SELECT JSON_OBJECT('id': ProductID, 'tags': JSON_ARRAY('new','sale')) AS json_row
FROM Production.Product;

-- GENERATE_SERIES: inline number/date sequences (replaces stacking CTE)
SELECT value AS N FROM GENERATE_SERIES(1, 100);
SELECT DATEADD(day, value, '2024-01-01') AS D FROM GENERATE_SERIES(0, 364);

-- XML compression for XML columns
ALTER TABLE Logs.Events REBUILD WITH (XML_COMPRESSION = ON);

-- S3 backup/restore (see Backup and Restore section)

-- Contained Availability Groups (see High Availability section)

-- APPROX_PERCENTILE_DISC / APPROX_PERCENTILE_CONT (T-Digest approximation)
SELECT APPROX_PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY Duration) OVER () AS p95
FROM Telemetry.RequestLog;
```

---

## SQL Server 2025 Features

```sql
-- VECTOR data type: store embedding vectors for AI applications
CREATE TABLE dbo.ProductEmbeddings (
    ProductID INT         NOT NULL PRIMARY KEY,
    Embedding VECTOR(768) NOT NULL   -- 768-dimension embedding (e.g. from Azure OpenAI text-embedding-3-small)
);

-- INSERT an embedding vector
INSERT INTO dbo.ProductEmbeddings (ProductID, Embedding)
VALUES (42, '[0.012, -0.034, 0.891, ...]');   -- JSON array format

-- Vector similarity search: cosine, dot product, Euclidean distance
SELECT TOP 10
    pe.ProductID,
    VECTOR_DISTANCE('cosine', pe.Embedding, CAST(@queryVector AS VECTOR(768))) AS distance
FROM dbo.ProductEmbeddings pe
ORDER BY distance ASC;   -- lower cosine distance = more similar

-- Vector index: ANN (approximate nearest neighbor) via DiskANN
CREATE VECTOR INDEX IX_ProductEmbeddings_Vector
    ON dbo.ProductEmbeddings (Embedding)
    WITH (METRIC = 'cosine', TYPE = 'diskann');

-- Native JSON functions expanded
SELECT JSON_PATH_EXISTS(Payload, '$.customer.address.city') AS has_city
FROM Logs.Events;

SELECT JSON_OBJECTAGG(ProductID: Name) AS product_map
FROM Production.Product WHERE IsActive = 1;
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Non-SARGable WHERE (`YEAR(col) = 2024`) | Range predicate: `col >= '2024-01-01' AND col < '2025-01-01'` |
| Implicit type conversion in WHERE (column is INT, param is VARCHAR) | Match parameter type to column type exactly |
| NOLOCK everywhere "for performance" | Enable RCSI : consistent reads without shared lock overhead |
| CTE referenced twice (scans source twice) | Materialize into a temp table |
| Scalar UDF in WHERE clause (prevents parallelism) | Replace with inline TVF + CROSS APPLY |
| IDENTITY for composite hierarchy child keys | Use max-plus-one functions scoped to parent key |
| Missing PARTITION BY in window functions on multi-entity data | Always `PARTITION BY entity_id` |
| Recursive CTE for number sequences | Use stacking CTE or GENERATE_SERIES (2022+) : 10-50x faster |
| Forgotten `WHERE N <= limit` in tally CTE | Always add the limit : without it materializes 65K rows |
| Key Lookup on high-row-count seek | Add SELECT columns to NCI INCLUDE list |
| Index fragmentation always triggering REBUILD | Skip < 5% fragmented or < 1,000 page indexes |
| Stats staleness on large tables (compat < 130) | Upgrade to compat 130+ for dynamic auto-update threshold |
| Single large DELETE bloating the log | Chunk with `DELETE TOP (5000)` in a WHILE loop |
| MERGE without OUTPUT makes errors hard to debug | Add `OUTPUT $action, INSERTED.*, DELETED.*` |
| FOR JSON AUTO in production | Use FOR JSON PATH : AUTO changes shape when aliases change |
| Dynamic SQL with string concatenation of user input | Always use sp_executesql with parameters |
| Temporal table queries with local time | FOR SYSTEM_TIME times are UTC : convert with AT TIME ZONE |
| TF 1117/1118 on SQL Server 2016+ | These are no-ops and should be removed |
