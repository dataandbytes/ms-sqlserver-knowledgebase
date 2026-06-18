---
topic: SQL Server 2025 : VECTOR data type, VECTOR_DISTANCE, DiskANN vector index
keywords: [SQL Server 2025, VECTOR, VECTOR_DISTANCE, DiskANN, vector index, ANN, approximate nearest neighbor, embedding, cosine similarity, AI]
use_when: Storing and searching embedding vectors in SQL Server 2025+ for semantic search or AI similarity queries.
---

# SQL Server 2025: Vector Search

```sql
-- VECTOR data type: store embedding vectors
CREATE TABLE dbo.ProductEmbeddings (
    ProductID INT         NOT NULL PRIMARY KEY,
    Embedding VECTOR(768) NOT NULL   -- 768-dimension (e.g., text-embedding-3-small)
);

-- Insert an embedding (JSON array format)
INSERT INTO dbo.ProductEmbeddings (ProductID, Embedding)
VALUES (42, '[0.012, -0.034, 0.891, ...]');

-- Vector similarity search: cosine distance (lower = more similar)
DECLARE @queryVector NVARCHAR(MAX) = '[0.015, -0.031, 0.887, ...]';
SELECT TOP 10
    pe.ProductID,
    VECTOR_DISTANCE('cosine', pe.Embedding, CAST(@queryVector AS VECTOR(768))) AS distance
FROM dbo.ProductEmbeddings pe
ORDER BY distance ASC;

-- Supported metrics: 'cosine', 'dot' (dot product), 'euclidean'

-- DiskANN vector index: approximate nearest neighbor (ANN) for fast large-scale search
CREATE VECTOR INDEX IX_ProductEmbeddings_Vector
    ON dbo.ProductEmbeddings (Embedding)
    WITH (METRIC = 'cosine', TYPE = 'diskann');

-- New JSON functions in 2025
SELECT JSON_PATH_EXISTS(Payload, '$.customer.address.city') AS has_city
FROM Logs.Events;

SELECT JSON_OBJECTAGG(ProductID: Name) AS product_map
FROM Production.Product WHERE IsActive = 1;
```

**Key rules:**
- Without a vector index, `VECTOR_DISTANCE` performs an exact brute-force scan : accurate but O(n). Use DiskANN for tables with >10,000 rows.
- DiskANN returns approximate results : a small fraction of the true nearest neighbors may be missed. Acceptable for semantic search; not for exact matching.
- The `VECTOR` type stores each dimension as a 4-byte float internally. A 768-dimension vector occupies ~3 KB per row.
- Combine vector search with standard SQL predicates: `WHERE CategoryID = 3 ORDER BY VECTOR_DISTANCE(...)` : the engine applies the filter first, then ranks by distance.
