# 06 — Crates, the environment abstraction, the simulation harness and the build

> Kind: **design**. Status: **pre-implementation** (2026-10-11). Fills the gap left by
> docs/00-overview.md §3.1 (D6, D18), §6, §7.1 and §9 for phase P0. It refines D6 (one
> workspace, hermetic build), D18 (public harnesses, private corpus), §6 (threads, memory,
> isolation, observability), §7.1 (the parser realisation, D19) and §9 (release mechanics).
> It depends on no sibling document. It is depended on by 01 (key encoding), 02 (segments,
> manifest, cache), 03 (ledger), 04 (transactions) and 05 (catalog): every one of them
> reaches time, storage, the network and task spawning only through the `Env` defined here,
> and every one names the crate this document assigns to it.

## 1. Workspace layout

### 1.1 One workspace, strict layers

Colibri is one Cargo workspace producing one binary. The crates exist to make the
dependency direction checkable, not to make deployable units: a crate may depend only on
crates below it in the table, and `cargo deny`'s `[bans]` plus a workspace test
(`tests/layering.rs`, which reads `cargo metadata` and fails on an upward or sideways edge)
enforce it.

| Layer | Crate | Responsibility | Public surface (one sentence) | May depend on (workspace) |
|---|---|---|---|---|
| 0 | `colibri-types` | Identifiers, timestamps, errors, deterministic collections | `DbId`, `Epoch`, `Lsn`, `CommitTs`, `SnapshotTs`, `TableId`, `IndexId`, `SegmentId`, `LedgerPos`, `ColibriError`, `DetHashMap`. | nothing |
| 1 | `colibri-env` | The `Env` trait, the production implementation, the simulation implementation, `MemoryBudget`, the counting allocator, the metrics registry | `Env` and its sub-traits (§11), `ProdEnv::new(config)`, `SimEnv` (constructed only by `colibri-sim`), `MemoryBudget`, `Counter`/`Histogram`. | types |
| 2 | `colibri-encoding` | 01: order-preserving key encoding, timestamp suffix, row value format | `EncodedKey::new`, `encode_tuple`, `decode_tuple`, `RowValue`. | types, env |
| 3 | `colibri-segment` | 02: segment writer/reader, bloom and index blocks, statistics, block cache (RAM + NVMe) | `SegmentBuilder`, `SegmentReader`, `BlockCache`, `trait VersionSource`, the merge iterator. | types, env, encoding |
| 3 | `colibri-manifest` | 02: manifest schema, conditional-PUT chain, hot-set object, segment GC sweep | `Manifest`, `ManifestStore::{load_current, put_if_match}`, `gc_sweep`. | types, env, encoding |
| 3 | `colibri-ledger` | 03: ledger client (writer side) and peer role (storage, recovery, truncation) | `LedgerClient::{recover, append, report_flushed}`, `LedgerPeer::serve`. | types, env, encoding |
| 4 | `colibri-txn` | 04: memtable, write buffer, writer token and admission, snapshots, HLC, sequences, GC watermark | `Memtable`, `Transaction`, `WriterToken`, `AdmissionQueue`, `Hlc`. | layers 0–3 |
| 5 | `colibri-catalog` | 05: internal catalog relations, OIDs, `CatalogSnapshot`, `pg_catalog` projections | `CatalogSnapshot`, `CatalogWriter`. | layers 0–4 |
| 6 | `colibri-sql` | Parser translation (§5), analyzer, planner, executor | `parse`, `analyze`, `plan`, `execute`; `ast::from_pg` is crate-private. | layers 0–5 |
| 7 | `colibri-db` | One open database: ties storage, txn, catalog, sql; flush and compaction scheduling; lease state | `Database::{open, session, close}`, `Session::{simple_query, prepare, execute}`. | layers 0–6 |
| 8 | `colibri-wire` | PostgreSQL v3 protocol, SCRAM, trusted-peer auth, COPY, cancel | `serve_connection(stream, auth_mode, databases)`. | layers 0–7 |
| 9 | `colibrid` | The daemon (lib + thin `main.rs`): config, roles, control socket, listeners, signals, drain | `Daemon::start(env, config) -> DaemonHandle`, the binary. | all of the above |
| — | `colibri-sim` | Scenario API, fault plans, invariant checkers, seed runner | `Scenario`, `FaultPlan`, `Invariant`, the `colibri-sim` runner binary. | everything; nothing depends on it |
| — | `colibri-regress` | pg_regress runner, isolation runner, driver suite runners, wire replay, ledger generators | Binaries `regress`, `isolation`, `drivers`, `wire-replay`, `ledger-render`. | `colibrid` (spawned as a process), types; nothing depends on it |

The three layer-3 crates are siblings and do not depend on each other. What they share
(`SegmentId`, `LedgerPos {epoch, lsn}`, the segment descriptor) lives in `colibri-types`.
Flushing is driven by `colibri-txn` for the memtable and by `colibri-db` for the manifest.

Three places would point upward; each is inverted by a trait or a value owned below:
(a) `colibri-segment` defines `trait VersionSource` (ordered iteration over encoded keys
with timestamps, point lookup); the merge iterator takes `&[&dyn VersionSource]` and the
memtable in `colibri-txn` implements it. (b) `SegmentBuilder` takes a `ColumnLayout` value
(column ids, types, the dropped-below-watermark set) supplied by `colibri-db`, so segments
and compaction never see the catalog. (c) `colibri-txn`'s commit path calls `trait
CommitObserver` with the batch before `last_committed_ts` advances; `colibri-db`
implements it to rebuild and publish the `CatalogSnapshot`.

`colibrid` is a library with a binary so that `colibri-sim` can instantiate several daemons
in one process (§4.5). `main.rs` constructs `ProdEnv` and calls `Daemon::start`; it contains
nothing else.

### 1.2 Directory tree

```
colibri/
  Cargo.toml                    workspace, [workspace.dependencies], [profile.release]
  Cargo.lock
  rust-toolchain.toml
  clippy.toml                   disallowed methods and types (§2.3)
  deny.toml
  .cargo/config.toml            vendored source replacement, target flags (§7)
  vendor/                       `cargo vendor` output, checked in
  third_party/libpg_query/      vendored C sources + LICENSE (PostgreSQL) (§5)
  crates/
    colibri-types/  colibri-env/  colibri-encoding/  colibri-segment/
    colibri-manifest/  colibri-ledger/  colibri-txn/  colibri-catalog/
    colibri-sql/  colibri-db/  colibri-wire/  colibrid/
    colibri-sim/  colibri-regress/
  tests/                        workspace integration tests (§8)
  regress/EXCLUSIONS.md         generated ledgers (00 §8), public (D18)
  isolation/EXCLUSIONS.md
  drivers/<name>.blocklist
  wire/EXCLUSIONS.md
  ci/                           build.sh, double-build compare, release signing glue
  docs/
```

### 1.3 Why one workspace and not services

00 §3 already fixes one binary with two roles (D7). A monorepo of services (a ledger
service, a storage service, a SQL front) would put a network boundary at every crate edge
and turn every protocol test into a multi-process test that the simulation could no longer
run in one address space. Rejected also: one crate with modules, because then the layering
is a convention rather than a compile error, and the `unsafe` boundary (§1.4) cannot be
stated per crate. One workspace gives one `Cargo.lock`, one vendored tree, one toolchain,
one reproducible artifact (D6), and compile-time layering.

### 1.4 Where `unsafe` is permitted

Every crate root carries `#![forbid(unsafe_code)]` except three, and the exceptions are
module-scoped:

