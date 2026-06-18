---
topic: SQL Agent : jobs, schedules, operators, alerts, monitoring
keywords: [SQL Agent, sp_add_job, sp_add_jobstep, sp_add_schedule, sp_add_operator, sp_add_alert, job monitoring, automation, sysjobs]
use_when: Creating automated maintenance or ETL jobs via T-SQL, setting up alerts, monitoring job history, or managing job schedules.
---

# SQL Agent

```sql
-- Create a job
EXEC msdb.dbo.sp_add_job
    @job_name = N'Daily Index Rebuild',
    @enabled  = 1,
    @description = N'Rebuild indexes with >30% fragmentation';

-- Add a job step (T-SQL)
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Daily Index Rebuild',
    @step_name = N'Rebuild Step',
    @subsystem = N'TSQL',
    @command   = N'
        DECLARE @SQL NVARCHAR(MAX);
        SELECT @SQL = STRING_AGG(
            ''ALTER INDEX '' + QUOTENAME(i.name) + '' ON '' + QUOTENAME(SCHEMA_NAME(t.schema_id)) + ''.'' + QUOTENAME(t.name) + '' REBUILD WITH (ONLINE = ON);'',
            CHAR(10))
        FROM sys.indexes i
        JOIN sys.tables t ON t.object_id = i.object_id
        WHERE i.type_desc = ''NONCLUSTERED''
          AND EXISTS (SELECT 1 FROM sys.dm_db_index_physical_stats(DB_ID(), i.object_id, i.index_id, NULL, ''LIMITED'') ps
                      WHERE ps.avg_fragmentation_in_percent > 30 AND ps.page_count > 1000);
        EXEC sp_executesql @SQL;',
    @database_name = N'YourDB';

-- Add a schedule (daily at 2 AM)
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Nightly 2AM',
    @freq_type     = 4,       -- daily
    @freq_interval = 1,
    @active_start_time = 020000;

-- Attach schedule to job
EXEC msdb.dbo.sp_attach_schedule
    @job_name    = N'Daily Index Rebuild',
    @schedule_name = N'Nightly 2AM';

-- Add an operator (email notification)
EXEC msdb.dbo.sp_add_operator
    @name     = N'DBA Team',
    @email_address = N'dba@company.com';

-- Add alert for severity 16 errors
EXEC msdb.dbo.sp_add_alert
    @name      = N'Severity 16 Errors',
    @severity  = 16,
    @enabled   = 1,
    @include_event_description_in = 1;   -- include in email

-- Notify operator on job failure
EXEC msdb.dbo.sp_update_job
    @job_name = N'Daily Index Rebuild',
    @notify_level_email = 2,     -- notify on failure
    @notify_email_operator_name = N'DBA Team';
```

**Job monitoring queries:**

```sql
-- Running jobs
SELECT j.name, ja.start_execution_date, ja.stop_execution_date
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobactivity ja ON ja.job_id = j.job_id
WHERE ja.start_execution_date IS NOT NULL
  AND ja.stop_execution_date IS NULL;

-- Last run status per job
SELECT j.name,
       last_run_outcome = CASE hj.run_status
           WHEN 0 THEN 'Failed' WHEN 1 THEN 'Succeeded'
           WHEN 2 THEN 'Retry'  WHEN 3 THEN 'Cancelled'
           WHEN 4 THEN 'In Progress' END,
       hj.run_date, hj.run_time
FROM msdb.dbo.sysjobs j
OUTER APPLY (SELECT TOP 1 * FROM msdb.dbo.sysjobhistory h
             WHERE h.job_id = j.job_id AND h.step_id = 0
             ORDER BY h.run_date DESC, h.run_time DESC) hj;
```

**Key rules:**
- Use `sp_add_jobstep` with `@subsystem = N'TSQL'` for T-SQL steps, `N'PowerShell'` for PowerShell scripts, `N'CmdExec'` for OS commands.
- Job history is stored in `msdb.dbo.sysjobhistory`. Set the history retention in SQL Agent properties (default 1000 rows per job).
- Enable Database Mail before using email notifications. SQL Agent cannot send email without it.
- For multi-step jobs, set `@on_success_action` and `@on_fail_action` on each step to control flow (1=quit success, 2=quit failure, 3=go next, 4=go step).
- SQL Agent must be running. Verify with: `SELECT status FROM sys.dm_server_services WHERE servicename LIKE 'SQL Server Agent%';`
