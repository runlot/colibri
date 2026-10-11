# Colibri documents

Numbered in the order they were written. A document is either **design** (authoritative
for what it decides, kept current) or a **journal** (closed, never rewritten). Each says
which it is at the top.

| | Document | Kind | What it decides |
|---|---|---|---|
| 00 | [Overview and architecture](00-overview.md) | design | Goals and their yardsticks, the twenty decisions (D1–D20), storage, lifecycle, runtime, SQL front, the compatibility ledgers, release, Runlot integration, phases, the name |
| 01 | [Key encoding, versions and row values](01-key-encoding.md) | design | The KV key space, the order-preserving encoding per type, the timestamp suffix, the row value format, tombstones, secondary index keys, the hidden rowid, collation sort keys |
| 02 | [Segments, the manifest, the block cache and segment GC](02-segments-manifest-cache.md) | design | The segment file format and statistics, levels and compaction, the manifest schema and conditional PUT, flush, the two-tier block cache, the hot set, branches and segment GC |
| 03 | [The ledger](03-ledger.md) | design | Record format, append and ack, epoch fencing, recovery on lease change, truncation, the three durability modes and their transitions, single-node |
| 04 | [Transactions](04-transactions.md) | design | The write buffer, the writer token and admission, snapshots per isolation level, savepoints, the commit path, sequences, the GC watermark, transactional DDL |
| 05 | [The catalog](05-catalog.md) | design | Internal relations and their OIDs, OID allocation, bootstrap, the catalog snapshot and schema versions, columns and drop, the `pg_catalog` projection, roles and privileges |
| 06 | [Crates, the environment abstraction, the simulation harness and the build](06-crates-and-environment.md) | design | The workspace and its layering, the `Env` trait, the production and simulation environments, the parser realisation, dependencies, reproducible build, test layout, the daemon's shape, memory accounting |

Documents 01–06 are the P0 set: together with 00 they are what an implementer needs to
start phase P0 without deciding a format or a protocol alone. Each refines named sections
of 00 and ends with what it does not decide.

Rules every document follows:

- **What was not measured is written down as not measured.** A number is either cited
  with its source or marked as a *target*.
- **An exclusion carries a reason, never a name alone.** Every ledger entry says why.
- **Three numbers travel together** on a compatibility change once there is code:
  ledger count, binary size, per-database resident size. Two getting worse while one
  improves is a trade, and a trade is recorded.
