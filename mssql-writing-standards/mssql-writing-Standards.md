# MSSQL Application Database Design — Knowledge Base

> Opinionated patterns for building SQL Server application databases: custom types, access
> control through views and procedures, transaction hierarchy, base/subtype inheritance,
> hierarchical composite keys, relational queues, idempotent migrations, and naming conventions.

## Scope

**Use this for:** designing a new SQL Server application schema from scratch; reviewing existing
T-SQL for adherence to type safety and access control; writing stored procedures, views,
migrations, or table hierarchies; implementing background job queues backed by relational tables.

**Not for:** ad-hoc reporting queries, read-only analytical databases, or non-SQL-Server engines.

## Contents

1. [The Two Access Rules](#the-two-access-rules)
2. [Custom Type System](#custom-type-system)
3. [Transaction Hierarchy](#transaction-hierarchy)
4. [Stored Procedure Structure](#stored-procedure-structure)
5. [DML Error Checking](#dml-error-checking)
6. [Functional Constraints](#functional-constraints)
7. [Base/Subtype Inheritance](#basesubtype-inheritance)
8. [Hierarchical Composite Keys](#hierarchical-composite-keys)
9. [Naming Conventions](#naming-conventions)
10. [Role-Scoped Views](#role-scoped-views)
11. [Relational Queues](#relational-queues)
12. [Application Settings](#application-settings)
13. [Idempotent Migrations](#idempotent-migrations)
14. [Error Codes](#error-codes)
15. [Query Patterns](#query-patterns)
16. [Common Mistakes](#common-mistakes)

---

## The Two Access Rules

These two rules are the foundation of the entire methodology:

1. **All reads go through views.** Never SELECT directly from tables in application code.
2. **All mutations go through stored procedures.** No ad-hoc INSERT, UPDATE, or DELETE.

Tables are an implementation detail. You can restructure them freely — rename columns, add audit columns, split a table — as long as views and procedures maintain their contracts. Application code never breaks because it never touched the tables.

```sql
-- WRONG: application code directly querying a table
SELECT CustomerNo, FullName, Email FROM Customer WHERE Email = @Email;

-- CORRECT: application code calls a procedure which reads through a view
EXEC FindCustomerByEmail_ut @Email = 'alice@example.com';

-- The procedure reads through the role-scoped view
SELECT CustomerNo, FullName, Email
FROM Admin_AllCustomers_V
WHERE Email = @Email;
```

---

## Custom Type System

Never use bare built-in types (`VARCHAR`, `INT`, `DATETIME`, `BIT`) for columns. Define named aliases that form a semantic layer. A column typed `Email` communicates intent and prevents swapping an `Email` column with a `Name` column by accident.

```sql
-- Define type aliases once per database
CREATE TYPE Email        FROM VARCHAR(100)   NOT NULL;
CREATE TYPE Name         FROM VARCHAR(100)   NOT NULL;
CREATE TYPE Description  FROM VARCHAR(500)   NOT NULL;
CREATE TYPE AccountNo    FROM INT            NOT NULL;
CREATE TYPE _Int         FROM INT            NOT NULL;
CREATE TYPE _Money       FROM DECIMAL(12,2)  NOT NULL;
CREATE TYPE _Bool        FROM BIT            NOT NULL;
CREATE TYPE _Timestamp   FROM DATETIME2      NOT NULL;
CREATE TYPE _Type        FROM VARCHAR(50)    NOT NULL;
CREATE TYPE DbUserID     FROM INT            NOT NULL;

-- Use types in table definitions — never bare built-ins
CREATE TABLE Customer (
    CustomerNo  AccountNo    PRIMARY KEY,
    FullName    Name         NOT NULL,
    Email       Email        NOT NULL,
    CreatedAt   _Timestamp   NOT NULL DEFAULT SYSDATETIME(),
    IsVerified  _Bool        NOT NULL DEFAULT 0,
    OwnerID     DbUserID     NOT NULL DEFAULT DATABASE_PRINCIPAL_ID()
);

-- NOT NULL by default. Nullable only with an explicit business reason.
-- A nullable Phone means "phone is optional for this customer" — document it.
ALTER TABLE Customer ADD Phone VARCHAR(20) NULL; -- nullable: phone is optional
```

**Type defaults** — attach a standard default to every type so no column ever needs an ad-hoc `DEFAULT`:

| Type | Default |
|---|---|
| `_Timestamp` | `SYSDATETIME()` |
| `_Bool` | `0` |
| `_Int` | `0` |
| `DbUserID` | `DATABASE_PRINCIPAL_ID()` |

---

## Transaction Hierarchy

Procedures are classified by their relationship to transactions using a suffix:

| Suffix | Role | Validates |
|---|---|---|
| `_trx` | Transaction owner — opens and commits/rolls back | `@@TRANCOUNT = 0` (refuses if already in a transaction) |
| `_utx` | Transaction participant — runs inside an existing transaction | `@@TRANCOUNT > 0` (refuses if NOT in a transaction) |
| `_ut` | Utility — no transaction requirement | No check |

The suffix is the contract. When you see `ProcessOrder_trx`, you know it owns the boundary. When you see `AddOrderLine_utx`, you know the caller must wrap it.

```sql
-- _trx opens the transaction and calls _utx procedures for subtasks
CREATE OR ALTER PROCEDURE ProcessOrder_trx
    @CustomerNo AccountNo,
    @LineItems  dbo.OrderLineTableType READONLY
AS BEGIN
    DECLARE @ErrNo INT;
    DECLARE @RowCnt INT;
    DECLARE @OrderNo _Int;

    IF (@@TRANCOUNT > 0) BEGIN
        RAISERROR(50012, 16, 1, 'ProcessOrder_trx'); GOTO EXIT_ERROR;
    END

    BEGIN TRANSACTION ProcessOrder_trx;

        EXEC @ErrNo = AddOrder_utx
            @CustomerNo = @CustomerNo,
            @OrderNo    = @OrderNo OUTPUT;
        IF (@ErrNo <> 0) GOTO EXIT_TRANSACTION;

        INSERT INTO OrderLine (CustomerNo, OrderNo, ProductID, Qty)
        SELECT @CustomerNo, @OrderNo, ProductID, Qty FROM @LineItems;
        SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
        IF (@ErrNo <> 0) GOTO EXIT_TRANSACTION;

    COMMIT TRANSACTION ProcessOrder_trx;
    RETURN 0;

EXIT_TRANSACTION:
    ROLLBACK TRANSACTION ProcessOrder_trx;
EXIT_ERROR:
    RETURN 1;
END;
```

---

## Stored Procedure Structure

Every procedure follows the same 5-block layout in order:

```
BLOCK 1 — DECLARATION   DECLARE @ErrNo INT; DECLARE @RowCnt INT;
BLOCK 2 — VALIDATION    Parameter NULL checks; @@TRANCOUNT check
BLOCK 3 — TRANSACTION   BEGIN TRANSACTION; XLOCK/HOLDLOCK; business logic
BLOCK 4 — COMMIT        COMMIT; RETURN 0;
BLOCK 5 — ERROR LABELS  EXIT_TRANSACTION: ROLLBACK; EXIT_ERROR: RETURN 1;
```

```sql
-- _utx example: participant procedure (no transaction open/close)
CREATE OR ALTER PROCEDURE AddOrder_utx
    @CustomerNo AccountNo,
    @OrderNo    _Int OUTPUT
AS BEGIN
    DECLARE @ErrNo INT;
    DECLARE @RowCnt INT;

    -- BLOCK 2: VALIDATION
    IF (@@TRANCOUNT = 0) BEGIN
        RAISERROR(50013, 16, 1, 'AddOrder_utx'); GOTO EXIT_ERROR;
    END
    IF @CustomerNo IS NULL BEGIN
        RAISERROR(50002, 16, 1, 'AddOrder_utx: CustomerNo'); GOTO EXIT_ERROR;
    END

    -- BLOCK 3: BUSINESS LOGIC (no BEGIN TRANSACTION — caller owns it)
    SET @OrderNo = ISNULL((SELECT MAX(OrderNo) FROM [Order]
                           WHERE CustomerNo = @CustomerNo), 0) + 1;

    INSERT INTO [Order] (CustomerNo, OrderNo, CreatedAt)
    VALUES (@CustomerNo, @OrderNo, SYSDATETIME());

    SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
    IF (@ErrNo <> 0) GOTO EXIT_ERROR;
    IF (@RowCnt <> 1) BEGIN
        RAISERROR(50004, 16, 1, 'AddOrder_utx: Order'); GOTO EXIT_ERROR;
    END

    RETURN 0;
EXIT_ERROR:
    RETURN 1;
END;
```

```sql
-- _ut example: utility/read procedure
CREATE OR ALTER PROCEDURE FindCustomerByEmail_ut
    @Email Email
AS BEGIN
    IF @Email IS NULL BEGIN
        RAISERROR(50002, 16, 1, 'FindCustomerByEmail_ut: Email'); GOTO EXIT_ERROR;
    END

    SELECT CustomerNo, FullName, Email
    FROM Admin_AllCustomers_V
    WHERE Email = @Email;

    RETURN 0;
EXIT_ERROR:
    RETURN 1;
END;
```

---

## DML Error Checking

After **every** INSERT, UPDATE, or DELETE, capture `@@ROWCOUNT` and `@@ERROR` in a single statement. Both values reset after any successful T-SQL statement — including `SET`.

```sql
-- Pattern: capture both in one SELECT, then check
UPDATE Account SET Balance = Balance - @Amount WHERE AccountNo = @AccountNo;
SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;

IF (@ErrNo <> 0) GOTO EXIT_TRANSACTION;      -- SQL error (constraint, deadlock, etc.)
IF (@RowCnt = 0) BEGIN                         -- row existed but didn't match
    RAISERROR(50005, 16, 1, 'DebitAccount_utx: Account');
    GOTO EXIT_TRANSACTION;
END

-- Row count expectations by DML type:
-- INSERT single row:  @RowCnt = 1   → RAISERROR 50004 (EXIT_NOT_ADDED)
-- UPDATE:             @RowCnt >= 1  → RAISERROR 50005 (EXIT_NOT_MODIFIED)
-- DELETE:             @RowCnt >= 1  → RAISERROR 50006 (EXIT_NOT_REMOVED)
```

**AddOrModify with MERGE** — when the caller does not know whether the record exists:

```sql
CREATE OR ALTER PROCEDURE AddOrModify_ProductCategory_trx
    @CategoryID _Int,
    @Name       Name,
    @ParentID   _Int
AS BEGIN
    DECLARE @ErrNo INT; DECLARE @RowCnt INT;
    IF (@@TRANCOUNT > 0) BEGIN RAISERROR(50012,16,1,'AddOrModify_ProductCategory_trx'); GOTO EXIT_ERROR; END

    BEGIN TRANSACTION AddOrModify_ProductCategory_trx;

        MERGE ProductCategory AS tgt
        USING (SELECT @CategoryID, @Name, @ParentID) AS src(CategoryID, Name, ParentID)
        ON tgt.CategoryID = src.CategoryID
        WHEN MATCHED THEN
            UPDATE SET Name = src.Name, ParentID = src.ParentID
        WHEN NOT MATCHED THEN
            INSERT (CategoryID, Name, ParentID) VALUES (src.CategoryID, src.Name, src.ParentID);

        SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
        IF (@ErrNo <> 0) GOTO EXIT_TRANSACTION;
        IF (@RowCnt <> 1) BEGIN
            RAISERROR(50005, 16, 1, 'AddOrModify_ProductCategory_trx');
            GOTO EXIT_TRANSACTION;
        END

    COMMIT TRANSACTION AddOrModify_ProductCategory_trx;
    RETURN 0;
EXIT_TRANSACTION: ROLLBACK TRANSACTION AddOrModify_ProductCategory_trx;
EXIT_ERROR: RETURN 1;
END;
```

---

## Functional Constraints

SQL's built-in constraints are single-table. Functional constraints extend this by calling a scalar function inside a CHECK constraint — enforcing cross-table business rules at the schema level.

```sql
-- Function: checks a fact about another table
CREATE OR ALTER FUNCTION dbo.Account_IsType_fn (
    @AccountNo   AccountNo,
    @ExpectedType _Type
) RETURNS BIT AS BEGIN
    IF EXISTS (SELECT 1 FROM Account
               WHERE AccountNo = @AccountNo AND [Type] = @ExpectedType)
        RETURN 1;
    RETURN 0;
END;

-- Constraint: enforces the type discriminator at insert time
-- A row in SavingsAccount can only reference an Account with Type = 'Savings'
CONSTRAINT SavingsAccount_IsAccountType
    CHECK (dbo.Account_IsType_fn(AccountNo, 'Savings') = 1)

-- State machine: only certain transitions are valid
CREATE OR ALTER FUNCTION dbo.Order_IsValidTransition_fn (
    @OrderNo   AccountNo,
    @NewStatus _Type
) RETURNS BIT AS BEGIN
    DECLARE @Current _Type;
    SELECT @Current = Status FROM [Order] WHERE OrderNo = @OrderNo;
    IF (@Current = 'Pending'    AND @NewStatus = 'Processing') RETURN 1;
    IF (@Current = 'Processing' AND @NewStatus = 'Shipped')    RETURN 1;
    IF (@Current = 'Processing' AND @NewStatus = 'Cancelled')  RETURN 1;
    IF (@Current = 'Shipped'    AND @NewStatus = 'Delivered')  RETURN 1;
    RETURN 0;
END;
```

Use functional constraints for: type discriminator enforcement across base/subtype tables;
cross-table existence checks; state machine transition validation; business rules that span
multiple tables.

---

## Base/Subtype Inheritance

When entities share common attributes but have specialized ones, use primary key inheritance
instead of nullable columns. A base table holds shared attributes and a type discriminator.
Each subtype table's PK is both the PK and FK to the base, enforced by a functional constraint.

```sql
-- Base table: shared attributes + type discriminator
CREATE TABLE Account (
    AccountNo  AccountNo  PRIMARY KEY,
    CustomerNo AccountNo  NOT NULL,
    [Type]     _Type      NOT NULL,
    Balance    _Money     NOT NULL DEFAULT 0,
    Status     _Type      NOT NULL DEFAULT 'Active',
    CreatedAt  _Timestamp NOT NULL DEFAULT SYSDATETIME(),

    CONSTRAINT Account_IsClassifiedBy_AccountType
        FOREIGN KEY([Type]) REFERENCES AccountType([Type]),
    CONSTRAINT Customer_Holds_Account
        FOREIGN KEY(CustomerNo) REFERENCES Customer(CustomerNo)
);

-- Reference table seeded immediately (FK can't work without data)
CREATE TABLE AccountType ([Type] _Type PRIMARY KEY);
INSERT INTO AccountType([Type]) VALUES ('Savings'), ('Checking'), ('MoneyMarket');

-- Subtype: PK = FK to base + functional type check
CREATE TABLE SavingsAccount (
    AccountNo    AccountNo    PRIMARY KEY,
    InterestRate DECIMAL(5,4) NOT NULL,
    MinBalance   _Money       NOT NULL,

    CONSTRAINT SavingsAccount_Is_Account
        FOREIGN KEY(AccountNo) REFERENCES Account(AccountNo),
    CONSTRAINT SavingsAccount_IsAccountType
        CHECK (dbo.Account_IsType_fn(AccountNo, 'Savings') = 1)
);

-- Foreign keys to subtype vs base:
-- Reference Account       → any account type
-- Reference SavingsAccount → savings accounts only
```

Benefits: every subtype has its own table with clean NOT NULL constraints. No nullable columns
representing "only relevant for type X". Joins are explicit and self-documenting.

---

## Hierarchical Composite Keys

Child tables inherit the full primary key of their parent and add a discriminator column.
The key is a path that encodes lineage from root to leaf.

```sql
-- Key hierarchy encodes the ownership chain
-- Customer(CustomerNo) → Order(CustomerNo, OrderNo) → OrderLine(CustomerNo, OrderNo, LineNo)

CREATE TABLE Customer (
    CustomerNo AccountNo PRIMARY KEY CLUSTERED
);

CREATE TABLE [Order] (
    CustomerNo AccountNo NOT NULL,
    OrderNo    _Int      NOT NULL,
    OrderDate  _Timestamp NOT NULL DEFAULT SYSDATETIME(),
    PRIMARY KEY CLUSTERED (CustomerNo, OrderNo),
    CONSTRAINT Customer_Places_Order FOREIGN KEY(CustomerNo) REFERENCES Customer(CustomerNo)
);

CREATE TABLE OrderLine (
    CustomerNo AccountNo NOT NULL,
    OrderNo    _Int      NOT NULL,
    LineNo     _Int      NOT NULL,
    ProductID  _Int      NOT NULL,
    Qty        _Int      NOT NULL,
    PRIMARY KEY CLUSTERED (CustomerNo, OrderNo, LineNo),
    CONSTRAINT Order_Contains_OrderLine
        FOREIGN KEY(CustomerNo, OrderNo) REFERENCES [Order](CustomerNo, OrderNo)
);
```

**Max-plus-one functions** replace IDENTITY — scoped to the parent key so children within one
order are numbered 1, 2, 3 independently of other orders:

```sql
-- Line number generator scoped to the parent (CustomerNo, OrderNo)
CREATE OR ALTER FUNCTION dbo.NextLineNo_fn (
    @CustomerNo AccountNo,
    @OrderNo    _Int
) RETURNS _Int AS BEGIN
    RETURN ISNULL(
        (SELECT MAX(LineNo) FROM OrderLine
         WHERE CustomerNo = @CustomerNo AND OrderNo = @OrderNo),
        0) + 1;
END;
```

Composite clustered keys group parent-child data contiguously on disk. All order lines for
`CustomerNo = 42, OrderNo = 7` are on adjacent pages — one sequential read retrieves the
full order.

---

## Naming Conventions

Everything is PascalCase. Underscores appear only in suffixes and structural separators.

| Object | Pattern | Examples |
|---|---|---|
| Table | `EntityName` | `Account`, `Customer`, `OrderLine` |
| View | `Role_Intent_V` | `Manager_TeamReport_V`, `Admin_AllCustomers_V` |
| Procedure | `Verb_Domain_{trx,utx,ut}` | `Add_OrderLine_trx`, `FindCustomer_ut` |
| Function | `Descriptive_fn` | `Account_IsType_fn`, `NextOrderNo_fn` |
| Constraint | `Subject_Predicate_Object` | `Customer_Rents_Vehicle`, `SavingsAccount_Is_Account` |

**Constraint names as business predicates** — readable as sentences in error logs:

```sql
-- BAD: describes mechanism, not meaning
CONSTRAINT FK_Rental_Customer FOREIGN KEY ...

-- GOOD: describes the business rule
CONSTRAINT Customer_Rents_Vehicle FOREIGN KEY (CustomerNo) REFERENCES Customer(CustomerNo)
CONSTRAINT Customer_MustHave_ValidEmail CHECK (Email LIKE '%_@_%.__%')
CONSTRAINT Customer_Email_IsUnique UNIQUE (Email)
CONSTRAINT SavingsAccount_Is_Account FOREIGN KEY(AccountNo) REFERENCES Account(AccountNo)
```

**Verbs for procedures** — avoid SQL keywords as procedure verbs:

| Avoid | Use |
|---|---|
| `Create` | `Add` |
| `Update` | `Modify` |
| `Delete` | `Remove` |
| `Select` | `Find` |

Prefer `AddOrModify_` with MERGE over separate `Add_` and `Modify_` procedures.

---

## Role-Scoped Views

Views are prefixed with the role they serve. Permissions are self-documenting from the name.

```sql
-- Manager: sees all customers and their orders
CREATE OR ALTER VIEW Manager_CustomerSummary_V AS
SELECT c.CustomerNo, c.FullName, c.Email,
       COUNT(o.OrderNo)         AS OrderCount,
       SUM(ol.Qty * p.Price)    AS TotalSpend
FROM Customer c
LEFT JOIN [Order]    o  ON o.CustomerNo = c.CustomerNo
LEFT JOIN OrderLine  ol ON ol.CustomerNo = o.CustomerNo AND ol.OrderNo = o.OrderNo
LEFT JOIN Product    p  ON p.ProductID  = ol.ProductID
GROUP BY c.CustomerNo, c.FullName, c.Email;

-- Customer: sees only their own orders (row-level security baked in)
CREATE OR ALTER VIEW Customer_MyOrders_V AS
SELECT o.CustomerNo, o.OrderNo, o.OrderDate,
       SUM(ol.Qty * p.Price) AS OrderTotal
FROM [Order]   o
JOIN OrderLine ol ON ol.CustomerNo = o.CustomerNo AND ol.OrderNo = o.OrderNo
JOIN Product   p  ON p.ProductID   = ol.ProductID
WHERE
    USER_NAME() IN ('__sysadmin', 'dbo', '__worker')
    OR IS_ROLEMEMBER('db_securityadmin') = 1
    OR o.CustomerNo = (SELECT CustomerNo FROM Customer WHERE OwnerID = USER_ID())
GROUP BY o.CustomerNo, o.OrderNo, o.OrderDate;

-- Worker: pending jobs it can claim
CREATE OR ALTER VIEW Worker_PendingJobs_V AS
SELECT JobID, JobType, ScheduledFor, QueuedAt, AttemptNum
FROM Job
WHERE Status = 'Pending'
  AND ScheduledFor <= SYSDATETIME();
```

---

## Relational Queues

A relational queue is a regular table with queue lifecycle columns. Workers claim items atomically using `READPAST` + `UPDLOCK` so multiple concurrent consumers never pick the same row.

```sql
CREATE TABLE NotificationJob (
    JobID       _Int       PRIMARY KEY,
    CustomerNo  AccountNo  NOT NULL,
    JobType     _Type      NOT NULL,
    Status      _Type      NOT NULL DEFAULT 'Pending',  -- Pending, Processing, Done, Failed
    AttemptNum  _Int       NOT NULL DEFAULT 0,
    ScheduledFor _Timestamp NOT NULL DEFAULT SYSDATETIME(),
    QueuedAt    _Timestamp NOT NULL DEFAULT SYSDATETIME(),
    StartedAt   _Timestamp NULL,
    FinishedAt  _Timestamp NULL,
    Response    Description NULL,
    Error       Description NULL,

    CONSTRAINT NotificationJob_Status_IsValid
        CHECK (Status IN ('Pending','Processing','Done','Failed'))
);

-- Worker claims the next available item atomically
CREATE OR ALTER PROCEDURE Next_NotificationJob_ut
    @JobID _Int OUTPUT
AS BEGIN
    SET @JobID = NULL;
    BEGIN TRANSACTION;
        SELECT TOP (1) @JobID = JobID
        FROM NotificationJob WITH (READPAST, ROWLOCK, UPDLOCK)
        WHERE Status = 'Pending' AND ScheduledFor <= SYSDATETIME()
        ORDER BY QueuedAt;

        IF @JobID IS NOT NULL
            UPDATE NotificationJob
            SET Status = 'Processing', StartedAt = SYSDATETIME(), AttemptNum = AttemptNum + 1
            WHERE JobID = @JobID;
    COMMIT;
    RETURN 0;
END;

-- Worker reports result
CREATE OR ALTER PROCEDURE Modify_NotificationJob_Done_utx
    @JobID    _Int,
    @Response Description
AS BEGIN
    DECLARE @ErrNo INT; DECLARE @RowCnt INT;
    IF (@@TRANCOUNT = 0) BEGIN RAISERROR(50013,16,1,'Modify_NotificationJob_Done_utx'); GOTO EXIT_ERROR; END

    UPDATE NotificationJob
    SET Status = 'Done', FinishedAt = SYSDATETIME(), Response = @Response
    WHERE JobID = @JobID AND Status = 'Processing';

    SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
    IF (@ErrNo <> 0) GOTO EXIT_ERROR;
    IF (@RowCnt <> 1) BEGIN RAISERROR(50005,16,1,'Modify_NotificationJob_Done_utx'); GOTO EXIT_ERROR; END

    RETURN 0;
EXIT_ERROR: RETURN 1;
END;
```

`READPAST` skips locked rows instead of waiting, so a second worker immediately moves to
the next available item. `UPDLOCK` acquires an update lock on the row it reads, preventing
another session from also selecting it before the UPDATE runs.

---

## Application Settings

A centralized table for runtime configuration: retry limits, feature flags, endpoints, batch sizes. Procedures read from it rather than hardcoding values.

```sql
CREATE TABLE AppSettings (
    Param    Name    PRIMARY KEY,
    ValBool  _Bool   NOT NULL DEFAULT 0,
    ValInt   _Int    NOT NULL DEFAULT 0,
    ValFloat FLOAT   NOT NULL DEFAULT 0,
    ValStr   Description NOT NULL DEFAULT ''
);

-- Namespaced keys prevent collisions between modules
INSERT INTO AppSettings (Param, ValInt, ValStr) VALUES
    ('notification.maxAttempts', 3, ''),
    ('notification.retryDelaySec', 300, ''),
    ('smtp.host', 0, 'smtp.internal.example.com'),
    ('batch.chunkSize', 5000, ''),
    ('feature.darkMode', 0, '');

-- Reading with COALESCE provides a default if the row is missing
DECLARE @MaxAttempts _Int;
SELECT @MaxAttempts = COALESCE(ValInt, 3)
FROM AppSettings WHERE Param = 'notification.maxAttempts';
```

---

## Idempotent Migrations

Every migration script must be safe to run multiple times. Wrap DDL in meta-function existence checks.

```sql
-- Meta functions query sys.* catalog views

CREATE OR ALTER FUNCTION dbo.ColumnExists_fn(@Table Name, @Column Name) RETURNS _Bool AS BEGIN
    IF EXISTS (SELECT 1 FROM INFORMATION_SCHEMA.COLUMNS
               WHERE TABLE_NAME = @Table AND COLUMN_NAME = @Column) RETURN 1;
    RETURN 0;
END;

CREATE OR ALTER FUNCTION dbo.TableExists_fn(@Table Name) RETURNS _Bool AS BEGIN
    IF EXISTS (SELECT 1 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = @Table) RETURN 1;
    RETURN 0;
END;

CREATE OR ALTER FUNCTION dbo.ForeignKeyExists_fn(@Name Name) RETURNS _Bool AS BEGIN
    IF EXISTS (SELECT 1 FROM sys.foreign_keys WHERE name = @Name) RETURN 1;
    RETURN 0;
END;

-- Usage: safe to rerun — only alters if not already present
IF dbo.ColumnExists_fn('Customer', 'PhoneVerified') = 0
    ALTER TABLE Customer ADD PhoneVerified _Bool NOT NULL DEFAULT 0;

IF dbo.TableExists_fn('CustomerSegment') = 0
BEGIN
    CREATE TABLE CustomerSegment (
        CustomerNo  AccountNo  PRIMARY KEY,
        Segment     _Type      NOT NULL DEFAULT 'Standard'
    );
END

IF dbo.ForeignKeyExists_fn('CustomerSegment_Is_Customer') = 0
    ALTER TABLE CustomerSegment ADD CONSTRAINT CustomerSegment_Is_Customer
        FOREIGN KEY(CustomerNo) REFERENCES Customer(CustomerNo);

-- Validation workflow after writing any migration:
-- 1. Run once — confirm no errors
-- 2. Run again — confirm idempotency (no errors, no duplicate objects)
-- 3. Verify: SELECT dbo.ColumnExists_fn('Customer', 'PhoneVerified')  -- must be 1
-- 4. Verify: SELECT dbo.ForeignKeyExists_fn('CustomerSegment_Is_Customer') -- must be 1
```

---

## Error Codes

Register semantic error codes via `sp_addmessage`. Each code names what went wrong — client code matches on the number to provide meaningful feedback.

```sql
EXEC sp_addmessage 50001, 16, 'EXIT_UNAUTHORISED: %s is not authorised to perform this action.';
EXEC sp_addmessage 50002, 16, 'EXIT_NULL_PARAM: %s is required and cannot be null.';
EXEC sp_addmessage 50003, 16, 'EXIT_NOT_FOUND: %s was not found.';
EXEC sp_addmessage 50004, 16, 'EXIT_NOT_ADDED: Failed to insert %s.';
EXEC sp_addmessage 50005, 16, 'EXIT_NOT_MODIFIED: Failed to update %s.';
EXEC sp_addmessage 50006, 16, 'EXIT_NOT_REMOVED: Failed to delete %s.';
EXEC sp_addmessage 50007, 16, 'EXIT_DUPLICATE: %s already exists.';
EXEC sp_addmessage 50008, 16, 'EXIT_CONSTRAINT: %s violates a business constraint.';
EXEC sp_addmessage 50009, 16, 'EXIT_INVALID_STATE: %s is in an invalid state for this operation.';
EXEC sp_addmessage 50010, 16, 'EXIT_INVALID_VALUE: %s has an invalid value.';
EXEC sp_addmessage 50011, 16, 'EXIT_INSUFFICIENT: %s has insufficient value.';
EXEC sp_addmessage 50012, 16, 'EXIT_IN_TRANSACTION: %s must not be called inside a transaction.';
EXEC sp_addmessage 50013, 16, 'EXIT_NOT_IN_TRANSACTION: %s must be called inside a transaction.';
EXEC sp_addmessage 50014, 16, 'EXIT_EXPIRED: %s has expired or exceeded its time limit.';
```

---

## Query Patterns

```sql
-- SARGable WHERE: never wrap the column in a function
-- BAD:  WHERE YEAR(CreatedAt) = 2025                    (forces scan)
-- GOOD: WHERE CreatedAt >= '2025-01-01' AND CreatedAt < '2026-01-01' (seeks)

-- NOT EXISTS instead of NOT IN (NOT IN silently returns no rows if subquery has NULLs)
-- BAD:  WHERE CustomerNo NOT IN (SELECT CustomerNo FROM BlackList)
-- GOOD:
SELECT CustomerNo, FullName
FROM Customer c
WHERE NOT EXISTS (
    SELECT 1 FROM BlackList b WHERE b.CustomerNo = c.CustomerNo
);

-- Window function: running total without a self-join
SELECT OrderNo, OrderDate, TotalAmt,
       SUM(TotalAmt) OVER (
           PARTITION BY CustomerNo
           ORDER BY OrderDate
           ROWS UNBOUNDED PRECEDING
       ) AS CustomerRunningTotal
FROM [Order];

-- Batch UPDATE: avoid a single statement locking the whole table
DECLARE @Rows INT = 1;
WHILE @Rows > 0
BEGIN
    UPDATE TOP (5000) Customer
    SET IsVerified = 1
    WHERE IsVerified = 0 AND VerifiedAt IS NOT NULL;
    SET @Rows = @@ROWCOUNT;
END;

-- Parameter sniffing fix: OPTIMIZE FOR UNKNOWN (preferred over local variable copy)
CREATE OR ALTER PROCEDURE FindOrdersByCustomer_ut @CustomerNo AccountNo
AS BEGIN
    SELECT OrderNo, OrderDate, Status
    FROM Admin_AllOrders_V
    WHERE CustomerNo = @CustomerNo
    OPTION (OPTIMIZE FOR (@CustomerNo UNKNOWN));
END;
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Bare `VARCHAR`, `INT`, `DATETIME` columns | Use named custom types: `Email`, `AccountNo`, `_Timestamp` |
| TRY-CATCH for deterministic DML | Use GOTO with `@@ROWCOUNT`/`@@ERROR` checks after every DML |
| Constraint names like `FK_Table_Other` | Use business predicates: `Customer_Rents_Vehicle` |
| IDENTITY for child table keys | Max-plus-one functions scoped to the parent key |
| Not seeding reference tables | INSERT known values in the same DDL script as the table |
| SELECT directly from tables in application code | All reads through views; all mutations through procedures |
| `Create`, `Update`, `Delete` as procedure verbs | Use `Add`, `Modify`, `Remove` — avoid SQL keyword collisions |
| Nullable columns by default | Types are NOT NULL; nullable only with an explicit business reason |
| Hardcoding config values in procedures | Read from `AppSettings` with a `COALESCE` default |
| Polymorphic nullable columns for subtypes | Use base/subtype with primary key inheritance |
| Wrapping column in function in WHERE | Apply the function to the parameter side — keep the column naked |
| `NOT IN` with a nullable subquery | Use `NOT EXISTS` — `NOT IN` returns nothing when subquery has NULLs |
| Non-idempotent migration DDL | Wrap every ALTER with a meta-function existence check |
| Global transaction in `_utx` called by a `_trx` | `_utx` must not open its own transaction — the caller owns the boundary |
