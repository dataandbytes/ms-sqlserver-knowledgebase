---
topic: Extended Events : creating sessions, ring buffer, event file, deadlock capture
keywords: [Extended Events, XE, event session, ring buffer, event file, sql_statement_completed, xml_deadlock_report]
use_when: Creating an XE session to capture slow queries or deadlocks, reading from a ring buffer.
---

# Extended Events

```sql
-- Capture statements over 5s and all deadlocks
CREATE EVENT SESSION [PerfTriage] ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    ACTION (sqlserver.sql_text, sqlserver.session_id, sqlserver.database_name)
    WHERE duration > 5000000   -- microseconds
),
ADD EVENT sqlserver.xml_deadlock_report (
    ACTION (sqlserver.sql_text, sqlserver.session_id)
)
ADD TARGET package0.ring_buffer (SET max_memory = 51200),
ADD TARGET package0.event_file  (SET filename = 'D:\XELogs\PerfTriage.xel', max_file_size = 100);
GO
ALTER EVENT SESSION [PerfTriage] ON SERVER STATE = START;

-- Read from ring buffer
SELECT xdr.value('@timestamp','datetime2') AS event_time,
       xdr.value('(data[@name="duration"]/value)[1]', 'BIGINT') / 1000 AS duration_ms,
       xdr.value('(action[@name="sql_text"]/value)[1]', 'NVARCHAR(MAX)') AS sql_text
FROM (SELECT CAST(target_data AS XML) AS td
      FROM sys.dm_xe_session_targets t
      JOIN sys.dm_xe_sessions s ON s.address = t.event_session_address
      WHERE s.name = 'PerfTriage' AND t.target_name = 'ring_buffer') d
CROSS APPLY td.nodes('//RingBufferTarget/event[@name="sql_statement_completed"]') AS e(xdr)
ORDER BY event_time DESC;
```

**Key rules:**
- Always include both ring buffer (quick lookup) and event file (persistent) targets during initial triage.
- Ring buffer data is lost on restart. Event files persist until the file retention limit is reached.
- Filter duration in microseconds: 5,000,000 = 5 seconds. Start high (5-10s) and lower as you narrow down.
- Use `system_health` (built-in) for deadlock graphs without creating a custom session. Custom sessions give you more control over filtering and targets.
