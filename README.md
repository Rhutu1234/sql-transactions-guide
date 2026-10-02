# SQL Transactions

*A deep-dive walkthrough of database transactions — covering the ACID properties as precise, individually meaningful guarantees, `BEGIN`/`COMMIT`/`ROLLBACK` mechanics and savepoints, isolation levels and the specific concurrency anomalies (dirty reads, non-repeatable reads, phantom reads) each one prevents or permits, locking and how it underlies isolation, deadlocks and how to reduce them, the write-ahead transaction log as the mechanism behind durability, the real costs of long-running transactions, and how all of this maps onto EF Core's `SaveChanges` and explicit transaction APIs.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [The Problem Transactions Solve](#1-the-problem-transactions-solve)
3. [ACID: Four Distinct Guarantees](#2-acid-four-distinct-guarantees)
4. [BEGIN, COMMIT, ROLLBACK](#3-begin-commit-rollback)
5. [Implicit vs. Explicit Transactions](#4-implicit-vs-explicit-transactions)
6. [Savepoints: Partial Rollback](#5-savepoints-partial-rollback)
7. [Concurrency Anomalies: What Isolation Protects Against](#6-concurrency-anomalies-what-isolation-protects-against)
8. [Isolation Levels](#7-isolation-levels)
9. [Locking: The Mechanism Underneath Isolation](#8-locking-the-mechanism-underneath-isolation)
10. [Snapshot Isolation and Row Versioning](#9-snapshot-isolation-and-row-versioning)
11. [Deadlocks](#10-deadlocks)
12. [The Transaction Log: How Durability Actually Works](#11-the-transaction-log-how-durability-actually-works)
13. [The Cost of Long-Running Transactions](#12-the-cost-of-long-running-transactions)
14. [Transactions in EF Core](#13-transactions-in-ef-core)
15. [Where Single-Database Transactions Stop](#14-where-single-database-transactions-stop)
16. [Common Pitfalls](#15-common-pitfalls)
17. [Quick Reference Table](#quick-reference-table)
18. [Conclusion](#conclusion)

---

## Introduction

A transaction groups several database operations into a single logical unit that either happens completely or not at all — the canonical example is transferring money between two accounts, where debiting one and crediting the other must both succeed or both be undone, never one without the other. But "all or nothing" is only the first of several guarantees transactions provide, and it's arguably the simplest: the harder, more consequential questions are what *other* concurrent transactions are allowed to see while yours is in flight (isolation), how the database guarantees a committed transaction survives a crash (durability), and what happens when two transactions each hold something the other needs (deadlock). This guide goes deep on all of that, and connects it directly to this series' EF Core guide's `SaveChanges` and `BeginTransaction` APIs, this series' High-Volume Transaction Processing guide's own concurrency concerns, and this series' Order Management guide's sagas — which are what you reach for at exactly the point where a single database's transactions stop being enough.

```sql
BEGIN TRANSACTION;
    UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;   -- debit
    UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;   -- credit
COMMIT;   -- BOTH changes become permanent, together — or, on failure/ROLLBACK, NEITHER does
```

---

## 1. The Problem Transactions Solve

### Without transactions, a failure between two related operations leaves the database in a genuinely inconsistent state

```sql
UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;  -- succeeds
-- ⚡ the application crashes, or the connection drops, RIGHT HERE
UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;  -- NEVER RUNS
-- Result: $100 has simply VANISHED from the system — debited, but never credited anywhere
```

This is the concrete, motivating failure transactions exist to prevent — each individual `UPDATE` statement is itself atomic (it either applies fully or not at all), but *two separate statements* have no such guarantee between them; a crash, an error, or a lost connection at exactly the wrong moment leaves the data in a state that violates a real-world invariant (money is neither created nor destroyed) even though every individual statement executed correctly.

### The second problem: concurrent transactions seeing each other's half-finished work

```plaintext
Even WITH all-or-nothing guaranteed, another concurrent query running
  BETWEEN the debit and the credit could observe the intermediate
  state — total money in the system temporarily appearing $100 short —
  which is a genuinely different problem than crash recovery, and
  precisely what ISOLATION (Sections 6-8) exists to control.
```

---

## 2. ACID: Four Distinct Guarantees

### The acronym names four separate properties, each solving a different problem

```plaintext
Atomicity:   all operations in the transaction succeed, or NONE take effect.
Consistency:  a transaction moves the database from one VALID state to
               another — every constraint, foreign key, and rule holds
               both before and after.
Isolation:    concurrent transactions don't interfere with each other in
               ways that produce incorrect results (the DEGREE of this is
               configurable — Section 7).
Durability:   once COMMITTED, the transaction's changes survive a crash,
               power loss, or restart.
```

It's worth treating these as four genuinely distinct guarantees rather than one blurred idea — a system can provide some without others (many NoSQL stores offer atomicity only within a single document, or relax isolation deliberately for throughput), and knowing precisely which of the four a given piece of infrastructure actually promises is what tells you what your own application code still has to handle.

### Consistency is the odd one out: partly the database's job, partly yours

```plaintext
The database enforces the constraints you DECLARE — primary keys,
  foreign keys, CHECK constraints, NOT NULL, UNIQUE (per this series'
  SQL Indexes guide's Section 10) — but it has NO knowledge of your
  BUSINESS invariants unless you encode them somehow ("total debits
  must equal total credits"). Atomicity, isolation, and durability are
  properties the database engine provides mechanically; consistency
  is a property that holds only to the degree your constraints and your
  transaction's logic actually express the rules that matter.
```

This is a genuinely useful distinction — three of the four letters describe machinery the engine implements for you, while the "C" is a shared responsibility: the transaction mechanism guarantees it won't *break* a consistent state through partial application, but whether the committed end state is *actually correct* depends on your code and your declared constraints.

---

## 3. BEGIN, COMMIT, ROLLBACK

### The three core statements

```sql
BEGIN TRANSACTION;

    INSERT INTO Orders (CustomerId, Total) VALUES (42, 99.99);
    UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductId = 7;

COMMIT TRANSACTION;   -- make everything above permanent, atomically
-- or:
ROLLBACK TRANSACTION; -- undo EVERYTHING since BEGIN, as if it never happened
```

`BEGIN` opens the transaction; every statement after it executes *within* it, its effects visible to itself but (per Section 8's isolation rules) not necessarily to other sessions; `COMMIT` makes the whole group permanent and visible; `ROLLBACK` discards every change made since the `BEGIN`, restoring the exact prior state.

### Error handling: rolling back on failure, in T-SQL

```sql
BEGIN TRY
    BEGIN TRANSACTION;
        UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;
        UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;  -- only roll back if a transaction is actually still open
    THROW;                                       -- re-raise, per this series' Exception Handling guide's
                                                  --  preserve-the-original-error discipline
END CATCH;
```

This mirrors, almost exactly, this series' Exception Handling guide's Section 1 `try`/`catch`/`finally` discipline, expressed in T-SQL — the `CATCH` block rolls back on any error, and `THROW` (bare, with no arguments) re-raises the original error with its context intact, the SQL-side equivalent of C#'s bare `throw;` preserving the original stack trace.

### `SET XACT_ABORT ON`: making certain errors abort the whole transaction automatically

```sql
SET XACT_ABORT ON;  -- any run-time error automatically rolls back the ENTIRE transaction
```

Worth knowing this setting exists because, by default in SQL Server, some errors abort only the *statement* that failed while leaving the transaction open and the earlier statements' changes still pending — which can leave a transaction in a half-applied state if error handling isn't written carefully. `XACT_ABORT ON` converts that into the safer, all-or-nothing behavior most stored procedures actually intend.

---

## 4. Implicit vs. Explicit Transactions

### Every single statement is already its own transaction, even without an explicit `BEGIN`

```sql
UPDATE Products SET Price = 29.99 WHERE Id = 42;
-- With no explicit BEGIN, this statement runs in an IMPLICIT, "autocommit" transaction —
-- it either fully applies and commits, or fully fails, immediately, on its own.
```

This is the default mode in most database engines (called autocommit) — each standalone statement is atomic and durable on its own; explicit `BEGIN`/`COMMIT` is what *extends* that guarantee across multiple statements. It's worth knowing precisely, since it explains why a single `UPDATE` affecting thousands of rows is atomic without any special syntax, while two separate `UPDATE` statements need an explicit transaction to be atomic *together*.

### Why this maps directly onto EF Core's own `SaveChanges` behavior

```plaintext
Per this series' EF Core guide's Section 9: a single SaveChanges() call
  wraps ALL of its accumulated INSERT/UPDATE/DELETE statements in ONE
  implicit transaction — the same principle as autocommit above,
  applied to a whole batch of tracked changes at once. Multiple
  SEPARATE SaveChanges calls are each their own transaction, exactly
  like separate autocommit statements — which is precisely why that
  guide's Section 12 introduces an explicit BeginTransaction for
  grouping several of them together.
```

---

## 5. Savepoints: Partial Rollback

### Rolling back part of a transaction without abandoning all of it

```sql
BEGIN TRANSACTION;
    INSERT INTO Orders (CustomerId, Total) VALUES (42, 99.99);

    SAVE TRANSACTION AfterOrderInsert;          -- a named checkpoint INSIDE the transaction

    UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductId = 7;
    -- suppose this turns out to be invalid for some reason...
    ROLLBACK TRANSACTION AfterOrderInsert;       -- undo ONLY back to the savepoint —
                                                   --  the Orders INSERT is still pending, intact
COMMIT TRANSACTION;                               -- commits everything up to (and NOT including) the rolled-back part
```

A savepoint lets a transaction undo just a portion of its work while keeping the rest — useful for a genuinely multi-step operation where one sub-step failing shouldn't necessarily doom the whole unit (a bulk import where individual bad rows should be skipped, say). Worth knowing it's a comparatively specialized tool: the plain full `ROLLBACK` is what most transactional code uses.

---

## 6. Concurrency Anomalies: What Isolation Protects Against

### Dirty read: seeing another transaction's UNCOMMITTED changes

```plaintext
Transaction A: UPDATE Accounts SET Balance = 500 WHERE Id = 1;   -- not yet committed
Transaction B: SELECT Balance FROM Accounts WHERE Id = 1;         -- reads 500 (!)
Transaction A: ROLLBACK;                                            -- the 500 NEVER officially existed
```

Transaction B acted on a value that was never actually committed and then vanished — it read data that, from the database's durable point of view, was never real. This is the most severe anomaly, since B may have made decisions or written further data based on a value that ceased to exist.

### Non-repeatable read: the SAME row, read twice in one transaction, returns DIFFERENT values

```plaintext
Transaction A: SELECT Balance FROM Accounts WHERE Id = 1;   -- reads 100
Transaction B: UPDATE Accounts SET Balance = 200 WHERE Id = 1; COMMIT;
Transaction A: SELECT Balance FROM Accounts WHERE Id = 1;   -- reads 200 — the SAME query, a DIFFERENT answer
```

Nothing was uncommitted here — B's change was legitimately committed — but from A's perspective, a row it already read changed underneath it *within its own transaction*, which can break any logic that assumed a value it read earlier would still hold when it acted on it later.

### Phantom read: the SAME query, run twice, returns a different SET of ROWS

```plaintext
Transaction A: SELECT COUNT(*) FROM Orders WHERE CustomerId = 42;   -- returns 5
Transaction B: INSERT INTO Orders (CustomerId, ...) VALUES (42, ...); COMMIT;
Transaction A: SELECT COUNT(*) FROM Orders WHERE CustomerId = 42;   -- returns 6 — a new "phantom" row appeared
```

This differs subtly from a non-repeatable read: no individual row A already read changed — instead, a *new row* now qualifies for A's query predicate. The distinction matters because preventing it requires protecting not just the rows A has already touched, but the *range* of rows that could satisfy A's query, which is a genuinely harder problem for the database to solve (Section 8's key-range locking exists for exactly this).

---

## 7. Isolation Levels

### The SQL standard's four levels, each preventing progressively more anomalies

```plaintext
                       Dirty Read   Non-Repeatable Read   Phantom Read
READ UNCOMMITTED         possible       possible             possible
READ COMMITTED           prevented      possible             possible
REPEATABLE READ          prevented      prevented            possible
SERIALIZABLE             prevented      prevented            prevented
```

Each step up buys stronger guarantees at the cost of more locking (or versioning), and therefore more potential for contention between concurrent transactions — which is the fundamental, unavoidable trade-off this entire section is about: correctness versus concurrency.

### Setting the isolation level

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- SQL Server's DEFAULT
BEGIN TRANSACTION;
    -- ...
COMMIT;
```

**READ COMMITTED** — SQL Server's default, and the PostgreSQL default too — is the practical sweet spot for most application code: it eliminates dirty reads (you only ever see committed data) while still permitting non-repeatable and phantom reads, which most business logic can tolerate or handle explicitly.

### READ UNCOMMITTED and `NOLOCK`: the dangerous convenience

```sql
SELECT * FROM Orders WITH (NOLOCK);   -- equivalent to READ UNCOMMITTED for this one table reference
```

This is worth a specific warning, because `WITH (NOLOCK)` is spread widely through real SQL Server codebases as a "make the query faster and stop blocking" trick — it works by permitting dirty reads (Section 6), which means a query can return uncommitted data that later rolls back, or (in a genuinely surprising side effect of how page reads interact with concurrent page splits) return the same row twice or skip rows entirely. It's occasionally defensible for rough, approximate reporting where exactness genuinely doesn't matter, and a real correctness bug anywhere else.

### SERIALIZABLE: the strongest guarantee, and its real cost

```plaintext
SERIALIZABLE guarantees the outcome is as if transactions had executed
  one at a time, in SOME serial order, even though they actually ran
  concurrently — the strongest, most intuitive correctness model. The
  cost is the most aggressive locking (including key-range locks that
  block INSERTs into a protected range, Section 8), which measurably
  reduces concurrency and increases both blocking and deadlock risk
  (Section 10) under real contention.
```

Worth reaching for deliberately, for specific operations where phantom-read protection genuinely matters (enforcing a uniqueness or capacity rule that can't be expressed as a simple constraint), rather than as a blanket default — which is exactly the same "apply the strongest tool narrowly, not everywhere" judgment this series' High-Volume Transaction Processing guide applies to strong consistency generally.

---

## 8. Locking: The Mechanism Underneath Isolation

### Shared and exclusive locks: the two fundamental modes

```plaintext
Shared (S) lock:     acquired when READING a row/page — MANY transactions can
                      hold a shared lock on the same resource simultaneously.
Exclusive (X) lock:  acquired when WRITING — only ONE transaction can hold it,
                      and it's INCOMPATIBLE with any other lock (shared or exclusive)
                      on the same resource.
```

This is the direct, mechanical basis for isolation: a writer's exclusive lock blocks other transactions from reading or writing the row until the writer commits or rolls back, which is precisely what prevents a dirty read (nobody can read a row mid-write), while multiple readers coexist happily via compatible shared locks.

### How isolation levels map onto lock duration

```plaintext
READ COMMITTED:  shared locks are held only WHILE the row is being read, then
                  released immediately — allowing a non-repeatable read (the row
                  can change once the reader has moved on).
REPEATABLE READ: shared locks are held until the END OF THE TRANSACTION — the row
                  can't change under the reader, preventing non-repeatable reads.
SERIALIZABLE:    additionally takes KEY-RANGE locks, protecting the RANGE of rows a
                  query's predicate covers, preventing phantom inserts into it.
```

This is the concrete, mechanical answer to "what does a higher isolation level actually *do*" — it holds locks longer, and locks broader ranges, which is exactly why each step up in Section 7's table costs more concurrency: locks held for the whole transaction block other transactions for that whole duration.

### Lock granularity and escalation

```plaintext
Locks can be held at row, page, or table granularity — SQL Server starts
  fine-grained (row locks) and may ESCALATE to a coarser lock (typically a
  table lock) once a single transaction holds a large number of fine-grained
  locks, trading concurrency for the memory overhead of tracking thousands
  of individual locks.
```

Worth knowing this exists because lock escalation is a real, sometimes surprising source of sudden blocking: a large batch update that quietly escalates to a table lock can freeze every other query touching that table for the duration, even though the same logic on a smaller batch caused no visible contention at all.

---

## 9. Snapshot Isolation and Row Versioning

### A fundamentally different approach: readers don't take shared locks at all

```sql
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON;   -- makes plain READ COMMITTED use row versioning
```

Instead of blocking readers with locks, the engine keeps **previous versions of modified rows** (in `tempdb`, for SQL Server), and a reader transparently sees the version of each row as it existed when its transaction (or statement) began — no shared locks, and therefore readers never block writers and writers never block readers. This is the same underlying idea as MVCC (multi-version concurrency control) that PostgreSQL uses by default for all its isolation levels.

### The trade-offs of row versioning

```plaintext
Wins:   readers and writers no longer block each other — often a dramatic
        reduction in blocking for read-heavy workloads.
Costs:  version storage overhead (in tempdb for SQL Server), and under full
        SNAPSHOT isolation, WRITE conflicts are detected at commit time —
        two transactions modifying the same row cause one to fail with an
        update-conflict error, rather than one waiting for the other.
```

That last point connects directly to this series' EF Core guide's Section 10 optimistic concurrency discussion — snapshot isolation's commit-time conflict detection is the same *philosophy* (assume conflicts are rare, detect them rather than prevent them with locks), just implemented by the database engine itself rather than via an explicit `RowVersion` column your application manages.

---

## 10. Deadlocks

### The classic circular wait, now across database locks instead of application locks

```plaintext
Transaction A: UPDATE Accounts SET ... WHERE Id = 1;   -- holds X lock on row 1
Transaction B: UPDATE Accounts SET ... WHERE Id = 2;   -- holds X lock on row 2
Transaction A: UPDATE Accounts SET ... WHERE Id = 2;   -- WAITS for B's lock on row 2
Transaction B: UPDATE Accounts SET ... WHERE Id = 1;   -- WAITS for A's lock on row 1  →  DEADLOCK
```

This is the exact same circular-wait structure this series' Threading guide's Section 5 describes for in-process `lock`s, occurring instead across database row locks — neither transaction can ever proceed, since each is waiting on a resource the other holds and will never release while waiting.

### The database detects and breaks it: one transaction becomes the "victim"

```plaintext
SQL Server runs a background deadlock monitor that periodically detects
  cycles in the lock-wait graph and resolves them by choosing one
  transaction as the DEADLOCK VICTIM — automatically rolling it back and
  raising error 1205 in that session, freeing its locks so the OTHER
  transaction can proceed.
```

This is worth internalizing as a fundamentally different failure mode from the in-process deadlock in this series' Threading guide, where a deadlocked `lock` simply hangs forever — the database *resolves* deadlocks automatically by sacrificing one participant, which means application code must be prepared for any transaction to be killed with a deadlock error at any time and, typically, simply **retry** it.

### Reducing deadlocks: the same discipline as the Threading guide's lock ordering

```plaintext
1. Access resources in a CONSISTENT ORDER across every transaction — if every
   transaction updates account 1 before account 2, the circular wait above
   becomes structurally impossible (this series' Threading guide's Section 5).
2. Keep transactions SHORT (Section 12) — less time holding locks means a
   smaller window for a conflicting transaction to collide with you.
3. Use appropriate INDEXES (this series' SQL Indexes guide) — a query that must
   scan a whole table locks far more rows than one that seeks precisely,
   dramatically widening the collision surface.
4. Consider row-versioning isolation (Section 9) so readers don't participate
   in lock cycles at all.
```

### Retry logic for deadlock victims, in C#

```csharp
for (int attempt = 0; attempt < 3; attempt++)
{
    try
    {
        await ExecuteTransactionalWorkAsync();
        break;
    }
    catch (SqlException ex) when (ex.Number == 1205 && attempt < 2) // 1205 = deadlock victim
    {
        await Task.Delay(TimeSpan.FromMilliseconds(50 * (attempt + 1))); // brief backoff, per this series'
    }                                                                        //  Resilience/retry discussions
}
```

Since a deadlock victim's transaction was fully rolled back and is safe to re-run, a bounded retry (with a short backoff) is the standard, correct response — worth noting the retry must re-execute the *entire* transaction from `BEGIN`, not just the failed statement, since the whole unit was undone.

---

## 11. The Transaction Log: How Durability Actually Works

### Write-ahead logging: changes are recorded in a sequential log BEFORE being applied to the data files

```plaintext
When a transaction modifies data, the engine FIRST writes a record
  describing the change to the TRANSACTION LOG (a sequential, append-only
  file) and forces it to durable storage — only THEN does the actual data
  page (in memory) get modified, with the on-disk data file updated
  LATER, lazily, during a background checkpoint.
```

This is the mechanism underneath the "D" in ACID, and it's a beautiful piece of design worth understanding precisely: data files are updated in a *random-access* pattern (slow to force to disk on every commit), while the log is written *sequentially* (fast); `COMMIT` only needs to wait for the small, sequential log write to be safely on disk, not for every modified data page to be flushed — which is what makes durable commits fast enough to be practical.

### Crash recovery: replaying (redo) committed work and undoing uncommitted work from the log

```plaintext
After a crash, the engine reads the log on startup:
  REDO:  reapply changes from transactions that COMMITTED (per the log) but
          whose data-page changes may not have reached the data files yet.
  UNDO:  reverse changes from transactions that were still IN FLIGHT (never
          committed) at the moment of the crash, using the log's records of
          what each change originally was.
```

This is exactly how atomicity and durability are jointly delivered after a failure — committed work is guaranteed to survive (redo), and uncommitted work is guaranteed to vanish (undo), both reconstructed purely from the log, without depending on the state the data files happened to be in at the moment of the crash.

### The log is also what powers `ROLLBACK`

```plaintext
The same log records that let crash recovery UNDO uncommitted work are what
  an explicit ROLLBACK uses to reverse a transaction's changes — this is why
  a very large transaction takes a long time to ROLL BACK (often comparable to
  the time it took to do the work), not an instant "undo" — the engine
  genuinely has to walk the log backwards, reversing each change.
```

---

## 12. The Cost of Long-Running Transactions

### Every second a transaction stays open, it holds locks (or version-store space) that other work may need

```plaintext
A long transaction:
- Holds its locks for its entire duration, blocking other transactions
  (and increasing deadlock probability, Section 10).
- Under row versioning (Section 9), prevents version-store cleanup for
  every version newer than its own start — bloating tempdb.
- Prevents the transaction log from being truncated past its own start
  point, causing the log file to grow without bound.
- Takes proportionally longer to ROLL BACK if it ultimately fails (Section 11).
```

This is a genuinely underappreciated cost — "the transaction is just waiting" isn't free; an idle-but-open transaction (a classic cause: application code that starts a transaction, then makes a slow external call, or waits on user input, before committing) still holds every lock it has acquired and pins the log and version store the whole time.

### The cardinal rule: never do slow, non-database work inside an open transaction

```csharp
// ❌ Holds locks (and pins the log) for the entire duration of an external HTTP call
using var transaction = await context.Database.BeginTransactionAsync();
context.Orders.Add(order);
await context.SaveChangesAsync();
await _paymentGateway.ChargeAsync(order);   // a slow, network-bound call, INSIDE the transaction
await transaction.CommitAsync();
```

This is the concrete, common way real applications accidentally create long-running transactions — mixing slow external I/O into the transactional scope. The correct restructuring keeps the transaction confined to genuinely database-only work, and handles the external call outside it (with compensation if a later step fails — which is exactly what Section 14's saga discussion covers).

---

## 13. Transactions in EF Core

### `SaveChanges`: one implicit transaction per call, per this series' EF Core guide's Section 9

```csharp
context.Products.Add(newProduct);
existingProduct.Price = 19.99m;
await context.SaveChangesAsync();   // all changes committed in ONE implicit transaction — atomic
```

Everything in Sections 2-4 applies directly: EF Core wraps the batch of generated `INSERT`/`UPDATE`/`DELETE` statements in a transaction at the database's default isolation level (`READ COMMITTED` for SQL Server), committing on success and rolling back if any statement fails.

### Explicit transactions, for spanning multiple `SaveChanges` calls

```csharp
await using var transaction = await context.Database.BeginTransactionAsync(IsolationLevel.Serializable);
try
{
    context.Orders.Add(newOrder);
    await context.SaveChangesAsync();          // needs the database-generated Order.Id...

    context.InventoryLog.Add(new InventoryLogEntry { OrderId = newOrder.Id });
    await context.SaveChangesAsync();          // ...to use here — both must commit together

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;                                       // bare throw — preserving the original stack trace
}
```

This is this series' EF Core guide's Section 12 pattern, now with the isolation-level parameter this guide's Section 7 explains — `BeginTransactionAsync` accepts an `IsolationLevel`, letting you request `Serializable` (or `Snapshot`, etc.) for a specific unit of work while leaving the application's default at `ReadCommitted` everywhere else.

### `TransactionScope`: ambient transactions, and why it's used more cautiously today

```csharp
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled); // the async flow option is REQUIRED
context.Orders.Add(order);
await context.SaveChangesAsync();
scope.Complete();   // marks the scope as successful — omitting this causes a ROLLBACK on dispose
```

`TransactionScope` creates an *ambient* transaction that any enlisted connection or resource joins automatically, without passing a transaction object around explicitly — convenient, but with real sharp edges: forgetting `TransactionScopeAsyncFlowOption.Enabled` in `async` code causes genuinely confusing failures (per this series' async/await guide's discussion of how continuations resume on different threads), and a scope that ends up enlisting *more than one* database connection can silently escalate to a distributed transaction (Section 14). Explicit `BeginTransactionAsync` is generally the clearer, safer default for new code.

---

## 14. Where Single-Database Transactions Stop

### A transaction can only span what one database engine controls

```plaintext
A database transaction guarantees atomicity across changes to the SAME
  database — but real business operations routinely span MORE than that: a
  payment gateway's API, a message broker, another service's own database. No
  database transaction can roll back an HTTP call to a payment provider that
  already charged a card.
```

This is precisely the boundary where this series' Order Management guide's Section 7 saga pattern takes over — its whole reason for existing is that "place an order" spans inventory, payment, and fulfillment services with separate databases, where no single ACID transaction can wrap them all, so coordination is instead achieved via a sequence of *local* transactions with explicit compensating actions for failure, exactly the "each step is its own local ACID transaction" structure this guide's Sections 2-4 describe, composed into a larger, eventually-consistent workflow.

### Distributed transactions (two-phase commit): technically possible, generally avoided

```plaintext
Two-phase commit (2PC, via the Distributed Transaction Coordinator on SQL
  Server / MSDTC) CAN coordinate a single atomic commit across multiple
  databases or resource managers — but couples every participant's
  availability and latency together, holds locks across network round trips,
  and is unsupported by many modern cloud databases and message brokers.
```

This echoes this series' High-Volume Transaction Processing guide's Section 8 argument precisely: at scale, sagas with local transactions plus compensation are generally preferred to 2PC, accepting eventual consistency in exchange for independence and throughput — which is why understanding single-database transactions thoroughly (this guide) is the foundation, and knowing where they end is what tells you when to reach for the saga pattern instead.

---

## 15. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Two related statements without an explicit transaction | Each runs in its own autocommit transaction — a failure between them leaves data inconsistent | Wrap related statements in an explicit transaction so they succeed or fail together (Section 4) |
| Using `WITH (NOLOCK)` as a blanket performance fix | Permits dirty reads, and can even return duplicated or missing rows | Use row-versioning isolation (`READ_COMMITTED_SNAPSHOT`) to avoid blocking without sacrificing correctness (Sections 7, 9) |
| Doing slow external work (HTTP calls, user waits) inside an open transaction | Holds locks and pins the log for the entire duration, blocking others and inflating deadlock risk | Keep transactions confined to database-only work; handle external calls outside with compensation on failure (Section 12) |
| Acquiring multiple resources in inconsistent order across transactions | The classic circular wait — a deadlock | Access resources in a consistent, agreed-upon order everywhere (Section 10) |
| Not retrying deadlock victims (error 1205) | A deadlock is a normal, expected event under contention; an unhandled one surfaces as a spurious user-facing failure | Wrap transactional work in bounded retry logic that re-runs the entire transaction (Section 10) |
| Defaulting to `SERIALIZABLE` "to be safe" | The strongest isolation carries the highest blocking and deadlock cost, throttling concurrency for guarantees most operations don't need | Use `READ COMMITTED` (or snapshot) by default; reserve `SERIALIZABLE` for specific operations that genuinely need phantom protection (Section 7) |
| Forgetting `TransactionScopeAsyncFlowOption.Enabled` in `async` code | The ambient transaction doesn't flow across `await`s, producing confusing failures | Prefer explicit `BeginTransactionAsync`; if using `TransactionScope`, always enable async flow (Section 13) |
| Assuming a database transaction can roll back an external side effect | Rollback only reverses changes within the database's own control — a charged card or a sent email stays done | Model cross-system workflows as sagas with compensating actions, not a single transaction (Section 14) |
| Ignoring that a huge transaction is slow to roll back | Rollback replays the log in reverse and can take as long as the original work — blocking resources meanwhile | Batch very large operations into smaller, independently-committed chunks where atomicity of the whole isn't required (Section 11-12) |

---

## Quick Reference Table

| Concept | Syntax / Mechanism | Purpose |
|---|---|---|
| Open / commit / undo | `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK` | Group statements into one all-or-nothing unit |
| Partial rollback | `SAVE TRANSACTION name` / `ROLLBACK TRANSACTION name` | Undo part of a transaction, keeping the rest |
| Auto-abort on error | `SET XACT_ABORT ON` | Ensures run-time errors abort the whole transaction, not just one statement |
| Isolation level | `SET TRANSACTION ISOLATION LEVEL ...` | Controls which concurrency anomalies are permitted |
| Row versioning | `READ_COMMITTED_SNAPSHOT` / `SNAPSHOT` | Readers and writers stop blocking each other |
| Deadlock victim error | SQL Server error `1205` | Signals a transaction was chosen to be rolled back; retry it |
| Durability mechanism | Write-ahead transaction log | Makes commits fast and crash-safe via sequential log writes |
| EF Core implicit transaction | `SaveChangesAsync()` | One atomic transaction per save call |
| EF Core explicit transaction | `BeginTransactionAsync(IsolationLevel...)` | Spans multiple `SaveChanges` calls, with a chosen isolation level |
| Beyond one database | Saga pattern (this series' Order Management guide) | Coordinates local transactions with compensation where one transaction can't span everything |

| Isolation Level | Prevents |
|---|---|
| READ UNCOMMITTED | Nothing (dirty reads permitted) |
| READ COMMITTED | Dirty reads |
| REPEATABLE READ | Dirty + non-repeatable reads |
| SERIALIZABLE | Dirty + non-repeatable + phantom reads |

---

## Conclusion

A transaction's "all or nothing" promise is really four separate guarantees working together — atomicity and durability delivered mechanically by the write-ahead log, isolation delivered by locking (or row versioning) at a level you choose, and consistency delivered jointly by the constraints you declare and the logic you write — and the practical craft is understanding the trade-offs among them. Higher isolation buys stronger correctness guarantees with more locking, which means more blocking and more deadlock risk; the standard, well-worn answer is `READ COMMITTED` (ideally with row versioning) by default, with `SERIALIZABLE` applied narrowly to the specific operations that genuinely need it, the same "strong guarantees where they matter, relaxed everywhere else" judgment this series' High-Volume Transaction Processing guide applies to consistency generally.

The two failure modes this guide spends the most effort on — deadlocks and long-running transactions — are really the same underlying issue seen from two angles: a transaction holds resources for as long as it's open, so the discipline that prevents both is identical, and simple to state: keep transactions short, confine them to database-only work, touch resources in a consistent order, and be prepared to retry. And knowing precisely where a single database's transactions *stop* — at the boundary of the database engine itself — is what tells you when to reach for this series' Order Management guide's saga pattern instead: local ACID transactions are the building block, and sagas are how you compose them into workflows that a single transaction can no longer contain.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the transaction-left-open-across-an-HTTP-call incident that brought the whole order table to a standstill and made "keep transactions short" feel like a hard rule rather than a style preference.*
