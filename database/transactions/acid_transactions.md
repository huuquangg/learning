# ACID Transactions

> **Goal:** understand why ACID exists, distinguish its four guarantees, and use
> transactions safely in practical SQL.

![ACID transaction map](./acid_transactions.excalidraw)

## Start from the problem

A **transaction** is a group of database operations treated as one logical
unit.

Consider transferring 100 units from account 1 to account 2:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;
```

Without additional guarantees, four different things can go wrong:

1. The system stops after the debit but before the credit.
2. The final data violates a rule such as `balance >= 0`.
3. Another transaction interferes while the transfer is running.
4. The database says "committed," then loses the change after a crash.

ACID is not an arbitrary list. Its four properties are answers to these four
failure modes.

| Risk | Property | Question it answers |
|---|---|---|
| Partial work | **Atomicity** | Can half of the transaction survive? |
| Invalid state | **Consistency** | Does committed data obey its invariants? |
| Concurrent interference | **Isolation** | Which concurrent outcomes are allowed? |
| Loss after success | **Durability** | Can an acknowledged commit disappear? |

## A - Atomicity: all or nothing

**Atomicity means that a transaction has one outcome: all its operations take
effect, or none of them do.**

If the second update fails, the first update must not remain:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

-- Imagine this fails because account 2 does not exist.
UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

ROLLBACK;
```

Atomicity protects the **transaction boundary**. It does not prove that the SQL
inside the boundary is logically correct.

For example, both updates could successfully affect zero rows because the IDs
were wrong. The transaction would still be atomic: all zero intended changes
were applied. Production code must verify affected-row counts.

### Savepoints are partial rollback markers

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT debit_complete;

-- Try optional work.
INSERT INTO transfer_notifications(account_id) VALUES (1);

-- Undo only work after the savepoint if needed.
ROLLBACK TO SAVEPOINT debit_complete;

COMMIT;
```

A savepoint does not weaken atomicity. The final `COMMIT` still publishes one
transaction outcome.

## C - Consistency: preserve declared invariants

An **invariant** is a rule that must be true for every valid database state.

```sql
CREATE TABLE accounts (
    id      bigint PRIMARY KEY,
    balance numeric(12, 2) NOT NULL CHECK (balance >= 0)
);
```

The `CHECK` constraint declares that a negative balance is invalid. A
transaction that violates it cannot commit that invalid row.

The key distinction is:

- The **database** enforces constraints actually declared in the schema.
- The **application transaction** must correctly enforce business rules that
  are not represented by constraints.

ACID cannot infer that money must be conserved, that a transfer needs approval,
or that the wrong `WHERE` clause was used. "Consistency" does **not** mean "the
database makes all business logic correct."

## I - Isolation: control concurrent interference

Transactions often overlap:

```text
time ------------------------------------------------>

Transaction A: read 100 ------------- write 90
Transaction B:       read 100 ------------- write 80
```

If both applications calculate a new absolute value from the stale balance
`100`, the final balance may become `80`. Transaction A's subtraction is lost.
Each transaction can be atomic while their interaction is still wrong.

**Isolation restricts what concurrent transactions may observe and which
combined outcomes are permitted.**

The strongest common model, **serializable isolation**, guarantees that the
committed result is equivalent to some serial order:

```text
A then B
```

or:

```text
B then A
```

The transactions may physically overlap; their observable result must behave
as though they ran one at a time.

### Isolation levels

The SQL standard describes progressively stronger guarantees. Database engines
can provide stronger behavior than the minimum, so always check the engine's
documentation.

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---:|---:|---:|
| Read uncommitted | possible | possible | possible |
| Read committed | prevented | possible | possible |
| Repeatable read | prevented | prevented | possible by SQL standard |
| Serializable | prevented | prevented | prevented |

Definitions:

- **Dirty read:** read another transaction's uncommitted value.
- **Non-repeatable read:** read one row twice and get different committed
  values.
- **Phantom read:** repeat a predicate query and get a changed set of rows.
- **Lost update:** one writer overwrites a result calculated by another writer.

### Why `READ COMMITTED` can return two answers

Many databases use `READ COMMITTED` by default. In PostgreSQL, each statement
gets a new snapshot:

```sql
-- Session A
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT balance FROM accounts WHERE id = 1; -- 100

