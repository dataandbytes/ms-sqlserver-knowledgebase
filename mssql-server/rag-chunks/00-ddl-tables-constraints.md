---
topic: DDL : creating tables, schemas, constraints, sequences
keywords: [CREATE TABLE, PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK, DEFAULT, IDENTITY, sequence, schema, ALTER TABLE, computed column]
use_when: You need to define or modify a table, add a column idempotently, create a sequence, or understand constraint types.
---

# DDL: Tables, Schemas, Constraints

```sql
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
               WHERE object_id = OBJECT_ID('Production.Product') AND name = 'Barcode')
    ALTER TABLE Production.Product ADD Barcode VARCHAR(50) NULL;

-- Persisted computed column (stored on disk, indexable)
ALTER TABLE Production.Product ADD
    FullLabel AS (SKU + ' - ' + Name) PERSISTED;

-- Sequence (shared counter across tables, no gaps on rollback)
CREATE SEQUENCE dbo.InvoiceSeq START WITH 100000 INCREMENT BY 1 CYCLE = OFF;
SELECT NEXT VALUE FOR dbo.InvoiceSeq AS NextInvoiceNo;
```

**Key rules:**
- Always name constraints explicitly : unnamed constraints get system-generated names that differ per environment, breaking idempotent scripts.
- `IDENTITY` is scoped to the table; `SEQUENCE` is scoped to the schema and can be shared.
- `PERSISTED` computed columns can be indexed; non-persisted cannot.
- Use `DATETIME2(0)` (seconds precision) not `DATETIME` for new columns : higher precision, no 1900 date quirk.
