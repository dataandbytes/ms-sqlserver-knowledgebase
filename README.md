# SQL Server Knowledge Base

A collection of standalone T-SQL reference files. Each file is a self-contained knowledge base
that can be loaded into a GitHub-backed Claude Project, used as RAG context, or read directly as a
reference during development.

## Knowledge Bases

| File | Topic | Key Areas |
|------|-------|-----------|
| [mssql-server/mssql-server-knowledge-base.md](mssql-server/mssql-server-knowledge-base.md) | Core T-SQL + Administration | DDL/DML/DQL, indexes, partitioning, transactions, security, temporal tables, JSON, backup/restore, HA, 2022/2025 features |
| [sql-server-performance/sql-server-performance-knowledge-base.md](sql-server-performance/sql-server-performance-knowledge-base.md) | Performance Tuning | Execution plans, wait stats, index strategy, statistics, parameter sniffing, locking, Query Store |
| [mssql-writing-guidelines/mssql-writing-guidelines-knowledge-base.md](mssql-writing-guidelines/mssql-writing-guidelines-knowledge-base.md) | Application Design Patterns | Custom types, stored procedures, transactions, relational queues, RLS views, idempotent migrations, error codes |
| [sql-bi-reporting/sql-bi-reporting-knowledge-base.md](sql-bi-reporting/sql-bi-reporting-knowledge-base.md) | Analytics and Reporting | ROLLUP/CUBE/GROUPING SETS, window functions, date bucketing, gaps and islands, pivot/unpivot, calendar tables |
| [sql-ml-features/sql-ml-features-knowledge-base.md](sql-ml-features/sql-ml-features-knowledge-base.md) | ML Feature Engineering | Numeric/categorical transforms, temporal features, RFM scoring, lag features, NULL imputation, train/test splitting, leakage prevention |

---

## What Each File Covers

### [mssql-server](mssql-server/mssql-server-knowledge-base.md)
The broadest file (30 topic sections). Covers the entire T-SQL and SQL Server surface: DDL,
query syntax including CROSS APPLY and PIVOT, DML with OUTPUT and MERGE, bulk operations,
CTEs (recursive and non-recursive), indexed views, stored procedures with TVPs, user-defined
functions, nonclustered and columnstore indexes, table partitioning, transactions and isolation
levels, error handling, DBCC commands, security (RLS, DDM, TDE), temporal tables, Change Data
Capture (CDC), JSON/XML, dynamic SQL, linked servers, performance diagnostics via DMVs, Query
Store, Extended Events, TempDB sizing, backup and restore including S3 (2022+), Always On
Availability Groups, SQL Agent, and dedicated sections for SQL Server 2022, 2025 (vector search,
DiskANN), and common mistakes.

### [sql-server-performancetuning](sql-server-performance/sql-server-performance-knowledge-base.md)
Focused on diagnosing and fixing slow queries. Covers reading execution plans (seek vs scan, key
lookup, hash/merge/nested-loop joins, parallelism), wait statistics methodology, index strategy
decisions, statistics freshness and manual UPDATE STATISTICS, parameter sniffing and mitigation,
blocking and locking patterns with RCSI, and chunked batch operations. Includes before/after
logical read examples and a full server-health diagnostic script.

### [mssql-writing-standards](mssql-writing-guidelines/mssql-writing-guidelines-knowledge-base.md)
Prescriptive patterns for writing SQL Server application code as a system rather than ad-hoc
queries. Covers: two access rules (reads via views, mutations via procedures), custom type system
with `CREATE TYPE`, the `_trx/_utx/_ut` transaction hierarchy with full 5-block templates,
DML error-checking with `@@ROWCOUNT` and `@@ERROR`, MERGE patterns, functional constraints,
base/subtype table inheritance, hierarchical composite keys with max-plus-one functions,
relational queues (READPAST + UPDLOCK), AppSettings table, idempotent migration meta-functions,
and semantic error codes 50001-50014.

### [sql-bi-reporting](sql-bi-reporting/sql-bi-reporting-knowledge-base.md)
Analytics queries for reporting and dashboards. Covers multi-level aggregation (ROLLUP, CUBE,
GROUPING SETS), GROUPING()/GROUPING_ID(), conditional aggregation, window functions (ROWS vs
RANGE frames, running totals, moving averages, YoY with LAG), date bucketing with DATETRUNC and
DATE_BUCKET (2022+) and DATEADD/DATEDIFF patterns for older versions, calendar table design and
seeding, gap filling via LEFT JOIN, tally tables and GENERATE_SERIES, gaps-and-islands technique,
pivot and unpivot, and FOR JSON PATH export.

### [sql-ml](sql-ml/sql-ml-features-knowledge-base.md)
Building ML training datasets in T-SQL. Covers the feature table workflow (#Base pattern),
numeric transforms (log, z-score, min-max, NTILE), categorical encoding (one-hot, ordinal,
frequency), temporal features keyed to a fixed `@SnapshotDate`, RFM scoring, rolling window
features with ROWS BETWEEN, lag features, NULL imputation strategies (mean, median, mode, forward
fill, indicator columns), HASHBYTES-based deterministic train/test splitting, stratified sampling,
time-based splitting with a gap buffer, and a detailed section on data leakage prevention
(temporal, target, and train/test leakage).

---

## How to Use

### GitHub-backed Claude Project

1. Push this repository to GitHub.
2. In Claude.ai, create a Project and connect the GitHub repository.
3. Claude will index all files. Reference a specific knowledge base in your prompt:

   > "Using the sql-bi-reporting knowledge base, write a query that shows 7-day rolling revenue
   > per region."

### Direct file reference

Each knowledge base is a single Markdown file. Open it in any editor or paste it into a chat
context alongside your question.

### AI Agent knowledge source

Each knowledge base is structured for direct consumption by an AI agent. The H2-chunked format,
keyword-labeled sections, and version-annotated examples mean an agent querying this repo gets
correct syntax instead of hallucinations. Drop the `rag-chunks/` directories into any vector
store or load the repo into a Claude Project for instant SQL Server context.

### RAG ingestion

Each file is structured with H2 section headings and fenced code blocks. Chunk by H2 section
(the `## ...` lines) for semantic retrieval — each chunk is a coherent topic with working SQL
examples.

---

## Design Notes

These files are written as standalone knowledge bases, not copies of the source skill files.
Each file:

- Uses concrete SQL examples rather than abstract descriptions.
- Includes before/after patterns and diagnostic queries where relevant.
- Avoids forward references — every section stands alone.
- Ends with a Common Mistakes table for quick error lookup.

SQL Server version coverage: SQL Server 2016 through 2025. Features specific to a version are
labeled inline (e.g., "2022+", "2019+").

---

## License

MIT License


Permission is hereby granted, free of charge, to any person obtaining a copy of this software
and associated documentation files (the "Software"), to deal in the Software without restriction,
including without limitation the rights to use, copy, modify, merge, publish, distribute,
sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or
substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING
BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---

## Authors

- **KK** — domain expertise, content direction, and review
- **Claude (Anthropic)** — knowledge base authoring and RAG chunk generation
