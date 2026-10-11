# Colibri — a PostgreSQL-dialect engine built for idle-zero, instant wake, and threaded throughput

> Status: **design, pre-implementation** (2026-10-11). The decisions are D1–D20 in §3.1:
> the licence is D17, the public/private test split is D18, the parser realisation is D19
> and the collation scope is D20, the last two added when the P0 design documents
> (`docs/01`–`docs/06`) were written; §14 lists what remains open.
>
> The rule every document here follows: **what was not measured is written down as not
> measured.** Every number below is either cited with its source or marked as a *target*.

## 0. One sentence

Colibri is a single-writer, multi-reader PostgreSQL-dialect database engine whose source of
truth is object storage, whose compute is a per-node daemon that opens and closes tenant
databases the way a browser opens tabs, and whose compatibility is a measured ledger
against external yardsticks rather than a property inherited from vendored code.

## 1. Why a new project and not pg-feather

Colibri is the next engine for Runlot's `env.db`. Its predecessor, pg-feather, vendors
PostgreSQL's upper half and runs one database per thread, which bounds throughput, density
and reclaim. Colibri reverses those three choices with its own upper half, log-structured
storage, and an out-of-process daemon.

## 2. Goals and non-goals

### 2.1 Goals, each with its yardstick

| | Goal | Target | How it is measured |
|---|---|---|---|
| **G1** | **Idle converges to the bucket.** A cold tenant's marginal cost is the bytes it holds in object storage and nothing else: no process, no RAM, no local disk, no node pinning | Marginal idle cost per tenant = storage bytes × bucket price; node pool sized by *active* tenants only | Count of node-resident resources held by a cold tenant is **zero** (test, not benchmark) |
| **G2** | **Instant wake.** From cold (nothing local) to first query answered | *Target* p50 ≤ 100 ms, p99 ≤ 500 ms, in-region object storage, 100 MB database | Wake drill: evict, drop cache, time `SELECT 1` and one indexed point lookup |
| **G3** | **Density.** Open tenants per node bounded by cache RAM, not by processes or slots | *Target* ≥ 1,000 open databases on a 64 GB node with a 32 MB per-tenant default cache budget; ≥ 10,000 cold tenants addressable per node with zero cost | Density drill: open N, measure RSS slope per tenant and p99 latency on a fixed workload |
| **G4** | **Threaded throughput.** Readers run on any core; one writer per database; many databases' writers share cores | *Target* read throughput on one database scales ≥ 0.7× linearly to 8 cores; write throughput of one database ≥ pg-feather's at equal durability | pgbench-shaped read-only and write workloads against the daemon |
| **G5** | **PostgreSQL compatibility that drivers and ORMs can feel.** Wire protocol v3, PG dialect, PG type OIDs and text/binary formats, `pg_catalog`/`information_schema` introspection, a native vector index with pgvector's surface | Every driver and ORM in the §8 suite green against Colibri; `pg_regress` diff ledger with every diff attached to a reason | §8 |
| **G6** | **Free branches and PITR.** A branch is a manifest pointer; restore to any commit inside retention | A `deploy --preview` branch costs O(1) regardless of database size | Branch drill |

### 2.2 Non-goals (explicitly given up)

- **Concurrent writers to one database.** Single writer per database is the skeleton, as in
  pg-feather. A write waits; it is not refused.
- **Byte-exact `pg_regress` parity.** Parity is a *ledger* (§8), not a property. The
  reachable ceiling for an own upper half is lower than a vendored one's and will be
  stated, not hidden.
- **C extensions and the fmgr ABI.** No `CREATE EXTENSION` from compiled C. The surfaces
  Runlot sells (vector, pgcrypto subset, uuid, trigram) are built in.
- **Tier-4 extensions** (Citus, Timescale core, PostGIS) and **logical replication** in the
  first three phases.
- **A WAL-compatible or on-disk-compatible format with PostgreSQL.** Migration is logical
  (`pg_dump`/`COPY`), never physical.
- **Running inside workerd.** Neither native nor wasm. The daemon is the only runtime.

