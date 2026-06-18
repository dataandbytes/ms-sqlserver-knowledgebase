---
topic: Base/subtype inheritance — shared primary key, AccountType reference table
keywords: [base table, subtype, inheritance, shared primary key, AccountType, reference table, one-to-one, FK to PK]
use_when: Modeling a type hierarchy (e.g., Account→SavingsAccount/LoanAccount) where subtypes share a primary key with the base table.
---

# Base/Subtype Inheritance

```sql
-- Reference table: the allowed types (seeded immediately)
CREATE TABLE Finance.AccountType (
    AccountTypeCode VARCHAR(20) NOT NULL,
    Category        VARCHAR(20) NOT NULL,    -- 'Savings', 'Loan', 'Checking'
    Description     NVARCHAR(200) NOT NULL,
    CONSTRAINT PK_AccountType PRIMARY KEY (AccountTypeCode)
);
INSERT INTO Finance.AccountType VALUES
    ('SAV001', 'Savings',  'Standard savings account'),
    ('LOA001', 'Loan',     'Personal loan account'),
    ('CHK001', 'Checking', 'Everyday transaction account');

-- Base table: common columns + type discriminator
CREATE TABLE Finance.Account (
    AccountNo       INT            NOT NULL IDENTITY(1,1),
    CustomerNo      INT            NOT NULL,
    AccountTypeCode VARCHAR(20)    NOT NULL,
    OpenedAt        dbo.Timestamp  DEFAULT SYSDATETIME(),
    IsActive        dbo.Flag       DEFAULT 1,
    CONSTRAINT PK_Account          PRIMARY KEY (AccountNo),
    CONSTRAINT FK_Account_Customer FOREIGN KEY (CustomerNo) REFERENCES Sales.Customer(CustomerNo),
    CONSTRAINT FK_Account_Type     FOREIGN KEY (AccountTypeCode) REFERENCES Finance.AccountType(AccountTypeCode)
);

-- Subtype table: shares primary key with base (FK-to-PK)
CREATE TABLE Finance.SavingsAccount (
    AccountNo    INT             NOT NULL,   -- same PK as Account
    InterestRate DECIMAL(5,4)   NOT NULL,
    MinBalance   dbo.Money      NOT NULL DEFAULT 0,
    CONSTRAINT PK_SavingsAccount PRIMARY KEY (AccountNo),
    CONSTRAINT FK_SavingsAccount_Account
        FOREIGN KEY (AccountNo) REFERENCES Finance.Account(AccountNo),
    CONSTRAINT CK_SavingsAccount_IsType
        CHECK (dbo.Account_IsType_fn(AccountNo, 'Savings') = 1)
);
```

**Key rules:**
- The subtype table's `AccountNo` is both the PK and an FK to the base table — enforced one-to-one relationship.
- The reference table must be seeded immediately after creation. A reference table with no data is incomplete.
- The functional constraint (`CK_SavingsAccount_IsType`) ensures only accounts of the correct type can have a SavingsAccount row.
- To read a complete savings account: `JOIN Finance.Account ON AccountNo + JOIN Finance.SavingsAccount ON AccountNo`.
- Use this pattern when subtypes have meaningfully different columns. For minor differences (one or two nullable columns), a single table with nullable subtype columns is simpler.
