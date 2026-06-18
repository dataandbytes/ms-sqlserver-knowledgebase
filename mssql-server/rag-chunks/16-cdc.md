---
topic: Change Data Capture (CDC) : enable, capture changes, net changes, cleanup
keywords: [CDC, change data capture, cdc.fn_cdc_get_all_changes, cdc.fn_cdc_get_net_changes, sys.sp_cdc_enable_table, LSN, change tracking]
use_when: Tracking row-level changes (INSERT, UPDATE, DELETE) on a table for ETL, auditing, or incremental loads.
---

# Change Data Capture (CDC)

```sql
-- Enable CDC at the database level
EXEC sys.sp_cdc_enable_db;

-- Enable CDC on a table (requires sysadmin)
EXEC sys.sp_cdc_enable_table
    @source_schema   = 'Sales',
    @source_name     = 'Orders',
    @role_name       = 'CDCReader',      -- NULL for unrestricted access
    @capture_instance = 'Sales_Orders';

-- CDC creates a change table: cdc.Sales_Orders_CT
-- Structure: __$start_lsn, __$operation, __$update_mask, followed by all source columns
-- __$operation: 1=DELETE, 2=INSERT, 3=UPDATE (before), 4=UPDATE (after)

-- Get all changes between two LSNs
DECLARE @from_lsn BINARY(10), @to_lsn BINARY(10);
SET @from_lsn = sys.fn_cdc_get_min_lsn('Sales_Orders');
SET @to_lsn   = sys.fn_cdc_get_max_lsn();

SELECT * FROM cdc.fn_cdc_get_all_changes_Sales_Orders(@from_lsn, @to_lsn, N'all');

-- Net changes (one row per source row, with the final operation)
SELECT * FROM cdc.fn_cdc_get_net_changes_Sales_Orders(@from_lsn, @to_lsn, N'all');

-- Map datetime to LSN for point-in-time CDC queries
DECLARE @lsn BINARY(10) = sys.fn_cdc_map_time_to_lsn('smallest greater than or equal', '2025-01-15 08:00:00');

-- Disable CDC
EXEC sys.sp_cdc_disable_table @source_schema = 'Sales', @source_name = 'Orders', @capture_instance = 'Sales_Orders';
EXEC sys.sp_cdc_disable_db;
```

**Key rules:**
- CDC uses the transaction log as its source. It does NOT read the data files, so there is no scan overhead on the source table.
- The change table is a relational table in the same database. Retention is managed by a cleanup job that runs every 5 minutes by default.
- `fn_cdc_get_all_changes` returns two rows per UPDATE (old + new values). `fn_cdc_get_net_changes` returns one row with the final state.
- CDC adds log reader overhead. Monitor log space and the capture job (`cdc.<db>_capture`) on high-volume tables.
- Minimum retention: 3 days for typical ETL. Adjust with `sys.sp_cdc_change_job @job_type='cleanup', @retention=4320` (minutes).
- CDC is available in Enterprise, Developer, and Standard Edition (2016 SP1+).
