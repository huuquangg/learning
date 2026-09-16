# Serializable Transaction Isolation — Summary Note

## Core idea

`Serializable` is the strongest standard transaction isolation level.

It guarantees that concurrent transactions produce the same result as if they had executed **one at a time in some serial order**.

Transactions may still run concurrently, but the database uses locking, validation, waiting, or transaction abortion to prevent invalid outcomes.

## Critical example: credit limit

Business rule:

```text
Total unpaid orders must not exceed $100.
```

Two concurrent transactions each attempt to create an `$80` order.

Without `Serializable`:

1. Transaction A reads total unpaid amount as `$0`.
2. Transaction B also reads `$0`.
3. Both determine that `$0 + $80 ≤ $100`.
4. Both insert their orders.
5. Both commit.

Final result:

```text
Total unpaid = $160
Credit limit = $100
```

Each transaction was individually valid based on what it read, but their combined result violated the business rule.

## How Serializable solves it

With `Serializable`, the database must make the transactions behave as if they ran one at a time:

1. Transaction A reads `$0`, inserts `$80`, and commits.
2. Transaction B then reads `$80`.
3. Transaction B calculates `$80 + $80 = $160`.
4. Transaction B rejects the order.

The database may block one transaction or abort it with a deadlock/serialization error. The application must be ready to retry the entire transaction.

## Repeatable Read versus Serializable

`Repeatable Read` protects existing rows that a transaction has read.

It prevents another transaction from:

* Updating those rows
* Deleting those rows

However, another transaction may still insert a **new row** that matches the query. This is a `Phantom Read`.

`Serializable` protects:

* Existing matching rows
* The complete query condition
* The range where new matching rows could appear

Mental model:

> Repeatable Read protects the rows you found.
> Serializable protects the question you asked.

| Isolation level | Protects existing rows | Prevents new matching rows |
| --------------- | ---------------------: | -------------------------: |
| Read Committed  |   During the statement |                         No |
| Repeatable Read | Until transaction ends |                         No |
| Serializable    | Until transaction ends |                        Yes |

## SQL example

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN TRANSACTION;

DECLARE @UsedCredit DECIMAL(18, 2);

SELECT @UsedCredit = COALESCE(SUM(Amount), 0)
FROM Orders
WHERE CustomerId = 42
  AND Status = 'Unpaid';

IF @UsedCredit + 80 <= 100
BEGIN
    INSERT INTO Orders(CustomerId, Amount, Status)
    VALUES (42, 80, 'Unpaid');
END;
ELSE
BEGIN
    THROW 50001, 'Credit limit exceeded.', 1;
END;

COMMIT;
```

## Trade-offs

Benefits:

* Prevents dirty, non-repeatable, and phantom reads
* Prevents many lost-update and write-skew problems
* Protects critical multi-row business rules
* Provides the strongest consistency guarantee

Costs:

* More blocking
* Lower concurrency
* Higher deadlock risk
* Transactions may fail and require retries
* Long transactions can significantly reduce performance

## When to use it

Use `Serializable` when a transaction makes a critical decision based on data that concurrent transactions must not invalidate, such as:

* Credit and spending limits
* Limited inventory
* Seat or appointment reservations
* Scheduling conflicts
* Maximum-user licensing
* Business rules involving multiple rows

For simpler rules, database constraints, unique indexes, or atomic updates may provide the same protection more efficiently.
