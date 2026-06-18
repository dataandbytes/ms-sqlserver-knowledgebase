---
topic: Minimal logging and BULK INSERT optimization
keywords: [minimal logging, BULK_LOGGED, SIMPLE recovery, TABLOCK, SELECT INTO, BULK INSERT, BATCHSIZE, bulk load]
use_when: You are doing a bulk load and want to reduce transaction log volume.
---

# Minimal Logging + BULK INSERT

Under `SIMPLE` or `BULK_LOGGED` recovery, certain bulk operations log only extent allocations, not individual rows. Conditions for INSERT...SELECT / SELECT INTO:

| Condition | Required? |
|---|---|
| Recovery model = SIMPLE or BULK_LOGGED | Yes — never under FULL |
| `TABLOCK` hint on target | Required for non-empty heap/clustered pre-2016 |
| Target has no nonclustered indexes | Required unless empty clustered index (2016+) |

2016+: an empty heap / empty clustered index with NCIs can still qualify with `TABLOCK` if the table was empty at the start (NCI entries are still fully logged).

```sql
INSERT INTO dbo.StagingFact WITH (TABLOCK)
SELECT FactID, ProductID, SaleDate, Amount FROM dbo.SourceFact WHERE SaleDate >= '2025-01-01';

SELECT FactID, ProductID, SaleDate, Amount INTO dbo.StagingFact
FROM dbo.SourceFact WHERE SaleDate >= '2025-01-01';
```

Switching recovery model for a bulk load (only if you can accept losing point-in-time restore during the window) — **do not skip the log backup afterward**:

```sql
ALTER DATABASE YourDatabase SET RECOVERY BULK_LOGGED;
-- ... bulk operation ...
ALTER DATABASE YourDatabase SET RECOVERY FULL;
BACKUP LOG YourDatabase TO DISK = '\\backupserver\YourDatabase_post_bulk.trn';
```

**BULK INSERT** is the fastest external-data load:

```sql
BULK INSERT dbo.StagingLoad FROM 'D:\data\load_file.csv'
WITH (FIELDTERMINATOR = ',', ROWTERMINATOR = '\n', FIRSTROW = 2,
      BATCHSIZE = 10000, TABLOCK, MAXERRORS = 0);
UPDATE STATISTICS dbo.StagingLoad WITH FULLSCAN;  -- bulk loads do NOT auto-update stats
```

`BATCHSIZE` controls commit frequency and restartability: 0 (default) loads the whole file in one transaction (fast, more log, not restartable); 10,000 commits every 10,000 rows (restartable at last committed batch). Minimal logging needs `TABLOCK`, SIMPLE/BULK_LOGGED, and no NCIs (or empty clustered index on 2016+).