## 3. Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│ clients: workerd env.db (unix socket) · external psql/drivers (node-   │
│          front TLS → daemon) · runlot-admin (control socket)           │
├────────────────────────────────────────────────────────────────────────┤
│ wire: PG v3 · SCRAM · extended query · pipeline · COPY · cancel ·      │
│       LISTEN/NOTIFY                                                     │
├────────────────────────────────────────────────────────────────────────┤
│ session: task per connection (async), bound to one database, no       │
│          thread affinity                                               │
├────────────────────────────────────────────────────────────────────────┤
│ SQL front: parser (PG gram.y-derived) → analyzer (own catalog) →       │
│            planner (rule + cost) → executor (pull, batched)            │
├────────────────────────────────────────────────────────────────────────┤
│ catalog: own, exposed as pg_catalog / information_schema views         │
├────────────────────────────────────────────────────────────────────────┤
│ storage: per-database MVCC KV, log-structured (memtable → segments),   │
│          manifest, compaction, vector index per segment                 │
├──────────────────────────┬─────────────────────────────────────────────┤
│ local NVMe cache (blocks)│ ledger client (commit log)                   │
├──────────────────────────┴─────────────────────────────────────────────┤
│ object storage (S3 / R2 / MinIO) — the source of truth                 │
└────────────────────────────────────────────────────────────────────────┘
```

Three kinds of process exist in a deployment, and only one binary is new:

| Process | Role | Where |
|---|---|---|
| `colibrid` | The engine daemon. Hosts N open databases, serves all protocols, owns the local cache | One per node |
| `colibrid --role ledger` | The commit-log tier (§4.5), the same binary in a second role. Small, replicated, append-only | Three peers per region; the one node on single-node on-prem |
| Object storage | Source of truth for every committed byte | Existing (MinIO on the CP today; R2/S3 elsewhere) |

Nothing else has state. node-agent, node-front and the CP keep their roles; the per-tenant
DB-owner process, the pgwire bridge, the workerd overlay and the backup chain disappear.

### 3.1 Decisions

**D1. Single writer, timestamp MVCC, log-structured.** Each database has one writer at a
time, holding a lease (§4.6). Commits carry a monotone commit timestamp. A key is
`(relation, index, user key, commit_ts)`; a version is a value or a tombstone. Readers take
a snapshot timestamp and see the latest version ≤ it. The garbage watermark is the oldest
active snapshot; compaction drops anything below it. This deletes VACUUM, XIDs, freeze,
wraparound, the visibility map, and the "alone" gate in one chain, without the heap
line-pointer invariant that brings reclaim back for an engine that keeps PostgreSQL's AMs.

**D2. Object storage is the truth; everything local is a cache.** A database is a
`manifest` object naming immutable `segment` objects plus a position in its commit log.
Committed data reaches the ledger before the commit is acknowledged and reaches the bucket
within a bounded lag. Any node can open any database by reading its manifest. A node's
disk holds only blocks it has read recently.

**D3. Reclaim is compaction and compaction is always online.** Segments are immutable, so
compaction writes new segments and swaps the manifest; readers holding old segments keep
them until their snapshot ends. There is no mode in which a maintenance task needs the
database to itself. The budget is a *rate* (bytes per second per tenant, scheduled against
foreground load), never a per-table timeout.

**D4. Own upper half, PG grammar.** The parser is generated from PostgreSQL's `gram.y` the
way `libpg_query` does it, so the *accepted language* is PostgreSQL's by construction and
dialect drift is confined to the analyzer and below. Everything after parse is ours and has
no global state: a session is a struct, a plan is a value, an executor is a tree of pull
operators over a context the caller owns.

**D5. The daemon is the only runtime; workerd is unpatched.** `env.db` from a worker is a
unix-socket connection to `colibrid` carrying the same v3 protocol with a pre-authenticated
identity handshake. External clients reach the same daemon through node-front. One wire,
one session model, one place to measure. The hop costs one local round trip; Runlot already
pays that hop in split mode.

**D6. Rust, hermetic, reproducible.** One crate workspace, vendored dependencies, `--locked`,
a pinned toolchain, deterministic build flags. The artifact identity is the sha256 of a
binary anyone can rebuild from the tag. There is no build host in the release path.

The ten decisions below closed the first draft's open items on 2026-10-11. Each names the
alternatives it rejected, so the reason survives the choice.

**D7. The ledger is a role of `colibrid`, written by us.** `colibrid --role ledger` runs the
commit-log tier (§4.5); there is no separate service to deploy, and a single-node on-prem
install is the N = 1 case of the same binary. The protocol needs no leader election: the
writer already holds the CP lease's epoch, so an append is "send the epoch-tagged batch to
three peers, wait for two fsync acks" — Neon's safekeeper shape, whose Rust source is public
under Apache 2.0 and is the reference. Rejected: a separate `colibri-ledger` service (one more
stateful thing to operate for no protocol gain), BookKeeper / JetStream / per-database Raft
(wrong weight, wrong fencing semantics, or too many groups at 10,000 tenants), and no ledger
at all with group commit straight to the bucket (a 10–30 ms floor per commit). That last one
survives as the **degraded mode**: when fewer than two peers are reachable, commits go to the
bucket directly and the manifest records that the database ran degraded.

**D8. Writes are pessimistic; the token is held from first write to COMMIT.** A transaction
takes the database's writer token at its first write and returns it at COMMIT or ROLLBACK;
other writers wait in the admission queue (§6). This is PostgreSQL's observable behaviour —
a write waits, and no SQLSTATE appears that an application written for PostgreSQL does not
already handle. The cost, one long interactive transaction stalling a database's writers, is
bounded by `idle_in_transaction_session_timeout` **defaulting to 30 s** and by exact
token-hold metrics per database. Rejected as the default: optimistic buffering with
validation at COMMIT, because its failure mode is `40001` on applications that run at READ
COMMITTED and never retry, and that defect lands in the customer's code. It may return as a
per-session knob once there is a measured need.

**D9. Isolation levels follow PostgreSQL's names and defaults.** READ COMMITTED is the
default and takes a fresh snapshot timestamp per statement; REPEATABLE READ and
SERIALIZABLE fix one at the transaction's first statement. SERIALIZABLE adds a read-set
check at the transaction's first write (`docs/04 §5.4`); with one writer it needs no
predicate-lock manager, and a SERIALIZABLE transaction that never writes pays nothing.
*(Amended 2026-10-11: the first draft said SERIALIZABLE "costs nothing beyond REPEATABLE
READ", which is false: one writer serialises writes, but a REPEATABLE READ transaction
reads at its snapshot and writes at commit, so two of them can commit write skew without
either tripping the write rule. `docs/04 §5.4` gives the trace.)*

**D10. The storage engine is ours.** The MVCC key layout, per-segment vector graphs,
per-tenant budgets and a bucket-native manifest are the centre of the project and cannot be
borrowed. SlateDB's manifest and conditional-PUT fencing design is the cited reference
for §4.1 and §4.6. Rejected: SlateDB as a dependency (young; one background worker set per
instance, which is the opposite of 10,000 databases sharing one cache and one pool; MVCC
version placement and segment HNSW would be deep hooks) and RocksDB (local-disk centred, no
tiering, per-instance cost that collides with G3).

**D11. `env.db` speaks v3 over the unix socket.** One wire and one session model, so
in-worker `pg` and `postgres.js` attach unmodified and every measurement has one place.
Authentication for same-host peers is **trusted-peer mode**: `SO_PEERCRED` plus a per-worker
token carried as a startup parameter; SCRAM is for external clients only. The worker binding
pools connections. Rejected: a private framing for the in-worker path, which is faster by a
small constant and gives the engine two wire paths to keep equal.

**D12. The vector index is ours, built for immutable input only.** Segments never change
after flush, so the graph is built once over a fixed point set and rebuilt at compaction;
concurrent insert and delete, the hard half of HNSW, does not exist here. Rejected: usearch
(C++, one more unsafe boundary, built for mutable graphs) and `hnsw_rs` (mutable-graph
machinery we would not use). Quantisation and disk-resident graphs (DiskANN-shaped) are
later work and not a design input now.

**D13. The lease lives in the CP; the bucket holds the fence.** The CP row `(db-id, epoch,
holder)` says who may write; renewals are free there, and the CP is already on the deploy
and wake paths. The bucket's conditional PUT on the manifest is the fence that makes a
wrong CP harmless (§4.6). Rejected: the lease as a bucket object, because object stores have
no TTL, so liveness becomes a PUT heartbeat — about 1,000 PUT/s on a node holding 10,000
tenants — and wake would still need placement from somewhere. The engine sees the lease
through one `LeaseProvider` trait (`docs/06 §9`); P0 ships a simulation provider and a
local-file provider for single-node use, and the CP provider arrives with P3.

**D14. Initial budgets, all *targets* until P0 measures them.** Cache per database 32 MB
default, 256 MB on Pro, plan ceilings set by the plan tier. Open databases
per node = (RAM − daemon fixed cost) ÷ (cache budget + per-tenant fixed overhead), with the
fixed overhead *targeted* at ≤ 2 MB. Compaction 8 MB/s burst per tenant, node-wide ceiling
50 % of measured disk bandwidth, scheduled behind foreground work. P0's density and wake
drills replace these numbers.

**D15. Planner constants start as PostgreSQL's, plus one term.** `seq_page_cost` 1,
`random_page_cost` 4 and the rest of PostgreSQL's defaults, so the planner makes the choices
applications are tuned for. One term is added that PostgreSQL does not have: the expected
cost of a cache miss is an object-store GET, so block cost is weighted by the measured
per-segment cache hit ratio. Constants are refitted from P1 measurements.

**D16. Migration is opt-in with a replay pre-flight.** A project moves only when its owner asks; before the move its recent query log is replayed against Colibri and every statement outside the compatibility ledger (§8) is reported, and a project with any such statement stays where it is until the ledger covers it.

**D17. Colibri Source License 1.0: source available, free below USD 10 M annual revenue,
no database service, four years to Apache-2.0.** The precedents the project takes
seriously are Elastic, MongoDB, Redis, HashiCorp and CockroachDB: all five began under
Apache or BSD and locked later, and the lock is what cost them — Terraform forked into
OpenTofu, Redis into Valkey. The direction is one way: closing an open project is the
expensive move, opening a closed one is free. So Colibri **starts** closed, and the shape
of the lock is CockroachDB's 2024 one, which taught a second lesson: its BSL years kept
cloud providers out but did not make large self-hosting companies pay, because BSL permits
self-hosted production use and its Covenants forbid adding a restriction to the Additional
Use Grant. CockroachDB therefore left BSL for its own licence with a revenue threshold, and
survived it without a fork. Colibri's licence has four clauses. **Free production use** for
individuals, educational and non-profit organisations, and organisations under USD 10 M
annual revenue (affiliates under common control counted together), assessed at first
production use of a version and never retroactive. **A commercial licence** above the
threshold, with a thirty-day evaluation. **No database service** at any revenue —
CockroachDB's definition, a service whose third-party users control the schemas — so
embedding in a product whose schemas the vendor controls stays free. **A Change Date** of
four years per version to Apache-2.0, the one thing taken from BSL, so the "eventually
open" promise that cost CockroachDB nothing (nobody starts production on a three-year-old
release) is kept. Three settings come with it: a **CLA** on every external contribution; a
**commercial licence** offered by Runlot; and **`cargo deny`** checked in before the first
dependency, with an allow list of permissive licences, so no copyleft dependency can make
the licence unenforceable. The name and logo are trademarks outside the licence. Two costs
are accepted: a licence with no SPDX identifier reads as `unknown` to licence scanners and
must be read once by every enterprise legal team, which CockroachDB's precedent shortens;
and friction with the PostgreSQL community, whose own licence is the most permissive there
is, so the compatibility rows of §8 run on our side first. Rejected: Apache-2.0 from the
start (the precedents), **BSL 1.1** (chosen first on 2026-10-11 and replaced the same day:
it cannot carry a revenue threshold), AGPL-3.0 (enterprise legal friction on the on-prem
path, and it does not stop a cloud provider that publishes its patches), ELv2 (no Change
Date), and staying private (a PostgreSQL-compatible engine whose source cannot be read
cannot have its compatibility claims checked). Licence terms are in `LICENSE`; how the licence is administered is an operational matter outside this document.

**D18. Test harnesses are public; the scenario corpus is private.** This repository carries the unit and integration tests, the compatibility harnesses (the `pg_regress` runner, the driver and ORM suite runners, the wire replay tool), the deterministic simulation harness, and the generated results of the compatibility ledgers (§8), so the claims §8 makes can be checked by anyone. The accumulated simulation scenarios, regression seeds, performance baselines and query corpora derived from customer workloads are kept out of this repository; CI runs them, and a pull request that fails only there is answered with a reduced reproduction.

**D19. The parser is libpg_query behind one translation module.** D4 says the parser is
"generated from `gram.y` the way `libpg_query` does it", and D6 says the binary is
hermetic Rust; the two meet in a C library. For P0 and P1 the parser **is** libpg_query,
through the `pg_query` crate: the C sources are vendored, compiled by the `cc` crate under
the fixed flags of §9, and the protobuf parse tree is translated into Colibri's own AST in
exactly one module (`docs/06 §5`). The C boundary is accepted because the parser has no
I/O and no state between calls: a string goes in and a tree comes out, which is a smaller
surface than the index AMs D12 refused to borrow. A parser generated in Rust from `gram.y`
remains an open item (§14); if it comes, only the translation module's producer changes.
Rejected: `sqlparser-rs` (its accepted language is its own, which D4 forbids), and
hand-porting `gram.y` before the first query runs (months of work for no P0 gain).

**D20. P0 has the `C` collation only; the key format already carries the others.**
`LC_COLLATE` and `LC_CTYPE` report `C`, text keys are their UTF-8 bytes, and `pg_regress`
runs with `--no-locale`, which is how PostgreSQL's own CI runs it. Non-`C` collations come
later with ICU4X (`icu_collator`, pure Rust, Unicode licence); the index key encoding for a
collated column is fixed now in `docs/01` (sort key in the key, original string in the
value) so that their arrival changes no on-disk format. Rejected: shipping `en_US`
semantics in P0 through the platform's libc (nondeterministic across hosts, which breaks
D6's reproducibility of test results), and ICU in C (one more foreign boundary for a
feature P0 does not need).

Two smaller settings are fixed with the same reasoning. `synchronous_commit` is a
per-session knob defaulting to **on**; `off` acknowledges before the ledger quorum and the
documentation states that the recovery point becomes the flush interval. One `colibrid` per
node is the default; a dedicated daemon per tenant is an Enterprise and on-prem knob, and
the blast radius of the shared default is stated on the product surface (§6).

## 4. Storage

### 4.1 Layout

```
bucket/
  tenants/<db-id>/
    manifest/<epoch>.<seq>.json        ← one per commit of the manifest (rare: flush, compact, branch)
    seg/<ulid>.sst                     ← immutable sorted segments, 4–64 MB
    vec/<ulid>.hnsw                    ← per-segment vector graphs (§4.4)
    hot/<epoch>.<seq>.json             ← the block set to prefetch on wake (§5)
    log/<epoch>.<from-lsn>.<to-lsn>    ← group-committed commit log, bucket-direct mode only (§4.5)
