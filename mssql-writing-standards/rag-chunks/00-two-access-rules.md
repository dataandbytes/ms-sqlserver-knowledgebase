---
topic: The two access rules — all reads via views, all mutations via procedures
keywords: [access rules, views, stored procedures, encapsulation, application design, no direct table access]
use_when: Understanding or explaining the core architectural rule of the mssql-writing-guidelines design pattern.
---

# The Two Access Rules

Every application database in this pattern follows two non-negotiable rules:

1. **All reads go through views.** Application code never selects directly from a base table. Views encapsulate joins, column subsets, and business logic. They also form the contract — renaming a column in the view is a breaking change, but renaming the underlying column is not.

2. **All mutations go through stored procedures.** No `INSERT`, `UPDATE`, or `DELETE` from application code against base tables. Every write operation is a named procedure call. This centralizes validation, error handling, and transaction management.

**Why this matters:**
- You can refactor the physical schema (rename tables, split columns, add audit columns) without touching application code.
- Permissions are simple: grant SELECT on views and EXECUTE on procedures. No table-level grants to applications.
- Every mutation is auditable by name — the call stack in SQL traces shows which procedure wrote which row.

**What this is not:**
- It is not an ORM replacement. The pattern complements an ORM or replaces raw ADO.NET queries.
- It is not about performance. The pattern has no meaningful overhead on modern SQL Server.

**Boundary exceptions:**
- Reporting and analytics tools that join many tables for ad-hoc queries may bypass the view rule for developer convenience — but they must use a read-only role and never write.
- ETL pipelines writing staging tables may write directly — staging tables are not first-class application tables.