| Crate | Module | Why | Stated invariant (in the module header) |
|---|---|---|---|
| `colibri-segment` | `cache::arena` | Block cache slabs reused without zeroing; aligned buffers for O_DIRECT-capable NVMe reads (00 §6) | A slab handle is valid while its pin count is non-zero; eviction never frees a pinned slab |
| `colibri-segment` | `reader::view` | Zero-copy views into a decoded block (00 §6) | A view never outlives the `Arc<Block>` it borrows; offsets were bounds-checked at block open |
| `colibri-env` | `alloc` | The counting `GlobalAlloc` wrapper (§10) is `unsafe impl` by definition | Forwards every call unchanged to the inner allocator; never allocates, never panics |

These crates use `#![deny(unsafe_code)]` at the root and `#[allow(unsafe_code)]` on the
listed module only, with `clippy::undocumented_unsafe_blocks` denied workspace-wide. A new
`unsafe` site is a design change to this table.

## 2. The `Env` abstraction

`Env` is the only door from engine code to time, randomness, object storage, local files,
the network, task spawning and metrics; §3 is the production implementation, §4 the
simulation, §11 the trait sketches.

### 2.1 `Arc<dyn Env>`, not generics

**Decision.** Engine code holds `Arc<dyn Env>` (cloned into each component at construction)
and the I/O methods return `BoxFuture<'_, T>` (`Pin<Box<dyn Future<Output = T> + Send +
'_>>`), written by hand, without the `async_trait` macro.

*Rejected: a generic `E: Env` parameter.* It would infect every type in every crate
(`Database<E>`, `Session<E>`, `Txn<E>`, …), monomorphise the whole engine twice (doubling
compile time and the size of test binaries), and make trait objects inside the engine
(executor operators, `Box<dyn Stream>`) awkward. Its gain — inlining and no allocation per
I/O call — is irrelevant on calls whose cost is an object-store GET or an fsync.

*Rejected: native `async fn` in traits* (not dyn-compatible once the future needs `Send`,
so it forces generics) and *`async_trait`* (the same boxed futures, but the macro hides
the `Send` bound and lifetimes that §11 exists to make visible).

### 2.2 What engine code may and may not use

Engine crates (layers 2–8) may use `Env`, `colibri-types`, `bytes`, `futures`,
`tokio::sync` (executor-independent) and pure computation crates; only `main.rs` and
`colibri-env::prod` touch the real world.

### 2.3 Enforcement: `clippy.toml`

The workspace `clippy.toml` (CI runs `cargo clippy --workspace -- -D clippy::disallowed_methods
-D clippy::disallowed_types`; `colibri-env::prod` and `crates/colibrid/src/main.rs` carry a
module-level `#[allow]` with a comment naming this section):

```toml
# Sketch of clippy.toml. Engine code reaches the world only through Env (06 §2).
disallowed-methods = [
  { path = "std::time::Instant::now",        reason = "use env.clock().now()" },
  { path = "std::time::SystemTime::now",     reason = "use env.clock().wall()" },
  { path = "std::thread::sleep",             reason = "use env.clock().sleep()" },
  { path = "std::thread::spawn",             reason = "use env.spawn()" },
  { path = "tokio::spawn",                   reason = "use env.spawn()" },
  { path = "tokio::task::spawn_blocking",    reason = "use env.spawn_cpu()" },
  { path = "tokio::time::sleep",             reason = "use env.clock().sleep()" },
  { path = "tokio::time::timeout",           reason = "use colibri_env::timeout(env, ..)" },
  { path = "std::fs::File::open",            reason = "use env.disk()" },
  { path = "std::fs::OpenOptions::open",     reason = "use env.disk()" },
  { path = "std::fs::rename",                reason = "use env.disk()" },
  { path = "std::env::var",                  reason = "config is read once in colibrid::config" },
  { path = "rand::thread_rng",               reason = "use env.rng()" },
  { path = "getrandom::getrandom",           reason = "use env.rng()" },
  { path = "std::collections::HashMap::new", reason = "RandomState; use DetHashMap" },
]
disallowed-types = [
  { path = "std::time::Instant",             reason = "use colibri_env::Instant (virtual)" },
  { path = "std::net::TcpStream",            reason = "use env.net()" },
  { path = "tokio::net::TcpStream",          reason = "use env.net()" },
  { path = "tokio::fs::File",                reason = "use env.disk()" },
  { path = "object_store::ObjectStore",      reason = "use env.object_store()" },
  { path = "std::collections::hash_map::RandomState", reason = "nondeterministic iteration" },
]
```

The `HashMap` rows exist because a randomised hash seed changes iteration order between
runs; `DetHashMap` is `HashMap` with a fixed `foldhash` (Zlib) seed, and maps keyed by
client-controlled strings use `BTreeMap`. Clippy cannot see into dependencies, so the
`determinism` job (§8.4) backs the lint: same seed, byte-identical trace.

## 3. The production `Env`

### 3.1 Runtime

`ProdEnv` owns a tokio multi-thread runtime built with `Builder::new_multi_thread()
.worker_threads(n).enable_io()` — `enable_time` too, used only inside `ProdEnv`'s clock.
`n` *defaults* to the number of CPUs available to the process (cgroup-aware
`std::thread::available_parallelism`), per 00 §6's fixed pool sized to cores. CPU-heavy work
that must not stall a worker (libpg_query parse of very large statements, LZ4 of a whole
segment, HNSW build in P2) goes through `Spawn::spawn_cpu`, which in production is tokio's
blocking pool capped at `n` threads and in simulation runs inline. The exact worker-count
policy (cores vs cores − 1 for the ledger role co-located on a node) is open (§12).

### 3.2 Object store

`ProdObjectStore` wraps `object_store::aws::AmazonS3Builder` (Apache-2.0) configured with
`with_conditional_put(S3ConditionalPut::ETagMatch)`, so that `PutMode::Create` maps to
`If-None-Match: *` and `PutMode::Update(version)` to `If-Match: <etag>`. One client per
bucket, shared by all databases on the node; `with_retry` is set to *zero* retries for
conditional puts (a retried conditional PUT whose first attempt actually succeeded would
return a spurious precondition failure, so the caller — 02's manifest commit — resolves the
ambiguity by re-reading) and to the crate's default backoff for GET/HEAD/LIST.

| Store | Configuration | Conditional put | Status before P0 exit |
|---|---|---|---|
| AWS S3 | `AmazonS3Builder` | `If-None-Match`, `If-Match` on PutObject | fencing suite must pass |
| MinIO | `AmazonS3Builder` with `with_endpoint`, `with_allow_http` for local CI only | `If-Match`, `If-None-Match` | fencing suite must pass |
| Cloudflare R2 | `AmazonS3Builder` with the account endpoint, region `auto` | 00 §11 item 4: the Workers binding calls it `onlyIf`; whether the S3 endpoint honours `If-Match` on PutObject identically is *not verified here* | fencing suite must pass |

A store is supported only after `tests/fencing_store.rs` (create and update races, stale
ETag, PUT-vs-HEAD ETag equality, LIST read-after-write) passes against it in the `stores`
job. `object_store` errors map to `StoreError` (§11.3): `Precondition`/`NotModified` to
`Precondition`, `AlreadyExists`, `NotFound`, the rest to `Transport`.

