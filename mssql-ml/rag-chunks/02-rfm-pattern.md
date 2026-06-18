---
topic: RFM scoring — Recency, Frequency, Monetary with NTILE quintiles
keywords: [RFM, recency, frequency, monetary, NTILE, quintile, customer scoring, segmentation, @SnapshotDate]
use_when: Computing RFM (Recency/Frequency/Monetary) scores for customer segmentation or ML features.
---

# RFM Pattern

```sql
DECLARE @SnapshotDate DATE = '2024-06-01';

WITH RFM_Raw AS (
    SELECT CustomerID,
           -- Recency: days since last order (lower = more recent = better)
           DATEDIFF(day, MAX(OrderDate), @SnapshotDate) AS Recency,
           -- Frequency: number of orders
           COUNT(*) AS Frequency,
           -- Monetary: total revenue
           SUM(TotalAmount) AS Monetary
    FROM Sales.Orders
    WHERE OrderDate < @SnapshotDate
    GROUP BY CustomerID
),
RFM_Scored AS (
    SELECT CustomerID, Recency, Frequency, Monetary,
           -- R_Score: 5 = most recent (lowest Recency value), so ORDER BY Recency DESC
           NTILE(5) OVER (ORDER BY Recency DESC) AS R_Score,
           NTILE(5) OVER (ORDER BY Frequency ASC) AS F_Score,
           NTILE(5) OVER (ORDER BY Monetary  ASC) AS M_Score
    FROM RFM_Raw
)
SELECT CustomerID, Recency, Frequency, Monetary,
       R_Score, F_Score, M_Score,
       R_Score + F_Score + M_Score AS RFM_Total,
       -- Segment string (for interpretability only — don't use as a model input)
       CAST(R_Score AS VARCHAR) + CAST(F_Score AS VARCHAR) + CAST(M_Score AS VARCHAR) AS RFM_Segment
FROM RFM_Scored
ORDER BY RFM_Total DESC;
```

**Key rules:**
- R_Score: `ORDER BY Recency DESC` because lower Recency (more recent) should score higher.
- F_Score and M_Score: `ORDER BY ... ASC` — higher values score higher.
- Use `R_Score`, `F_Score`, `M_Score` as separate numeric features in the model — they carry more signal than the combined `RFM_Total` or `RFM_Segment` string.
- `RFM_Segment` (e.g., `'554'`) is useful for business dashboards but not for ML models — ordinal encoding of the string adds no value the component scores don't already provide.
- NTILE distributes rows as evenly as possible. Ties may cause slightly uneven buckets.
