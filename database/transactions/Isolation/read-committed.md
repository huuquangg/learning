# Read Committed — Study Note

## 1. Definition

`Read Committed` is a transaction isolation level that guarantees:

> A transaction can only read data committed by other transactions.

It prevents **Dirty Read**, but does not guarantee that data remains unchanged during the whole transaction.

## 2. Core behavior

When transaction A updates `Product 1`:

```text
Transaction A updates Product 1
→ Database immediately acquires an exclusive lock

Transaction B tries to update Product 1
→ Transaction B waits

Transaction A commits or rolls back
→ Lock is released

Transaction B continues
```

The database acquires the lock immediately. It does not wait for another transaction to create a conflict.

Transactions accessing unrelated data can continue:

```text
Transaction A updates Product 1
Transaction C updates Product 2
→ Both can normally execute concurrently
```

The actual lock might cover an index key, row, page, or table depending on the database, query plan, indexes, and lock escalation.

## 3. Lock lifetime

Generally, in locking-based `Read Committed`:

* Write locks are held until `COMMIT` or `ROLLBACK`.
* Read locks are usually released after the statement finishes.
* Readers may wait for uncommitted writes.

With MVCC or row versioning:

* Readers may read an older committed version.
* Readers do not necessarily block writers.
* Writers still need coordination when modifying the same data.

## 4. What it prevents

### Dirty Read

Transaction A changes stock but has not committed:

```sql
BEGIN TRANSACTION;

UPDATE Products
SET Stock = 0
WHERE Id = 1;
```

Transaction B cannot read that uncommitted value under `Read Committed`.

If A rolls back, B never observes a value that did not become permanent.

## 5. What it does not prevent

### Non-repeatable Read

The same transaction may read different committed values:

```text
Transaction A reads Stock = 10
Transaction B changes Stock to 9 and commits
Transaction A reads again and sees Stock = 9
```

### Phantom Read

Repeating a query may return additional or missing rows:

```text
First query:  10 pending orders
Another transaction inserts and commits an order
Second query: 11 pending orders
```

### Lost Update

This read–calculate–write pattern can be unsafe:

```text
A reads Stock = 1
B reads Stock = 1
A calculates and writes 0
B calculates and writes 0
```

Two orders may be created while the final stock is still `0`.

`Read Committed` fulfilled its guarantee because both transactions read committed data. It does not automatically protect an entire multi-statement business workflow.

## 6. Safe update pattern

Keep the validation and update in one atomic statement:

```sql
UPDATE Products
SET Stock = Stock - @Quantity
WHERE Id = @ProductId
  AND Stock >= @Quantity;
```

Then verify the affected rows:

```sql
IF @@ROWCOUNT = 0
    THROW 50001, 'Insufficient stock.', 1;
```

This avoids separating:

```text
Read stock → Calculate new value → Write new value
```

## 7. Transaction versus Read Committed

They solve different problems.

### Transaction

Ensures multiple operations succeed or fail together:

```sql
BEGIN TRANSACTION;

UPDATE Products
SET Stock = Stock - 1
WHERE Id = 1;

INSERT INTO Orders (ProductId, Quantity)
VALUES (1, 1);

COMMIT;
```

If the insert fails, `ROLLBACK` restores the stock.

This provides **Atomicity**.

### Read Committed

Controls what concurrent transactions may read.

It prevents them from reading changes that have not committed.

This provides a level of **Isolation**.

## 8. Without an explicit transaction

Every SQL statement still runs in an implicit auto-commit transaction.

```sql
UPDATE Products SET Stock = Stock - 1;
-- Committed immediately

INSERT INTO Orders (...);
-- Separate transaction
```

If the application crashes between the statements:

```text
Stock reduced
Order not created
```

Use an explicit transaction when multiple operations form one business unit.

## 9. Framework and database responsibilities

| Layer           | Responsibility                                   |
| --------------- | ------------------------------------------------ |
| Application     | Defines business rules and transaction boundary  |
| ADO.NET/EF Core | Begins, commits and rolls back transactions      |
| Database driver | Sends transaction commands                       |
| RDBMS           | Implements isolation, locking, MVCC and recovery |

Example with ADO.NET:

```csharp
await using SqlTransaction transaction =
    connection.BeginTransaction(IsolationLevel.ReadCommitted);

try
{
    // Execute commands using the same connection and transaction.

    await transaction.CommitAsync(cancellationToken);
}
catch
{
    await transaction.RollbackAsync(cancellationToken);
    throw;
}
```

No custom database function is required. A stored procedure is optional, not necessary.

## 10. When to use Read Committed

It is suitable for:

* Normal CRUD APIs.
* Short transactions.
* Independent row updates.
* Atomic `UPDATE` statements.
* Workflows where seeing newly committed data is acceptable.

Use additional protection for:

* Inventory reservation.
* Financial balance calculations.
* Seat allocation.
* Read–validate–write workflows.
* Business rules spanning multiple rows.

Possible protections include:

* Atomic conditional updates.
* Optimistic concurrency with a version column.
* Explicit locking such as `UPDLOCK`.
* `Repeatable Read`.
* `Serializable`.

## Final mental model

> `Read Committed` does not synchronize all transactions at one global moment. The database coordinates operations that conflict over the same resources, prevents dirty reads, and allows unrelated work to continue concurrently.

And:

> The RDBMS supplies the concurrency mechanism, the framework exposes transaction APIs, and the application must define and protect its business invariants.
