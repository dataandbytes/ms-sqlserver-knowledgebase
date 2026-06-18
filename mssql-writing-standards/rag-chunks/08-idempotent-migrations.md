---
topic: Idempotent migrations — meta-functions to check existence before ALTER
keywords: [idempotent migration, ColumnExists_fn, TableExists_fn, ForeignKeyExists_fn, IF NOT EXISTS, schema migration, safe migration]
use_when: Writing database migration scripts that can be safely re-run without errors.
---

# Idempotent Migrations

```sql
-- Meta-functions: return 1 if the object exists, 0 otherwise
CREATE OR ALTER FUNCTION dbo.ColumnExists_fn (
    @TableName  NVARCHAR(128),
    @ColumnName NVARCHAR(128)
) RETURNS BIT AS BEGIN
    RETURN (
        SELECT CASE WHEN EXISTS (
            SELECT 1 FROM sys.columns
            WHERE object_id = OBJECT_ID(@TableName)
              AND name = @ColumnName
        ) THEN CAST(1 AS BIT) ELSE CAST(0 AS BIT) END
    );
END;

CREATE OR ALTER FUNCTION dbo.TableExists_fn (
    @TableName NVARCHAR(128)
) RETURNS BIT AS BEGIN
    RETURN (
        SELECT CASE WHEN OBJECT_ID(@TableName, 'U') IS NOT NULL
            THEN CAST(1 AS BIT) ELSE CAST(0 AS BIT) END
    );
END;

CREATE OR ALTER FUNCTION dbo.ForeignKeyExists_fn (
    @FKName NVARCHAR(128)
) RETURNS BIT AS BEGIN
    RETURN (
        SELECT CASE WHEN EXISTS (
            SELECT 1 FROM sys.foreign_keys WHERE name = @FKName
        ) THEN CAST(1 AS BIT) ELSE CAST(0 AS BIT) END
    );
END;

-- Usage in migration scripts
IF dbo.ColumnExists_fn('Sales.Customer', 'LoyaltyTier') = 0
    ALTER TABLE Sales.Customer ADD LoyaltyTier VARCHAR(20) NULL;

IF dbo.TableExists_fn('Messaging.NotificationJob') = 0
    CREATE TABLE Messaging.NotificationJob (...);

IF dbo.ForeignKeyExists_fn('FK_Order_Customer') = 0
    ALTER TABLE Sales.Order ADD CONSTRAINT FK_Order_Customer
        FOREIGN KEY (CustomerNo) REFERENCES Sales.Customer(CustomerNo);
```

**Key rules:**
- Every migration script must be re-runnable. Running it twice should produce the same result as running it once.
- Use `CREATE OR ALTER` for procedures, functions, and views — no existence check needed.
- Use `IF NOT EXISTS` guards for tables, columns, indexes, and constraints.
- Use the meta-functions (`ColumnExists_fn`, etc.) for readability — they make the intent explicit versus embedding `sys.columns` queries inline.
- Log migration execution in a `dbo.SchemaVersion` table with the migration name, applied timestamp, and checksum.
