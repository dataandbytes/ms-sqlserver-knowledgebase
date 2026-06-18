# RAG Chunks Index: BI and Reporting

10 topic chunks for analytics and reporting queries. Each file is self-contained with YAML frontmatter (`topic`, `keywords`, `use_when`) for AI context selection.

| # | File | Topic | Use When |
|---|------|-------|----------|
| 00 | [multi-level-aggregation](00-multi-level-aggregation.md) | ROLLUP, CUBE, GROUPING SETS, GROUPING(), GROUPING_ID() | Producing subtotals, grand totals, or multi-dimensional aggregations |
| 01 | [window-functions](01-window-functions.md) | ROW_NUMBER, RANK, running totals, moving averages, LAG/LEAD | Ranking rows, computing running totals, moving averages, YoY comparisons |
| 02 | [date-bucketing](02-date-bucketing.md) | DATETRUNC, DATE_BUCKET, DATEADD/DATEDIFF patterns | Truncating dates, grouping into time buckets, SARGable date filters |
| 03 | [time-series-patterns](03-time-series-patterns.md) | YTD, cohort analysis, temporal table point-in-time joins | Computing YTD totals, building cohort retention tables |
| 04 | [calendar-table](04-calendar-table.md) | Design, seeding, business day calculations, gap filling | Creating a calendar table or calculating business days between dates |
| 05 | [tally-tables](05-tally-tables.md) | Stacking CTE, GENERATE_SERIES (2022+), number/date sequences | Generating sequences, splitting strings, date spine for gap filling |
| 06 | [gaps-and-islands](06-gaps-and-islands.md) | ROW_NUMBER difference technique, streaks, session analysis | Finding consecutive sequences or identifying gaps in a series |
| 07 | [pivot-unpivot](07-pivot-unpivot.md) | PIVOT operator, dynamic PIVOT, CROSS APPLY VALUES unpivot | Rotating rows to columns or columns to rows |
| 08 | [common-mistakes](08-common-mistakes.md) | Common BI/reporting mistakes and their fixes | Quick reference for known BI query pitfalls |
| 09 | [real-world-scenarios](09-real-world-scenarios.md) | Executive dashboard, SaaS MRR bridge, retention cohort, RFM | Building a complete, business-framed report combining multiple BI techniques |
