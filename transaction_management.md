# Transaction Management

## 1. Why Transactions Matter

Sometimes a single task requires updating a database in multiple places. If only some of those updates happen (because of a crash, a bug, or a conflict with another user), the database ends up in a state that doesn't make sense.

The classic example is a bank transfer. Transferring £20 from Alice to Bob involves two writes: debit Alice's account, credit Bob's account. If the system crashes between the debit and the credit, £20 has vanished. The database is inconsistent — the total money in the system has changed, which should never happen.

A **transaction** groups these related operations into a single logical unit. The DBMS guarantees that either *all* of the operations happen, or *none* of them do. This is the foundation of database reliability.


## 2. Two Scenarios That Create Challenges

### System failure

A failure (power cut, crash, bug) can strike at any moment — including halfway through a transaction. Even if a transaction has finished its work, the results may still be sitting in temporary memory (buffers) and not yet written to disk. So a crash can lose completed work as well as incomplete work.

### Concurrency

In a busy database, many transactions run at the same time. Their operations get interleaved — the database doesn't finish one transaction before starting the next. This interleaving can cause interference between transactions. Three classic problems illustrate this:

**The lost update problem.** Two transactions both read the same balance (£100). T1 subtracts £20 and writes £80. T2 adds £20 and writes £120. But T2's write overwrites T1's — the £20 debit is lost. The balance should be £100 (both changes applied), but it's £120.

**The uncommitted dependency (dirty read) problem.** T1 updates a balance from £100 to £80, but then aborts (rolls back). Before the rollback, T2 reads the £80 value and uses it. T2 is now working with a value that was never meant to exist in the database.

**The inconsistent analysis problem.** T1 is summing up all account balances. Meanwhile T2 is transferring money between two accounts. T1 reads one account before the transfer and another account after the transfer, producing a total that's wrong — it has counted the transferred money twice (or not at all).


## 3. The ACID Properties

For transactions to protect us from these problems, the DBMS should guarantee four properties — collectively known as ACID:

**Atomicity** — "all or nothing." A transaction either completes entirely or has no effect at all. If it fails partway through, all its changes are undone. This is the responsibility of the *recovery subsystem*.

**Consistency** — a transaction takes the database from one valid state to another valid state. The DBMS enforces integrity constraints (foreign keys, check constraints, etc.), but consistency also depends on the application logic being correct. If a programmer debits the wrong account, the DBMS can't catch that.

**Isolation** — the partial, in-progress work of one transaction is invisible to other transactions. Each transaction behaves as if it's the only one running. This is the responsibility of the *concurrency control subsystem*.

**Durability** — once a transaction commits, its effects are permanent. Even if the system crashes one millisecond after commit, the changes survive. A committed transaction can only be reversed by running a separate *compensating transaction*. This is the responsibility of the *recovery subsystem*.


## 4. Transaction States

A transaction moves through these states:

- **Active** — the transaction is executing its operations. It hasn't committed or aborted yet.
- **Partially committed** — the final operation has executed, but the DBMS hasn't confirmed that everything is safely on disk yet. If something goes wrong at this point (a constraint violation is discovered, or the system crashes before flushing buffers), the transaction moves to the failed state.
- **Committed** — the transaction has completed successfully and its effects are permanent. It can no longer be rolled back.
- **Failed** — something has gone wrong and the transaction cannot continue.
- **Aborted** — the transaction has been rolled back. All its changes have been undone and the database is back to the state before the transaction started. The transaction may be restarted later.


## 5. The Transaction Management Subsystem

The DBMS has several components working together:

- **Transaction manager** — coordinates transactions on behalf of application programs.
- **Scheduler (or lock manager)** — decides how to interleave operations from multiple concurrent transactions, maximising concurrency while preventing interference.
- **Recovery manager** — restores the database to a consistent state after a failure.
- **Buffer manager** — handles the transfer of data between disk and main memory (buffers), deciding when to flush buffers to disk.


---


## 6. Concurrency Control — Core Concepts

### Schedules

A **schedule** is a sequence showing the interleaved operations of a set of concurrent transactions, preserving the internal order of each transaction. In exam notation, it might look like:

```
T1: Read(A)
T2: Read(B)
T1: Write(A)
T2: Write(B)
```

