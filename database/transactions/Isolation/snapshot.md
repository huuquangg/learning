The main use case for **Snapshot Isolation** is:

> A feature must read multiple pieces of related data consistently, while allowing other users to continue updating the database without being blocked.

## Practical feature examples

### 1. Financial reports and account statements

Suppose a report executes:

```sql
SELECT SUM(Amount) FROM Orders;
SELECT SUM(Amount) FROM Payments;
SELECT SUM(Amount) FROM Refunds;
```

Without one consistent snapshot, new payments could commit between these queries. The report may combine data from different moments.

With Snapshot Isolation, all three queries see the database as it existed when the transaction began.

Suitable features:

* Monthly financial reports
* Customer account statements
* Balance reconciliation
* Audit reports
* Invoice generation

---

### 2. Dashboards containing multiple queries

A dashboard may load:

* Total orders
* Orders by status
* Revenue
* Top customers
* Recent transactions

If each query sees a different database state, the numbers may contradict one another.

Snapshot Isolation gives the entire dashboard calculation a consistent view:

```text
Total orders:       1,000
Completed:            700
Pending:              200
Cancelled:            100
                      ─────
                       1,000
```

Meanwhile, other users can continue creating and updating orders.

---

### 3. Data export

For example, exporting customers and their orders:

```text
Read customers
Read orders
Read payments
Generate CSV
```

Without a snapshot, an order could be updated after its customer was read, producing an inconsistent export.

Common examples:

* CSV/Excel exports
* Data migration
* ETL jobs
* Regulatory reports
* Database backup processes

For extremely large exports, however, a long-running snapshot can consume considerable version-storage space.

---

### 4. Loading an aggregate or complete API response

Consider an API:

```http
GET /api/orders/123
```

It loads:

```text
Order
├── Order items
├── Payments
├── Shipment
└── Customer
```

If these require several queries, concurrent updates could cause the response to contain:

* The old order total
* New order items
* An old payment status
* New shipment information

A snapshot can ensure that the complete response represents one logical moment.

This could apply to your PRM features when loading:

* Page plus controls and widgets
* Profile plus header and content sections
* Navigation tree plus permissions
* Design draft plus its configuration

However, Snapshot Isolation may be unnecessary if everything is obtained using one SQL statement because a single statement already receives a consistent view under common MVCC configurations.

---

### 5. Read-modify-write workflows on the same record

Example:

```text
T1 reads Document version 5
T2 reads Document version 5

T1 updates it → version 6 and commits
T2 tries to update it
```

Under Snapshot Isolation, the database can detect that T2 is updating a row changed since its snapshot. T2 fails and must retry.

Suitable features include:

* Editing documents
* Updating page configurations
* Updating user profiles
* Modifying application settings
* Reserving a specific resource row

This provides a form of optimistic concurrency, although explicit version columns are often clearer at the application level.

## When Snapshot Isolation is not enough

Avoid relying only on Snapshot Isolation when correctness depends on a condition across multiple rows.

Examples:

* “Never sell more items than total inventory.”
* “A room must not have overlapping bookings.”
* “At least one administrator must remain active.”
* “Total approved credit must not exceed the customer’s limit.”
* “Menu updates must never create a cycle.”

These can experience **Write Skew**, because two transactions may modify different rows without creating a direct write conflict.

Such features may require:

* `Serializable`
* Explicit locking
* Unique/check constraints
* Atomic conditional updates
* Application-level concurrency control

## Selection guide

| Feature                                           | Suitable isolation               |
| ------------------------------------------------- | -------------------------------- |
| Normal CRUD API                                   | Read Committed                   |
| Report using several related queries              | Snapshot                         |
| Dashboard requiring internally consistent numbers | Snapshot                         |
| Export of related tables                          | Snapshot                         |
| Read an object graph consistently                 | Snapshot                         |
| Update a single record with conflict detection    | Snapshot or version column       |
| Inventory deduction                               | Atomic update or Serializable    |
| Booking without overlap                           | Serializable or explicit locking |
| Cross-row business invariant                      | Usually Serializable/locking     |

## SQL Server example with .NET

```csharp
await using DbTransaction transaction =
    await connection.BeginTransactionAsync(IsolationLevel.Snapshot);

try
{
    // All queries observe the same database snapshot.
    await LoadOrderAsync(connection, transaction);
    await LoadPaymentsAsync(connection, transaction);
    await LoadShipmentAsync(connection, transaction);

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

In SQL Server, Snapshot Isolation must first be enabled for the database.

The simplest rule is:

> Use Snapshot when you need a stable, multi-query view of data with minimal reader/writer blocking. Use Serializable or explicit protection when you must enforce a business rule across concurrently changing rows.
