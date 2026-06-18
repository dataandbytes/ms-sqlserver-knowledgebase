---
topic: Custom type system — CREATE TYPE aliases for domain types
keywords: [CREATE TYPE, type alias, user-defined type, Email, PhoneNo, Money, domain type, named type]
use_when: Defining column types using the custom type system, or understanding why columns use type names like Email instead of VARCHAR(100).
---

# Custom Type System

```sql
-- Define domain types once in the database
CREATE TYPE dbo.Email      FROM VARCHAR(100)      NOT NULL;
CREATE TYPE dbo.PhoneNo    FROM VARCHAR(20)        NOT NULL;
CREATE TYPE dbo.PersonName FROM NVARCHAR(100)      NOT NULL;
CREATE TYPE dbo.Money      FROM DECIMAL(18,4)      NOT NULL;
CREATE TYPE dbo.Flag       FROM BIT                NOT NULL;
CREATE TYPE dbo.Timestamp  FROM DATETIME2(0)       NOT NULL;

-- Use in table definitions
CREATE TABLE Sales.Customer (
    CustomerNo   INT            NOT NULL IDENTITY(1,1),
    FullName     dbo.PersonName,
    Email        dbo.Email,
    Phone        dbo.PhoneNo,
    Balance      dbo.Money,
    IsActive     dbo.Flag       DEFAULT 1,
    CreatedAt    dbo.Timestamp  DEFAULT SYSDATETIME(),
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerNo)
);

-- Use in procedure parameters
CREATE OR ALTER PROCEDURE Sales.FindCustomerByEmail
    @Email dbo.Email
AS ...
```

**Why this matters:**
- A column defined as `dbo.Email` is self-documenting. `VARCHAR(100)` is not.
- All `Email` columns are guaranteed to be the same type. Changing the type is one `ALTER TYPE` plus regenerating the affected objects.
- Parameters in stored procedures automatically match the column type — no implicit conversion, no truncation bugs.
- The type name becomes part of the schema contract: if `dbo.Email` is 100 chars today and 200 tomorrow, the change propagates systematically.

**Dropping and recreating types:** SQL Server does not support `ALTER TYPE` for alias types. To change a type, drop dependent objects, drop the type, recreate the type, and recreate the objects. Use idempotent migration scripts with `DROP TYPE IF EXISTS`.