A **serial schedule** runs transactions one at a time with no interleaving — all of T1, then all of T2 (or vice versa). Serial schedules are always correct because no interference is possible. However, they're slow because the CPU sits idle during I/O waits instead of doing useful work on other transactions.

A **nonserial schedule** interleaves operations from multiple transactions. Most nonserial schedules are fine, but some can produce incorrect results. The goal is to identify which nonserial schedules are safe.

### Serializability

A nonserial schedule is **serializable** if it produces the same final result as *some* serial schedule. It doesn't matter which serial order — just that at least one exists. If a schedule is serializable, we can trust it.

There are different ways to test for serializability, each with different trade-offs.


## 7. Simple Serializability

The most conservative approach. It says:

- Read-only transactions can run concurrently with any other read-only transaction (reads never conflict).
- Transactions that touch completely different data items can run concurrently.
- Everything else must wait — if two transactions share any data item and at least one writes, they cannot be interleaved.

This is easy to check and always safe, but it's overly restrictive. Many interleavings that would be perfectly safe are blocked. We can do better.


## 8. Conflict Serializability

Conflict serializability is the most widely used test. It asks: can we rearrange this interleaved schedule into a serial one by swapping only non-conflicting adjacent operations?

### What is a conflict?

Two operations **conflict** if all three conditions hold:

1. They belong to **different transactions**.
2. They access the **same data item**.
3. **At least one is a write**.

If both are reads, or if they touch different data, the order doesn't matter and we can freely swap them.

### The precedence graph

Rather than trying all possible swaps, we build a **precedence graph** (or serialization graph):

1. Create a **node** for each transaction.
2. For each pair of conflicting operations where Ti's operation comes before Tj's in the schedule, draw a **directed edge Ti → Tj**.

An edge Ti → Tj means "in any equivalent serial schedule, Ti must come before Tj."

**If the graph has no cycle**, the schedule is conflict serializable. The equivalent serial schedule(s) can be found by topological sorting — listing the nodes in an order that respects all arrows.

**If the graph has a cycle**, the schedule is not conflict serializable. There's no serial ordering that's consistent with all the constraints, because the constraints contradict each other.

### Worked example

```
T1: Read(A)
T2: Read(A)
T1: Write(A)
T2: Write(A)
```

Conflicts on data item A:
- T1 Read(A) before T2 Write(A) → **T1 → T2** (T1 read the value before T2 overwrote it)
- T2 Read(A) before T1 Write(A) → **T2 → T1** (T2 read the value before T1 overwrote it)

The precedence graph has a cycle: T1 → T2 → T1. This schedule is **not** conflict serializable.

### What to do about a non-serializable interleaving

If a cycle is discovered, the DBMS must abort and roll back one of the transactions involved in the cycle. This breaks the cycle and allows the remaining transactions to proceed.


## 9. View Serializability

View serializability asks a more fundamental question than conflict serializability: does each transaction *see* the same values, and does the database end up in the same final state, as in some serial schedule?

Two schedules are **view equivalent** if:

1. **Same initial reads.** Any transaction that reads the initial (unmodified) value of a data item in one schedule also reads the initial value in the other.
2. **Same reads-from relationships.** If Ti reads a value written by Tj in one schedule, then Ti must also read the value written by Tj in the other.
3. **Same final writes.** Whichever transaction performs the last write to each data item must be the same in both schedules.

A schedule is **view serializable** if it is view equivalent to some serial schedule.

### Relationship to conflict serializability

Every conflict serializable schedule is automatically view serializable. But some schedules are view serializable without being conflict serializable. The difference is always due to **blind writes** — writes that happen without the transaction having read the data item first.

When a blind write occurs, its value might be immediately overwritten by another transaction and never read by anyone. Conflict serializability still sees a write-write conflict and insists on an ordering. View serializability is smarter — it recognises that if nobody ever used the value, the ordering of that write doesn't matter.

### Practicality

Testing for view serializability is **NP-complete**, meaning there's almost certainly no efficient general algorithm. That's why practical database systems use protocols (like locking or timestamping) that guarantee conflict serializability rather than trying to achieve the broader view serializability.

### The Venn diagram

Think of the relationship as concentric sets:

- All schedules (outermost — most are not safe)
  - View serializable schedules (safe)
    - Conflict serializable schedules (safe and efficiently testable)

