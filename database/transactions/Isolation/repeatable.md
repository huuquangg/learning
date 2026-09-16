# Non-repeatable Read — Summary Note

## Definition

A **Non-repeatable Read** occurs when one transaction reads the same row twice but receives different values because another transaction updates and commits that row between the two reads.

```text
Transaction A reads Price = $100
Transaction B updates Price = $120 and commits
Transaction A reads Price again = $120
```

Both values are committed and valid, but Transaction A does not see a stable version of the row.

## When it becomes critical

It becomes dangerous when an application follows this pattern:

```text
Read → Make decision → Data changes → Perform action
```

Typical critical cases include:

* Payments and account balances
* Inventory reservation
* Order approval or cancellation
* Permission and authorization checks
* Financial calculations and reports
* Business workflow transitions

### Example

Transaction A:

1. Reads price `$100`.
2. Deducts `$100` from the customer.
3. Transaction B changes the price to `$120`.
4. Transaction A reads the price again.
5. Records a payment of `$120`.

Final inconsistent data:

```text
Amount deducted: $100
Payment recorded: $120
```

The transaction is atomic, but its business result is inconsistent.

## Under `Read Committed`

In a traditional locking implementation:

```text
Read row
→ release shared lock after the statement
→ another transaction updates and commits
→ read the new value
```

`Read Committed` prevents **Dirty Reads**, but it does not prevent Non-repeatable Reads.

It is still suitable when:

* A row is read only once.
* The latest committed value is desired.
* Changes between reads are acceptable.
* The operation is informational.
* Higher concurrency is more important than a stable transaction view.

## Using `Repeatable Read`

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN TRANSACTION;

SELECT Price
FROM Products
WHERE Id = 1;

-- Business operations

SELECT Price
FROM Products
WHERE Id = 1;

COMMIT;
```

In a locking database such as SQL Server:

```text
Transaction A reads row
→ keeps shared lock until COMMIT
→ Transaction B attempts update
→ Transaction B waits
→ Transaction A reads the same value
→ Transaction A commits
→ Transaction B can update
```

Therefore, the row cannot be modified between Transaction A’s reads.

## Trade-offs

| Level                    | Benefit                              | Cost                                      |
| ------------------------ | ------------------------------------ | ----------------------------------------- |
| `Read Committed`         | Better concurrency                   | Values may change between reads           |
| `Repeatable Read`        | Previously read rows remain stable   | More blocking and possible deadlocks      |
| Snapshot-based isolation | Stable data without blocking writers | Requires row-version storage              |
| `Serializable`           | Strongest transaction isolation      | Lowest concurrency and greater contention |

`Repeatable Read` protects rows already read, but it may not prevent new matching rows from appearing—known as a **Phantom Read**.

## Alternative solutions

Higher isolation is not always necessary.

### Read once and reuse

```sql
SELECT @Price = Price
FROM Products
WHERE Id = 1;

-- Reuse @Price for the entire calculation
```

### Conditional update

```sql
UPDATE Orders
SET Status = 'Paid'
WHERE Id = 42
  AND Status = 'Pending';
```

Check the affected-row count:

```text
1 row → transition succeeded
0 rows → state changed; reject or retry
```

Other options include:

* `rowversion` or version-column optimistic concurrency
* `SELECT ... FOR UPDATE`
* SQL Server update locks
* Snapshot isolation
* `Serializable`

## Core takeaway

> Non-repeatable Read is not automatically an error. It becomes critical when business logic assumes that a value read earlier will remain unchanged until the transaction finishes.

Use `Repeatable Read` when the same rows must remain stable, but prefer reading once or using atomic conditional updates when they express the business rule more directly.
