# RAG Chunks Index

30 topic chunks for semantic retrieval. Each file is self-contained with YAML frontmatter (`topic`, `keywords`, `use_when`) for AI context selection.

| # | File | Topic | Use When |
|---|------|-------|----------|
| 00 | [ddl-tables-constraints](00-ddl-tables-constraints.md) | CREATE TABLE, constraints, sequences, computed columns | Defining or modifying a table, adding a column idempotently |
| 01 | [dql-select-syntax](01-dql-select-syntax.md) | SELECT, OFFSET FETCH, CROSS APPLY, PIVOT, set operators | Writing SELECT queries with pagination or set operations |
| 02 | [dml-insert-update-delete-merge](02-dml-insert-update-delete-merge.md) | INSERT/UPDATE/DELETE with OUTPUT, MERGE upsert | Auditing DML with OUTPUT or upsert patterns |
| 03 | [bulk-operations](03-bulk-operations.md) | BULK INSERT, OPENROWSET BULK, BCP, minimal logging | Loading large datasets efficiently |
| 04 | [ctes](04-ctes.md) | Non-recursive and recursive CTEs, MAXRECURSION | Structuring multi-step queries or hierarchy traversal |
| 05 | [views](05-views.md) | Standard, indexed (materialized), WITH CHECK OPTION | Creating views, indexing aggregates on disk |
| 06 | [stored-procedures](06-stored-procedures.md) | TVP parameters, EXECUTE AS, parameter sniffing | Creating procedures with TVPs or fixing parameter sniffing |
| 07 | [user-defined-functions](07-user-defined-functions.md) | Inline TVF vs scalar UDF, UDF inlining | Replacing scalar UDFs with inline TVFs for performance |
| 08 | [indexes](08-indexes.md) | Covering, filtered, rebuild vs reorganize, missing index DMVs | Fixing Key Lookups or choosing index maintenance |
| 09 | [columnstore-indexes](09-columnstore-indexes.md) | Clustered/nonclustered columnstore, delta store, rowgroups | Adding analytics performance to DW or OLTP tables |
| 10 | [partitioning](10-partitioning.md) | Partition function/scheme, sliding window, SWITCH | Partitioning large tables or implementing sliding window archival |
| 11 | [transactions-locking](11-transactions-locking.md) | Isolation levels, RCSI, lock hints, deadlock detection | Writing transactions or choosing isolation level |
| 12 | [error-handling](12-error-handling.md) | TRY/CATCH, THROW, RAISERROR, XACT_ABORT | Wrapping T-SQL in error handling |
| 13 | [dbcc-commands](13-dbcc-commands.md) | CHECKDB, OPENTRAN, SHRINKFILE, SQLPERF | Running consistency checks or finding open transactions |
| 14 | [security](14-security.md) | Logins, roles, RLS, dynamic data masking, TDE | Setting up permissions, RLS, or encryption |
| 15 | [temporal-tables](15-temporal-tables.md) | System-versioned, FOR SYSTEM_TIME, retention | Tracking row history or querying point-in-time data |
| 16 | [cdc](16-cdc.md) | Change Data Capture, LSN functions, net changes | Tracking row changes for ETL or incremental loads |
| 17 | [json-xml](17-json-xml.md) | FOR JSON PATH, OPENJSON, JSON_VALUE, FOR XML PATH | Exporting JSON or parsing JSON payloads |
| 18 | [dynamic-sql](18-dynamic-sql.md) | sp_executesql, dynamic WHERE, dynamic PIVOT | Building queries with variable structure |
| 19 | [linked-servers](19-linked-servers.md) | OPENQUERY, OPENROWSET, four-part names, security | Querying across SQL Server instances or OLE DB sources |
| 20 | [performance-diagnostics](20-performance-diagnostics.md) | Top CPU queries, wait stats, blocking, spills | Diagnosing high CPU or finding blocking chains |
| 21 | [query-store](21-query-store.md) | Enable, find regressed queries, force plans | Finding regressed plans or forcing known-good plans |
| 22 | [extended-events](22-extended-events.md) | XE sessions, ring buffer, event file, deadlock capture | Capturing slow queries or deadlocks |
| 23 | [tempdb](23-tempdb.md) | File sizing, version store, long-running transactions | Sizing TempDB or diagnosing version store bloat |
| 24 | [backup-restore](24-backup-restore.md) | Full/differential/log backup, point-in-time restore, S3 | Running or scripting backups, point-in-time restore |
| 25 | [high-availability](25-high-availability.md) | Always On AG, readable secondary, Contained AG | Creating or monitoring Availability Groups |
| 26 | [sql-agent](26-sql-agent.md) | Jobs, schedules, operators, alerts, monitoring | Creating automated jobs or setting up alerts |
| 27 | [sql-server-2022-features](27-sql-server-2022-features.md) | IS DISTINCT FROM, GREATEST/LEAST, GENERATE_SERIES, etc. | Using SQL Server 2022+ syntax |
| 28 | [sql-server-2025-vector-search](28-sql-server-2025-vector-search.md) | VECTOR type, VECTOR_DISTANCE, DiskANN | Storing and searching embedding vectors |
| 29 | [common-mistakes](29-common-mistakes.md) | SARGable, implicit conversion, NOLOCK, UDF, etc. | Quick reference for known T-SQL pitfalls |