<peer data dir>/ledger/log/<seq>.log   ← commit log on the ledger-role peers: one shared log per
                                         peer carrying every database's records, addressed by
                                         (db-id, epoch, lsn), rotated and dropped once flushed (docs/03 §2)
```

A `manifest` lists: the segment set per level, the ledger position every segment set is
consistent with, the schema version, the lease epoch that wrote it, the durability mode
and its history, dropped key ranges, retention pointers, branch parentage, and the
previous manifest's name (`docs/02 §4`). Each manifest version is its own object;
`manifest/CURRENT` names the latest one and is the object the conditional PUT of §4.6 is
made against. A manifest is *estimated* at about 330 KB at a thousand segments with key
bounds truncated to 48 bytes (`docs/02 §4.3`; the first draft said "kilobytes", which was
wrong), and is the unit of consistency.

### 4.2 Write path

1. A transaction's writes go to a **per-transaction write buffer**, not to the memtable:
   the commit timestamp does not exist until `COMMIT`, and a `ROLLBACK` must be able to
   discard everything. Reads inside the transaction merge the buffer over the snapshot
   (`docs/04 §3`). *(Corrected 2026-10-11; the first draft sent writes to the memtable.)*
2. At `COMMIT`, the writer assigns the commit timestamp, serialises the buffer into one
   batch, and appends it to the ledger; the ledger acknowledges after replication to a
   quorum with fsync. Only then is the batch applied to the memtable, so the memtable holds
   committed versions only and visibility is a timestamp comparison. Commit returns.
   **Commit latency floor = ledger round trip**, which is a local-network RTT plus one
   fsync, not an object-store PUT. With `synchronous_commit = off` (per session, default
   on) the commit returns before the quorum ack and the recovery point is the flush
   interval.
3. The memtable flushes to an L0 segment in the bucket when it reaches its size budget or
   its age budget (*target* 64 MB or 30 s, whichever first). The manifest is rewritten
   (conditional PUT, §4.6) to include it. The ledger is truncated behind the flushed
   position.
4. Compaction merges segments across levels on a per-tenant rate budget.

Durability on single-node on-prem: the ledger is a local fsync'd file; the mode is declared
in the manifest and the product surface says so.

### 4.3 Read path

A read resolves a key through the memtable, then segments newest to oldest, pruned by
per-segment key ranges and bloom filters. Blocks come from the local NVMe cache or from the
bucket on miss. Readers run on a shared thread pool; a read never takes the writer's lock.
Scans use segment index blocks for range bounds. Statistics for the planner are stored per
segment at flush time and merged, so `ANALYZE` is a no-op the planner does not depend on.

### 4.4 Indexes

Secondary indexes are key ranges in the same KV space (`(relation, index-id, index key,
primary key, commit_ts)`), so they are MVCC, log-structured and compacted like data. Unique
indexes are enforced by the writer at the statement that writes, against the newest
committed versions plus the transaction's own buffer, so `23505` comes from the inserting
statement as it does in PostgreSQL (`docs/04 §6.3`); the writer token makes this a local
check that cannot be invalidated before commit.

Vector indexes follow Lucene's proven shape: each segment carries its own HNSW graph built
at flush or compaction time, queries fan out across segments and merge by distance, and
compaction rebuilds the graph for the merged segment. Building a graph for a 64 MB segment
is bounded work on an immutable input, scheduled on the compaction budget — there is no
"repair" step and nothing that must wait for the database to be idle. The tombstone
accumulation of a mutable HNSW graph cannot occur: a deleted vector is a
tombstone in the KV space and disappears from the graph at the next compaction of its
segment. The cost is higher query fan-out on a database with many small segments, which
the compaction policy bounds (*target* ≤ 8 segments carrying vectors after steady state).

### 4.5 The ledger

The ledger is the only new stateful tier, and it is a **role of the same binary**
(`colibrid --role ledger`, D7), not a separate service. Per database it holds an append-only
log addressed by `(epoch, lsn)`, accepts appends only from the current epoch or higher
(§4.6), replicates to a quorum of two out of three peers, and truncates below a position
the owning daemon reports as flushed. It is the shape of Neon's safekeeper and of the
replicated log under Cloudflare's Durable Object storage; its job is to turn the commit
latency floor from an object-store PUT (*not measured here*; tens of milliseconds in-region
is the usual figure) into a local-network round trip plus one fsync.

Its capacity is small by construction: it holds at most "unflushed data across all
databases", which the flush age budget bounds to seconds of writes. Because a node that
drains carries its peers' unflushed log for those seconds, a drain waits for the owning
daemons to flush before the role stops.

**Degraded mode.** When fewer than two ledger peers are reachable, a database's commits go
to the bucket directly as group-committed log objects and the manifest records
`durability: bucket-direct` for the interval. Commit latency rises to the PUT floor; nothing
is lost. The product surface exposes the mode as a per-database counter.

### 4.6 Fencing and the lease

A database has exactly one writer because:

1. The CP grants a **lease** `(db-id, epoch)` to one daemon; epochs only increase.
2. Every manifest commit ends in a **conditional PUT** of `manifest/CURRENT` against the
   version the writer last saw (ETag / `If-Match`), carrying the epoch; the manifest body
   itself is created under a fresh name first (`docs/02 §4.5`). S3, R2 and MinIO all
   support this.
3. The ledger refuses appends from an epoch lower than the highest it has seen.

A daemon that lost its lease and did not notice is stopped at both doors; its commits were
never acknowledged past a ledger that already moved on, and its manifest write fails. There
is no rewind to witness, so the rewind witnesses and lease quarantine a file-based engine
needs have no equivalent here.

### 4.7 Branches, PITR, backups

A **branch** is a new database id whose first manifest names the parent's segment set at a
manifest and a ledger position. Segments are shared by reference; reference counts live in
the manifests and the GC of unreferenced segments is a bucket-side sweep. `deploy
--preview` therefore gets a database branch in one manifest write (a branch has its own id from birth).

**PITR** is retention of manifests and ledger: restore to a manifest, then replay ledger
records up to a timestamp. **Backup** is not a subsystem; it is the retention policy on the
bucket plus cross-region replication of the bucket if the plan buys it. Retention costs
bucket bytes, not control-plane disk.

## 5. Lifecycle: open, idle, cold, wake

| State | What is held on the node | Marginal cost | Transition |
|---|---|---|---|
| **open** | memtable, cache budget, session tasks, lease | RAM budget, CPU on demand | → idle after `idle_after` with no sessions (*default* 60 s) |
| **idle** | nothing in RAM; NVMe cache blocks retained; lease retained | NVMe bytes | → cold on cache pressure or `cold_after` (*default* 24 h); → open on first connection in ≤ one cache read |
| **cold** | nothing | **bucket bytes only** (G1) | → open on first connection (wake) |

**Wake** is: lease CAS at the CP (one round trip), `GET manifest`, `GET hot` (the block set
recorded at the last idle transition), parallel `GET` of those blocks into cache, open. The
first query then runs against a warm cache for its usual working set. Target G2 is stated
for a 100 MB database with a 2 MB hot set; the drill measures both the first `SELECT 1`
and the first indexed lookup.

Idle transition records the hot set from the cache's recency list, bounded (*default*
8 MB). A database that was never idle has no hot set and wakes block by block; the drill
measures that case too.

There is no node pinning. The CP's placement for a cold tenant is "nowhere"; wake chooses
the least-loaded node in the region. A node that disappears loses only caches; every
acknowledged commit is in the ledger quorum or the bucket.

## 6. Runtime and tenancy inside `colibrid`

- **Threads.** A fixed worker pool sized to cores. Sessions are async tasks; a statement's
  execution is a task that may run on any worker. Each open database has one **writer
  token**, taken at a transaction's first write and held to COMMIT or ROLLBACK (D8); a
  database's write serialization is therefore a queue of transactions, not a thread, and
  the token's hold time is an exact per-database counter.
- **Memory.** Every allocation that outlives a statement is charged to a database's budget
  (cache, memtable, prepared plans). Budgets come from the plan and
  are enforced by the engine, not by cgroups. A database over budget is
  flushed and shrunk, never killed.
- **CPU fairness.** Per-database weighted scheduling on the task level, with an admission
  queue per database carrying request classes (interactive > app > maintenance), bounded
  depth and overflow fail-fast with the documented SQLSTATE. The admission queue is shared
  across cores.
- **Isolation.** A Rust panic unwinds one task and closes one session; the database stays
  open. Memory-unsafe code is confined to `unsafe` blocks in the block cache and the
  segment reader, each with a stated invariant. A fault in the daemon takes every open
  database on the node with it; the mitigation is a knob — an Enterprise or on-prem tier
  may run its own `colibrid` instance (the setting after D20) — not the design. The blast
  radius is stated on the product surface.
- **Observability.** Every database exposes exact counters (open sessions, queue depth per
  class, cache bytes, ledger lag, segment count, compaction debt) over the control socket
  and as Prometheus series, so every counter is readable where it is
  counted.

## 7. SQL front

### 7.1 Parser

PostgreSQL's `gram.y` and `scan.l` (PostgreSQL licence) as `libpg_query` packages them,
used through the `pg_query` crate (D19). The raw parse tree is translated into Colibri's
own AST in one place. Keeping the grammar means `psql`, ORMs' generated SQL and every
migration tool parse identically; what they then *mean* is the analyzer's ledger.

### 7.2 Types

PostgreSQL's type OIDs, names, text I/O and binary I/O formats are preserved for every
supported type, because drivers key on OIDs and ORMs key on `format_type`. Phase 1 types:
bool, int2/4/8, float4/8, numeric, text/varchar/char, bytea, uuid, date, time, timetz,
timestamp, timestamptz, interval, json, jsonb, arrays of the above, vector. Phase 2:
inet/cidr, range types, composite types from `CREATE TYPE`, enums, domains. Each type
ships with its PG operator and function surface listed in the ledger.

### 7.3 Catalog and introspection

`pg_catalog` and `information_schema` are the compatibility surface ORMs and tools actually
read (`psql \d`, Prisma introspection, Drizzle introspect, pgAdmin). They are views over
Colibri's own catalog, and their column sets match PostgreSQL 18's for the tables in scope:
`pg_class`, `pg_attribute`, `pg_type`, `pg_namespace`, `pg_index`, `pg_constraint`,
`pg_proc` (listing what exists), `pg_database`, `pg_roles`, `pg_settings`, and the
`information_schema` tables those same tools read. `current_database()` returns the
database's name (there is no template database).

### 7.4 Planner and executor

A rule-based planner with a small cost model over per-segment statistics: index selection,
join order for ≤ 8 relations by DP, hash and merge joins, nested loop with index lookup,
sort, aggregate (hash and sorted), window functions, CTEs (inlined or materialised per
PG's rules), subquery decorrelation for the common shapes. The executor is pull-based with
row batches so a scan amortises segment block decoding. Prepared statements cache plans
per session with invalidation on catalog change.

### 7.5 Procedural code and triggers

Phase 2: `CREATE TRIGGER` with row and statement triggers calling PL/pgSQL functions,
where PL/pgSQL is a bounded interpreter for the subset ORMs and app migrations use:
assignment, `IF`, loops, `RETURN`, `NEW`/`OLD`, `RAISE`, `PERFORM`, `EXECUTE` with
`USING`, cursors in phase 3. SQL-language functions are phase 1 (the `c_proc` relation exists from P0, `docs/05 §2.9`; `CREATE FUNCTION` lands in P1).

### 7.6 Built-in extensions

`CREATE EXTENSION` accepts a fixed list and is otherwise a clean error: `vector` (type,
operators `<->`, `<=>`, `<#>`, HNSW via §4.4, pgvector's `vector_dims`/`vector_norm`
surface), `pgcrypto` (digest, hmac, gen_random_uuid, crypt/gen_salt), `uuid-ossp`,
`pg_trgm` (phase 3, GIN-shaped over the KV space), `citext`.

