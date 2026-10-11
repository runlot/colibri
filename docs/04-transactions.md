# 04 — Transactions: the write buffer, the writer token, snapshots, sequences and the GC watermark

> Kind: **design**. Status: **pre-implementation** (2026-10-11). Fills the gap left by
> docs/00-overview.md §3.1 (D1, D8, D9), §4.2 steps 1–2 and §6 for phase P0. It
> **corrects** 00 §4.2 step 1: a statement's writes do not go to the memtable before
> commit; they go to a per-transaction write buffer, and the memtable holds only committed
> versions (the orchestrator edits 00 to match). It also **corrects** D9's sentence "with
> one writer, SERIALIZABLE costs nothing beyond REPEATABLE READ" (§5.4) and 00 §4.4's
> "unique indexes are enforced by the writer at commit" (§6.3). Depends on 01 (key
> encoding, tombstones, hidden rowid), 02 (memtable flush, segment `max_commit_ts`,
> compaction's use of the watermark), 03 (append, ack, epoch fencing, tail replay,
> degraded mode, record kinds, and the `TxnBatch`/`SeqReserve` layouts of 03 §2.2) and 05
> (catalog rows, `schema_version`, `CatalogSnapshot`).
> 02 depends on this document for the GC watermark (§9) and the memtable interface
> (§13); 05 depends on it for transactional DDL (§10); 06 places these types, the
> memtable included, in `colibri-txn`.

## 1. The transaction state machine

### 1.1 Shapes a transaction takes

A session is bound to one database (00 §6) and has at most one top-level transaction.

| Start | How it ends | PostgreSQL equivalent |
|---|---|---|
| **Implicit**: a statement arrives with no block open | At the end of the statement (simple query) or at `Sync` (extended query); commit on success, rollback on error | Same |
| **Explicit**: `BEGIN` / `START TRANSACTION` | `COMMIT` / `END`, `ROLLBACK` / `ABORT`, or the session ending (rollback) | Same |

Autocommit is the absence of `BEGIN`, as in PostgreSQL. The wire maps onto this:

- **Simple query** (`Query` message): every statement in the string runs in one implicit
  transaction unless the string itself contains `BEGIN`/`COMMIT`. PostgreSQL's rule is
  kept exactly: `"INSERT ...; INSERT ..."` is atomic, and an error in the second rolls
  back the first.
- **Extended query**: an implicit transaction spans every `Execute` up to the next `Sync`,
  which commits it; an error discards messages up to `Sync`. Pipelining clients depend on
  this boundary, and it is identical. `ReadyForQuery` carries `I`, `T` or `E` (§1.3).

### 1.2 Savepoints and the aborted state

Inside an explicit block, `SAVEPOINT s` pushes a level, `RELEASE s` pops it (and every
level above it) keeping the writes, and `ROLLBACK TO s` undoes every write since `s` and
leaves `s` in place, as PostgreSQL does. Savepoint names may repeat; the innermost wins.
§7 specifies the undo mechanism. Savepoints are P0 (00 §12): the write buffer's undo log
(§3, §7) is shaped for them from the first line.

Any error inside an explicit block puts the transaction in the **aborted** state. Every
statement other than `ROLLBACK`, `ROLLBACK TO`, `COMMIT` (which then performs a rollback
and reports the `ROLLBACK` command tag, as PostgreSQL does) and `ABORT` fails with
SQLSTATE `25P02` *current transaction is aborted, commands ignored until end of
transaction block*. `ROLLBACK TO s` returns the block from aborted to in-progress.

### 1.3 The states as a Rust enum (sketch)

```rust
pub enum TxnState {
    Idle,                        // 'I'
    Implicit(Box<Transaction>),  // one simple-query string, or extended query up to Sync
    InBlock(Box<Transaction>),   // BEGIN ...; 'T'
    Aborted(Box<Transaction>),   // 'E'; kept for ROLLBACK TO, token stays held (§7.3)
}
```

