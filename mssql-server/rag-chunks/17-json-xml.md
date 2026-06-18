---
topic: JSON and XML : FOR JSON PATH, OPENJSON, JSON_VALUE, FOR XML PATH
keywords: [FOR JSON PATH, FOR JSON AUTO, OPENJSON, JSON_VALUE, JSON_OBJECT, JSON_ARRAY, FOR XML PATH, ROOT, JSON index, computed column]
use_when: Exporting query results as JSON, parsing incoming JSON payloads, indexing a JSON property, or generating XML output.
---

# JSON and XML

```sql
-- FOR JSON PATH: controlled nesting via dot notation
SELECT o.OrderID   AS "id",
       o.OrderDate AS "date",
       c.Name      AS "customer.name",
       c.Email     AS "customer.email"
FROM Sales.Orders o JOIN Sales.Customers c ON c.CustomerID = o.CustomerID
FOR JSON PATH, ROOT('orders');

-- JSON_OBJECT and JSON_ARRAY (SQL Server 2022+)
SELECT JSON_OBJECT('id': ProductID, 'name': Name, 'price': Price) AS json_row
FROM Production.Product WHERE IsActive = 1;

-- Parse JSON input with OPENJSON
DECLARE @json NVARCHAR(MAX) = '[{"id":1,"name":"Widget"},{"id":2,"name":"Gadget"}]';
SELECT j.id, j.name
FROM OPENJSON(@json)
WITH (id INT '$.id', name NVARCHAR(100) '$.name') j;

-- Index a JSON property via persisted computed column
ALTER TABLE Logs.Events ADD Payload NVARCHAR(MAX);
ALTER TABLE Logs.Events ADD EventType AS JSON_VALUE(Payload, '$.type') PERSISTED;
CREATE INDEX IX_Events_Type ON Logs.Events (EventType);

-- FOR XML PATH (use only when downstream strictly requires XML)
SELECT c.Name AS "customer/@name", o.OrderID AS "order/@id", o.TotalAmount AS "order"
FROM Sales.Orders o JOIN Sales.Customers c ON c.CustomerID = o.CustomerID
FOR XML PATH('row'), ROOT('orders');
```

**Key rules:**
- Use `FOR JSON PATH`, not `FOR JSON AUTO`. `AUTO` derives structure from table aliases : it breaks silently when you rename a column or alias.
- `OPENJSON` with an explicit `WITH` clause is strongly typed and far more performant than reading individual `JSON_VALUE()` calls in a WHERE clause.
- `JSON_VALUE` returns `NVARCHAR(4000)`. If the value exceeds 4,000 characters, use `JSON_QUERY` (returns `NVARCHAR(MAX)` fragment).
- A JSON column is just `NVARCHAR(MAX)`. Add a `CHECK (ISJSON(Payload) = 1)` constraint to enforce valid JSON.
