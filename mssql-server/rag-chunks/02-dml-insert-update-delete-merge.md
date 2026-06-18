---
topic: DML : INSERT, UPDATE, DELETE with OUTPUT clause, MERGE upsert
keywords: [INSERT, UPDATE, DELETE, MERGE, OUTPUT, upsert, INSERTED, DELETED, $action, archive rows]
use_when: Writing INSERT/UPDATE/DELETE with auditing via OUTPUT, or MERGE for upsert/sync patterns.
---

# DML: INSERT, UPDATE, DELETE, MERGE

```sql
-- INSERT with OUTPUT : capture inserted IDs into audit table
INSERT INTO Sales.Orders (CustomerID, OrderDate, TotalAmount)
OUTPUT INSERTED.OrderID, INSERTED.CustomerID, GETDATE() INTO Sales.OrderAudit
VALUES (42, SYSDATETIME(), 150.00);

-- UPDATE with OUTPUT : see before and after values
UPDATE Production.Product SET Price = Price * 1.10
OUTPUT DELETED.ProductID, DELETED.Price AS OldPrice, INSERTED.Price AS NewPrice
WHERE CategoryID = 3 AND IsActive = 1;

-- DELETE with OUTPUT : archive before deleting
DELETE TOP (1000) FROM Sales.OldOrders
OUTPUT DELETED.* INTO Sales.ArchivedOrders
WHERE OrderDate < '2020-01-01';

-- MERGE : update, insert, and conditionally deactivate in one statement
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
```

**Key rules:**
- `OUTPUT` captures the state of rows at the moment of the DML. `INSERTED` = new/updated row; `DELETED` = old/deleted row.
- `$action` in a MERGE OUTPUT returns `'INSERT'`, `'UPDATE'`, or `'DELETE'` : useful for auditing.
- `MERGE` requires the source to produce at most one matching row per target row. Duplicates in source cause a runtime error.
- Always add `OUTPUT` to MERGE in production : it's the only way to know which rows were affected and how.
