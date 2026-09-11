ACID
 │
 ├── Atomicity
 │      └── Transaction Log
 │             └── Undo / Rollback
 │
 ├── Durability
 │      └── WAL
 │             └── Flush / Checkpoint / Recovery
 │
 ├── Isolation
 │      └── Concurrent Transactions
 │              ├── Dirty Read
 │              ├── Non-repeatable Read
 │              ├── Phantom Read
 │              └── Lost Update
 │
 │      └── Isolation Levels
 │              ├── Read Uncommitted
 │              ├── Read Committed
 │              ├── Repeatable Read
 │              ├── Serializable
 │              └── Snapshot
 │
 │      └── Implementation
 │              ├── Locking
 │              │     ├── Blocking
 │              │     └── Deadlock
 │              │
 │              └── MVCC
 │
 └── Consistency
        ├── Constraints
        ├── Business invariants
        └── Interaction with concurrency