### 3.3 Disk

`ProdDisk` is rooted at one `DiskRoot` directory (§11.1) and rejects any path that
escapes it. It uses `std::fs` in blocking sections rather than `tokio::fs`:
`append` writes at the file end, `sync` is `fdatasync(2)` (`File::sync_data`), `sync_dir`
opens the directory and calls `fsync(2)` on it, which is required after `create`, `rename`
and `remove` for the change to survive power loss. 03's group fsync is built from one
`sync` per file in the window; `Disk` does not batch on its own.

### 3.4 Network

`ProdNet` provides byte streams (`AsyncRead + AsyncWrite`) over tokio's `TcpStream` and
`UnixStream`. Inter-node streams (ledger appends, ledger recovery reads) are TLS 1.3 via
`tokio-rustls` (Apache-2.0/ISC/MIT), mutually authenticated with **Runlot's node
certificates** — the node identity the CP already issues; this is a stated dependency on
Runlot's node PKI for multi-node deployments, and single-node on-prem needs no TLS. The
rustls crypto provider is discussed in §6.3. Names are `NodeName` strings resolved through
the static peer map in the config (00 §4.5 peer sets come from the CP's lease grant as
names); there is no DNS lookup inside the engine. `peer_cred()` returns `SO_PEERCRED` for
unix streams, which D11's trusted-peer mode needs.

### 3.5 Metrics: an own registry

**Decision.** `colibri-env::metrics` is an own minimal registry: `Counter` (atomic `u64`),
`Gauge` (atomic `i64`), `Histogram` (fixed log-linear buckets), each registered with a name
and a label set, of which the database id is the common one. It renders Prometheus text
exposition on the control socket's `counters` method (§9.3) and, optionally, on an HTTP
port; the same values are returned as JSON per database.

*Rejected: `metrics` + `metrics-exporter-prometheus`.* Its process-global recorder would
be shared across simulated daemons, and its handles are write-only. 00 §6 designs out
"counted but never readable": token-hold time, queue depth and ledger lag are read by the
engine itself (admission, throttling, D17 telemetry) and by tests.

## 4. The deterministic simulation `Env` (D18)

### 4.1 Lineage and purpose

FoundationDB's simulation (one process, one thread, simulated I/O and time, a seed per run)
and TigerBeetle's VOPR (a seeded fault-injecting cluster with continuous invariants). Every
P0 multi-node protocol (03 append, recovery, fencing, degraded transitions; 02/04 flush and
GC) runs under it; a failing seed is a reproduction.

### 4.2 Executor and clock

`SimEnv` runs a single-threaded executor. Every task belongs to a simulated node and has a
name. Each step picks one task uniformly at random from the ready set (scheduler RNG),
polls it once, appends `(step, node, task)` to the trace and runs the per-step invariant
checkers; when nothing is ready it advances virtual time to the earliest timer; when no
timer remains the run is quiescent; past the time bound it stops. Virtual time advances **only** when no task is runnable, and jumps directly to the next
deadline, so a scenario spanning simulated hours runs in seconds. `Clock::now()` returns
virtual time; `Clock::wall()` returns `sim_epoch + virtual elapsed + node skew`, where each
node has a seeded constant skew and optional drift, so the HLC seeding rule (brief:
`max(wall, manifest.max_commit_ts, ledger tail ts) + 1`) is exercised against clocks that
disagree. Randomising the pick among ready tasks is what explores interleavings; the seed
fixes the pick sequence.

### 4.3 Randomness

One `ChaCha8Rng` (`rand_chacha`, MIT/Apache) seeded from the scenario seed drives the
scheduler; each node and each fault source gets its own child RNG derived from
`(seed, stream id)`, so adding a fault source does not shift the random sequence a node's
engine code sees. `Env::rng()` in engine code returns the node's stream. Production uses
`ChaCha12Rng` seeded from the OS once at startup, behind the same trait; values that must be
unpredictable to an attacker (SCRAM nonces, cancel keys, ULID randomness) come from
`Rng::fill_secure`, which in production reads the OS CSPRNG on each call and in simulation
reads the node's stream.

### 4.4 Simulated object store, disk and network

