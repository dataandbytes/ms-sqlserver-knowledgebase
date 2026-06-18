---
topic: Semantic error codes — sp_addmessage, error code registry 50001-50014
keywords: [sp_addmessage, error codes, semantic errors, RAISERROR, 50001, 50002, error registry, custom messages]
use_when: Using or registering the application-level error code registry, or raising a named semantic error from a stored procedure.
---

# Error Codes

Register all application error messages once at deployment time using `sp_addmessage`:

```sql
EXEC sp_addmessage 50001, 16, 'Record not found: %s';
EXEC sp_addmessage 50002, 16, 'Duplicate record: %s already exists';
EXEC sp_addmessage 50003, 16, 'Invalid state: %s cannot transition to %s';
EXEC sp_addmessage 50004, 16, 'Insufficient balance: account %d has insufficient funds';
EXEC sp_addmessage 50005, 16, 'Access denied: %s';
EXEC sp_addmessage 50006, 16, 'Validation failed: %s';
EXEC sp_addmessage 50007, 16, 'Concurrency conflict: record %s was modified by another user';
EXEC sp_addmessage 50008, 16, 'Quota exceeded: %s';
EXEC sp_addmessage 50009, 16, 'Dependency conflict: %s is referenced by %s';
EXEC sp_addmessage 50010, 16, 'Transaction error: %s';
EXEC sp_addmessage 50011, 16, 'Configuration missing: %s';
EXEC sp_addmessage 50012, 16, 'External service error: %s';
EXEC sp_addmessage 50013, 16, 'Data integrity error: %s';
EXEC sp_addmessage 50014, 16, 'Timeout: %s exceeded maximum wait';
```

**Usage in procedures:**

```sql
-- Raise by message ID (number is stable across environments)
RAISERROR(50001, 16, 1, 'Customer #42');    -- "Record not found: Customer #42"
RAISERROR(50003, 16, 1, 'Pending', 'Shipped'); -- "Invalid state: Pending cannot transition to Shipped"

-- Application code catches by error number, not message text
-- This means: don't parse the message string in application code
```

**Key rules:**
- Register at deployment, not dynamically. Scripts run at startup should be idempotent: `IF NOT EXISTS (SELECT 1 FROM sys.messages WHERE message_id = 50001) EXEC sp_addmessage ...`
- Severity 16 = user-correctable errors (standard for application logic errors).
- Reserve 50015–50099 for future core errors; start domain-specific codes at 50100+.
- The message ID (50001, etc.) is the stable API contract. Message text can be updated without breaking application code.
