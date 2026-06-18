---
topic: Functional constraints — scalar functions used in CHECK constraints
keywords: [functional constraint, CHECK constraint, scalar function, IsType_fn, business rule, constraint function]
use_when: Enforcing a cross-table business rule or complex validation logic as a database constraint rather than application code.
---

# Functional Constraints

SQL Server CHECK constraints can call scalar functions — this allows complex multi-column or cross-table validation to be enforced at the database level:

```sql
-- Function that returns 1 (valid) or 0 (invalid)
CREATE OR ALTER FUNCTION dbo.Account_IsType_fn (
    @AccountNo   INT,
    @AccountType VARCHAR(20)
)
RETURNS BIT AS BEGIN
    RETURN (
        SELECT CASE WHEN EXISTS (
            SELECT 1 FROM Finance.Account a
            JOIN Finance.AccountType t ON t.AccountTypeCode = a.AccountTypeCode
            WHERE a.AccountNo = @AccountNo AND t.Category = @AccountType
        ) THEN CAST(1 AS BIT) ELSE CAST(0 AS BIT) END
    );
END;

-- Use in a CHECK constraint on a dependent table
CREATE TABLE Finance.SavingsAccount (
    AccountNo   INT NOT NULL,
    InterestRate DECIMAL(5,4) NOT NULL,
    CONSTRAINT PK_SavingsAccount PRIMARY KEY (AccountNo),
    CONSTRAINT FK_SavingsAccount_Account
        FOREIGN KEY (AccountNo) REFERENCES Finance.Account(AccountNo),
    CONSTRAINT CK_SavingsAccount_IsType
        CHECK (dbo.Account_IsType_fn(AccountNo, 'Savings') = 1)
);
```

**Key rules:**
- The function must be deterministic or explicitly non-trusted (non-trusted constraints are still enforced but excluded from certain optimizer optimizations).
- Functional constraints are checked on INSERT and UPDATE, not on the function's underlying data changes. If referenced data changes in a way that would violate the constraint, the constraint is not re-checked automatically.
- Use this pattern for: subtype enforcement, cross-table state validation, date range overlap checks.
- For simple column-level rules (range, enum), use a plain CHECK expression without a function — it is cheaper and clearer.
