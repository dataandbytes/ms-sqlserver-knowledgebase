# RAG Chunks Index: Performance Tuning

29 topic chunks for diagnosing and fixing slow SQL Server queries. Each file is self-contained with YAML frontmatter (`topic`, `keywords`, `use_when`) for AI context selection.

| # | File | Topic | Use When |
|---|------|-------|----------|
| 00 | [diagnostic-flowchart](00-diagnostic-flowchart.md) | Slow query diagnostic sequence | A query is slow and you need a starting methodology |
| 01 | [capturing-execution-plans](01-capturing-execution-plans.md) | Capturing plans and STATISTICS IO/TIME | You need a query's plan or I/O/CPU measurements |
| 02 | [plan-operators](02-plan-operators.md) | Scan/seek, key lookup, joins, sort, spool, parallelism | Interpreting operators in an execution plan |
| 03 | [cardinality-estimation](03-cardinality-estimation.md) | CE versions, estimated vs actual rows | Estimated rows differ from actual rows |
| 04 | [intelligent-query-processing](04-intelligent-query-processing.md) | IQP features and compatibility levels | Using or disabling IQP features |
| 05 | [query-store-plan-regression](05-query-store-plan-regression.md) | Plan regression detection and plan forcing | A query that was fast regressed |
| 06 | [plan-cache-analysis](06-plan-cache-analysis.md) | Expensive queries server-wide, ad-hoc bloat | Finding most expensive queries or diagnosing cache bloat |
| 07 | [clustered-index-selection](07-clustered-index-selection.md) | Choosing a clustered key | Designing a table's clustered key |
| 08 | [covering-and-composite-indexes](08-covering-and-composite-indexes.md) | Covering indexes, INCLUDE, composite key ordering | A query has a Key Lookup; deciding column order |
| 09 | [filtered-indexes](09-filtered-indexes.md) | Indexing subsets of rows, partial unique constraints | Indexing active/pending rows or partial uniqueness |
| 10 | [over-indexing](10-over-indexing.md) | Finding unused and duplicate indexes | Writes are slow; finding indexes safe to drop |
| 11 | [fragmentation-fill-factor](11-fragmentation-fill-factor.md) | Rebuild vs reorganize, fill factor | Running index maintenance |
| 12 | [missing-index-dmvs](12-missing-index-dmvs.md) | Missing index DMVs and limitations | Evaluating optimizer index recommendations |
| 13 | [statistics-and-histogram](13-statistics-and-histogram.md) | How statistics drive plans, histogram, DBCC SHOW_STATISTICS | Reading a statistics histogram |
| 14 | [stats-auto-update-thresholds](14-stats-auto-update-thresholds.md) | Legacy vs dynamic thresholds, freshness monitoring | Auto-update never fires on a large table |
| 15 | [ascending-key-and-stats-fixes](15-ascending-key-and-stats-fixes.md) | Ascending key problem, UPDATE STATISTICS options | Recent-date queries estimate 1 row |
| 16 | [wait-stats-methodology](16-wait-stats-methodology.md) | Baseline methodology, benign waits to exclude | Running a wait-stats analysis |
| 17 | [wait-type-reference](17-wait-type-reference.md) | Wait type causes and fixes | You have a dominant wait type |
| 18 | [tempdb-and-memory-config](18-tempdb-and-memory-config.md) | TempDB file config, max server memory | PAGELATCH_EX contention or configuring tempdb |
| 19 | [isolation-levels-and-rcsi](19-isolation-levels-and-rcsi.md) | Isolation levels, RCSI | Choosing an isolation level or eliminating reader/writer blocking |
| 20 | [lock-modes-escalation-hints](20-lock-modes-escalation-hints.md) | Lock compatibility, escalation thresholds, hints | Reasoning about lock behavior |
| 21 | [deadlocks-and-blocking](21-deadlocks-and-blocking.md) | Deadlock diagnosis, blocking chains, NOLOCK dangers | Diagnosing deadlocks (error 1205) or blocking chains |
| 22 | [chunked-dml](22-chunked-dml.md) | Large INSERT/UPDATE/DELETE batching | Writing large DML that bloats the log |
| 23 | [minimal-logging-and-bulk-insert](23-minimal-logging-and-bulk-insert.md) | BULK INSERT optimization, minimal logging | Reducing log volume during bulk loads |
| 24 | [partition-switching-and-log-management](24-partition-switching-and-log-management.md) | Fast archive via SWITCH, log latency | Archiving large data or managing log size |
| 25 | [sargability](25-sargability.md) | Writing predicates that allow seeks | A query scans because of WHERE clause structure |
| 26 | [parameter-sniffing](26-parameter-sniffing.md) | Sniffing in stored procedures, mitigations | A procedure is fast for some values, slow for others |
| 27 | [temp-tables-vs-table-variables](27-temp-tables-vs-table-variables.md) | Temp tables vs table variables vs CTEs | Choosing an intermediate result container |
| 28 | [common-mistakes](28-common-mistakes.md) | Common performance mistakes and fixes | Quick checklist of frequent anti-patterns |
