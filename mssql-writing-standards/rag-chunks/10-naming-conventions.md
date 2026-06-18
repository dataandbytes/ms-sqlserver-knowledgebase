---
topic: Naming conventions — PascalCase, view/procedure/function naming rules
keywords: [naming convention, PascalCase, Role_Intent_V, Verb_Domain_trx, stored procedure naming, view naming, function naming]
use_when: Naming a new table, view, stored procedure, function, or constraint following the convention.
---

# Naming Conventions

| Object | Pattern | Examples |
|---|---|---|
| Table | `PascalCase` noun | `Customer`, `OrderLine`, `NotificationJob` |
| View | `Role_Intent_V` | `Customer_Active_V`, `Order_Pending_V` |
| Stored procedure | `Verb_Domain_{trx,utx,ut}` | `TransferFunds_trx`, `AddOrderLine_utx`, `FindCustomerByEmail_ut` |
| Scalar function | `Domain_Intent_fn` | `NextLineNo_fn`, `Account_IsType_fn` |
| Inline TVF | `Domain_Intent_fn` | `GetCustomerOrders_fn` |
| Primary key | `PK_TableName` | `PK_Customer`, `PK_OrderLine` |
| Foreign key | `FK_ChildTable_ParentTable` | `FK_Order_Customer`, `FK_OrderLine_Order` |
| Unique constraint | `UQ_TableName_ColumnName` | `UQ_Customer_Email` |
| Check constraint | `CK_TableName_RuleName` | `CK_SavingsAccount_IsType` |
| Index | `IX_TableName_Columns` | `IX_Order_CustomerNo_OrderDate` |
| Custom type | `PascalCase` noun | `Email`, `PhoneNo`, `Money` |

**Verb vocabulary for procedures:**

| Verb | Meaning |
|---|---|
| `Add` | Insert a new record |
| `Modify` | Update an existing record |
| `Remove` | Delete a record |
| `Transfer` | Move a value between records |
| `Find` / `Get` | Read / query |
| `Validate` | Check without writing |
| `Process` | Multi-step workflow |

**Key rules:**
- Schema name prefix goes before the object name: `Sales.AddOrder_trx`, not `dbo.Sales_AddOrder_trx`.
- Suffixes (`_trx`, `_utx`, `_ut`, `_V`, `_fn`) are structural — they tell the caller what contract the object has.
- Abbreviations are banned unless universally understood (SQL, ID, No, URL). `Cust` is not `Customer`. `Ord` is not `Order`.