**Object store.** An in-memory map with a monotone version counter as ETag; seeded faults:
latency, transport errors, **ambiguous success** (applied, but the caller sees an error —
why §3.2 does not retry), **precondition races** (a conditional PUT held until another
node's PUT to the path lands), and optional LIST staleness for negative tests.

**Disk.** Per-node in-memory files with two images: *durable* bytes and *pending* bytes.
`append` and `write` change pending; `sync` copies pending to durable; directory entries
(`create`, `rename`, `remove`) are pending until `sync_dir`. A **simulated crash** discards
every pending change. The **torn-write** option instead keeps a random prefix of the last
unsynced append, cut at a seeded byte offset, so 03's per-record `crc32c` and tail
truncation are tested against partial records. 

**Network.** Streams between named nodes go through a simulated link per ordered pair with
seeded delay, and the fault plan may partition (drop all), isolate one node, delay, drop,
duplicate, or reorder **frames**. Because a TCP stream does not reorder or duplicate bytes,
reorder and duplication apply only to connection-level events (a connection is reset and
re-established) and to frames of the `Framed` helper (§11.5); byte streams are delivered in
order or cut, which is what a real network can do above TCP.

### 4.5 The process model

A simulated node is a `SimNode { name, disk: SimDisk, daemon: Option<DaemonHandle> }`.
`start` calls the same `colibrid::Daemon::start(env, config)` that `main.rs` calls, with a
`SimEnv` view scoped to that node. `kill` drops every task tagged with the node (their
futures are dropped without running further), closes its streams with a reset, and applies
the disk crash rule. `restart` is `kill` followed by `start` against the surviving durable
disk image. The object store and the simulated CP lease table are cluster-wide and survive
any node's death.

The CP is simulated by `SimControlPlane` (§9.5), a CAS lease table `(db_id -> epoch,
holder)` that calls the chosen node's control `open` method (§9.3).

### 4.6 One engine, zero `cfg` branches

Engine crates contain no `#[cfg(test)]` or feature branches that change behaviour under
simulation. They receive an `Arc<dyn Env>`; which implementation it is cannot be observed
from the engine. The only branches are the `Env` implementations themselves in
`colibri-env::{prod, sim}`, both always compiled (no features, so the production binary is
the same artifact whether or not tests ran; the unused simulation code is removed by the
linker). A component that needs a timeout calls `colibri_env::timeout(env, dur, fut)`, which
races `fut` against `env.clock().sleep(dur)`; there is no second path.

### 4.7 The scenario API

```rust
// Sketch.
let report = Scenario::new(seed)
    .with_nodes(3)                          // n1..n3, each runs both node and ledger roles
    .with_store(StoreFaults::default().ambiguous_put(0.01))
    .with_fault_plan(FaultPlan::random(Intensity::Moderate)) // or an explicit timeline
    .with_workload(Workload::bank(accounts = 100, clients = 8))
    .invariant(inv::AckedCommitsSurvive)
    .invariant(inv::OneWriterPerEpoch)
    .invariant(inv::ManifestChain)
    .invariant(inv::GcNeverDeletesReferenced)
    .run_until(Stop::Quiescent.or(Stop::VirtualTime(Duration::from_secs(600))));
assert!(report.ok(), "{}", report.reproduction());   // prints seed + scenario name
```

`FaultPlan` is a list of timed actions (`at(t, Fault::Partition{a, b})`,
`Fault::Kill(node)`, `Fault::Restart(node)`, `Fault::LeaseMove{db, to}`,
`Fault::StoreOutage(d)`) or a seeded generator of them. The invariants shipped with P0:

| Invariant | Checked | Statement |
|---|---|---|
| `AckedCommitsSurvive` | at quiescence and after every restart | Every commit acknowledged to a client is visible to a snapshot taken after recovery, at the same `CommitTs` |
| `OneWriterPerEpoch` | every step | No two nodes hold an open writable database with the same `(db_id, epoch)`; no ledger record or manifest of epoch *e* is accepted after an epoch *e' > e* was adopted by a majority/written |
| `ManifestChain` | every manifest PUT | The successful manifests of a database form one chain by `prev`, epochs non-decreasing, `ledger_position` non-decreasing |
| `GcNeverDeletesReferenced` | every DELETE in the store | No deleted segment is named by any retained manifest of the database or of any descendant branch (02 §9.1) |
| `HlcMonotone` | every commit | `CommitTs` strictly increases per database across epochs |

Each invariant is a struct implementing `Invariant` (§11.6) that observes the event trace
(store operations, ledger acks, client acks, lease grants) rather than poking engine
internals, so it checks the protocol, not the implementation.

### 4.8 What is public and what is private (D18)

`colibri-sim` publicly carries the executor (in `colibri-env::sim`), the scenario API, the
fault plan vocabulary, the invariant checkers, a small set of example scenarios (one per
protocol, enough for a contributor to see the shape), and the runner
(`colibri-sim run --scenario <file> --seed <n>`, `colibri-sim explore --seeds 10000`). The
private corpus carries the scenario files accumulated from investigations and the seed
fixed for every defect ever found, each with its investigation note. Scenario files
are TOML with a format version, so the public runner reads private scenarios without a code
dependency in either direction.

## 5. Parser realisation (D19)

### 5.1 The crate and the vendored sources

`colibri-sql` uses the `pg_query` crate (MIT) for its bindings and parse API, with the
crate's build script replaced: the workspace patches `pg_query` (a `[patch.crates-io]` entry
pointing at `vendor/pg_query` with a modified `build.rs`) so that libpg_query's C sources are
taken from `third_party/libpg_query/` — the exact upstream release tag for PostgreSQL 18,
recorded in `third_party/libpg_query/VERSION`, with its `LICENSE` (PostgreSQL licence for
the PostgreSQL-derived sources, BSD-3-Clause for libpg_query's own) — and compiled by the
`cc` crate with fixed flags: `-O2 -fno-omit-frame-pointer -fPIC`, `-ffile-prefix-map=<abs
source dir>=/third_party/libpg_query`, no `-march`, and `SOURCE_DATE_EPOCH` honoured so
`__DATE__`/`__TIME__` do not leak. The protobuf bindings are pre-generated and checked in,
so the build needs no `protoc`.

### 5.2 One consumer

`colibri-sql::ast::from_pg` is the only module that names `pg_query::protobuf` types. It
translates the parse tree into Colibri's own AST (`colibri-sql::ast`) and rejects node types
outside P0's scope with SQLSTATE `0A000` (`feature_not_supported`) naming the construct.
Nothing downstream — analyzer, planner, executor, catalog — depends on `pg_query`, which is
what makes the parser replaceable (§5.5).

### 5.3 Fingerprint and normalise for D16

libpg_query exposes `pg_query_fingerprint` (a stable hash of a statement's structure,
ignoring literal values and whitespace) and `pg_query_normalize` (literals replaced by
`$n`). The `pg_query` crate exposes both as `fingerprint()` and `normalize()`.
`colibri-sql` re-exports them as `statement_fingerprint` and `normalize_statement`. D16's
pre-flight replay groups a project's query log by fingerprint, so a ledger verdict is
per statement shape rather than per literal, and the shadow-read comparison (D16) keys its
exclusions the same way. The fingerprint version is libpg_query's; a libpg_query upgrade
that changes it is recorded in `VERSION` and invalidates stored fingerprints.

### 5.4 Why the C boundary is acceptable

The parser is a pure function: a string goes in and a protobuf comes out. libpg_query's
documentation states it is thread-safe (its memory contexts and error state are
thread-local), it performs no file or network I/O, and it keeps no state across calls that
could carry one tenant's input into another's parse. The residual risk, a memory-safety defect in C reached by a hostile
statement, is bounded by upstream fuzzing and a nightly `cargo fuzz` target (`fuzz/parse`).

### 5.5 The open item: a Rust-generated parser

D4 says the parser is *generated from `gram.y`*. A pure-Rust parser generated from
PostgreSQL's `gram.y` and `scan.l` (a Bison-to-Rust LALR translation) would remove the C
boundary and the `cc` build step. What changes when it arrives: `colibri-sql::ast::from_pg`
is replaced by the generator's tree-to-AST translation, `third_party/libpg_query` is
removed, and fingerprinting is reimplemented over Colibri's AST (and D16's stored
fingerprints are recomputed). Nothing else changes. Its timing is open (§12).

## 6. Dependency policy

### 6.1 The allow-list