## 8. Compatibility as a ledger

Compatibility is measured, never claimed. Four yardsticks, each a CI row with an
exclusions file where every entry carries a *reason*, never a name alone:

| Yardstick | What it runs | Ledger |
|---|---|---|
| **pg_regress** | PostgreSQL 18's `src/test/regress` against `colibrid`, the way CockroachDB's `pg_regress` roachtest does: clone, patch the harness, diff | `regress/EXCLUSIONS.md`: every diff hunk has a signature and a reason (`unsupported`, `intended`, `defect`); `defect` count is the backlog |
| **isolation** | PostgreSQL's isolation specs, with the single-writer contract (D8) as the expected behaviour where PG's differs | `isolation/EXCLUSIONS.md` |
| **drivers and ORMs** | Each driver's and ORM's *upstream* test suite — pgx, node-postgres, postgres.js, psycopg, asyncpg, Npgsql, pgjdbc; Prisma, Drizzle, Kysely, TypeORM, Sequelize, Knex, SQLAlchemy, ActiveRecord — the way CockroachDB's roachtests run them with blocklists | `drivers/<name>.blocklist` with a reason per entry; the runlot landing page's logo list is the required set |
| **wire replay** | The same wire conversations sent to PostgreSQL 18 and to `colibrid`, byte-compared | `wire/EXCLUSIONS.md` |

