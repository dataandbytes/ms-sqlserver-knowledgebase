---
topic: High availability : Always On Availability Groups, monitoring AG health, readable secondary
keywords: [Availability Group, Always On, AG, readable secondary, ApplicationIntent, synchronous commit, automatic failover, HADR, Contained AG, 2022]
use_when: Creating or monitoring an Always On AG, routing read queries to a readable secondary, or using Contained AGs (SQL Server 2022+).
---

# High Availability: Always On Availability Groups

```sql
-- Create AG
CREATE AVAILABILITY GROUP [YourAG]
WITH (AUTOMATED_BACKUP_PREFERENCE = SECONDARY, DB_FAILOVER = ON)
FOR DATABASE YourDB
REPLICA ON
    N'Primary\SQL1' WITH (
        ENDPOINT_URL = 'TCP://primary.internal:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC),
    N'Secondary\SQL2' WITH (
        ENDPOINT_URL = 'TCP://secondary.internal:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        READABLE_SECONDARY = YES,
        SEEDING_MODE = AUTOMATIC);

-- Monitor AG health
SELECT ag.name, ars.role_desc, ar.replica_server_name,
       drs.synchronization_state_desc, drs.synchronization_health_desc,
       drs.log_send_queue_size, drs.redo_queue_size
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ar.group_id = ag.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ars.replica_id = ar.replica_id
JOIN sys.dm_hadr_database_replica_states drs ON drs.replica_id = ar.replica_id;

-- Contained AG (SQL Server 2022+): AG-level logins, no dependency on master
ALTER AVAILABILITY GROUP [YourAG] SET (CONTAINED = ON);
```

**Connection string for read routing:**
```
Server=ag-listener;Database=YourDB;ApplicationIntent=ReadOnly
```
The AG listener routes `ApplicationIntent=ReadOnly` connections to the readable secondary automatically.

**Key rules:**
- `log_send_queue_size` and `redo_queue_size` measure replication lag in KB. Alert if either exceeds your RPO threshold.
- Synchronous commit + automatic failover: zero data loss but adds latency on every write (waits for secondary acknowledgment).
- Asynchronous commit: no write latency penalty but potential data loss on failover.
- `READABLE_SECONDARY = YES` allows SELECT queries on the secondary, but only at snapshot isolation : no shared locks.
- Contained AGs (2022+) move logins, credentials, and SQL Agent jobs inside the AG, eliminating the need to sync them separately to the secondary.