`deny.toml` (checked in before the workspace, D17) allows Apache-2.0 (with or without the
LLVM exception), MIT, BSD-2-Clause, BSD-3-Clause, ISC, PostgreSQL, Zlib, Unicode-3.0,
Unicode-DFS-2016, CC0-1.0, 0BSD and MPL-2.0; everything else is denied. This document
aligns with it and adds one restriction by policy rather than by tool: **MPL-2.0 is
accepted only on the parser path** (the brief's rule); a dependency elsewhere under MPL-2.0
needs a design note. The workspace's own crates declare `license-file = "LICENSE"` and are
covered by `[licenses] private = { ignore = true }` so that
`cargo deny` does not flag the unidentified Colibri Source License (D17's accepted cost).
`[sources]` already denies unknown registries and git sources; vendored crates come from
crates.io only.

### 6.2 The initial dependency set

| Crate | Licence | Used by | Why |
|---|---|---|---|
| `tokio` | MIT | env (prod), `tokio::sync` everywhere | 00 §6 runtime |
| `futures` | MIT/Apache-2.0 | all | `BoxFuture`, `AsyncRead`/`AsyncWrite`, combinators |
| `object_store` | Apache-2.0 | env (prod) | S3/R2/MinIO with conditional put (brief) |
| `bytes` | MIT | all | Shared immutable buffers for blocks and frames |
| `lz4_flex` | MIT | segment | Block compression (brief, 02); pure Rust |
| `xxhash-rust` (xxh3) | BSL-1.0 | segment | Block checksums, bloom hashing (02). **BSL-1.0 (Boost) is not in the allow-list**; it is permissive and must be added to `deny.toml` before the first build (§6.4) |
| `crc32c` | Apache-2.0/MIT | ledger | Record checksum (03) |
| `ulid` | MIT | types | `DbId`, `SegmentId` |
| `serde`, `serde_json` | MIT/Apache-2.0 | manifest, control socket, sim | Manifest JSON (02), control protocol (§9.3) |
| `foldhash` | Zlib | types | `DetHashMap` |
| `rand_core`, `rand_chacha` | MIT/Apache-2.0 | env | Seeded RNG (§4.3) |
| `getrandom` | MIT/Apache-2.0 | env (prod) | OS entropy at startup and `fill_secure` |
| `rustls`, `tokio-rustls` | Apache-2.0/ISC/MIT | env (prod) | Inter-node TLS (§3.4) |
| `rustls-rustcrypto` | MIT/Apache-2.0 | env (prod) | rustls crypto provider, §6.3 |
| `pg_query` | MIT (+ vendored libpg_query: PostgreSQL, BSD-3-Clause) | sql | D19 |
| `sha2`, `hmac`, `pbkdf2` (RustCrypto) | MIT/Apache-2.0 | wire | SCRAM-SHA-256 |
| `subtle` | BSD-3-Clause | wire | Constant-time proof comparison |
| `ed25519-dalek` | BSD-3-Clause | build-dependency of colibrid | Verifying the official build mark (§7.6) |
| `tracing`, `tracing-subscriber` | MIT | all, colibrid | Structured logs (§9.6) |
| `mimalloc` | MIT | env (alloc inner) | Inner allocator, §7.4 |

**Wire protocol: own, not `postgres-protocol`.** `postgres-protocol` (MIT/Apache-2.0, so
allowed either way) is a client-oriented message codec; its server-side backend messages
are partial and its SCRAM implementation is the client half. `colibri-wire` writes its own
v3 message parser and writer, because the server needs control over buffer reuse under
pipelining (P1), exact error-field ordering for the wire-replay ledger (00 §8), and the
server half of SCRAM with its iteration count and channel binding policy. `tokio-postgres`
and `postgres-protocol` may appear as **dev-dependencies** of `colibri-wire` to cross-check
the codec in tests. **SCRAM** is assembled from RustCrypto `sha2`, `hmac` and `pbkdf2` plus
`subtle`; rejected: `ring` (it would be the only C/assembly crypto in the binary for three
primitives RustCrypto provides) and a `scram` crate (thin, client-oriented, not maintained
with a server focus).

### 6.3 The rustls crypto provider

rustls requires a crypto provider. Its defaults are `aws-lc-rs` (C and assembly, a CMake
build) and `ring` (C and assembly, built by `cc`). Both complicate a reproducible
`musl` static build (§7.3), `aws-lc-rs` severely. **Decision for P0:** `rustls-rustcrypto`,
which keeps the binary free of C crypto. Its cost is stated: it is younger and less audited
than either default, and its performance is *not measured here*. The inter-node TLS it
serves carries ledger appends between Runlot's own nodes, not customer-facing TLS (node-front
terminates that, 00 §10). If a security review requires `ring` or `aws-lc-rs`, that is the
§12 open item on musl vs glibc.

### 6.4 Vendoring, locking and adding a dependency

`cargo vendor` output is checked in under `vendor/` and `.cargo/config.toml` replaces
crates.io with it; every build runs with `--locked --offline`. `cargo deny check` (licences,
bans, advisories, sources) runs in CI on every PR. A PR that adds or upgrades a dependency
must state, in its description, the crate, its licence as `cargo deny` reads it, the reason
it is needed, and the alternative considered (including "write it ourselves"); a licence
not in the allow-list requires a change to `deny.toml` in the same PR with the licence text
reviewed. `xxhash-rust`'s BSL-1.0 (Boost Software License, permissive, no copyleft — not
to be confused with BUSL-1.1, which stays denied) is the first such change. The vendored
tree is re-generated only by `ci/vendor.sh`, so a reviewer sees the diff in `vendor/`.

## 7. Build and reproducibility (00 §9)

### 7.1 Toolchain

`rust-toolchain.toml` pins one exact stable release (`channel = "1.95.0"` at workspace
creation, never a channel name), components `clippy` and `rustfmt`, both musl targets,
profile `minimal`; a bump is a PR that passes the double build.

### 7.2 Profile and flags

`[profile.release]`: `opt-level = 3`, `lto = "fat"`, `codegen-units = 1`,
`panic = "unwind"`, `strip = "symbols"`, `debug = false`, `incremental = false`.
`.cargo/config.toml` replaces crates.io with `vendor/` and sets, per musl target,
`-C target-feature=+crt-static` and `-C link-arg=-Wl,--build-id=none`.

`panic = "abort"` is rejected because 00 §6's isolation rests on unwinding: the session task
catches the unwind at its top (`FutureExt::catch_unwind` around the session future),
returns SQLSTATE `XX000` to that client, releases the writer token if held (the token's
`Drop` does it), and the database stays open. Path remapping needs absolute paths that a
config file cannot spell, so `ci/build.sh` sets `RUSTFLAGS+=" --remap-path-prefix=$PWD=/colibri
--remap-path-prefix=$CARGO_HOME=/cargo"`, the matching `-ffile-prefix-map` in `CFLAGS`, and
`SOURCE_DATE_EPOCH` to the commit time of the tag (`git log -1 --format=%ct`). Local
developer builds may differ in paths; only `ci/build.sh` output is claimed reproducible.

### 7.3 Static linking: musl