| From | Event | To |
|---|---|---|
| `Idle` | statement / `BEGIN` | `Implicit` / `InBlock` |
| `Implicit` | end of string / `Sync`, no error | commit → `Idle` |
| `Implicit` | error | rollback → `Idle` (extended: skip to `Sync`) |
| `Implicit` | `BEGIN` inside a simple-query string | `InBlock` (PG's behaviour: the block absorbs the implicit transaction) |
| `InBlock` | error | `Aborted` |
| `InBlock` | `COMMIT` | commit (§6) → `Idle`; if the commit itself fails, rollback → `Idle` |
| `InBlock`, `Aborted` | `ROLLBACK` | rollback (§7) → `Idle` |
| `Aborted` | `ROLLBACK TO s` | `InBlock` |
| `Aborted` | any other statement | `25P02`, stays `Aborted` |
| any | session ends | rollback → (gone) |

## 2. Snapshots

### 2.1 What a snapshot is

Each open database holds one atomic `last_committed_ts: AtomicU64`. It is advanced only by
the commit path (§6) with `Release` ordering after the transaction's versions are in the
memtable. A snapshot is

```
SnapshotTs = last_committed_ts.load(Acquire)
```

and a read at `snapshot_ts` sees, per key, the newest version with
`commit_ts <= snapshot_ts` (01 owns how the `!commit_ts` suffix makes "newest" the first
in key order). No transaction id, no in-progress list, no visibility map: because the
memtable holds only committed versions, visibility is a single comparison (D1).

### 2.2 When a snapshot is taken, per isolation level

| Level | Snapshot | Rule for writing statements |
|---|---|---|
| READ COMMITTED (default) | A fresh one at the start of every statement | A statement classified as writing (§5.1) acquires the token **before** taking its snapshot |
| REPEATABLE READ | Fixed at the transaction's first statement that needs one (a `SELECT` counts; `BEGIN`, `SET`, `SET TRANSACTION`, `SAVEPOINT` do not) | Token acquired at the first writing statement; the snapshot does not move; conflicting writes raise `40001` (§5.3) |
| SERIALIZABLE | As REPEATABLE READ | As REPEATABLE READ, plus the read-set check (§5.4) |
| READ UNCOMMITTED | Treated as READ COMMITTED, as in PostgreSQL | |

`SET TRANSACTION ISOLATION LEVEL` after the snapshot has been taken fails with `25001`
exactly as PostgreSQL does.

### 2.3 Read-only transactions never take the token

A transaction with no writing statement never touches the token and reads at its snapshot
on any worker (00 §4.3, §6), so one long writer never slows a reader.

### 2.4 What a snapshot pins

A live snapshot is registered in the database's `SnapshotRegistry` (§9, §13), a multiset
of `SnapshotTs` values. The GC watermark (§9) is the minimum of that multiset and the PITR
horizon; compaction (02) never drops a version a registered snapshot can see.

Registration loads `last_committed_ts` under the registry's lock, and the watermark is
computed under the same lock; otherwise a reader could load 10, compaction could compute a
watermark of 11 and drop the version at 10, and only then would the reader register. The
critical section is one map increment; its cost is not measured. Rejected for now: a
sharded register-then-revalidate registry (more code for an unmeasured gain).

| Level | Registered | Unregistered |
|---|---|---|
| READ COMMITTED | At statement start | At statement end (portal close for extended query) |
| REPEATABLE READ, SERIALIZABLE | At the first statement | At transaction end, after the commit or rollback is complete |
| Cursor `WITH HOLD` | Not after commit: the result is materialised (§11.1) | |

## 3. The write buffer

### 3.1 Shape

A transaction's writes go to its `WriteBuffer`, an ordered map from the **encoded user key
without timestamp** (`prefix || user key`, 01) to a pending version:

```
WriteBuffer (in memory)                   undo log
┌──────────────────────────────┬────────┐  ┌────────────────────────────────┐
│ key (no ts)                  │ entry  │  │ (key, previous Option<Entry>)  │
├──────────────────────────────┼────────┤  │ ...                            │
│ 16384|0|pk=7                 │ V(row) │  └────────────────────────────────┘
│ 16384|16390|email=a@x|pk=7   │ V()    │  savepoint marks: [wseq_at_s1, ...]
│ 16384|0|pk=9                 │ Tomb   │
└──────────────────────────────┴────────┘
entry = { version: Value(bytes) | Tombstone, wseq: u64, depth: u16, fresh: bool,
          stamp_offsets: Vec<u32> }
```

- `wseq` is a per-transaction write sequence number, incremented per write. Savepoint
  marks and spilled runs (§3.5) are expressed in `wseq`.
- `depth` is the savepoint depth at which the entry was last written. A second write to
  the same key at the same depth does not add an undo record (the first one already
  restores the state before the level began).
- `fresh` records that, when the key was first written in this transaction, it had no
  live committed version (absent, or newest version a tombstone). A key that is `fresh`
  and ends as a `Tombstone` is dropped from the commit batch: an `INSERT` followed by a
  `DELETE` in one transaction writes nothing.
- `stamp_offsets` (usually empty) lists byte offsets in the value where the commit path
  writes the 8-byte big-endian `commit_ts` at commit (§6.1 step 3), for values that must
  record their own commit time but are written before it is known. The one P0 user is
  `TRUNCATE`'s `truncated_at` (05 §7.4).

Secondary index entries are written into the same buffer as ordinary keys (01's index key
layout); an `UPDATE` that changes an indexed column writes a tombstone for the old index
key and a value for the new one. Catalog rows for DDL are ordinary keys too (§10).

### 3.2 Read-your-writes

Every read inside the transaction is a merge of two sources: the buffer, and the snapshot.
For a point read the buffer is consulted first; a hit (value or tombstone) is the answer.
For a range scan, the executor's KV iterator merges a buffer range iterator over the
snapshot iterator; on equal user keys the buffer wins, and a buffer tombstone hides the
snapshot's row. The snapshot used is the statement's (READ COMMITTED) or the
transaction's (REPEATABLE READ, SERIALIZABLE).

`UPDATE` of a row the same transaction inserted is an ordinary overwrite of the buffer
entry: the scan finds the row in the buffer, the new version replaces the entry, and the
undo log records the previous entry if the depth changed. `fresh` is kept, so the row is
still an insert at commit. An `UPDATE` that changes the primary key writes a tombstone at
the old key and a value at the new one, as PostgreSQL's observable behaviour requires.

PostgreSQL's "Halloween" rule (a statement does not see its own writes) is kept by the
executor, not by the buffer: a writing statement's scan iterates a view of the buffer
frozen at the statement's start `wseq` (entries with a larger `wseq` are skipped and the
undo log supplies their earlier state). This is the equivalent of PostgreSQL's command id;
it is per statement and costs one comparison per buffer entry visited.

### 3.3 Memory and the cap

The buffer's in-memory bytes are charged to the database's memory budget (00 §6), next to
the cache and the memtable, because they outlive statements. Two limits apply, both
*defaults* that P0 measurements may change:

| Limit | *Default* | Effect when crossed |
|---|---|---|
| `colibri.txn_buffer_memory` | 64 MiB per transaction, or less if the database budget has less headroom | The in-memory part spills to a local sorted run (§3.5); no error |
| `colibri.txn_max_size` | 1 GiB of encoded keys and values per transaction | The statement fails with `54000` *program_limit_exceeded*, message "transaction write set exceeds colibri.txn_max_size"; the transaction enters the aborted state |

PostgreSQL has no such cap: its writes reach shared buffers and WAL as they happen and a
commit is one small record. Here a transaction is one atomic batch through a ledger whose
capacity is small by construction (00 §4.5) and one atomic memtable apply; an unbounded
transaction makes both unbounded. The cap is a *default* the operator can change; crossing
it is a clean error, recorded as `intended` in the regress ledger. Rejected: `53200`
*out_of_memory* (the transaction did not run out of memory; it hit a limit the operator
set), and no cap with an unbounded spill (moves the failure to the ledger peers' disks,
where it hurts every database on the node).

### 3.4 COPY

