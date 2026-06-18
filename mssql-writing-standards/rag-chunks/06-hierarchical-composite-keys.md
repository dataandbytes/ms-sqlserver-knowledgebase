---
topic: Hierarchical composite keys and max-plus-one functions
keywords: [composite key, hierarchical key, OrderNo, OrderLineNo, max-plus-one, NextLineNo_fn, CustomerNo, natural key]
use_when: Designing child tables whose keys are scoped to a parent, or generating the next line number within an order.
---

# Hierarchical Composite Keys

```sql
-- Three-level hierarchy: Customer → Order → OrderLine
CREATE TABLE Sales.Customer (
    CustomerNo INT NOT NULL IDENTITY(1,1),
    CONSTRAINT PK_Customer PRIMARY KEY (CustomerNo)
);

CREATE TABLE Sales.Order (
    CustomerNo INT NOT NULL,
    OrderNo    INT NOT NULL,   -- scoped to CustomerNo
    OrderDate  dbo.Timestamp,
    CONSTRAINT PK_Order          PRIMARY KEY (CustomerNo, OrderNo),
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerNo) REFERENCES Sales.Customer(CustomerNo)
);

CREATE TABLE Sales.OrderLine (
    CustomerNo INT NOT NULL,
    OrderNo    INT NOT NULL,   -- scoped to CustomerNo
    LineNo     INT NOT NULL,   -- scoped to CustomerNo + OrderNo
    ProductNo  INT NOT NULL,
    Qty        INT NOT NULL,
    CONSTRAINT PK_OrderLine       PRIMARY KEY (CustomerNo, OrderNo, LineNo),
    CONSTRAINT FK_OrderLine_Order FOREIGN KEY (CustomerNo, OrderNo)
        REFERENCES Sales.Order(CustomerNo, OrderNo)
);

-- Max-plus-one function: next LineNo within an order
CREATE OR ALTER FUNCTION Sales.NextLineNo_fn (
    @CustomerNo INT, @OrderNo INT
)
RETURNS INT AS BEGIN
    RETURN (
        SELECT ISNULL(MAX(LineNo), 0) + 1
        FROM Sales.OrderLine
        WHERE CustomerNo = @CustomerNo AND OrderNo = @OrderNo
    );
END;

-- Usage inside a _utx procedure
DECLARE @LineNo INT = Sales.NextLineNo_fn(@CustomerNo, @OrderNo);
INSERT INTO Sales.OrderLine (CustomerNo, OrderNo, LineNo, ProductNo, Qty)
VALUES (@CustomerNo, @OrderNo, @LineNo, @ProductNo, @Qty);
```

**Key rules:**
- The child table's PK includes the parent's full key — this enforces that `LineNo` is scoped to the order, not globally unique.
- Max-plus-one is not safe under concurrent inserts into the same order — serialize with `UPDLOCK` on the parent row or use a `SEQUENCE` scoped per order.
- This pattern is preferred over IDENTITY on child tables because the keys are meaningful and portable (the OrderLine can be identified without knowing its surrogate ID).
