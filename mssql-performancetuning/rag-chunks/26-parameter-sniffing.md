---
topic: Parameter sniffing in stored procedures
keywords: [parameter sniffing, OPTIMIZE FOR UNKNOWN, OPTION RECOMPILE, local variable, skewed distribution, PSPO, stored procedure plan]
use_when: A stored procedure is fast for some parameter values and slow for others.
---

# Parameter Sniffing

SQL Server compiles a stored-procedure plan on first execution using the actual parameter values then, caches it, and reuses it — even for later calls with different values. Beneficial ~90% of the time; it hurts when the distribution is skewed and the sniffed values are atypical. A single query with many distinct cached plan_ids in Query Store signals instability.

**Ranked strategies:**

**1. OPTIMIZE FOR UNKNOWN** — stable plan from average distribution. Preferred when many values are possible and none dominates:

```sql
CREATE PROCEDURE dbo.GetOrders_ut @CustomerID INT
AS BEGIN
    SET NOCOUNT ON;
    SELECT OrderID, OrderDate, TotalAmt FROM dbo.Orders WHERE CustomerID = @CustomerID
    OPTION (OPTIMIZE FOR (@CustomerID UNKNOWN));
END;
```

**2. OPTION (RECOMPILE)** — fresh plan per execution embedding the actual value; optimal plan every time at CPU compile cost. Use for infrequent but highly variable procs (nightly reports, admin tools):

```sql
SELECT OrderID, OrderDate, TotalAmt FROM dbo.Orders WHERE CustomerID = @CustomerID
OPTION (RECOMPILE);   -- statement-level, NOT WITH RECOMPILE on the proc
```

**3. Local variable copy — avoid.** Assigning the parameter to a local variable defeats sniffing entirely, even when sniffing would help. `OPTIMIZE FOR UNKNOWN` achieves the same statistical behavior with explicit, documented intent.

On SQL Server 2022 (compat 160) with Query Store, Parameter-Sensitive Plan Optimization (PSPO) can automatically maintain multiple plan variants for skewed distributions — but it is not a substitute for the above on 2019 and earlier.
