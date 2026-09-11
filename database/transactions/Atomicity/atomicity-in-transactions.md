# Atomicity in Transactions

Atomicity is the **all-or-nothing** property of a database transaction:

- If every operation succeeds, the database makes the transaction's changes
  visible as one unit.
- If any operation fails, the database removes every change made by that
  transaction.

The database must never expose a half-completed transaction.

## Example: transferring money

Suppose Alice transfers 100 to Bob:

1. Subtract 100 from Alice's account.
2. Add 100 to Bob's account.

The valid outcomes are:

| Outcome | Alice | Bob | Meaning |
| --- | ---: | ---: | --- |
| Commit | -100 | +100 | The complete transfer happened. |
| Rollback | 0 | 0 | Neither account was changed. |

An outcome where Alice loses 100 but Bob receives nothing violates atomicity.
The same idea applies to an order, its inventory reservation, and its payment:
the application should not leave only some of those changes behind.

## The pieces involved

The companion diagram, [atomicity-in-transaction.excalidraw](./atomicity-in-transaction.excalidraw),
shows the normal relationship between the transaction, transaction log, and
data pages.

- **Transaction**: the logical unit of work, such as the transfer.
- **Data page**: an in-memory or on-disk page containing table or index data.
- **Transaction log**: an append-only record of changes and transaction state.
  A common implementation uses write-ahead logging (WAL).
- **Commit record**: a log record stating that the transaction completed.
- **Rollback/undo**: restoration of the before-images (or an equivalent
  compensation mechanism) for changes made by an uncommitted transaction.

## Typical workflow

1. The transaction begins.
2. It reads the required rows and changes the affected data pages in memory.
3. For each change, the database records enough information in the transaction
   log to redo or undo the change.
4. Before reporting success, the database flushes the transaction's log records
   and commit record according to its durability rules. This is the
   **write-ahead** part: the log must be safe before the corresponding dirty
   data pages are allowed to be written.
5. The database returns `COMMIT` and later flushes dirty data pages.
6. If an error occurs before commit, the database writes or performs rollback
   work, undoing all changes belonging to the transaction, and returns an error.

The exact ordering and recovery algorithm differ between database engines, but
the atomicity contract remains the same.

## Crash recovery intuition

After a crash, recovery examines the log:

- A transaction with a durable **commit record** is treated as committed. Its
  changes are redone if its data pages were not written yet.
- A transaction with no durable commit record is treated as incomplete. Its
  changes are undone.

This is why the log is more than an audit trail: it is the source of truth
needed to reconstruct a consistent state.

## Pseudocode

```text
BEGIN
try:
    UPDATE accounts SET balance = balance - 100 WHERE id = 'alice'
    UPDATE accounts SET balance = balance + 100 WHERE id = 'bob'
    COMMIT
except error:
    ROLLBACK
    raise
```

Applications should keep the transaction boundary around the complete unit of
work and should not catch an error and continue as though the transaction
partially succeeded.

## Atomicity is not durability

These ACID properties are related but different:

- **Atomicity** answers: “Do all operations happen, or none?”
- **Durability** answers: “After commit succeeds, will the result survive a
  crash?”

The transaction log supports both: undo information helps atomicity, while
redo information and a durable commit record help recovery and durability.

## Practical checklist

- Identify the smallest business operation that must be all-or-nothing.
- Put every required write inside one transaction boundary.
- Check that errors cause rollback rather than partial success.
- Do not report success before commit succeeds.
- Test failures between each pair of writes and during crash recovery.
- Remember that atomicity does not by itself prevent lost updates or guarantee
  valid business rules; isolation and constraints address those concerns.