-- Session B commits: UPDATE accounts SET balance = 70 WHERE id = 1;

-- Session A gets a newer statement snapshot.
SELECT balance FROM accounts WHERE id = 1; -- 70
COMMIT;
```

Session A did not perform a dirty read: `70` was committed. But its read was
non-repeatable because the transaction did not retain one snapshot for both
statements.

This is the important boundary:

> Preventing dirty reads is weaker than making a whole transaction behave as
> if it ran alone.

### Safer update patterns

Prefer a relative update over client-side read-modify-write:

```sql
UPDATE accounts
SET balance = balance - 20
WHERE id = 1
  AND balance >= 20
RETURNING balance;
```

For workflows spanning multiple rows, choose based on the invariant:

- Lock the rows you will base decisions on with `SELECT ... FOR UPDATE`.
- Use optimistic concurrency with a version column and reject stale writes.
- Use `SERIALIZABLE` and retry transactions rejected with serialization
  failures.
- Encode rules as constraints where the database can express them.

Higher isolation is not "free safety." It can increase waiting, deadlocks,
aborts, and retries. Choose the weakest level that still preserves the
workflow's invariants.

## D - Durability: commit survives failure

**Durability means that after the database acknowledges a successful commit,
the committed result survives failures covered by its durability model.**

Databases commonly use a write-ahead log (WAL):

1. Describe the change in a log record.
2. Make the required log record durable.
3. Acknowledge `COMMIT`.
4. Write modified data pages later.

After a crash, recovery can replay committed log records. The exact guarantee
still depends on configuration and infrastructure. Asynchronous commit, broken
storage, or an unprotected regional disaster can create loss modes outside the
chosen guarantee.

Durability is different from:

- **Atomicity:** whether the outcome is all or nothing.
- **Replication:** whether copies exist elsewhere.
- **Backup:** whether an older recoverable copy exists.

## Putting all four together

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1
  AND balance >= 100;
-- Application must verify exactly one row changed.

UPDATE accounts
SET balance = balance + 100
WHERE id = 2;
-- Application must verify exactly one row changed.

COMMIT;
```

What each property contributes:

- **Atomicity:** debit and credit become visible together or not at all.
- **Consistency:** constraints reject invalid committed states; application
  checks complete the business invariant.
- **Isolation:** concurrent transfers behave like a permitted serial order, or
  one is rejected and must be retried.
- **Durability:** after successful commit acknowledgment, recovery preserves
  the transfer within the configured failure model.

## A compact mental model

Think of a transaction as crossing a guarded bridge:

- **Atomicity:** you reach the other side or return to the start.
- **Consistency:** guards reject a destination that breaks declared rules.
- **Isolation:** other travelers cannot force an invalid crossing.
- **Durability:** once arrival is confirmed, a crash does not erase it.

Or remember the four risks, which are more useful than memorizing the letters:

```text
partial work -> invalid state -> concurrency interference -> post-commit loss
     A               C                   I                     D
```

## Practical checklist

Before shipping a transaction, ask:

1. What exact operations form the atomic unit?
2. Which invariants are encoded as database constraints?
3. Which invariants remain application responsibilities?
4. What concurrent reads and writes can touch the same data?
5. Does the selected isolation level permit a harmful outcome?
6. Are lock waits, deadlocks, and serialization failures retried correctly?
7. Does the durability configuration match the acceptable loss window?
8. Does the application verify affected-row counts before committing?

## Self-check

1. A transfer debits one account, then crashes before crediting the other.
   Which property prevents the half-transfer from surviving?
2. A transaction reads the same committed row twice and sees two values. Is
   that a dirty read or a non-repeatable read?
3. Why can two individually atomic transactions still produce a wrong result?
4. Does a `COMMIT` prove that the transaction's business logic was correct?
5. Why might a serializable transaction need to be retried?

<details>
<summary>Answers</summary>

1. Atomicity.
2. A non-repeatable read. A dirty read observes an uncommitted value.
3. Atomicity governs each transaction's own all-or-nothing outcome; isolation
   governs interaction between transactions.
4. No. Commit says the database accepted the transaction under its constraints
   and isolation rules, not that every application intention was correct.
5. The database may reject one of several concurrent transactions when their
   combined execution cannot safely be represented as a serial order.

</details>
