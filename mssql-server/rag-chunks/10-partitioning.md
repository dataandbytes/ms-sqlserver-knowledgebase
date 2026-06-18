---
topic: Table partitioning : partition function, scheme, sliding window, partition switching
keywords: [partitioning, PARTITION FUNCTION, PARTITION SCHEME, RANGE RIGHT, SWITCH, SPLIT, MERGE, sliding window, partition elimination]
use_when: Partitioning a large table by date or key range, implementing a sliding window for data archival, or switching in/out partitions.
---

# Partitioning

```sql
-- Partition function: defines the boundary values
CREATE PARTITION FUNCTION PF_Orders_Monthly (DATE)
AS RANGE RIGHT FOR VALUES (
    '2023-01-01','2023-02-01','2023-03-01','2023-04-01',
    '2023-05-01','2023-06-01','2023-07-01','2023-08-01',
    '2023-09-01','2023-10-01','2023-11-01','2023-12-01','2024-01-01'
);
-- RANGE RIGHT: boundary value is the first row of the next partition

-- Partition scheme: maps each partition to a filegroup
CREATE PARTITION SCHEME PS_Orders_Monthly
    AS PARTITION PF_Orders_Monthly ALL TO ([PRIMARY]);

-- Partitioned table (no syntax difference : just specifies ON scheme(column))
CREATE TABLE Sales.Orders (
    OrderID   INT  NOT NULL,
    OrderDate DATE NOT NULL,
    ...
    CONSTRAINT PK_Orders PRIMARY KEY (OrderID, OrderDate)
) ON PS_Orders_Monthly (OrderDate);

-- Sliding window: archive old partition, add new one
-- Step 1: switch old partition to archive table (metadata-only, milliseconds)
ALTER TABLE Sales.Orders SWITCH PARTITION 1 TO Sales.OrdersArchive_2022_12;

-- Step 2: add new partition for next month
ALTER PARTITION SCHEME PS_Orders_Monthly NEXT USED [PRIMARY];
ALTER PARTITION FUNCTION PF_Orders_Monthly() SPLIT RANGE ('2024-02-01');
```

**Key rules:**
- The partition key must be part of the primary key and all unique constraints.
- `SWITCH` is a metadata operation : both tables must have identical structure, indexes, and constraints, and the target must be empty.
- Partition elimination: the optimizer skips partitions that cannot contain rows matching the WHERE predicate : but only if the partition column is in the WHERE clause as a SARGable predicate.
- `RANGE RIGHT`: boundary value is the leftmost row of the new partition (most natural for date-range tables). `RANGE LEFT`: boundary value is the rightmost row of the previous partition.
