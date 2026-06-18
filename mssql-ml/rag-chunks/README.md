# RAG Chunks Index: ML Feature Engineering

8 topic chunks for building ML training datasets in T-SQL. Each file is self-contained with YAML frontmatter (`topic`, `keywords`, `use_when`) for AI context selection.

| # | File | Topic | Use When |
|---|------|-------|----------|
| 00 | [feature-table-workflow](00-feature-table-workflow.md) | 7-step pattern, #Base table, snapshot date | Building a feature table for ML training from SQL Server data |
| 01 | [numeric-features](01-numeric-features.md) | Log, z-score, min-max, NTILE bucketing | Applying transforms or bucketing a numeric column into deciles |
| 02 | [rfm-pattern](02-rfm-pattern.md) | Recency, Frequency, Monetary with NTILE quintiles | Computing RFM scores for customer segmentation or ML features |
| 03 | [temporal-features](03-temporal-features.md) | Recency, frequency windows, tenure, calendar features | Engineering time-based features for ML models |
| 04 | [null-imputation](04-null-imputation.md) | Mean, median, mode, forward fill, NULL indicator column | Handling NULL values in feature columns before ML training |
| 05 | [sampling-train-test-split](05-sampling-train-test-split.md) | HASHBYTES deterministic split, stratified sampling, time-based | Splitting data into train/validation/test sets correctly |
| 06 | [data-leakage-prevention](06-data-leakage-prevention.md) | Temporal, target, and train/test leakage | Auditing a feature table for data leakage |
| 07 | [common-mistakes](07-common-mistakes.md) | Common ML feature engineering mistakes and fixes | Quick reference for known ML feature engineering pitfalls |
