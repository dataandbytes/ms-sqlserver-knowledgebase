---
topic: Backup and restore : full, differential, log, point-in-time, S3 (2022+), VERIFYONLY
keywords: [BACKUP DATABASE, BACKUP LOG, RESTORE DATABASE, RESTORE LOG, NORECOVERY, RECOVERY, STOPAT, differential, S3, VERIFYONLY, point-in-time restore]
use_when: Running or scripting backups, restoring a database to a point in time, verifying backup integrity, or using S3 as backup target (SQL Server 2022+).
---

# Backup and Restore

```sql
-- Full backup
BACKUP DATABASE YourDB
    TO DISK = '\\backupserver\backups\YourDB_full.bak'
    WITH COMPRESSION, CHECKSUM, STATS = 10, INIT;

-- Differential backup (since last full)
BACKUP DATABASE YourDB TO DISK = '\\backupserver\backups\YourDB_diff.bak'
    WITH DIFFERENTIAL, COMPRESSION, CHECKSUM;

-- Transaction log backup (required for point-in-time under FULL recovery model)
BACKUP LOG YourDB TO DISK = '\\backupserver\backups\YourDB_log_20240615_1400.trn'
    WITH COMPRESSION, CHECKSUM;

-- Point-in-time restore sequence
RESTORE DATABASE YourDB_Test FROM DISK = '\\backupserver\backups\YourDB_full.bak'
    WITH NORECOVERY,
         MOVE 'YourDB'     TO 'D:\data\YourDB_Test.mdf',
         MOVE 'YourDB_log' TO 'D:\log\YourDB_Test.ldf',
         REPLACE;

RESTORE LOG YourDB_Test FROM DISK = '\\backupserver\backups\YourDB_log_20240615_1400.trn'
    WITH NORECOVERY;

-- Final log backup: stop at exact point in time then bring online
RESTORE LOG YourDB_Test FROM DISK = '\\backupserver\backups\YourDB_log_20240615_1500.trn'
    WITH RECOVERY, STOPAT = '2024-06-15T14:42:00';

-- S3 backup (SQL Server 2022+)
BACKUP DATABASE YourDB TO URL = 's3://my-backup-bucket/YourDB_full.bak'
    WITH COMPRESSION, CHECKSUM, CREDENTIAL = 'S3BackupCred';

-- Verify integrity without restoring
RESTORE VERIFYONLY FROM DISK = '\\backupserver\backups\YourDB_full.bak' WITH CHECKSUM;
```

**Key rules:**
- Always include `CHECKSUM` : it validates the backup at write time. `VERIFYONLY` re-checks at any time.
- Restore chain: Full → (optional Differential) → all subsequent Log backups in order. Any gap in the log chain breaks point-in-time restore.
- `NORECOVERY` leaves the database in restoring state, ready for the next backup. `RECOVERY` brings it online (use only on the final restore step).
- Test restores regularly : an untested backup is not a backup.
