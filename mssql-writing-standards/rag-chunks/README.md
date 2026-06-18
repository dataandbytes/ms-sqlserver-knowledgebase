# RAG Chunks Index: Writing Standards and Application Design

11 topic chunks for writing SQL Server application code as a system rather than ad-hoc queries. Each file is self-contained with YAML frontmatter (`topic`, `keywords`, `use_when`) for AI context selection.

| # | File | Topic | Use When |
|---|------|-------|----------|
| 00 | [two-access-rules](00-two-access-rules.md) | All reads via views, all mutations via procedures | Understanding the core architectural rule |
| 01 | [custom-type-system](01-custom-type-system.md) | CREATE TYPE aliases for domain types | Defining column types using domain-specific type names |
| 02 | [transaction-hierarchy](02-transaction-hierarchy.md) | _trx/_utx/_ut naming, 5-block procedure structure | Writing a procedure that needs a transaction |
| 03 | [dml-error-checking](03-dml-error-checking.md) | @@ROWCOUNT and @@ERROR after every DML statement | Checking for errors or unexpected zero-row results in procedures |
| 04 | [functional-constraints](04-functional-constraints.md) | Scalar functions used in CHECK constraints | Enforcing cross-table business rules as database constraints |
| 05 | [base-subtype-inheritance](05-base-subtype-inheritance.md) | Shared primary key, AccountType reference table | Modeling a type hierarchy with shared base table keys |
| 06 | [hierarchical-composite-keys](06-hierarchical-composite-keys.md) | Max-plus-one functions, child keys scoped to parent | Generating the next line number within an order |
| 07 | [relational-queues](07-relational-queues.md) | READPAST + UPDLOCK + ROWLOCK for concurrent consumers | Implementing concurrent queue dequeue without blocking |
| 08 | [idempotent-migrations](08-idempotent-migrations.md) | Meta-functions to check existence before ALTER | Writing migration scripts safe to re-run |
| 09 | [error-codes](09-error-codes.md) | sp_addmessage, error code registry 50001-50014 | Raising named semantic errors from procedures |
| 10 | [naming-conventions](10-naming-conventions.md) | PascalCase, view/procedure/function naming rules | Naming tables, views, procedures, and constraints |
