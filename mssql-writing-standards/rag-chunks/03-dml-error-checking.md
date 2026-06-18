---
topic: DML error checking — @@ROWCOUNT and @@ERROR after every DML statement
keywords: [@@ROWCOUNT, @@ERROR, DML error checking, error handling, UPDATE, INSERT, DELETE, row count check]
use_when: Writing INSERT/UPDATE/DELETE inside a stored procedure and needing to check for errors or unexpected zero-row results.
---

# DML Error Checking

The pattern reads `@@ROWCOUNT` and `@@ERROR` together in a single `SELECT` immediately after every DML statement:

```sql
UPDATE Sales.Order SET Status = 'Shipped', ShippedAt = SYSDATETIME()
WHERE OrderNo = @OrderNo AND CustomerNo = @CustomerNo;
SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
IF @ErrNo <> 0 OR @RowCnt = 0 GOTO ERROR_TRX;
```

**Why a single `SELECT`:**
`@@ERROR` is cleared to 0 by the very next statement. If you read them separately:

```sql
-- WRONG — @@ERROR is 0 by the time you read it
UPDATE ...;
SET @RowCnt = @@ROWCOUNT;    -- this clears @@ERROR
SET @ErrNo  = @@ERROR;       -- always reads 0
```

**Full pattern in context:**

```sql
-- Inside _trx procedure, after BEGIN TRANSACTION
INSERT INTO Sales.OrderLine (OrderNo, CustomerNo, ProductNo, Qty, UnitPrice)
VALUES (@OrderNo, @CustomerNo, @ProductNo, @Qty, @UnitPrice);
SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
IF @ErrNo <> 0 GOTO ERROR_TRX;   -- system error (constraint violation, etc.)

UPDATE Inventory.Stock SET QtyOnHand = QtyOnHand - @Qty
WHERE ProductNo = @ProductNo AND QtyOnHand >= @Qty;
SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
IF @ErrNo <> 0 OR @RowCnt = 0 GOTO ERROR_INSUFFICIENT_STOCK;
```

**When to check `@RowCnt = 0`:**
- UPDATE or DELETE where zero affected rows means a business rule was violated (row not found, concurrent delete, optimistic lock mismatch).
- INSERT into a table with a unique constraint where zero rows means a duplicate — though usually the error is caught via `@ErrNo`.

**When not to check `@RowCnt = 0`:**
- INSERT into a table where zero rows is not an error (conditional insert with WHERE NOT EXISTS — zero rows is the success path).