**Decision.** Release binaries are `x86_64-unknown-linux-musl` and
`aarch64-unknown-linux-musl`, fully static. Rejected: glibc static (glibc does not support
static linking of NSS and `dlopen` paths and warns on it; the binary would still depend on
the host's glibc behaviour), and glibc dynamic (the artifact identity would then depend on
the host's libc, which D6 forbids). The precondition is the absence of C crypto (§6.3); the
only C in the binary is libpg_query and the allocator (§7.4), both of which build cleanly with a musl cross `cc`.

### 7.4 The allocator

musl's built-in `malloc` is widely reported as slow under multithreaded load; this is *not
measured here*. The global allocator is `colibri_env::alloc::Counting<Inner>` (§10.2), with
`Inner = mimalloc::MiMalloc` (MIT, C compiled by `cc` under the same fixed flags) as the
P0 default. The wrapper is the part that is decided, because D14's per-tenant overhead
measurement depends on it; the inner allocator is open (§12) and changes in one line.

### 7.5 Double build and publication

The `double-build` job builds a tag with `ci/build.sh` on two runners of different
providers, hostnames and checkout paths and compares sha256 per target; a mismatch fails
the release. The release job publishes each binary to the OCI registry under its digest
with SLSA provenance from the CI builder identity (00 §9). `colibrid`'s build script
(§7.6) reads only listed files and variables and emits `rerun-if-*` for each.

### 7.6 The official build mark (D17 enforcement)

`server_version` is `18.0 (colibri/<version>)` for an official build and
`18.0 (colibri/<version> unofficial)` otherwise. The mark is a construction of
`crates/colibrid/build.rs`:

1. It computes the **source digest**: sha256 over the sorted list of `(path, sha256(content))`
   for every file tracked under `crates/`, `third_party/`, `vendor/`, `Cargo.toml`,
   `Cargo.lock` and `rust-toolchain.toml` (the file list comes from a manifest generated
   at tag time, `ci/source-manifest.txt`, so the script does not depend on `git`).
2. It reads `COLIBRI_BUILD_MARK` if set: an Ed25519 signature over
   `"colibri-build-mark-v1" || version || source_digest`.
3. It verifies the signature against the release public key embedded in `build.rs`.
4. It emits `cargo:rustc-env=COLIBRI_OFFICIAL=1` and the signature only if verification
   succeeds; otherwise `COLIBRI_OFFICIAL=0`. The wire's `ParameterStatus` and
   `SELECT version()` read it through `env!`.

The signing step exists only in the release CI job, which holds the private key as a secret,
computes the digest, signs, and passes the mark to both double-build runners. Because
Ed25519 signatures are deterministic, both runners embed the same bytes. The mark is
**published** next to the release: it is a signature, not a secret, and anyone rebuilding
the unmodified tag with it obtains the official binary byte for byte, which is what D6
promises. A patched source tree changes the digest, so the published mark no longer
verifies and the build says `unofficial`; a patch that removes the check is the deliberate
act D17 accepts it cannot prevent. Local builds never set the variable.

## 8. Test layout (D18)

### 8.1 Where tests live

| Kind | Location | Runs under | Example |
|---|---|---|---|
| Unit | `#[cfg(test)] mod tests` in each crate | plain `cargo test`; I/O through a single-node `SimEnv` | 01's encoding round-trips, 02's block decode |
| Integration | `tests/` at the workspace root | `SimEnv` multi-node, or `ProdEnv` against MinIO | `tests/fencing_store.rs`, `tests/wake.rs` |
| Simulation | `crates/colibri-sim/scenarios/` (public examples), the private corpus | `colibri-sim run` | ledger recovery with partitions |
| Compatibility | `colibri-regress` binaries against a running `colibrid` | real processes | pg_regress, isolation, drivers, wire replay |

### 8.2 The compatibility harnesses and their ledgers

`colibri-regress` contains the runners and the ledger renderer. Each runner writes a
machine-readable result, and `ledger-render` turns it into the public ledger files (00 §8),
which are committed so that the diff of a ledger is reviewable in a PR:

| Runner | Input | Ledger file |
|---|---|---|
| `regress` | PostgreSQL 18 `src/test/regress`, fetched at a pinned tag by checksum, run with `--no-locale` (D20) | `regress/EXCLUSIONS.md` |
| `isolation` | PostgreSQL 18 isolation specs, single-writer expected outputs where PG's differ | `isolation/EXCLUSIONS.md` |
| `drivers` | Each driver's/ORM's upstream suite at a pinned tag, in a container | `drivers/<name>.blocklist` |
| `wire-replay` | Recorded v3 conversations, sent to PostgreSQL 18 and `colibrid`, byte-compared | `wire/EXCLUSIONS.md` |

### 8.3 The private corpus in CI

The private corpus is checked out in CI with a deploy key held as a CI secret, available
only to workflows on branches of this repository (not to workflows triggered from forks). It
contributes private scenarios with their seeds, customer-derived query corpora for the D16
replay, and performance baselines. A PR from a fork runs the public jobs; a maintainer
triggers the private jobs on it after review (a `corpus` label on a mirrored branch). When a
private job fails, the public PR sees only "private corpus: failed (scenario id)"; a
maintainer reduces the failure to a public scenario or test that reproduces it without
customer data, and posts it on the PR (CONTRIBUTING.md).

### 8.4 CI jobs

| Job | Runs | Gates |
|---|---|---|
| `fmt-clippy` | `cargo fmt --check`, `cargo clippy` with the §2.3 lints | every PR |
| `deny` | `cargo deny check` | every PR |
| `layering` | `tests/layering.rs` over `cargo metadata` | every PR |
| `unit` | `cargo test --workspace --locked --offline` | every PR |
| `sim-public` | public scenarios at their fixed seeds + 1,000 fresh seeds | every PR |
| `determinism` | every public scenario twice per seed, trace compared byte for byte | every PR |
| `stores` | `tests/fencing_store.rs` against MinIO (every PR) and S3 and R2 (nightly and on tag) | PR (MinIO); store support (S3, R2) |
| `regress`, `isolation`, `drivers`, `wire-replay` | the compatibility runners | PR (ledger must not regress) |
| `sim-private` | private scenarios at their seeds + seed exploration | merge to main, tag |
| `corpus-replay` | D16 replay of private query corpora | merge to main, tag |
| `fuzz` | 30 minutes per target | nightly; a crash opens an issue |
| `double-build` | §7.5 | tag (release) |
| `release` | sign mark, build, publish OCI + provenance | tag, after all above |

## 9. The daemon's process shape

### 9.1 Roles and configuration

`colibrid --role node` (default) hosts open databases and serves the wire; `colibrid --role
ledger` runs only the ledger peer; `--role node,ledger` runs both in one process, which is
the single-node on-prem and the usual co-located case. Configuration is a TOML file
(`--config`, *default* `/etc/colibri/colibrid.toml`) overlaid by environment variables
`COLIBRI_<SECTION>_<KEY>` (00 §13), read once at startup by `colibrid::config` into a
`Config` struct passed to every component; nothing else reads the environment. The
config names the bucket, the disk roots (§11.1), the node name, the peer map, the
listeners, the TLS material paths and the licence key path.

### 9.2 Listeners

| Listener | Transport | Auth | Who connects |
|---|---|---|---|
| Wire, local | unix socket, *default* `/run/colibri/pg.sock` | trusted-peer mode: `SO_PEERCRED` + per-worker token as a startup parameter (D11) | workerd's `env.db` binding |
| Wire, front | TCP, *default* `127.0.0.1:5433` or a private address | SCRAM-SHA-256 (no TLS in `colibrid`; node-front terminates it, 00 §10) | node-front proxying external clients |
| Ledger | TCP + mTLS (§3.4) | node certificate | other nodes' ledger clients |
| Control | unix socket `/run/colibri/control.sock`, mode 0600 | filesystem permission + `SO_PEERCRED` uid check | node-agent, `runlot-admin` |

### 9.3 The control protocol

Newline-delimited JSON-RPC-shaped requests (`id`, `method`, `params`; reply `result` or
`error {code, message}`), no batches. Rejected: gRPC (a code generator and HTTP/2 for five
methods) and an admin database over v3 (node management would depend on the SQL front).

| Method | Params | Result | Notes |
|---|---|---|---|
| `open` | `db_id`, `epoch`, `peers[]`, `budget {cache_bytes, …}`, `role: writer\|reader` | `{state, opened_at}` | Called by node-agent after the CP lease CAS; idempotent per `(db_id, epoch)` |
| `close` | `db_id`, `mode: idle\|cold` | `{flushed_lsn}` | Flushes, writes the hot set (02), releases RAM; `cold` also drops cache blocks |
| `status` | `db_id?` | node or per-database state, durability mode, epoch | |
| `counters` | `db_id?`, `format: json\|prometheus` | the §3.5 registry | 00 §6 observability |
| `drain` | `deadline_ms` | `{remaining_dbs}` | §9.4 |

### 9.4 Signals and drain

`SIGTERM` and the `drain` method start a drain: the node stops accepting new wire
connections, closes each database after its sessions end or the deadline passes (sessions
past the deadline receive SQLSTATE `57P01` `admin_shutdown`), and each close flushes the
memtable and writes the manifest. The **ledger role** drains last: it stops only when every
database for which it holds unflushed records has reported `flushed_lsn` past them, or when
the CP has moved those databases' leases and their new writers have recovered (00 §4.5).
`SIGINT` is the same as `SIGTERM`; a second `SIGTERM` exits immediately, which is safe
because every acknowledged commit is in the ledger quorum or the bucket. `SIGHUP` reloads
the TLS material and the licence key only. Signals are received by `main.rs`, which turns
them into a `DaemonHandle::drain()` call; engine code never installs a signal handler.

### 9.5 Leases: `LeaseProvider`

`colibri-db` obtains its epoch through `trait LeaseProvider { acquire(db) -> (epoch, peers);
renew(db); release(db) }`. Three implementations: `SimControlPlane` (§4.5, simulation);
`LocalLease`, a file under `DiskRoot::Ledger` holding the last epoch per database, for
development and single-node on-prem; and the Runlot CP client (P3). P0 ships
`SimControlPlane` and `LocalLease`.

### 9.6 Logging

`tracing` with a JSON layer: one line per event to stderr with `ts` (from `Env::wall`, so
simulated logs carry virtual time), `level`, `target`, an English `msg` without
interpolated values, and fields (`db_id`, `epoch`, `lsn`, `session`, `task`).

## 10. Memory accounting (00 §6, D14)

### 10.1 Two layers

Memory accounting has an **enforced** layer and a **measuring** layer. The enforced layer is
`MemoryBudget`: components that hold memory beyond a statement charge it explicitly, so the
budget is exact for what it covers and never depends on allocator heuristics. The measuring
layer is the counting allocator: it attributes every allocation to a database tag, so the
D14 drill can measure the per-tenant fixed overhead the budget does not cover (session
structs, catalog snapshots' small allocations, task state) and cross-check it against the
RSS slope 00 §2.1 G3 names.

### 10.2 The database tag and the counting allocator

Every task spawned for a database carries a `BudgetTag` (a small index into the node's
budget table, `0` = daemon). `Spawn::spawn` wraps the future in `Tagged<F>`, whose `poll`
sets a thread-local `CURRENT_TAG` on entry and restores the previous value on exit; this is
used instead of `tokio::task_local!` because a `GlobalAlloc` cannot safely enter tokio's
task-local machinery, while a `Cell<u32>` thread-local is a plain load. `Counting<Inner>`
on `alloc` adds the size to a per-thread delta for `CURRENT_TAG`, on `dealloc` subtracts it,
and folds the delta into the tag's shared atomic counter when its magnitude passes 64 KiB
(a *default*), so the common path touches no shared cache line.

Attribution is by the tag current at the call, so memory allocated by A and freed by a
daemon task (the node-wide evictor) drifts from A to tag 0; the drift is reported, not
hidden. Rejected: a per-allocation tag header (8–16 bytes each, distorting the very overhead
measured) and per-database arenas (shared structures such as the block cache would each
need an arena discipline).

### 10.3 `MemoryBudget`

One `MemoryBudget` per open database, created by `open` from the plan's budget (D14:
*default* 32 MB cache, Pro 256 MB). It has a limit and a set of named sub-accounts, each with
a soft share of the limit:

| Charged to the database | Owner | Charged when / uncharged when |
|---|---|---|
| Block cache RAM tier | 02 | block inserted / evicted |
| Memtable | 04 | commit applies a batch / flush completes and the memtable is dropped |
| Write buffers of open transactions | 04 | write buffered / commit or rollback |
| Prepared plans and the plan cache | `colibri-sql` | plan cached / invalidated or evicted |
| `CatalogSnapshot` | 05 | snapshot built (estimated size) / last `Arc` dropped |
| Sort and hash spill buffers of a statement | `colibri-sql` | charged against the database while the statement runs; not "outliving a statement" but bounded by the same budget |

**Daemon-fixed** (charged to tag 0, counted in D14's "daemon fixed cost"): the tokio
runtime, the object-store client and its connection pool, the metrics registry, the ledger
peer's buffers (when the role is co-located), listener state, the NVMe cache index for
blocks of *closed* databases.

```rust
// Sketch.
pub struct MemoryBudget { /* limit, per-account atomics, pressure notifier */ }
impl MemoryBudget {
    pub fn try_charge(&self, acct: Account, bytes: usize) -> Result<Charge, OverBudget>;
    pub fn charge(&self, acct: Account, bytes: usize) -> Charge; // may exceed; signals pressure
    pub fn used(&self) -> usize;
    pub fn limit(&self) -> usize;
    pub fn pressure(&self) -> PressureWatch;                    // awaitable level changes
}
pub struct Charge { /* budget, account, bytes */ }               // uncharges on Drop
impl Charge { pub fn grow(&mut self, bytes: usize); pub fn shrink(&mut self, bytes: usize); }
```

### 10.4 Over budget: flush and shrink, never kill

When `used > limit`, the database's pressure watch fires and a per-database reclaimer task
acts in a fixed order: shrink the block cache share to its floor (evict), evict cached
plans, force a memtable flush (02/04), and finally make `try_charge` for new write-buffer
growth wait on the pressure watch, which backpressures writers through admission (00 §6)
rather than failing them. A statement whose own working memory cannot be charged spills
(sort, hash) or, where spilling does not exist in P0, fails with SQLSTATE `53200`
`out_of_memory` — the statement fails, the database stays open (00 §6: "never killed").

## 11. Type sketches: the contract

These are sketches; names and error variants are binding, exact signatures may tighten in
implementation.

### 11.1 `Env`

```rust
pub type BoxFuture<'a, T> = Pin<Box<dyn Future<Output = T> + Send + 'a>>;

pub trait Env: Send + Sync + 'static {
    fn clock(&self) -> &dyn Clock;
    fn rng(&self) -> &dyn Rng;
    fn object_store(&self) -> &dyn ObjectStore;
    fn disk(&self, root: DiskRoot) -> Arc<dyn Disk>;
    fn net(&self) -> &dyn Net;
    fn spawn(&self) -> &dyn Spawn;
    fn metrics(&self) -> &Registry;       // own registry, §3.5 (concrete type)
    fn node(&self) -> &NodeName;
}
pub enum DiskRoot { Cache, Ledger, Spill }
```

Production maps the roots to `COLIBRI_CACHE_DIR`, `COLIBRI_LEDGER_DIR` and
`COLIBRI_SPILL_DIR` (*default* `<cache-dir>/spill`). Spill has a node-level reservation,
*default* 20 % of the cache volume (02 §6.2), shared by 04's transaction spill, `CREATE
INDEX` backfill and `WITH HOLD` cursors; a spill beyond the free reservation fails with
`53100` *disk_full*. 04's per-transaction cap (*default* 1 GiB) applies within it.

### 11.2 `Clock` and `Rng`

```rust
#[derive(Copy, Clone, Eq, Ord, PartialEq, PartialOrd)]
pub struct Instant(u64);                  // nanoseconds since Env start; virtual in sim

pub trait Clock: Send + Sync {
    fn now(&self) -> Instant;             // monotone; for timeouts, budgets, latency
    fn wall(&self) -> SystemTime;         // for the HLC physical part (brief) and logs only
    fn sleep(&self, d: Duration) -> BoxFuture<'static, ()>;
}

pub trait Rng: Send + Sync {
    fn next_u64(&self) -> u64;
    fn fill(&self, buf: &mut [u8]);           // seeded; deterministic in sim
    fn fill_secure(&self, buf: &mut [u8]);    // OS CSPRNG in prod (nonces, cancel keys)
}
```

### 11.3 `ObjectStore`

```rust
pub struct ObjPath(String);               // "tenants/<db>/manifest/…"; validated, no ".."
#[derive(Clone, Eq, PartialEq)]
pub struct ObjVersion { pub etag: String, pub version: Option<String> }

pub enum PutMode { Create, Update(ObjVersion), Overwrite }

pub struct ObjMeta { pub path: ObjPath, pub size: u64, pub version: ObjVersion,
                     pub last_modified: SystemTime }

pub enum StoreError {
    NotFound,
    AlreadyExists,                         // PutMode::Create lost the race
    Precondition,                          // PutMode::Update: version moved (the fence, 00 §4.6)
    Transport { retryable: bool, ambiguous: bool, detail: String },
}

pub trait ObjectStore: Send + Sync {
    fn get(&self, p: &ObjPath) -> BoxFuture<'_, Result<(Bytes, ObjVersion), StoreError>>;
    fn get_range(&self, p: &ObjPath, range: Range<u64>)
        -> BoxFuture<'_, Result<(Bytes, ObjVersion), StoreError>>;
    fn put(&self, p: &ObjPath, body: Bytes, mode: PutMode)
        -> BoxFuture<'_, Result<ObjVersion, StoreError>>;
    fn put_multipart(&self, p: &ObjPath, body: Box<dyn futures::io::AsyncRead + Send + Unpin>)
        -> BoxFuture<'_, Result<ObjVersion, StoreError>>;     // parts default 8 MiB
    fn head(&self, p: &ObjPath) -> BoxFuture<'_, Result<ObjMeta, StoreError>>;
    fn list(&self, prefix: &ObjPath)
        -> BoxFuture<'_, Result<Vec<ObjMeta>, StoreError>>;   // full listing, sorted by path
    fn delete(&self, p: &ObjPath) -> BoxFuture<'_, Result<(), StoreError>>;
}
```

### 11.4 `Disk`

```rust
pub struct DiskPath(String);              // relative to the Disk root; validated
pub trait Disk: Send + Sync {
    fn open(&self, p: &DiskPath, opts: OpenOpts) -> BoxFuture<'_, io::Result<Box<dyn DiskFile>>>;
    fn rename(&self, from: &DiskPath, to: &DiskPath) -> BoxFuture<'_, io::Result<()>>;
    fn remove(&self, p: &DiskPath) -> BoxFuture<'_, io::Result<()>>;
    fn list(&self, dir: &DiskPath) -> BoxFuture<'_, io::Result<Vec<DiskPath>>>;
    fn sync_dir(&self, dir: &DiskPath) -> BoxFuture<'_, io::Result<()>>; // makes create/rename/remove durable
}
pub struct OpenOpts { pub create: bool, pub truncate: bool, pub write: bool }
pub trait DiskFile: Send + Sync {
    fn read_at(&self, off: u64, len: usize) -> BoxFuture<'_, io::Result<Bytes>>;
    fn write_at(&self, off: u64, data: Bytes) -> BoxFuture<'_, io::Result<()>>;
    fn append(&self, data: Bytes) -> BoxFuture<'_, io::Result<u64>>;   // returns offset written
    fn sync(&self) -> BoxFuture<'_, io::Result<()>>;                    // data durable
    fn truncate(&self, len: u64) -> BoxFuture<'_, io::Result<()>>;      // ledger tail repair (03)
    fn punch_hole(&self, offset: u64, len: u64) -> BoxFuture<'_, io::Result<()>>;
}
```

### 11.5 `Net` and `Spawn`

```rust
pub struct NodeName(String);
pub enum Endpoint { Node { node: NodeName, service: Service }, Unix(String), Tcp(SocketAddr) }
pub enum Service { Ledger, Wire, Control }

pub trait ByteStream: futures::io::AsyncRead + futures::io::AsyncWrite + Send + Unpin {
    fn peer(&self) -> PeerInfo;           // node name (mTLS) or SO_PEERCRED {pid, uid, gid}
}
pub trait Net: Send + Sync {
    fn connect(&self, to: &Endpoint) -> BoxFuture<'_, io::Result<Box<dyn ByteStream>>>;
    fn listen(&self, at: &Endpoint) -> BoxFuture<'_, io::Result<Box<dyn Listener>>>;
}
pub trait Listener: Send {
    fn accept(&mut self) -> BoxFuture<'_, io::Result<Box<dyn ByteStream>>>;
}

pub struct TaskSpec { pub name: &'static str, pub tag: BudgetTag, pub class: TaskClass }
pub enum TaskClass { Interactive, App, Maintenance, Daemon }   // 00 §6 request classes
pub trait Spawn: Send + Sync {
    fn spawn(&self, spec: TaskSpec, fut: BoxFuture<'static, ()>) -> TaskHandle;
    fn spawn_cpu(&self, spec: TaskSpec, f: Box<dyn FnOnce() + Send>) -> TaskHandle;
}
pub struct TaskHandle { /* join() -> BoxFuture<Result<(), Panicked>>, abort() */ }
```

Framing (`Framed<S>`, `u32` length prefix) is a helper over `ByteStream` used by the
ledger and control socket; the wire uses the raw stream. The 16 MiB limit applies to the
ledger record carried in an `Append` (03 §2.2's cap), and the framing header and the
`Append` message fields ride on top of it, so the hard frame maximum is *default* 16 MiB +
64 KiB and a 16 MiB record always fits.

### 11.6 Simulation

`Scenario` is the builder shown in §4.7 (`new`, `with_nodes`, `with_store`,
`with_fault_plan`, `with_workload`, `invariant`, `run_until`, plus `from_file` for the
private corpus format). The checker contract:

```rust
// Sketch.
pub trait Invariant {
    fn name(&self) -> &'static str;
    fn on_event(&mut self, ev: &TraceEvent, world: &WorldView) -> Result<(), Violation>;
    fn at_end(&mut self, world: &WorldView) -> Result<(), Violation>;
}
pub struct Report { pub seed: u64, pub steps: u64, pub virtual_time: Duration,
                    pub violations: Vec<Violation>, pub trace_digest: [u8; 32] }
```

`trace_digest` is what the `determinism` job compares.

## 12. What this document does not decide

- **musl vs glibc**, if a security review requires `ring` or `aws-lc-rs` as the rustls
  provider instead of `rustls-rustcrypto` (§6.3). `ring` under musl is likely workable;
  `aws-lc-rs` may force a glibc target. Decided by that review, not by measurement here.
- **The inner allocator.** `mimalloc` is the P0 default behind the counting wrapper (§7.4);
  musl's allocator, jemalloc (`tikv-jemallocator`) and mimalloc are compared in the P0
  density drill. The wrapper (§10.2) is decided; the inner is not.
- **The exact tokio worker-count policy**: all cores, cores minus a reserve for the
  co-located ledger role's group fsync, or a separate runtime for the ledger role. Decided
  from the P0 throughput and ledger-latency measurements.
- **The heap-attribution drift** (§10.2): whether it is small enough for D14, checked
  against the RSS slope in the density drill.
- **R2's conditional-put behaviour on the S3 endpoint** (§3.2): not verified here; the
  fencing suite decides whether R2 is supported.

- **The frame-size and accounting-batch defaults** (16 MiB + 64 KiB frame, 64 KiB allocator fold):
  defaults, not measurements.