Two-phase locking and timestamping each produce subsets of schedules that fall within the conflict serializable region. They overlap with each other but neither is a subset of the other — some schedules are achievable with locking but not timestamping, and vice versa. With the "ignore obsolete write" rule, timestamping can additionally produce some view serializable schedules that go beyond conflict serializability.


## 10. Recoverability

Serializability ensures the *effect* on the database is correct. But we also need to worry about what happens when transactions fail. This is where **recoverability** comes in.

### The problem

Suppose T1 writes a value, then T2 reads that value and commits. Later, T1 aborts. The atomicity property says we must undo all of T1's effects. But T2 has already committed using T1's value, and the durability property says committed transactions can't be undone. We're stuck — atomicity and durability contradict each other. This is a **non-recoverable schedule**, and it must never be allowed.

### The rule

A schedule is **recoverable** if, whenever Tj reads a data item previously written by Ti, Ti commits before Tj commits. In notation:

> If Ti.Write(x) happens before Tj.Read(x), then Ti.Commit must happen before Tj.Commit.

This ensures that if Ti aborts, we can safely cascade that abort to Tj (since Tj hasn't committed yet).

### Cascading rollbacks

Even with recoverable schedules, we can end up with **cascading rollbacks** — one abort triggers another, which triggers another. This is undesirable because it wastes a lot of work. To prevent cascading rollbacks, we use stricter scheduling rules (like rigorous 2PL, discussed below).


---


## 11. Locking

Locking is the most widely used concurrency control technique. The idea is simple: before accessing a data item, a transaction must claim a lock on it. The lock prevents other transactions from interfering.

### Lock types

**Shared lock (read lock)** — the transaction intends to read the item. Multiple transactions can hold shared locks on the same item simultaneously (reads don't conflict). But no transaction can obtain an exclusive lock while shared locks are held.

**Exclusive lock (write lock)** — the transaction intends to write the item. Only one transaction can hold an exclusive lock on an item at a time. No other transaction can read or write the item until the lock is released.

Some systems allow **lock upgrading** (shared → exclusive) and **downgrading** (exclusive → shared) during a transaction.

### Locking alone is not enough

Using locks doesn't automatically guarantee serializability. If a transaction releases a lock on one item and then acquires a lock on another, another transaction can slip in between and create interference. The textbook gives a detailed example (Example 22.5) where two transactions use locks correctly on individual items but still produce a non-serializable result because locks are released too early.

### Two-Phase Locking (2PL)

To guarantee conflict serializability, transactions must follow the **two-phase locking protocol**:

1. **Growing phase** — the transaction acquires all the locks it needs. It may acquire locks at different times (not all at once), but it never releases any lock during this phase.
2. **Shrinking phase** — once the transaction releases its first lock, it enters the shrinking phase. It can release locks but can never acquire new ones.

The key rule: **all lock acquisitions must happen before any lock release.**

If every transaction follows 2PL, the resulting schedule is guaranteed to be conflict serializable.

### The cascading rollback problem with basic 2PL

Basic 2PL guarantees serializability but can still allow cascading rollbacks. Consider: T1 acquires a lock on X, writes X, then releases the lock on X (entering the shrinking phase) while still doing other work. T2 immediately locks X and reads the value T1 wrote. If T1 later aborts, T2 has used a dirty value and must also be rolled back — and any transaction that read T2's output must also be rolled back, and so on.

### Rigorous 2PL

The solution is **rigorous 2PL**: don't release *any* lock until the transaction commits or aborts. This means no other transaction can see a transaction's changes until they're permanent, which eliminates cascading rollbacks entirely. Most real database systems use rigorous 2PL (or the slightly less strict "strict 2PL" which holds only exclusive locks until the end).


## 12. Deadlock and Livelock

Deadlock and livelock are problems unique to locking-based systems.

### Deadlock

**Deadlock** occurs when two or more transactions are each waiting for locks held by the other, creating a circular wait. Neither can proceed.

Example: T1 holds an exclusive lock on X and requests a lock on Y. T2 holds an exclusive lock on Y and requests a lock on X. Both wait forever.

The only way to break a deadlock is to **abort one or more** of the involved transactions, releasing their locks so others can proceed.

### Three techniques for handling deadlock

**Timeouts.** Each lock request has a time limit. If the lock isn't granted within that period, the DBMS assumes deadlock and aborts the transaction. Simple and practical — used by many commercial systems — but can abort transactions that aren't actually deadlocked.

**Prevention.** Use policies that make deadlock structurally impossible:

- *Wait-Die*: an older transaction may wait for a younger one, but a younger transaction requesting a lock held by an older one must abort ("die") and restart with the same timestamp.
- *Wound-Wait*: a younger transaction may wait for an older one, but if an older transaction requests a lock held by a younger one, the younger one is forced to abort ("wounded").

Both schemes prevent circular waits by imposing a consistent ordering based on transaction age.

- *Conservative 2PL*: acquire all locks at the start of the transaction. If any lock is unavailable, release everything and retry. This prevents deadlock but is impractical because transactions often don't know upfront which locks they'll need, so they over-lock, reducing concurrency.

**Detection and recovery.** Let deadlocks happen, but detect them periodically:

- Build a **wait-for graph (WFG)**: nodes are transactions, and there's an edge Ti → Tj if Ti is waiting for a lock held by Tj.
- A **cycle in the WFG** means deadlock exists.
- Choose a victim (preferring younger, smaller, or less-progressed transactions) and abort it.
- Care must be taken to avoid **starvation** — the same transaction being chosen as victim repeatedly.

### Livelock (starvation)

A transaction can be stuck waiting indefinitely even without deadlock if the scheduling algorithm is unfair. For example, if new transactions keep jumping the queue. Solution: use a priority scheme (e.g., first-come-first-served) so that waiting transactions eventually get served.


## 13. Timestamping

Timestamping is a fundamentally different approach to concurrency control. Instead of locks, each transaction is assigned a unique **timestamp** when it starts (typically from a logical counter or the system clock). The DBMS uses these timestamps to ensure that transactions execute in an order consistent with their timestamps.

**Key advantage: no locks means no deadlock.** Conflicts are resolved by aborting and restarting one of the conflicting transactions, not by making it wait.

### How it works

Each data item X in the database maintains two timestamps:

- **read_timestamp(X)** — the timestamp of the most recent transaction to read X.
- **write_timestamp(X)** — the timestamp of the most recent transaction to write X.

When a transaction T (with timestamp ts(T)) tries to access X, the system checks for conflicts:

**T wants to Read(X):**

- If ts(T) < write_timestamp(X): a younger (later) transaction has already overwritten X. T would be reading a stale value that shouldn't exist anymore in the assumed serial order. **Abort T** and restart it with a new timestamp.
- Otherwise: the read is safe. Proceed and update read_timestamp(X) = max(ts(T), read_timestamp(X)).

**T wants to Write(X):**

- If ts(T) < read_timestamp(X): a younger transaction has already read the current value of X. If T overwrites it now, that younger transaction's read becomes invalid. **Abort T** and restart.
- If ts(T) < write_timestamp(X): a younger transaction has already written a newer value. T's write is obsolete. Under the basic protocol, T would be aborted. But under **Thomas's write rule** (the "ignore obsolete write" rule), we can simply **skip T's write** — it would have been overwritten anyway. This allows greater concurrency and can produce some view serializable schedules that aren't conflict serializable.
- Otherwise: the write is safe. Proceed and update write_timestamp(X) = ts(T).

### Comparison with locking

| | Locking | Timestamping |
|---|---|---|
| Conflict handling | Transaction waits | Transaction is aborted and restarted |
| Deadlock | Possible | Impossible (no waiting) |
| Restart overhead | Low (transactions wait rather than restart) | Higher (aborted transactions must redo all work) |
| Schedule space | Subset of conflict serializable | Overlapping but different subset |


## 14. Multiversion Timestamp Ordering

A refinement of timestamping that keeps **multiple versions** of each data item. When a transaction reads X, the system gives it the correct version based on its timestamp — the most recent version whose write timestamp is ≤ ts(T).

The key benefit: **reads never fail.** A read can always find an appropriate historical version. Only writes can trigger an abort (if a younger transaction has already read the version that would be affected).

This trades increased **storage** (keeping old versions around) for increased **concurrency** (fewer aborts, especially for read-heavy workloads). Oracle uses a variant of this approach (multiversion read consistency).


## 15. Optimistic Concurrency Control

Both locking and timestamping are **pessimistic** — they assume conflicts will happen and take precautions in advance. Optimistic methods take the opposite approach: assume conflicts are rare and let transactions run freely, only checking for problems at commit time.

### Three phases

1. **Read phase** — the transaction reads from the database and makes all modifications to a *local copy* of the data (not the database itself).
2. **Validation phase** — before committing, the system checks whether this transaction's reads and writes conflict with any other transaction. If there's a conflict, the transaction is aborted and restarted.
3. **Write phase** — if validation passes, the local changes are applied to the actual database.

### When it works well

Optimistic control is ideal when conflicts are genuinely rare (e.g., a database where most transactions touch different data). The majority of transactions sail through without any delays. However, if conflicts are common, the repeated abort-and-restart overhead can be worse than the delay caused by locking.

Because changes are made to local copies, aborting a transaction is cheap — there's no need to undo changes in the database, and there are no cascading rollbacks.


## 16. Granularity of Data Items

All concurrency control techniques operate on "data items," but how big is a data item? The **granularity** — the size of the unit being locked or timestamped — has a significant effect on performance.

The spectrum from coarse to fine:

| Granularity | Example | Concurrency | Overhead |
|---|---|---|---|
| Entire database | Lock the whole DB | Very low | Very low |
| Table/file | Lock the Staff table | Low | Low |
| Page | Lock a disk block | Medium | Medium |
| Row/record | Lock one staff member | High | High |
| Field | Lock one column of one row | Very high | Very high |

**Coarser granularity** means fewer locks to manage, but more data is blocked — other transactions can't touch anything in the locked region even if they only need a small part of it.

**Finer granularity** means more concurrency, but more locks to track and a higher chance of deadlock.

### Multiple-granularity locking

A practical approach: represent the database as a **hierarchy** (database → files → pages → records → fields). When a transaction locks a node, all its descendants are implicitly locked too.

To avoid searching the entire tree to check for conflicts, the system uses **intention locks** — lightweight flags placed on ancestor nodes to indicate that some descendant is locked. For example, if T1 locks a specific row, intention locks are placed on the containing page, file, and database. When T2 later tries to lock the whole file, it sees the intention lock and knows a descendant is already in use.


---


## 17. Recovery

Recovery is about restoring the database to a correct state after a failure. The recovery manager must ensure atomicity (undo incomplete transactions) and durability (don't lose committed transactions).

### Types of failure

- **Loss of main memory** — system crash, power failure. Database buffers (which may contain uncommitted or recently committed changes) are lost. The data on disk survives but may be inconsistent.
- **Loss of secondary storage** — disk crash, corruption. The database itself is damaged or destroyed. Requires restoring from backups.

Other causes include software bugs, human error, and physical disasters. The DBMS must handle all of these.

### The fundamental problem

Writing data involves several steps: compute the new value, write it to a buffer in memory, and eventually flush the buffer to disk. A crash can happen between any of these steps. So:

- A committed transaction's changes might not have reached disk yet → must be **redone** (rollforward).
- An uncommitted transaction's changes might have reached disk → must be **undone** (rollback).

### Recovery tools

**Backups.** Periodic copies of the entire database (or incremental copies of changes since the last backup). Used when disk storage is damaged. Should be stored separately from the main database.

**Log file (journal).** A sequential record of every database modification. Each log entry contains:

- Transaction ID
- Type of operation (start, insert, update, delete, commit, abort)
- Data item affected
- **Before-image** — the old value (for updates and deletes)
- **After-image** — the new value (for inserts and updates)
- Pointers to the previous and next log entries for the same transaction

The log is critical for recovery. It's often duplicated or triplicated for safety and should be stored on a different disk from the database.

**Checkpoints.** Periodic synchronisation points where:

1. All log records in memory are written to disk.
2. All modified database buffers are written to disk.
3. A checkpoint record is written to the log, listing all currently active transactions.

Checkpoints limit how far back in the log we need to search during recovery. Without checkpoints, we'd potentially have to scan the entire log from the beginning.

### Recovery with checkpoints

After a failure, the recovery manager looks at the most recent checkpoint and categorises transactions:

- **Committed before the checkpoint** → no action needed; their changes are safely on disk.
- **Committed after the checkpoint** → must be **redone**; their log records have the after-images needed.
- **Active (uncommitted) at the time of failure** → must be **undone**; their log records have the before-images needed.


## 18. Recovery Techniques

The approach to recovery depends on *when* updates are written to the database.

### Deferred update

Under **deferred update**, changes are written only to the log until the transaction commits. The actual database is not modified until after commit. This means:

- If a transaction fails before committing, the database hasn't been touched → **no undo needed**.
- Committed transactions may need to be **redone** if their changes haven't reached disk yet.
- Recovery is simpler: scan the log, redo committed transactions, ignore uncommitted ones.

This corresponds to a **no-steal** policy (the buffer manager never writes uncommitted data to disk).

### Immediate update

Under **immediate update**, changes are written to the database as they happen (whenever buffers are flushed), even before the transaction commits. This means:

- Uncommitted changes may be on disk → **undo may be needed**.
- Committed changes may not be fully on disk → **redo may be needed**.
- Recovery requires both undo and redo.

This requires a **write-ahead log (WAL) protocol**: the log record for any change must be written to disk *before* the actual change is written to the database. This ensures that if a crash occurs, the log always has enough information to undo any change that reached the database.

Undos are applied in **reverse order** (most recent change first). Redos are applied in **forward order**. Both operations are **idempotent** — applying them multiple times has the same effect as applying them once, so if a second crash occurs during recovery, the process can simply restart.

### Shadow paging

An alternative to log-based recovery. The system maintains two page tables:

- **Shadow page table** — a frozen copy from the start of the transaction. Never modified.
- **Current page table** — records all updates during the transaction.

If the transaction commits, the current page table becomes the new shadow. If it aborts, the shadow page table is already there as a clean backup — just discard the current page table.

**Advantages:** no log file overhead, instant recovery (just revert to shadow).
**Disadvantages:** data fragmentation, need for garbage collection of abandoned pages, doesn't extend well to concurrent transactions.


---


## 19. Advanced Transaction Models

Traditional flat transactions work well for short, simple operations (bank transfers, ticket bookings). But some real-world tasks are long-running, involve multiple independent subtasks, or require collaboration between different systems. For these, the strict ACID model can be too restrictive.

### Nested transactions

A transaction is decomposed into a tree of **subtransactions**. Each subtransaction can itself contain further subtransactions. For example:

```
Book trip (T1)
├── Book flights (T2)
│   ├── Book London → Paris (T3)
│   └── Book Paris → New York (T4)
├── Book hotel (T5)
└── Book rental car (T6)
```

Key properties:

- Subtransactions can run **concurrently** with each other (T5 and T6 can execute in parallel).
- If a subtransaction fails, the parent can decide how to handle it: retry, ignore (if non-vital), run an alternative, or abort the whole thing.
- **Commits bubble up**: a subtransaction's commit is conditional — its effects only become permanent when the top-level transaction commits.
- The **top-level transaction** still obeys ACID properties from the outside world's perspective.

### Sagas

A simpler alternative where a transaction is a flat sequence of subtransactions T1, T2, ..., Tn, each with a corresponding **compensating transaction** C1, C2, ..., Cn that can undo its effect. If Ti fails:

- Execute Ci-1, Ci-2, ..., C1 to undo all previously completed subtransactions.

Sagas relax isolation (intermediate results are visible) but are practical when subtransactions are independent and compensating actions are straightforward (e.g., cancelling a flight booking).


---


## 20. Transaction Management in Oracle (Brief Overview)

Oracle uses **multiversion read consistency**: each transaction sees a consistent snapshot of the data as it existed when the transaction (or statement) started. Reads never block writes and writes never block reads.

Oracle implements **row-level locking** automatically — users don't need to manage locks manually (though they can override defaults). Lock information is stored directly in data blocks rather than in a separate lock table, so Oracle never needs to escalate locks from row to table level.

Oracle detects deadlock automatically and resolves it by rolling back one of the involved statements.

For recovery, Oracle uses **redo logs** (to redo committed transactions after a crash) and **undo segments** (to undo uncommitted transactions and provide read consistency). It supports checkpoints, point-in-time recovery, standby databases, and Flashback Technology for rewinding tables or the entire database to a previous state.
