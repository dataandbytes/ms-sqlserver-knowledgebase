---
topic: Security : logins, roles, RLS, dynamic data masking, TDE
keywords: [CREATE LOGIN, CREATE USER, GRANT, DENY, REVOKE, row-level security, RLS, dynamic data masking, TDE, transparent data encryption, SESSION_CONTEXT]
use_when: Setting up database users and roles, restricting row access with RLS, masking PII columns, or enabling encryption at rest.
---

# Security

```sql
-- Login + user + role-based permissions
CREATE LOGIN AppUser WITH PASSWORD = 'S3cur3P@ss!';
CREATE USER  AppUser FOR LOGIN AppUser;
CREATE ROLE  ReportReader;
GRANT SELECT ON SCHEMA::Reporting TO ReportReader;
ALTER ROLE ReportReader ADD MEMBER AppUser;
DENY SELECT ON Sales.Orders TO AppUser;   -- DENY overrides GRANT

-- Row-Level Security: filter rows by tenant from session context
CREATE FUNCTION Security.fn_TenantFilter (@TenantID INT)
RETURNS TABLE WITH SCHEMABINDING AS RETURN (
    SELECT 1 AS allow
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT)
       OR IS_ROLEMEMBER('db_owner') = 1
);
CREATE SECURITY POLICY Sales.TenantPolicy
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID) ON Sales.Orders,
ADD BLOCK  PREDICATE Security.fn_TenantFilter(TenantID) ON Sales.Orders AFTER INSERT
WITH (STATE = ON);

-- Set tenant context at connection time (application code sets this)
EXEC sys.sp_set_session_context N'TenantID', 7;

-- Dynamic Data Masking: underlying data unchanged, display masked per user privilege
ALTER TABLE Sales.Customers ALTER COLUMN Email
    ADD MASKED WITH (FUNCTION = 'email()');
ALTER TABLE Sales.Customers ALTER COLUMN Phone
    ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)');
GRANT UNMASK TO PrivilegedReportUser;

-- TDE: encrypt the data file at rest
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongKeyPass1!';
CREATE CERTIFICATE TDECert WITH SUBJECT = 'TDE Certificate';
CREATE DATABASE ENCRYPTION KEY WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE TDECert;
ALTER DATABASE YourDB SET ENCRYPTION ON;
SELECT name, encryption_state_desc, percent_complete
FROM sys.dm_database_encryption_keys;
```

**Key rules:**
- Grant permissions to roles, never directly to users : it makes auditing and changes manageable.
- RLS filter predicates run for every SELECT on the table; keep them simple (single equality check) to avoid performance impact.
- Dynamic Data Masking does not encrypt data : a user with SELECT can read values via side channels. It is a display-layer control.
- Back up the TDE certificate immediately after creating it. Without it, the database cannot be restored on another server.