The published compatibility page is generated from the ledgers; a feature appears there
only when its row is green.

## 9. Release

- One Rust workspace; `cargo vendor`; `--locked`; pinned toolchain in `rust-toolchain.toml`;
  `-C codegen-units=1`, deterministic linking flags, `SOURCE_DATE_EPOCH`. A tag builds the
  same bytes on any host; the CI job proves it by building twice on different runners and
  comparing hashes.
- The artifact is a static binary per `(os, arch)` published to an OCI registry under its
  digest, with SLSA provenance. Runlot's release lock names the digest; there is no
  pointer file, no attestation bound to a host, nothing that dies with a VM.
- Runlot's node package installs `colibrid` from the registry; the ledger is the same
  binary in its second role.
  node-agent's verify step compares the installed digest with the lock. Because workerd is
  unpatched, a Colibri release never rebuilds workerd and a workerd release never rebuilds
  Colibri.
- The acceptance matrix (§8 plus Runlot's gate and soak) runs in CI on every tag. There
  is no build host in the release path and therefore no cycle that must finish inside a
  build window.

## 10. Runlot integration

Runlot opens and closes databases through `colibrid`'s control socket, terminates TLS in
front of it, and reaches it from workers over the unix socket (D5, D11). The per-tenant
process, the park/wake timers, the backup chain and the lease quarantine of the previous
engine have no counterpart; the control plane keeps placement, leases and retention policy.
Migration of an existing project is logical: `pg_dump` from the previous engine, restore
into a Colibri database, and the pre-flight of D16.

## 11. Risks, in order of expected effect

1. **The compatibility long tail.** This is the whole cost of an own upper half.
   CockroachDB has run `pg_regress` in CI since 2024 and is still classifying diffs. The
   mitigation is structural (PG's grammar, PG's OIDs and formats, PG's catalog shapes) and
   procedural (the ledgers are the roadmap and the published claim is generated from them).
   The risk is not that the tail exists but that it is hidden; §8 exists so it cannot be.
2. **Query fan-out on vector search with many segments.** Bounded by the compaction policy;
   measured by the vector drill at 100 k and 1 M rows, the sizes at which
   mutable-graph reclaim was measured to fail.
3. **The ledger tier is a new stateful service to operate.** Small, append-only, bounded by
   flush age; still a quorum to keep healthy. Single-node mode exists for on-prem and must
   state its durability honestly.
4. **Conditional PUT semantics differ across stores.** S3 (`If-Match` on ETag), R2
   (`onlyIf`), MinIO (`If-Match`) agree on the primitive; the fencing test suite runs
   against all three before any is supported.
5. **Blast radius of one daemon per node.** Stated on the surface; a per-tier daemon is the
   knob. The alternative (process per tenant) is exactly the density ceiling this project
   exists to remove.
6. **Reproducible Rust builds.** Achievable with the flags in §9; proc-macro and build-script
   nondeterminism is the usual leak and the double-build CI row is the gate.
7. **Write throughput of one database is one writer.** The same contract as every
   single-writer engine; readers scale, writers of *different* databases scale, one
   database's writer does not. Stated as the contract.

## 12. Phases

| Phase | Delivers | Exit criterion |
|---|---|---|
| **P0 — storage and wire** | KV MVCC, memtable/segments/manifest, ledger (quorum and single-node modes, `docs/03`), cache, lease and fencing (the simulation and local-file lease providers, `docs/06 §9`), v3 wire with SCRAM, simple and extended query protocol with per-session prepared statements, transactions with savepoints, DDL/DML for phase-1 types, B-tree secondary indexes, `COPY`, the `pg_catalog` subset that `psql`'s `\d`, `\dt`, `\di`, `\ds`, `\l` and `\dn` read | Wake, density and fencing drills green at their stated targets; `psql` and node-postgres run a dogfood app |
| **P1 — the surface ORMs read** | The full `pg_catalog`/`information_schema` surface ORM introspection reads, pipelining, SQL-language functions, privilege enforcement, foreign-key enforcement, `pg_dump` compatibility, the driver and ORM rows of §8 | Every driver and ORM row of §8 green; Prisma and Drizzle introspect a real schema |
| **P2 — what apps write** | Triggers, PL/pgSQL subset, expression indexes, a real background `CREATE INDEX CONCURRENTLY`, vector index, pgcrypto/uuid, `LISTEN/NOTIFY`, branches and PITR | `pg_regress` ledger with zero `defect` entries in the phase-2 scope; vector drill at 1 M rows; branch drill O(1) |
| **P3 — Runlot** | `env.db` over the socket, node-agent open/close, the CP lease provider and cold placement, preview branches, the opt-in migration tool, release through the registry | A real customer project migrated and served for a week with the §6 counters readable in Grafana |
| **P4 — on-prem** | Packaging of the single-node mode (the mode itself is P0, `docs/03 §9`), `charts.runlot.io` chart and the deb/rpm packages carrying `colibrid` | `make onprem-drill` and `make helm-drill` green with Colibri as the engine |

## 13. Name

**Colibri** — the hummingbird. Three facts about the bird are the three goals: it is the
smallest bird (density), it enters torpor at night and drops its metabolism to near zero
(idle converges to the bucket), and it wakes from torpor and is at full wingbeat within
seconds (wake). The name is one word, pronounceable in English, Korean (콜리브리) and
Japanese (コリブリ), has no database or data-infrastructure project on it, and sits beside
`pg-feather` without inheriting its name.

What the name binds:

| Thing | Name |
|---|---|
| Repository | `colibri` |
| Daemon binary and its roles | `colibrid`, `colibrid --role ledger` |
| Crate workspace prefix | `colibri-*` |
| Environment prefix | `COLIBRI_*` |
| Control socket | `/run/colibri/control.sock` |
| Application name on the wire | `colibri/<version>` in `server_version` suffix, `server_version_num` tracks the PG dialect version implemented (18xxxx) |

Alternatives considered: **Alula** (the feather group that lets a bird land slowly — wake,
but it names only one goal), **Pinion** (flight feather and gear — continuity with feather,
but it also means to restrain), **Fledge** (leaving the nest — taken by an IoT framework).

## 14. What this document does not decide

The first draft's four open items, and six more found on re-reading, were decided on
2026-10-11 and are D7–D16 (§3.1). The P0 design documents `docs/01`–`docs/06` fix the
formats and protocols below this document, and added D19 and D20. What remains open is
what only measurement or a later phase can close:

- The **values** behind D14's budgets and D15's constants, and every number the P0
  documents mark as a *default* or *target* (block size, bloom bits, compaction ratios,
  flush thresholds, the ledger's group-fsync window). The decisions fix the shape and
  the starting numbers; P0 and P1 replace the numbers.
- Whether degraded mode (D7) should be allowed to persist beyond a stated interval before
  the CP moves the database to a node with quorum; decided from P0's fencing drill.
- Whether optimistic write mode (rejected as the default in D8) is ever worth adding as a
  knob; decided only if a migrated project asks for it with a measured stall.
- **Access control.** P0 stores roles, SCRAM verifiers and GRANT/REVOKE in the catalog and
  reports them through `pg_catalog`, but enforces nothing beyond login (`docs/05 §10`).
  Privilege enforcement is P1, default privileges P2, row-level security P3. Until P1 a
  database's roles are a naming surface, not a security boundary, and the product surface
  says so.
- **Collation beyond `C`** (D20): which collations ship, and whether a database's default
  collation can be anything other than `C`, are decided when ICU4X lands; the key encoding
  is already fixed.
- **A Rust-generated parser** replacing libpg_query (D19): decided when a measured reason
  appears (build time, a grammar divergence, or the C boundary itself); only the AST
  translation module changes.
- **Foreign keys in P0.** `REFERENCES` is accepted and recorded in P0; referential checks
  run at the end of each writing statement from P1 (`docs/04 §6.4`).
