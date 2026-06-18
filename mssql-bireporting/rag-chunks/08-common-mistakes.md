---
topic: Common BI/reporting mistakes and their fixes
keywords: [common mistakes, HAVING vs WHERE, window function in WHERE, RANGE frame, dynamic pivot injection, FOR JSON AUTO, gaps, tally CTE limit]
use_when: Quick reference for known BI query pitfalls when reviewing or debugging reporting queries.
---

# Common BI Reporting Mistakes

| Mistake | Fix |
|---|---|
| `WHERE YEAR(col) = 2024` makes predicate non-SARGable | Use `col >= '2024-01-01' AND col < '2025-01-01'` |
| Window function in WHERE clause | Wrap in CTE or subquery, then filter |
| `OVER (ORDER BY col)` without explicit ROWS/RANGE frame | Add `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` for running totals |
| RANGE frame on `SUM` doubles values on tied dates | Switch to `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |
| Stacking CTE missing `WHERE N <= @Count` | Without it materializes 65,536 rows before trim |
| Dynamic PIVOT with string concatenation of column names | Always use `QUOTENAME()` to prevent injection |
| `FOR JSON AUTO` in production queries | Use `FOR JSON PATH` — AUTO breaks when aliases change |
| `UNPIVOT` silently drops NULL rows | Use `CROSS APPLY VALUES` and filter NULLs explicitly |
| Cohort `MonthOffset` computed from local time | Use UTC or normalize before `DATEDIFF` |
| `GROUPING(col)` used in WHERE clause | Move the GROUPING check to HAVING |
| Calendar gap fill uses function on join column | Join on raw date column: `CAST(OrderDate AS DATE) = c.DateKey` — keep it SARGable |
| `COUNT(*)` vs `COUNT(col)` confusion | `COUNT(*)` includes NULLs; `COUNT(col)` excludes them — choose deliberately |
| YTD not resetting at year boundary | Add `PARTITION BY YEAR(OrderDate)` to the window |
| Moving average with RANGE instead of ROWS | RANGE includes all tied dates in the window — often unintentional |
| Session analysis: gap threshold hardcoded | Store session timeout in AppSettings and parameterize the `DATEDIFF` threshold |
| `LAST_VALUE` returning wrong value | Requires explicit `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` frame |
