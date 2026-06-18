---
topic: Transaction hierarchy — _trx, _utx, _ut naming and 5-block procedure structure
keywords: [_trx, _utx, _ut, transaction, stored procedure, naming convention, 5-block, DECLARATION, VALIDATION, TRANSACTION, COMMIT, ERROR LABELS]
use_when: Writing a stored procedure that needs a transaction, understanding the _trx/_utx/_ut suffix naming system, or implementing the 5-block procedure structure.
---

# Transaction Hierarchy

## Naming Convention

| Suffix | Meaning | Owns transaction |
|---|---|---|
| `_trx` | Full transaction procedure. Manages BEGIN/COMMIT/ROLLBACK. Called by application or other `_trx` procedures. | Yes |
| `_utx` | Unit of work sub-procedure. Called inside an existing transaction. Does not BEGIN its own transaction. | No — borrows caller's |
| `_ut` | Read-only utility procedure. No writes, no transaction management. | No |

## 5-Block Structure for `_trx`

```sql
CREATE OR ALTER PROCEDURE Sales.TransferFunds_trx
    @FromAccountNo  INT,
    @ToAccountNo    INT,
    @Amount         dbo.Money
AS BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    -- 1. DECLARATION
    DECLARE @ErrNo  INT, @RowCnt INT;
    DECLARE @Msg    NVARCHAR(2048);

    -- 2. VALIDATION
    IF @Amount <= 0 BEGIN
        SET @Msg = 'Amount must be positive';
        GOTO ERROR_VALIDATION;
    END;
    IF NOT EXISTS (SELECT 1 FROM Finance.Account WHERE AccountNo = @FromAccountNo)
        GOTO ERROR_NOT_FOUND;

    -- 3. TRANSACTION
    BEGIN TRANSACTION;
        UPDATE Finance.Account SET Balance = Balance - @Amount
        WHERE AccountNo = @FromAccountNo;
        SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
        IF @ErrNo <> 0 OR @RowCnt = 0 GOTO ERROR_TRX;

        UPDATE Finance.Account SET Balance = Balance + @Amount
        WHERE AccountNo = @ToAccountNo;
        SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR;
        IF @ErrNo <> 0 OR @RowCnt = 0 GOTO ERROR_TRX;

    -- 4. COMMIT
    COMMIT;
    RETURN 0;

    -- 5. ERROR LABELS
    ERROR_VALIDATION:
        RAISERROR(@Msg, 16, 1); RETURN 1;
    ERROR_NOT_FOUND:
        RAISERROR(50003, 16, 1, @FromAccountNo); RETURN 1;
    ERROR_TRX:
        IF @@TRANCOUNT > 0 ROLLBACK;
        RAISERROR(50010, 16, 1); RETURN 1;
END;
```

**Key rules:**
- `SELECT @RowCnt = @@ROWCOUNT, @ErrNo = @@ERROR` must be the very next statement after every DML — `@@ERROR` clears on the next statement.
- `GOTO ERROR_TRX` always checks `@@TRANCOUNT > 0` before ROLLBACK.
- `_utx` procedures skip blocks 3 and 4 — they assume a transaction is already open and only do the DML + error check.
- `_ut` read procedures have no transaction blocks at all.