`COPY FROM` commits in one transaction, as PostgreSQL does; a failure anywhere rolls
back the whole load. It streams rows through the same buffer, so a large `COPY` crosses
`txn_buffer_memory` and spills, and a load above `txn_max_size` fails with `54000`.
Loading more than the cap means splitting the input into several `COPY` statements, each
its own transaction. A bulk-ingest path that writes segments directly and commits them
with a manifest change plus a small ledger record (RocksDB's `IngestExternalFile` shape)
would remove the cap for `COPY`; it needs 02 and 03 to agree on the record and is deferred
(§15).

### 3.5 Spilling to sorted runs

When the in-memory part reaches `txn_buffer_memory`, it is written, in key order, to a
**run** file on `env.disk(DiskRoot::Spill)` (06 §11.1), at
`txn/<db-id>/<session-id>/<n>.run`. Spill draws on the node-level spill reservation
(*default* 20 % of the cache volume, 02 §6.2), shared with `CREATE INDEX` backfill and
`WITH HOLD` cursors; a spill that would exceed the free reservation fails the statement
with `53100` *disk_full*, and the transaction enters the aborted state. A run is a
sequence of LZ4-compressed blocks (*default* 32 KiB) of `(key, wseq, entry)` triples,
plus an in-memory sparse index (first key of each block) kept by the transaction. Blocks carry an xxh3-64 checksum and
nothing more: runs live only as long as their transaction, carry no durability
obligation, and are deleted at its end or, after a crash, when the daemon clears the
directory at start.

A spill writes every current entry **and** every previous entry still held by the undo log
(each with its own `wseq`), so the run preserves every state a `ROLLBACK TO` can return to;
then the in-memory map and undo log are emptied (§7.2 explains rollback across runs). A read consults the memory map, then runs newest to oldest; the first hit with a
`wseq` that is not rolled back wins. Above 8 runs (*default*) they are merged k-way into one,
keeping per key the versions some live savepoint mark can still return to. At commit the runs and the memory map are merged
in key order into the commit batch (§6).

## 4. The writer token and admission

### 4.1 The token

Each open database has one `WriterToken`, held from the first writing statement (§5.1)
to the end of the top-level transaction (D8). Only its holder appends to the ledger,
advances the HLC, applies to the memtable or advances `last_committed_ts`; every
correctness argument below rests on that.

### 4.2 The admission queue

Waiters for the token sit in the database's `AdmissionQueue`, one FIFO per request class
from 00 §6:

| Class | Who | Default for |
|---|---|---|
| `interactive` | Sessions whose startup parameter `colibri.request_class=interactive`, intended for human `psql` sessions and admin tools | Explicit opt-in |
| `app` | Application sessions | Every session without the parameter, including `env.db` |
| `maintenance` | Internal writers and sessions that opt in (bulk loads, migrations) | Explicit opt-in |

Dispatch is strict priority between classes and FIFO within a class, with one ageing rule:
a `maintenance` or `app` waiter that has waited longer than `colibri.admission_max_starve`
(*default* 5 s) is dispatched before any newer waiter of a higher class. Strict priority
without ageing was rejected because a steady interactive stream would starve a migration
forever, and pure FIFO was rejected because 00 §6 asks for classes.

Queue depth is bounded per database (*default* 256 waiters). A writer arriving at a full
queue fails fast with `53400` *configuration_limit_exceeded*, message "too many writers
waiting for database". This is the documented SQLSTATE 00 §6 promises.

### 4.3 Timeouts

| Setting | *Default* | Applies to | Effect |
|---|---|---|---|
| `lock_timeout` | 0 (no limit), as PostgreSQL | Time a statement waits in the admission queue | `55P03` *lock_not_available*, "could not obtain writer token"; the transaction aborts as PG's does on a lock timeout |
| `statement_timeout` | 0, as PostgreSQL | A statement's whole run, including token wait | `57014` *query_canceled* |
| `idle_in_transaction_session_timeout` | **30 s** (D8; PostgreSQL's default is 0) | A session in `InBlock` or `Aborted` with no statement running | The session is terminated with `25P03` *idle_in_transaction_session_timeout*, its transaction rolled back, its token released and its snapshot unregistered |
| `transaction_timeout` | 0, as PostgreSQL 17+ | Whole transaction | `25P04` *transaction_timeout*, session terminated |

The 30 s default applies to every idle-in-transaction session, not only token holders,
because an idle REPEATABLE READ reader pins the GC watermark (§9) as an idle writer pins
the token.

### 4.4 The hold-time counter

The token records, per acquisition, the acquisition instant from `Env`'s monotonic clock
and, on release, adds the hold time to an exact per-database total and to a histogram
(§12). "Exact" means every acquisition is counted, not sampled; 00 §6 calls it the exact
per-database counter and D8 relies on it to make the cost of long transactions visible.

### 4.5 Deadlock is impossible on the token

A deadlock needs a cycle of waits on at least two resources. A session waits for at most
one thing in this design, the token of its own database; it holds at most one token, its
own database's, because a session is bound to one database and there are no
cross-database transactions. A session that holds the token never waits for the token. So
no cycle exists and no deadlock detector is needed. Two caveats:
application-level cycles (a token holder waiting, in client code, for a second
connection queued for the token) are invisible to the server, as in PostgreSQL, and end at
`lock_timeout` or `statement_timeout`; and advisory locks would add a second resource, so
they fail with `0A000` in P0 and arrive later with a wait-for graph raising `40P01`.

### 4.6 Row locking clauses and LOCK TABLE

`SELECT ... FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` and `FOR KEY SHARE` are writing
statements: they acquire the token. Under REPEATABLE READ and SERIALIZABLE they apply the
`40001` rule of §5.3 to every row they lock. With one token, "locking a row" is "holding
the database's write side", so these clauses serialise more than PostgreSQL's row locks do:
two transactions locking disjoint rows run one after the other here and concurrently in
PostgreSQL. `NOWAIT` fails with `55P03` if the token is not immediately available; `SKIP
LOCKED` waits for the token like a plain `FOR UPDATE` (once it holds the token no row is
locked by anyone else, so it skips nothing). Queue-consumer workloads built on `SKIP
LOCKED` therefore run one consumer at a time per database. Every such diff is recorded in
the isolation ledger (00 §8) with reason `intended`, citing this section.

`LOCK TABLE` in `SHARE` mode or stronger acquires the token; weaker modes (`ACCESS SHARE`,
`ROW SHARE`, `ROW EXCLUSIVE`) are accepted and do nothing, because no lock they conflict
with exists outside the token. The diff in `deadlock-simple`-shaped specs, where PG grants
two `ACCESS SHARE` locks and then deadlocks, is likewise `intended`.

## 5. Isolation semantics mapped to PostgreSQL's

### 5.1 Which statements write

The analyzer classifies a statement as **writing** if it is `INSERT`, `UPDATE`, `DELETE`,
`MERGE`, `COPY FROM`, any DDL, a `SELECT` with a locking clause, `LOCK TABLE` in a mode
that takes the token, a statement containing a data-modifying CTE, or a call to a
user-defined function whose body writes (SQL-language functions, P1, are inspected; a
volatile function the analyzer cannot inspect, P2's PL/pgSQL, is treated as writing).
`nextval` and `setval` are not classified; they take the token at call time (§8.2). The
classification happens after analysis and before execution, so a writing statement has
executed nothing when it joins the queue.

### 5.2 READ COMMITTED: token before snapshot, and why no re-check is needed

PostgreSQL's READ COMMITTED `UPDATE` scans at the statement's snapshot, and when it finds a
target row that a concurrent transaction has updated since, it waits for that transaction
and then re-evaluates the `WHERE` clause against the newest version (EvalPlanQual).
Colibri needs no re-check. The claim and its proof:

**Claim.** If a READ COMMITTED writing statement acquires the token before it reads
`last_committed_ts`, then for every key, the newest version visible at its snapshot is
the newest committed version of that key, and it stays so until the token is released.

**Proof.** (1) Only the token holder appends to the ledger, applies to the memtable and
advances `last_committed_ts` (§4.1, §6); `SeqReserve` records are appended only under the
token too (§8). (2) The previous holder released the token only after applying its batch
and advancing `last_committed_ts` (§6 steps 6–9), so when our statement acquires the token
every committed version has `commit_ts <= last_committed_ts`, and none is missing from the
memtable or segments. (3) Our snapshot is that value, so every committed version is
visible to it; the newest visible version of a key is the newest committed one. (4) No
other session can commit until we release the token, so nothing newer can appear during
the statement or the rest of the transaction. ∎

Hence the statement's scan already sees the version EvalPlanQual would fetch, and its
`WHERE` was evaluated against it. The observable result is the one PostgreSQL produces
when the conflicting transaction committed before ours started waiting, which is one of
the outcomes PostgreSQL itself permits; the difference is only that our wait happens
before the scan instead of in the middle of it. A READ COMMITTED transaction that read
before its first write took older snapshots for those reads, exactly as PostgreSQL's
earlier statements did.

### 5.3 REPEATABLE READ: the 40001 rule

A REPEATABLE READ transaction's snapshot was fixed before it acquired the token, so other
transactions may have committed in between. The rule, applied by the writer at the moment
of the write (the token makes the check final, by the same argument as §5.2 step 4):

| Write | Condition | Result |
|---|---|---|
| `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE/SHARE`, `MERGE` matched action, `INSERT ... ON CONFLICT DO UPDATE` on a conflicting row | The target key's newest committed version has `commit_ts > snapshot_ts` | `40001` *could not serialize access due to concurrent update* (PG's message, verbatim; for `DELETE`, PG's "concurrent delete") |
| `INSERT` into a unique key (primary or unique index) | A live committed version with `commit_ts > snapshot_ts` exists for the unique key | REPEATABLE READ: `23505` *unique_violation*, as PostgreSQL; SERIALIZABLE: `40001`, as PostgreSQL since 9.6 does when the conflicting row is invisible to the snapshot |
| `INSERT` into a key whose newest version is a tombstone newer than the snapshot | | Allowed: the row is gone, and PG's unique check also passes against a dead tuple |

Targets are found by scanning at the snapshot; a row inserted after the snapshot is not a
target, as in PostgreSQL. The check reads the newest version of each target key, which the
token makes a stable local lookup (memtable, then segments whose `max_commit_ts >
snapshot_ts`; 02's per-segment statistics make the segment test a header comparison).

### 5.4 SERIALIZABLE: why REPEATABLE READ is not enough, and the read-set check

D9 says that with one writer SERIALIZABLE costs nothing beyond REPEATABLE READ. That is
true for the writes and **false for the reads**, and this section corrects it.

The argument for D9 goes: commit order is a serial order, because writers hold the token
one at a time; every snapshot is a prefix of commit order; so reads see a serial history.
The flaw is that a REPEATABLE READ transaction's reads and writes happen at different
points of that order. Its reads see the prefix at `snapshot_ts`; its writes land at
`commit_ts`; anything committed in between is invisible to its reads but precedes its
writes. Write skew follows:

```
T1 (SERIALIZABLE)             T2 (SERIALIZABLE)
BEGIN; SELECT x  (snap=10)
                              BEGIN; SELECT y  (snap=10)
UPDATE y  -- token, y@10 ok
COMMIT    (commit_ts=11)
                              UPDATE x  -- token; newest x is @<=10: no 40001 under §5.3
                              COMMIT    (commit_ts=12)
```

T2 read `y` before T1's write, so in any serial order T2 precedes T1; T1 read `x` before
T2's write, so T1 precedes T2. The cycle is the classic write skew (PostgreSQL's
`simple-write-skew` spec), and §5.3 does not catch it because the key T2 *writes* was not
touched by T1; the key it *read* was.

**Decision: SSI-lite, a read-set check under SERIALIZABLE.** A SERIALIZABLE transaction
records what it read: point keys (encoded, without timestamp) and key ranges `[start,
end)` per index, including whole-index ranges for sequential scans. Then:

1. **At token acquisition** (its first write), if `last_committed_ts == snapshot_ts`
   nothing has committed since the snapshot and the check is skipped — the common,
   uncontended case costs one comparison. Otherwise, for every recorded key and range, the
   writer asks whether any committed version in it has `commit_ts > snapshot_ts`; if one
   does, the statement fails with `40001` *could not serialize access due to read/write
   dependencies among transactions* (PG's SSI message).
2. **After token acquisition**, each further read under the transaction's old snapshot is
   checked at read time against the same condition (still a stable local check, since
   nothing commits while the token is held). Without this, T2 could take the token first
   and then read the stale `y`.
3. A SERIALIZABLE transaction that never writes never takes the token, and its read set is
   discarded at commit: it saw a prefix of a serial order, which is serialisable. This
   also covers PostgreSQL's read-only anomaly (`read-only-anomaly` specs): the anomaly
   requires two writers whose commit order disagrees with their dependency order, and the
   check above prevents such writers from both committing.

The proof that this suffices: a writing SERIALIZABLE transaction T commits at `commit_ts`
and every value it read is, by steps 1–2, unchanged between `snapshot_ts` and the moment
it took the token, and nothing changes after that moment until T commits. So every read
of T returns what it would have returned at `commit_ts - 1`; T is equivalent to running
alone at its commit point, and the commit order is a serial order for all writers. Readers
that never write sit at their snapshot point in that order.

Checking a range asks "is there any version in `[start, end)` with `commit_ts >
snapshot_ts`": a range scan of the memtable plus the segments whose `max_commit_ts >
snapshot_ts`, stopping at the first hit. Versions newer than a live snapshot are above the
GC watermark by construction, so compaction cannot have removed the evidence. Recording
the scanned range rather than the predicate yields false positives (PostgreSQL's SSI has
them too, at page and relation granularity); it never yields a false negative.

**Bound.** The read set holds at most `colibri.serializable_read_set_per_index` entries
per index (*default* 1,024). Above that, the entries for that index are replaced by one
whole-index range: the check becomes "anything committed in this table or index since my
snapshot", the analogue of PostgreSQL promoting predicate locks to the relation. Rejected:
failing with `40001` when the bound is crossed (a retry would hit the same bound on the
same query and never succeed), and silently downgrading to REPEATABLE READ (dishonest:
the transaction would report SERIALIZABLE and could commit write skew). Memory for the
read set is charged to the database budget with the buffer.

**What it costs.** Recording reads costs memory and some CPU per scan, and a contended
writer pays one validation pass, all bounded and paid only under SERIALIZABLE. D9 should
read: "SERIALIZABLE adds a read-set check at the first write (04 §5.4); with one
writer it needs no predicate-lock manager." The orchestrator amends 00.

**When 40001 is raised differs from PostgreSQL.** PG's SSI usually raises at the commit of
the pivot transaction or at a later read; ours raises at the first write of the
transaction whose read went stale. Both are allowed by the SQL standard and by PG's own
documentation ("applications must be prepared to retry"), but isolation spec expected
outputs will differ in which step fails. Those diffs are recorded with reason `intended`.

## 6. The commit path

### 6.1 Steps

```
session          WriterToken / HLC        ledger (03)          memtable    readers
   │ COMMIT            │                      │                    │           │
   │──1 final checks──▶│ (deferred unique, P1 deferred FK)         │           │
   │──2 commit_ts ────▶│ hlc.next() > last_committed_ts            │           │
   │──3 merge runs + memory → TxnBatch (sorted, fresh tombs dropped)│          │
   │──4 append ───────────────────────────────▶│ quorum fsync       │           │
   │◀─5 ack (Lsn) ─────────────────────────────│                    │           │
   │──6 apply batch at commit_ts ───────────────────────────────────▶│          │
   │──7 CommitObserver: publish CatalogSnapshot if DDL (§10)                     │
   │──8 last_committed_ts.store(commit_ts, Release) ─────────────────────────────▶│ new snapshots see it
   │──9 release token ─▶│ next waiter dispatched                                  │
   │──10 unregister snapshot; deliver NOTIFY (P2); delete run files               │
   │──11 CommandComplete "COMMIT", ReadyForQuery 'I'                              │
```

1. **Final checks.** Unique constraints are checked at write time (§6.3), so at commit
   only `DEFERRABLE INITIALLY DEFERRED` unique constraints (P1) and deferred foreign keys
   (P1) are checked here. Nothing in P0 runs at this step.
2. **Assign `commit_ts`.** `hlc.next()` returns `max(physical_ms << 16, last + 1)` using
   `Env`'s wall clock; it is strictly greater than `last_committed_ts` and therefore than
   every snapshot (all snapshots are `<= last_committed_ts`).
3. **Build the batch.** A k-way merge of the runs and the memory map, in key order,
   skipping rolled-back `wseq` intervals (§7.2), keeping the newest version per key, and
   dropping `fresh` tombstones (§3.1); each value's `stamp_offsets` (§3.1) are filled
   with `commit_ts`. The result is the `TxnBatch` payload, whose layout, fragmenting
   above the record cap (*default* 16 MiB) and complete-run replay rule are 03 §2.2's;
   bit 0 of `flags` is set when the batch contains catalog rows. An empty batch (a
   transaction whose writes all cancelled) appends nothing and skips to step 9;
   `last_committed_ts` does not move.
4. **Append.** The batch goes to the ledger client (03), tagged with the lease epoch.
5. **Await the quorum ack** (two fsync acks of three, or one on single-node; 03). The ack
   returns the record's `Lsn`.
6. **Apply** every pair to the memtable at `commit_ts` (§13). Application cannot fail for
   any reason other than process death: the memtable is charged to the budget, and a
   memtable over budget is flushed afterwards (02), never refused.
7. **Publish the catalog** if the batch contains catalog rows: the commit path calls
   `CommitObserver` (06 §1.1), implemented by `colibri-db`, which builds and publishes
   the new `CatalogSnapshot` (§10). It runs **before** step 8, so no statement can hold
   a `snapshot_ts` whose catalog is not yet published.
8. **Advance `last_committed_ts`** with `Release`. Every snapshot taken from now on sees
   the transaction. A reader that loaded the old value while step 6 ran saw none of the
   new versions, because they carry a larger timestamp than its snapshot; a partially
   applied batch is therefore never visible.
9. **Release the token**; 10. unregister the snapshot, queue NOTIFY (P2, §11.2), delete
   runs; 11. reply `CommandComplete` and `ReadyForQuery`.

### 6.2 Failure at each step

| Step | Failure | Outcome | Client sees |
|---|---|---|---|
| 1 | Deferred constraint violation (P1) | Rollback (§7) | `23505` / `23503`, then `ReadyForQuery 'I'` |
| 3 | Run file unreadable (local disk error) | Rollback | `58030` *io_error* |
| 4–5 | Ledger answers `Fenced` (a higher epoch exists; 03) | The database is closed on this node: every session of it is terminated, the token is never released to another local waiter, the memtable is discarded. Whether this record survives depends on the new writer's recovery (03), so the outcome is unknown | `08007` *transaction_resolution_unknown*, connection closed; other sessions get `57P01` *admin_shutdown* |
| any (background) | A flush's manifest `CURRENT` PUT gets `412` (02 §4.5): another writer has the database | Same closure as `Fenced`: the failed flush is abandoned and the memtable dropped | Same codes: `08007` for a session mid-commit, `57P01` for every other session |
| 4–5 | Ack timeout | 03's degraded-mode transition: the append is retried bucket-direct after the forced flush; the commit waits through it | Higher latency, then success; if the transition itself fails, `08007` and the database closes |
| after 5, before 11 | Process crash | The record is on a quorum, so it is committed; the next writer's tail replay (03) applies it to the memtable | Connection drop with no reply, as PostgreSQL after a WAL flush and before the reply |

The token is never released between steps 4 and 8 on any path: either the commit
completes, or the database closes on this node. A state in which another local writer
proceeds while a commit's outcome is unknown does not exist.

### 6.3 Unique checks happen at write time

00 §4.4 says unique indexes are enforced "at commit". PostgreSQL reports `23505` from the
statement that inserts the duplicate, and applications (`INSERT ... ON CONFLICT`, ORM
error mapping) depend on that, so Colibri checks at write time: when a row is written, for
each unique index the writer looks for a live entry with the same index columns among the
newest committed versions (memtable and segments) merged with its own buffer, excluding the
row's own primary key. NULLs are distinct unless the index is `NULLS NOT DISTINCT` (PG 15
syntax, accepted). The token makes the result stable until commit (§5.2), so no commit-time
re-check is needed. The orchestrator corrects 00 §4.4's wording.

### 6.4 Foreign keys

PostgreSQL enforces foreign keys with internal triggers at the end of each statement (or at
commit for deferred constraints). In **P0, foreign keys are not enforced**: `CREATE TABLE
... REFERENCES` and `ALTER TABLE ... ADD FOREIGN KEY` are accepted, recorded in the
catalog (05), and reported through `pg_constraint` and `information_schema`, so ORMs that
introspect them work; inserting an orphan succeeds. The regress ledger marks every
resulting diff `unsupported`, and the product surface states it. In **P1**, the writer
checks foreign keys at the end of each writing statement (and at step 1 for deferred ones)
against the newest committed versions plus the buffer, which, as with unique checks, the
token makes a local, stable lookup. Rejected for P0: enforcing through the P2 trigger
machinery (it does not exist yet) and refusing `REFERENCES` (breaks every ORM migration).

### 6.5 `synchronous_commit = off`

With the session knob off (D-setting in 00 §3.1), steps 4 and 5 split: the batch is
**sent** at step 4, steps 6–11 run immediately, and the ack is awaited in the background.
Appends stay ordered because only the token holder appends and the ledger client keeps a
per-database send order. Other sessions can see, and build on, a commit that is not yet on
a quorum; if the ack never comes (fencing or crash), those commits are lost, as with
PostgreSQL's asynchronous commit. A fenced background ack closes the database exactly as
in §6.2. On degraded entry an `off` commit already in the memtable is covered by the
forced flush and is not re-submitted (03 §8); its session sees nothing, since its commit
already returned. The recovery point is the unacknowledged window, which the documentation states
as the flush interval (00 §4.2).

### 6.6 What the commit path costs a single database

With `synchronous_commit = on` the token is held across one ledger round trip, so one
database commits at most `1 / (ledger RTT + apply)` transactions per second; neither term
is measured. Pipelined commit, releasing the token before the ack, is deferred (§15).

## 7. Rollback and savepoints

### 7.1 Undo log

Each write that changes a buffer entry at a deeper savepoint level than the entry's
`depth` (or creates it) appends `(key, previous: Option<Entry>)` to the undo log.
`SAVEPOINT s` records a mark `(undo_len, wseq)`. `ROLLBACK TO s` pops undo records down to
`undo_len`, restoring each previous entry (or removing the key), and leaves `s` in place.
`RELEASE s` drops the mark; the undo records stay, attributed to the enclosing level, so a
later `ROLLBACK TO` an outer savepoint still undoes them. Top-level `ROLLBACK` discards the
buffer, undo log, marks and runs at once.

### 7.2 Rollback across spilled runs

A run is immutable, so `ROLLBACK TO s` cannot edit it. Instead the transaction keeps a list
of **rolled-back `wseq` intervals**: rolling back to `s` adds `(s.wseq, current_wseq]`.
Reads and the commit merge ignore any run entry whose `wseq` falls in a rolled-back
interval, which exposes the newest earlier version of the key, in an older run or below
it in the same run, exactly as the undo log would have. The in-memory part is restored by
the undo log as in §7.1. Run merges (§3.5) apply the intervals and drop the ignored
entries, so the list stays short.

### 7.3 The aborted state and the token

An error inside a block leaves the buffer as it was before the failing statement started
(the statement's own writes are rolled back to an implicit per-statement mark, so `ROLLBACK
TO` an earlier savepoint resumes from a clean state) and moves the transaction to
`Aborted`. The **token stays held** until the top-level transaction ends, mirroring
PostgreSQL, where an aborted transaction keeps its locks until `ROLLBACK`. Releasing it
early was rejected: `ROLLBACK TO` resumes the transaction with its earlier writes still in
the buffer, and §5.2's argument requires that nothing has committed since the token was
taken. The 30 s idle timeout (§4.3) bounds an abandoned aborted block.

If the error happened while waiting for the token (`55P03`, `57014`), the transaction never
held it, and nothing changes. A rolled-back transaction appends nothing to the ledger and
leaves no trace except metrics.

## 8. Sequences

### 8.1 State

A sequence is a catalog object (05) and its state is one KV row under the sequence's OID
(`prefix(seq_oid, 0)`, empty user key), holding `{last_value: i64, is_called: bool,
reserved_upto: i64}` (05 §2.8).
While the database is open the authoritative counter is in memory, in a `SequenceState`
per sequence, created on first use from the row's newest version. `CREATE SEQUENCE`'s
`START`, `INCREMENT`, `MINVALUE`, `MAXVALUE`, `CYCLE` are catalog properties (05).

### 8.2 `nextval`

`nextval` is non-transactional, as in PostgreSQL: a value handed out is never returned,
whether the transaction commits or not. It requires the writer token because it changes
durable state. If the calling transaction already holds the token (any `INSERT` with a
`SERIAL` default does), nothing more happens. If it does not, `nextval` acquires the token
for the duration of the call only and releases it after; the transaction does **not**
become a writer, its snapshot does not change, and §5's rules do not apply to it, because a
sequence value is outside the transactional state.

Under the token, `nextval` increments the in-memory counter. When the counter reaches
`reserved_upto`, the holder appends a `SeqReserve { seq_oid, reserved_upto, is_called }`
ledger record (payload layout: 03 §2.2), with its own `commit_ts` from the HLC, **outside** any transaction batch, waits for
its ack, applies it to the memtable as a new version of the sequence row, and advances
`last_committed_ts` to its `commit_ts`. `reserved_upto` advances by `reserve * increment`,
where `reserve = max(32, CACHE)`. The 32 is PostgreSQL's `SEQ_LOG_VALS`: PostgreSQL WAL-logs
32 values ahead for the same reason. `CREATE SEQUENCE ... CACHE n` is honoured as the
reserve size when larger. PostgreSQL's per-session caching is not emulated: values are
globally increasing here, a subset of what PostgreSQL permits.

A `SeqReserve` inside a writing transaction's token hold is appended in its own record
before the transaction's batch; it is not rolled back if the transaction is. It costs one
ledger round trip per `reserve` values. The `SeqReserve` path is the one place where
`last_committed_ts` moves by a record that is not a `TxnBatch`; §5.2's proof covers it
because it happens only under the token.

### 8.3 `currval`, `lastval`, `setval`, `ALTER SEQUENCE`

`currval(seq)` and `lastval()` are per-session values held in the session, error `55000`
when undefined, as in PostgreSQL. `setval(seq, v [, is_called])` takes the token like
`nextval`, appends a `SeqReserve { seq_oid, reserved_upto: v, is_called }` (03 §2.2),
and sets the counter; it is non-transactional as in PostgreSQL. `ALTER SEQUENCE ...
RESTART` changes the catalog row inside the transaction and resets the counter at commit
step 7, as PostgreSQL 10+ makes it transactional.

### 8.4 Identity, SERIAL and the hidden rowid

`SERIAL`/`BIGSERIAL` create an owned sequence and a `nextval` default, as PostgreSQL.
`GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY` creates an internal sequence (05 records
the dependency). The hidden `rowid int8` of a table with no primary key (01) is allocated
from a per-table internal sequence through the same `nextval` machinery; its OID and
visibility to `pg_class` are 05's.

### 8.5 Gaps and wake

A clean close (00 §5) appends, for every touched sequence, `SeqReserve{reserved_upto =
last_value, is_called}`, so no value is lost, as PostgreSQL's clean shutdown loses none;
after wake `nextval` starts from `reserved_upto + increment` when `is_called`. After a
crash the counter resumes after the newest `reserved_upto` in segments or the replayed
ledger tail; the values between the last handed out and it are lost. **The gap is at
most `reserve` values per sequence per crash**, which with the default equals PostgreSQL's
32; §14.3 checks it.

### 8.6 What `nextval` costs others

Readers never take the token, so `nextval` never blocks a reader. A writer already holds
the token, so `nextval` inside an `INSERT` costs a memory increment and, every 32 values,
one ledger round trip inside a hold the `INSERT` already pays. The cost lands on a
read-only transaction that calls `nextval` while another transaction holds the token: it
waits for the whole writer transaction, which PostgreSQL would not make it do. This liveness
difference is `intended`; token-less `nextval` is
deferred (§15).

## 9. The GC watermark

```
gc_watermark = min( min(SnapshotRegistry), pitr_horizon_ts )      if the registry is non-empty
             = min( last_committed_ts,     pitr_horizon_ts )      otherwise
```

`pitr_horizon_ts` comes from the manifest's retention (02). A database has one open
instance (00 §4.6), so the node's registry holds every snapshot in existence. The watermark is published as an atomic that compaction (02) reads at the
start of each job and holds for the job's duration. It is monotone non-decreasing: every
newly registered snapshot is `last_committed_ts`, which is `>=` the current watermark, and
the registry lock (§2.4) orders registration against computation. A retention increase
cannot resurrect versions already compacted away.

A long-running snapshot (an open REPEATABLE READ transaction, a slow analytic query)
holds the watermark back, and compaction keeps every version above it. Segments grow,
reads scan more versions, and bucket bytes rise. PostgreSQL's `old_snapshot_threshold`,
which cut such snapshots off, was removed in PostgreSQL 17; Colibri does not have one.
The mitigations are the 30 s idle-in-transaction timeout (§4.3), `statement_timeout` and
`transaction_timeout` where the operator sets them, and visibility: the counters
`oldest_snapshot_age_seconds` and `gc_watermark_lag_seconds` (§12) are exposed per
database, so a stuck watermark is readable instead of discovered through bucket growth.

## 10. Transactional DDL

DDL is ordinary writes. A `CREATE TABLE` writes catalog rows (05) into the buffer and
takes the token like any writing statement. A transaction sees its own uncommitted DDL
through 05 §5.3's private `CatalogSnapshot`, re-derived after each DDL statement, so
`CREATE TABLE t; INSERT INTO t` works in one block and `ROLLBACK` makes `t` never have
existed. Rejected: a transaction-local catalog overlay consulted on every lookup (05 §5.3
gives the reasons). Every DDL transaction also writes the `schema_version` row incremented
by one (05). At commit step 7, after the memtable apply (step 6) and before
`last_committed_ts` advances (step 8), `CommitObserver` publishes the new `CatalogSnapshot` (§6.1); plan caches key on
`schema_version` (05 owns the invalidation). Since `next_oid` is a catalog row advanced in
the buffer, rolled-back DDL consumes no OIDs that anyone else can see (05's never-reuse
rule applies to committed allocations).

A statement binds names against the catalog as of its own `snapshot_ts` (05 §5.2). A
running statement keeps its catalog and data snapshot, so a concurrent `DROP TABLE` never
breaks it; unlike PostgreSQL, `DROP TABLE` does not wait for readers' `ACCESS SHARE`
locks. Under REPEATABLE READ, a relation whose descriptor a later committed DDL changed
raises `40001` at the transaction's first write to it; `42P01`-style errors arise only
when the snapshot itself has no such object (`intended`).

**`CREATE INDEX` backfill.** Under the token, the DDL transaction scans the table at
`last_committed_ts` (not at a REPEATABLE READ transaction's older snapshot, which would
miss rows committed since; with the token held the two can differ only in that direction),
merged with its own buffer, and writes one index entry per live row into the buffer. A
unique index reports `23505` on the first duplicate. The backfill streams through the
buffer, so it spills (§3.5) and is bounded by `txn_max_size` (§3.3): an index whose entries
exceed the cap fails with `54000`. That is a real limit on index size in P0, and it is the
same limit that §3.4's ingest path would lift. Backfill runs spill to
`DiskRoot::Spill` under §3.5's reservation. `CREATE INDEX CONCURRENTLY` runs as a plain
`CREATE INDEX` under the token in P0 (writers wait, readers do not; `intended`); 05 §6.6
plans a real background build in P2.

## 11. Cursors, notifications and two-phase commit

### 11.1 Cursors

`DECLARE` cursors (P1) read at the snapshot of the `DECLARE` statement (the transaction's,
under REPEATABLE READ and SERIALIZABLE), registered while the cursor is open, and see the
buffer as of their opening `wseq` (§3.2). `WITH HOLD` cursors are materialised at commit,
as PostgreSQL does, into a session-owned file on `DiskRoot::Spill` under §3.5's
reservation; the snapshot is then unregistered so the cursor does not pin the watermark.

### 11.2 LISTEN/NOTIFY (P2)

`NOTIFY` queues the notification in the transaction state (not in the KV buffer; they are
neither data nor durable). At commit step 10, after the ack and the apply, the queue goes
to the database's notification hub in commit order, so no listener hears a notification
before its commit is durable; it is delivered after `CommitComplete`. A rolled-back queue
is discarded; duplicates are folded as PostgreSQL does. The hook is in P0's commit path;
the hub is P2.

### 11.3 Two-phase commit

`PREPARE TRANSACTION` is not supported. `max_prepared_transactions` reports 0, and the
statement fails with PostgreSQL's own response to that setting: `55000` *prepared
transactions are disabled*, after which the transaction is rolled back. Drivers and
transaction managers already handle this message. Recorded as `unsupported`.

## 12. Observability

Exposed per database over the control socket and as Prometheus series (00 §6):

| Series | Kind | Meaning |
|---|---|---|
| `colibri_writer_token_hold_seconds` | histogram | Hold time per acquisition, labelled by request class |
| `colibri_writer_token_hold_seconds_total` | counter | Exact sum of hold times (§4.4) |
| `colibri_writer_token_wait_seconds` | histogram | Time from enqueue to dispatch, per class |
| `colibri_admission_queue_depth` | gauge | Waiters per class |
| `colibri_txn_buffer_bytes` | gauge | In-memory buffer bytes across open transactions |
| `colibri_txn_spill_total`, `colibri_txn_spill_bytes_total` | counters | Spill events and bytes written to runs |
| `colibri_serialization_failures_total` | counter | `40001`, labelled `rule=write` (§5.3) or `rule=read_set` (§5.4) |
| `colibri_oldest_snapshot_age_seconds` | gauge | Age of the oldest registered snapshot |
| `colibri_gc_watermark_lag_seconds` | gauge | `last_committed_ts - gc_watermark`, physical part |
| `colibri_commit_seconds` | histogram | Commit steps 2–11 |

## 13. Rust type sketches

Sketches, not code.
```rust
pub struct Transaction {
    isolation: IsolationLevel,              // ReadCommitted | RepeatableRead | Serializable
    read_only: bool,                        // SET TRANSACTION READ ONLY
    snapshot: Option<SnapshotGuard>,        // RR/SER: fixed; RC: per statement, held by the portal
    token: Option<TokenGuard>,              // Some from first write to top-level end
    buffer: WriteBuffer,
    read_set: Option<ReadSet>,              // Some under SERIALIZABLE until the token is taken + validated
    savepoints: Vec<SavepointMark>,         // { name, undo_len, wseq }
    catalog_delta: Option<CatalogDelta>,    // private snapshot source after DDL (05 §5.3)
    notifies: Vec<Notification>,            // P2
}

pub struct WriteBuffer {
    mem: BTreeMap<Box<[u8]>, Entry>,        // key without ts → entry (§3.1)
    undo: Vec<(Box<[u8]>, Option<Entry>)>,
    runs: Vec<SpilledRun>,                  // newest last
    rolled_back: Vec<(u64, u64)>,           // (from_wseq, to_wseq] intervals (§7.2)
    next_wseq: u64,
    mem_bytes: usize,                       // charged to DbBudget
    total_bytes: u64,                       // checked against txn_max_size
}

pub struct Entry { version: PendingVersion, wseq: u64, depth: u16, fresh: bool,
                   stamp_offsets: Vec<u32> /* §3.1 */ }
pub enum PendingVersion { Value(Bytes), Tombstone }

pub struct WriterToken {
    holder: Mutex<Option<SessionId>>,
    acquired_at: AtomicU64,                 // Env monotonic, for the exact counter
    hold_total: AtomicU64,
    queue: AdmissionQueue,
}
pub struct TokenGuard { /* releases on drop; drop after commit step 8 only */ }

pub struct AdmissionQueue { classes: [VecDeque<Waiter>; 3], max_depth: usize, max_starve: Duration }

pub struct SnapshotRegistry {
    inner: Mutex<BTreeMap<SnapshotTs, u32>>,
    last_committed_ts: AtomicU64,           // read under `inner` for registration (§2.4)
    pitr_horizon_ts: AtomicU64,
    gc_watermark: AtomicU64,                // published to compaction (02)
}
pub struct SnapshotGuard { ts: SnapshotTs /* unregisters on drop */ }

pub struct SequenceState { seq_oid: u32, last_value: i64, is_called: bool,
                            reserved_upto: i64, reserve: i64 /* max(32, CACHE) */, increment: i64 }

/// Defined and implemented in `colibri-txn`; also implements 02's `VersionSource`
/// (06 §1.1), which the merge iterator consumes. 03's tail replay applies through it too.
pub trait Memtable: VersionSource + Send + Sync {
    /// Token holder or pre-open recovery only; infallible apart from process death.
    fn apply(&self, ts: CommitTs, lsn: Lsn, batch: &[(Bytes, PendingVersion)]);
    fn get(&self, key: &[u8], ts: SnapshotTs) -> Option<Version>;
    fn range(&self, start: &[u8], end: &[u8], ts: SnapshotTs) -> MemtableIter<'_>;
    /// Any version in [start, end) with commit_ts > ts (§5.3, §5.4).
    fn has_newer(&self, start: &[u8], end: &[u8], ts: SnapshotTs) -> bool;
    fn approx_bytes(&self) -> usize;
    /// Highest lsn of the batches applied; a flush records it at the cut (02 §5, 03 §8).
    fn max_lsn(&self) -> Lsn;
}

/// Called by the commit path before `last_committed_ts` advances (§6.1 step 7);
/// `colibri-db` implements it to rebuild and publish the `CatalogSnapshot`.
pub trait CommitObserver: Send + Sync { fn on_commit(&self, ts: CommitTs, batch: &TxnBatch); }
```

## 14. Test plan

### 14.1 PostgreSQL's isolation specs

PostgreSQL 18's `src/test/isolation` runs against `colibrid` as 00 §8's isolation row,
with the single-writer contract (00 §2.2, D8) as the expected behaviour where
PostgreSQL's differs: a blocked write step is reported as **waiting** (the session is in the
admission queue), never as `55P03`; reads never block. The queue knows who
waits, so no polling of wait events is needed.

Expected outcomes, **not measured**; P0's run produces the real ledger:

| Spec group | Expectation |
|---|---|
| Read never blocks, insert/insert on unique keys (`insert-conflict-do-nothing`, `read-write-unique*`) | Pass, possibly with different waiting points |
| REPEATABLE READ update conflicts (`*-update-conflict*`-shaped, `lock-update-delete`) | Pass: §5.3 matches PG's `40001` |
| SSI specs (`simple-write-skew`, `receipt-report`, `two-ids`, `classroom-scheduling`, `total-cash`, `project-manager`, `temporal-range-integrity`, `read-only-anomaly*`) | The anomaly is prevented; the step that gets `40001` may differ (§5.4): `intended` |
| Row-lock concurrency (`FOR UPDATE` on disjoint rows, `skip-locked*`, `nowait*`) | Serialised by the token: `intended` (§4.6) |
| Deadlock specs (`deadlock-*`) | No `40P01`; the second session waits: `intended` |
| DDL concurrent with a REPEATABLE READ writer | `40001` at the first write to a relation whose descriptor changed after the snapshot (05 §5.2, §10): `intended` |
| FK (`fk-*`, `ri-trigger`), `prepared-transactions*`, advisory locks | `unsupported` in P0 (§6.4, §11.3, §4.5) |

### 14.2 Property tests

- **Read-your-writes.** Random inserts, updates, deletes, point reads and scans in one
  transaction against a `BTreeMap` model, with `txn_buffer_memory` at a few KiB so spills
  and run merges occur.
- **Savepoints.** Random interleavings of writes, `SAVEPOINT`, `RELEASE`, `ROLLBACK TO`,
  and spills; after each operation the buffer's merged view equals the model's. At the end,
  the commit batch equals the model's net write set, including `fresh` tombstone elision.
- **Serializability.** Random workloads of SERIALIZABLE transactions in the simulation
  `Env` with interleaved statements; the committed history, ordered by `commit_ts`, is
  replayed serially and every committed transaction's reads must equal the serial replay's
  (the check that would have caught §5.4's write skew). The same workload under REPEATABLE
  READ must exhibit write skew on some seed, as a guard that the test can see it.

### 14.3 Simulation seeds (06's harness, D18)

- **Crash between ack and apply.** Crash the writer after step 5 at a random point in step
  6; the next writer's tail replay must make the transaction fully visible, and no
  snapshot ever sees part of it.
- **Fenced during commit.** A second lease holder bumps the epoch while a commit is
  between steps 4 and 5; the old writer must return `08007`, close the database, and never
  release the token locally.
- **Sequence gap bound.** Many sessions call `nextval`; crash at random points; after
  recovery, the next value minus the last value any client received is `<= reserve` per
  sequence, and no value is ever handed out twice. A clean close followed by reopen has a
  gap of zero.

## 15. What this document does not decide

- **The buffer limits' values.** `txn_buffer_memory` (64 MiB) and `txn_max_size` (1 GiB)
  are *defaults* chosen to stay well inside the ledger's "seconds of writes" capacity; P0
  measures ledger peer disk and memtable apply time for large batches and resets them.
- **A bulk-ingest path** for `COPY` and `CREATE INDEX` that writes segments directly and
  commits them by manifest change, removing `txn_max_size` for those statements (§3.4).
  Needs 02 and 03.
- **The SSI read-set bound** (1,024 entries per index before promotion) and whether
  recording ranges rather than predicates produces enough false `40001`s to matter; decided
  from the isolation run and the first SERIALIZABLE workload.
- **Foreign keys in P0.** Specified as absent with `REFERENCES` accepted (§6.4); enforcement
  is P1. The orchestrator may pull it forward if the dogfood app needs it.
- **Pipelined commit** (§6.6): releasing the token before the ledger ack. The main lever
  for one database's write throughput; not designed here.
- **Token-less `nextval`** from a reserved range (§8.6).
- **The optimistic write mode** rejected as the default by D8; still decided only on a
  measured stall in a migrated project (00 §14).
- **Advisory locks** and the wait-for graph they need (§4.5).

Items for the orchestrator to carry into 00: the correction of §4.2 step 1 (writes go to
the buffer), the amendment of D9 (§5.4), the wording of §4.4 on unique checks (§6.3), the
statement that foreign keys are unenforced in P0 (§6.4) for 00 §14.
