# 02 — Segments, the manifest, the block cache and segment GC

> Kind: **design**. Status: **pre-implementation** (2026-10-11). Fills the gap left by
> docs/00-overview.md §4.1, §4.2 steps 3–4, §4.3, §4.7 and §5 for phase P0. It fixes the
> segment (SST) byte format, the manifest JSON schema and its fencing, the hot-set object,
> the two-tier block cache, compaction policy and the segment GC sweep across branches.
> It depends on **01** for the key encoding (the `prefix || user key || !commit_ts` layout,
> the tombstone marker, column-id-tagged row values) and is depended on by **03** (flush
> reports `flushed_lsn`; forced flush at durability transitions), **04** (the GC watermark,
> snapshot reads through the merge iterator) and **06** (the `Env` object-store and disk
> methods this crate calls). Crates: `colibri-segment` (segments, compaction, cache) and
> `colibri-manifest` (manifest, hot set, GC).
>
> Every number below is a *default* or a *target*. None has been measured; P0's wake,
> density and compaction drills replace them.

## 1. Segment file format

A segment is an immutable object `tenants/<db-id>/seg/<ulid>.sst`, *target* 4–64 MB
(00 §4.1). It holds a sorted run of encoded keys (01) with their versions, for any mix of
tables and indexes of one database. The ULID is generated from the `Env` clock and
randomness at the moment the builder starts; it is globally unique, which lets branches
share a segment and the node cache share its blocks (§6).

### 1.1 Layout

```
offset 0
┌──────────────────────────────────────────────────────────────┐
│ header (16 B): magic "COLIBSEG" (8) │ format_version u32 = 1 │ │
│                reserved u32 = 0                              │
├──────────────────────────────────────────────────────────────┤
│ data block 0  ─┐                                             │
│ data block 1   │  each: payload │ type u8 │ xxh3-64 u64      │
│ ...            │  (payload LZ4-compressed or raw)            │
│ data block N  ─┘                                             │
├──────────────────────────────────────────────── meta_offset ─┤
│ filter partition 0..P   (bloom, one per index partition)     │
│ index partition 0..P    (block handles + first/last key)     │
│ statistics block        (per (table, column))                │
│ properties block                                             │
│ top-level index         (one entry per partition)            │
├──────────────────────────────────────────────────────────────┤
│ footer (64 B, fixed)                                         │
│   top_index   handle (offset u64, size u32)                  │
│   properties  handle (offset u64, size u32)                  │
│   statistics  handle (offset u64, size u32)                  │
│   meta_offset u64                                            │
│   reserved    (zero, 12 B)                                   │
│   format_version u32 │ footer xxh3-64 u64 │ magic (8)        │
└──────────────────────────────────────────────────────────────┘
```

All integers are little-endian except inside encoded keys, which are memcomparable (01).
Every block, data or meta, has the same trailer: one compression type byte (`0` raw,
`1` LZ4 block format) and an xxh3-64 over `payload || type`. A **block handle** is
`(offset u64, size u32)`, where `size` is the payload length without the 9-byte trailer.
Meta blocks are written contiguously after the last data block so that a reader opening
the segment can fetch footer, top-level index and properties in **one** ranged GET of the
tail (the manifest records `meta_offset` and the tail length per segment, §4.2, so even the
footer read is not a separate round trip). Filter and index partitions are fetched lazily
and cached like data blocks.

### 1.2 Data blocks

A data block is filled up to *default* 32 KiB of uncompressed payload (a single entry
larger than that gets its own block). Entries are LevelDB-shaped:

```
entry := shared_len varint │ unshared_len varint │ value_len varint
         │ key[shared_len..] │ value
block := entry* │ restart_offset u32 × R │ R u32
```

Every 16th entry (*default* restart interval 16) is a **restart point** stored with
`shared_len = 0`; the restart array allows binary search inside the block before a linear
scan of at most 15 entries. The key is the full encoded key including the 8-byte
`!commit_ts` suffix; the value is the version bytes defined by 01 (format byte, column
tagged row, or the one-byte tombstone). The payload is compressed with LZ4 (`lz4_flex`,
MIT) and stored raw when compression saves less than 12.5 %.

**Why 32 KiB rather than 4 KiB.** RocksDB's 4 KiB default is tuned for local SSD where a
read costs one page. Here a miss costs an object-store GET whose fixed latency dominates
the transfer of a few tens of kilobytes, and whose price is per request; a larger block
amortises both and halves the index size per doubling. The cost is read amplification for
point reads on a warm cache (decompressing 32 KiB to return one row) and coarser cache
residency. 32 KiB is a **target to measure in P0**: the wake drill and a point-read
microbenchmark run at 8, 16, 32 and 64 KiB, and the format records the block size only
implicitly (via handles), so changing the default changes no reader. Rejected: 4 KiB
(GET count on cold scans is 8× higher), 256 KiB (point-read decompression and cache
granularity too coarse for a 32 MB per-database RAM budget, which would hold only 128
blocks).

### 1.3 Two-level index

Each **index partition** is a block of entries `(first_key, last_key, handle)`, one per
data block, prefix-compressed with restart interval 1 (keys are full encoded keys). A
partition closes at *default* 32 KiB. The **top-level index** has one entry per partition:
`(first_key, last_key, index_handle, filter_handle)`. A 64 MB segment has about 2,000 data
blocks; at roughly 100 bytes per index entry its index is about 200 KB in some 6–7
partitions, and the top-level index is well under 1 KB. Keeping the top level tiny is what
lets the open path read it with the footer; a single flat index would put 200 KB on the
wake path for every segment touched. Rejected: a flat index (wake cost), a three-level
index (no segment is large enough to need it at the 64 MB target).

`last_key` is stored per data block (not only `first_key` as LevelDB does) so a range scan
can decide that a block lies wholly inside or outside a bound without fetching it.

### 1.4 Bloom filter

The filter covers **user keys without the timestamp**: for each entry, the encoded key
minus its trailing 8 bytes (`prefix || user key`, which 01 makes prefix-free so the strip is
unambiguous). Consecutive versions of one user key are added once. The filter is
partitioned in step with the index partitions, so a point read fetches only the one filter
partition its key falls into.

Bits per key *default* 10, with k = 7 hash probes (round(10 · ln 2)) derived from one
xxh3-64 by double hashing. The false-positive rate this implies is
`(1 − e^(−k/10))^k = (1 − e^(−0.7))^7 ≈ 0.82 %`, the textbook figure that LevelDB and RocksDB
also quote for 10 bits per key; it is computed, not measured here. Rejected: ribbon or
xor filters (about 30 % smaller at the same rate, but build memory and code size for no P0
gain; a later format version may adopt one), and no filter (every point read on a miss
would touch every overlapping segment's index).

### 1.5 Properties block

A JSON-free fixed record, encoded as a sequence of `(tag u16, len u32, bytes)` fields so
that later versions can add fields readers skip:

| Field | Type | Meaning |
|---|---|---|
| `min_key`, `max_key` | bytes | Smallest and largest encoded key (with suffix) |
| `min_commit_ts`, `max_commit_ts` | u64 | Over all versions in the segment |
| `entry_count` | u64 | Versions, values plus tombstones |
| `tombstone_count` | u64 | Tombstone versions |
| `user_key_count` | u64 | Distinct user keys |
| `level` | u8 | Level the segment was written for |
| `created_epoch` | u64 | Lease epoch of the writer that built it |
| `created_by` | enum | `flush` or `compaction` |
| `schema_version` | u64 | Catalog `schema_version` (05) at flush or compaction start |
| `column_set` | list of `(table_id u32, attnum i16)` | Columns that appear in row values |
| `block_size_target` | u32 | The builder's setting, for the drill reports |
| `prefix_sizes` | list of `(table_id u32, index_id u32, bytes u64, rows u64)` | One entry per key prefix present in the segment: `bytes` is the compressed on-disk size of the data blocks whose keys fall under the prefix (a block spanning prefixes is apportioned by key count), `rows` the versions under it. 05's `relpages`, `reltuples` and `pg_relation_size` sum these across the manifest's segments |

### 1.6 Statistics block

00 §4.3 makes per-segment statistics the planner's only source; `ANALYZE` is a no-op. A
segment holds many tables, so statistics are keyed by `(table_id, attnum)` and computed at
build time over the **newest version of each primary-store key in the segment**, tombstones
excluded (secondary-index ranges contribute only an entry count per index). Per column:

| Statistic | Representation |
|---|---|
| `row_count` | u64 rows that carried this table |
| `null_frac` | f32 |
| `min`, `max` | encoded datum bytes (01's per-type order-preserving encoding) |
| `distinct` | HyperLogLog, precision 12 (4,096 one-byte registers; standard error 1.04/√4096 ≈ 1.6 %) |
| `histogram` | equi-depth, *default* 100 buckets: 101 encoded boundary datums |
| `avg_width` | f32 bytes |

Text and bytea boundaries are truncated to 64 bytes (lower bounds truncated, upper bounds
rounded up), so a wide column cannot inflate the block.

**Merging across segments.** The planner (in `colibri-sql`) keeps a per-table merged
statistic, rebuilt when the manifest changes. Row counts add; `null_frac` is the row-weighted
mean; min and max take the extremes; HLL sketches merge by register-wise maximum, which is
exact for the union; histograms merge by weighted quantiles: each input bucket is treated
as `row_count / 100` rows uniformly spread between its bounds, all bounds are merged in
order with their cumulative weights, and 100 new equi-depth bounds are read off the
combined cumulative distribution. The approximation is admitted: a key updated in an L0
segment and present in an older L1 segment is counted twice until compaction merges them.
The bottom level holds most data in steady state (§3), so the overcount is bounded by the
upper levels' share. Rejected: sampling at query time (a GET on the planning path) and a
separate `ANALYZE` (D15 and 00 §4.3 already removed it).

## 2. Reading a segment and the merge iterator

### 2.1 Opening

`SegmentReader::open` needs the manifest entry only: it issues one ranged GET for
`[meta_offset, len)`, verifies the footer checksum and magic, and parses the top-level
index and properties; statistics are parsed on demand by the planner. Readers are cached
per node, keyed by segment ULID, with their meta bytes charged to the opening database.

### 2.2 Point read

For a key `k` (without timestamp) at `snapshot_ts`: prune segments whose
`[min_key, max_key]` from the manifest excludes `k`; in each remaining segment, binary
search the top-level index, fetch the filter partition and test `k`; on a positive, fetch
the index partition, binary search the data block whose `[first_key, last_key]` contains
the seek key `k || !snapshot_ts`, and seek inside the block via restart points. Because the
suffix is the bitwise NOT of the commit timestamp (01), the seek lands directly on the
newest version with `commit_ts <= snapshot_ts`.

### 2.3 The merge iterator

Reads and compaction both run a **merge iterator** over the active memtable, any frozen
memtable being flushed, and the segments, all seen through one trait, `VersionSource`
(ordered iteration over encoded keys with their timestamps, and point lookup), which
`colibri-segment` defines and the memtable in `colibri-txn` implements (06 §1.1); the
iterator takes `&[&dyn VersionSource]`. Sources are ordered newest to oldest (memtables, then
L0 newest first, then L1, L2, ...). A binary heap orders child iterators by encoded key;
on equal encoded keys (possible only while a frozen memtable and its freshly flushed
segment are both visible) the newer source wins and the other is skipped, the two being
byte-identical. Since the newest version of a user key sorts first, the stream yields per
user key a run of versions from newest to oldest.

The **visibility filter** sits above the merge: for each user key it skips versions with
`commit_ts > snapshot_ts`, takes the first version with `commit_ts <= snapshot_ts`, and
then seeks every child past `user key || 0xFF×8` (the suffix of `commit_ts = 0`, the
largest possible). If the version taken is a tombstone, the key is absent and nothing is
emitted. The filter also consults the manifest's `dropped_ranges` (§4.2), but only for a
reader whose `snapshot_ts >= dropped_at_ts`: for it, a version under a listed prefix with
`commit_ts <= dropped_at_ts` (`<` when `strict`) is treated as absent, and a scan whose
bounds lie wholly under such a prefix returns nothing without touching a segment. Such a
reader's catalog (05 §5.2) has no such table or index anyway, so the mask is a safety net.
A reader whose snapshot predates the drop (a REPEATABLE READ transaction begun before the
DDL commit, whose catalog still names the table) is not masked and reads the versions,
which compaction keeps for it (§3.4). Rejected: masking for every reader, because it would
make such a snapshot see an existing table as empty. A range scan sets lower and upper user-key bounds; children prune whole blocks
using `last_key`, and a scan uses readahead (§6.3). A transaction's own writes merge over
this stream in 04's write buffer, not here.

Compaction uses the same merge with a different filter (§3.4): it sees all versions and
decides which to keep, so snapshot semantics and GC semantics share one ordering.

## 3. Levels and compaction

### 3.1 Shape

**L0** holds flushed memtables; its segments overlap and are searched newest first.
**L1 and below** are leveled: within a level, segments are disjoint and sorted by key, so
one segment per level is consulted for a point read. The size ratio between adjacent
levels is *default* 10. Level targets are computed **dynamically from the bottom**
(RocksDB's `level_compaction_dynamic_level_bytes` shape): the bottom level's target is its
actual size, each level above is one tenth of the level below, and a level whose target
falls below *default* 64 MB (one target segment) is left empty, so a 100 MB database has
L0 plus one level rather than an empty L1 to L4. Rejected: fixed targets from L1 upward
(small databases, the common tenant, would carry most of their data in upper levels and pay
space amplification), and universal (tiered) compaction (lower write amplification, but
worse read fan-out and space amplification, and the vector segment bound of §3.5 is easier
to hold with levels).

Output segments are split at *default* 64 MB and at user-key boundaries (all versions of
one user key stay in one segment, so tombstone reasoning in §3.4 is per segment).

### 3.2 Picking

Each level gets a **score**: for L0, `max(segment_count / 4, bytes / 256 MB)` (four L0
segments, *default*, is the trigger); for Ln, `bytes / target`. The level with the highest
score above 1.0 is compacted; ties go to the level whose oldest segment (smallest
`created` ULID timestamp) is oldest. From L0 all segments are taken together with the
overlapping L1 range; from Ln the input is the segment with the oldest ULID (a round-robin
by age, which spreads rewrites across the key space) plus its overlap in Ln+1. A
compaction whose inputs exceed *default* 1 GB is split by key range.

### 3.3 Rate budget

D14 fixes compaction at 8 MB/s burst per tenant and a node-wide ceiling of 50 % of measured
disk bandwidth, behind foreground work. This is realised as two **token buckets**: a
per-database bucket refilled at 8 MB/s with capacity 8 MB (one second of burst), and a
node bucket refilled at half the disk bandwidth measured at daemon start (a sequential
read/write probe through `Env`, rerun on device change) with one second of capacity. A
compaction charges `bytes read + bytes written` to both before each block I/O and sleeps
on `Env` time when either is empty. Compaction tasks run in the **maintenance** request
class (00 §6), so a busy database's foreground statements preempt them. Compaction debt
(sum of score excess in bytes) is exposed per database. All these values are *targets* to
replace from the P0 drills.

### 3.4 Version dropping and the GC watermark

04 owns the **GC watermark**: min over active snapshot timestamps on the node and the
manifest's `retention.pitr_horizon_ts`. Compaction reads it once at start (a later, higher
watermark is merely unused, never unsafe) and, per user key in its merged input:

1. Keeps every version with `commit_ts > watermark`.
2. Keeps the newest version with `commit_ts <= watermark` (it is what a reader at the
   watermark sees) and drops every older one.
3. If the kept version from step 2 is a **tombstone**, drops it too only when no older
   version of the key can exist below: the output is the bottom level, or no segment in
   any lower level overlaps the user key (checked against the manifest's key ranges and,
   if those overlap, the segments' bloom filters; a bloom positive keeps the tombstone).

This is CockroachDB's and RocksDB's rule. Dropped columns are stripped from row values in
the same pass, which is how `DROP COLUMN` bytes are reclaimed, but only for a column whose
drop committed at or below the watermark (05 §6.3 owns the rule); the column set to strip
arrives in the `ColumnLayout` the caller in `colibri-db` hands the `SegmentBuilder`
(06 §1.1), so compaction never reads the catalog.

**Dropped tables and indexes.** `DROP TABLE`, `TRUNCATE` and `DROP INDEX` (05) write no
tombstones over the dropped key range; the next manifest rewrite after the DDL commit,
whether a flush or a compaction, adds an entry to `dropped_ranges` (§4.2). The source is
the open database's in-memory list of pending drops, which the DDL commit's
`CommitObserver` (04 §6.1) appends to and ledger replay rebuilds from the catalog rows in
the replayed batches, so a crash between the commit and the rewrite loses nothing. Before
the entry exists compaction cannot discard the range, since it reads only the manifest. Its `prefix` is the `TableId` alone for `DROP TABLE` and
`TRUNCATE`, which covers the primary store, every secondary index and the overflow-chunk
side range `IndexId = 0xFFFF_FFFF` (01), or `TableId || IndexId` for `DROP INDEX`. Once the
watermark this compaction read is `>= dropped_at_ts`, compaction drops every version under
the prefix with `commit_ts <= dropped_at_ts` (`< dropped_at_ts` when the entry is `strict`,
`TRUNCATE`'s form, 05 §7.4) before applying steps 1–3 and emits nothing in its place, at
any level. While the watermark is below `dropped_at_ts`, a snapshot taken before the drop
may still read the table (§2.3), so the versions follow steps 1–3 like any others.
Rejected: discarding regardless of the watermark (the earlier rule), because a REPEATABLE
READ snapshot older than the DDL commit would lose rows its catalog still names. Versions
under the prefix with a later `commit_ts` (a reused prefix is impossible because OIDs are
never reused, 05, but the rule does not rely on it) follow steps 1–3. Once the watermark
allows the discard, compaction picking raises the score of any level holding a segment
that overlaps a dropped prefix, so the bytes are reclaimed promptly. An entry is removed from `dropped_ranges` by the first
manifest write after which no segment in any level overlaps its prefix (checked against
the manifest's key bounds). Rejected: a range tombstone per drop (RocksDB's
`DeleteRange`), because it would add a second tombstone kind to every read path and to
01's format for a case the manifest can carry in one line; and per-key tombstones (a
`DROP TABLE` would cost as much I/O as the table).

### 3.5 Vector segments (P2 policy input)

00 §4.4 *targets* at most 8 segments carrying vectors in steady state. In P2 the picker adds
a term: when the count of segments with a `vec` entry exceeds 8, the score of the level
holding the most such segments is raised above 1.0. P0 carries the `vec` field in the
manifest (absent) and nothing else; the graph format is P2.

### 3.6 Installing the result

Output segments are written to the bucket first, each completely (multipart upload,
*default* 8 MiB parts). Only then is a new manifest written (§4) that removes the inputs and
adds the outputs in one swap. A crash before the swap leaves orphan outputs that the sweep
(§9) removes; the inputs remain referenced by retained manifests until retention expires,
and readers on the node holding the old manifest keep reading them (§9.3).

## 4. The manifest

### 4.1 Names and the `CURRENT` pointer

Each manifest version is an immutable object `tenants/<db>/manifest/<epoch>.<seq>.json`,
with `epoch` and `seq` rendered as 20-digit zero-padded decimals so lexicographic order is
numeric order. `seq` starts at 1 for each epoch and increments per manifest write.

**Decision: a `manifest/CURRENT` object holds the name of the latest manifest, and the
conditional PUT is on `CURRENT`.** A commit of the manifest is two PUTs:

1. PUT the new body at its new name with `PutMode::Create` (`If-None-Match: *`). A failure
   here means another writer of the same epoch and seq exists, which is a bug; the daemon
   closes the database.
2. PUT `CURRENT` (a small JSON `{"name": "...", "epoch": n}`) with
   `PutMode::Update(etag)` (`If-Match`), where `etag` is the version of `CURRENT` the writer
   last wrote or read at open. This is the **commit point** and the **fence**.

The reason is that 00 §4.6's fence needs one object that every writer contends on. Each
manifest version has a distinct name, so a conditional create on a new name cannot fence
two writers of different epochs: both creates succeed. A single mutable `CURRENT` makes the
`If-Match` meaningful, and it is SlateDB's pattern (D10). A reader finds the latest
manifest with two GETs (`CURRENT`, then the body), never a LIST. Rejected: **listing** with
a sortable name (LIST is paginated, costs per call and grows with retention, gives no
fence, and a LIST racing a PUT may miss it on stores whose listing lags writes); and
**`CURRENT` holding the whole manifest** (one PUT instead of two, but retention and PITR
need immutable history, so every manifest would be written twice anyway, and a
hundreds-of-kilobytes conditional PUT on the commit path is slower to retry). An orphan
body from a failed step 2 is unreferenced and the sweep removes it (§9.4).

### 4.2 Schema

| Field | Type | Meaning |
|---|---|---|
| `format` | u32 | Manifest format version, `1` |
| `db_id` | ULID string | The database |
| `epoch` | u64 | Lease epoch of the writer |
| `seq` | u64 | Sequence within the epoch |
| `written_at_ms` | u64 | `Env` wall clock at write, informational |
| `prev` | string or null | Name of the previous manifest of this database |
| `ledger_position` | `{epoch: u64, lsn: u64}` | Every record at or below this LSN is in the segments named here; replay starts above it (03) |
| `max_commit_ts` | u64 | Highest `commit_ts` in any named segment; seeds the HLC (brief) |
| `schema_version` | u64 | Catalog version (05) consistent with the segments |
| `catalog_format_version` | u32 | Format of the catalog rows (05 §1, §13); checked at open beside `schema_version` |
| `durability` | `{mode, history[]}` | `mode` ∈ `quorum`, `bucket-direct`, `single`; `history` lists `{mode, from_lsn, from_ts, to_lsn?, to_ts?}` intervals, newest last, trimmed with retention |
| `levels` | array of `{level: u8, segments: [Segment]}` | L0 newest first; Ln sorted by `min_key` |
| `hot` | string or null | Name of the hot-set object (§7) |
| `retention` | `{pitr_horizon_ts: u64, keep_manifests_after_ts: u64, window_ms: u64}` | §4.4 |
| `branch` | `{parent_db, parent_manifest, parent_ledger_position{epoch,lsn}, as_of_ts?}` or null | §8 |
| `dropped_ranges` | array of `{prefix: hex string, dropped_at_ts: u64, strict: bool}` | Key prefixes dropped by DDL (05). A prefix is either `TableId` alone (4 bytes as 8 hex digits: the whole table, every index and the overflow-chunk side range `IndexId = 0xFFFF_FFFF` of 01 included, which is what `DROP TABLE` and `TRUNCATE` write) or `TableId || IndexId` (8 bytes as 16 hex digits, what `DROP INDEX` writes). Masked for reads and discarded by compaction at or below `dropped_at_ts` (§3.4); `strict` is `TRUNCATE`'s read-time rule (05 §7.4) |
| `next_segment_hint` | null | Reserved |

`Segment`:

| Field | Type | Meaning |
|---|---|---|
| `db` | ULID string | The database whose prefix holds the object; differs from `db_id` for inherited segments |
| `id` | ULID string | Object is `tenants/<db>/seg/<id>.sst` |
| `min_key`, `max_key` | base64url | User-key bounds (suffix stripped), truncated to 48 bytes: min truncated, max rounded up to the next prefix |
| `min_ts`, `max_ts` | u64 | Commit timestamp range |
| `bytes` | u64 | Object length |
| `meta_offset` | u64 | Start of the meta region (§1.1) |
| `rows` | u64 | Entry count |
| `tombstones` | u64 | Tombstone count |
| `vec` | object, absent in P0 | P2: `{id, bytes}` of the `vec/<ulid>.hnsw` graph |

Fields are serialised with `serde_json`; unknown fields are rejected by readers of the same
`format` and that number is bumped on any addition, so a reader never silently ignores a
field it should honour.

### 4.3 Example

```json
{
  "format": 1,
  "db_id": "01JA7Q4M6Z3T8C2X9V5N1B0K7H",
  "epoch": 7,
  "seq": 42,
  "written_at_ms": 1791700000000,
  "prev": "manifest/00000000000000000007.00000000000000000041.json",
  "ledger_position": { "epoch": 7, "lsn": 918273 },
  "max_commit_ts": 117420000000000123,
  "schema_version": 19,
  "catalog_format_version": 1,
  "durability": {
    "mode": "quorum",
    "history": [
      { "mode": "quorum", "from_lsn": 1, "from_ts": 117300000000000000,
        "to_lsn": 800100, "to_ts": 117380000000000000 },
      { "mode": "bucket-direct", "from_lsn": 800101, "from_ts": 117380000000000001,
        "to_lsn": 800950, "to_ts": 117381000000000000 },
      { "mode": "quorum", "from_lsn": 800951, "from_ts": 117381000000000001 }
    ]
  },
  "levels": [
    { "level": 0, "segments": [
      { "db": "01JA7Q4M6Z3T8C2X9V5N1B0K7H", "id": "01JA9ZB0P1C2D3E4F5G6H7J8K9",
        "min_key": "AABAAAAAAAAB", "max_key": "AABAAAAAAAAC",
        "min_ts": 117419000000000000, "max_ts": 117420000000000123,
        "bytes": 21733120, "meta_offset": 21480011, "rows": 190233, "tombstones": 1204 } ] },
    { "level": 1, "segments": [
      { "db": "01J9X0PARENT0000000000000Q", "id": "01J9X1A2B3C4D5E6F7G8H9J0K1",
        "min_key": "AAAAAAQAAAAA", "max_key": "AABAAAAAAAAD",
        "min_ts": 117300000000000000, "max_ts": 117350000000000000,
        "bytes": 67100000, "meta_offset": 66420000, "rows": 610000, "tombstones": 0 } ] }
  ],
  "hot": "hot/00000000000000000007.00000000000000000040.json",
  "retention": { "pitr_horizon_ts": 116900000000000000,
                 "keep_manifests_after_ts": 116900000000000000, "window_ms": 604800000 },
  "branch": { "parent_db": "01J9X0PARENT0000000000000Q",
              "parent_manifest": "manifest/00000000000000000003.00000000000000000017.json",
              "parent_ledger_position": { "epoch": 3, "lsn": 55120 } },
  "dropped_ranges": [
    { "prefix": "00004002", "dropped_at_ts": 117410000000000007, "strict": false },
    { "prefix": "0000400700004009", "dropped_at_ts": 117415000000000002, "strict": false }
  ],
  "next_segment_hint": null
}
```

The first `dropped_ranges` entry is a `DROP TABLE` of table `0x4002`: its 8-hex-digit
prefix covers the primary store, every index and the overflow range `IndexId =
0xFFFF_FFFF`. The second is a `DROP INDEX` of index `0x4009` on table `0x4007`.

Two earlier fields are gone. `catalog_root: null` is replaced by `catalog_format_version`:
the reserved root had no use (catalog rows live in the KV space, 05), while 05 needs the
format version in the manifest. `log_objects` is removed: recovery of a `bucket-direct`
database finds degraded log objects by LISTing `tenants/<db>/log/` above `ledger_position`
(03 §7.4) and the sweep deletes covered ones (§9.4), so the list duplicated what the bucket
already says and had to be kept in step with it.

**Size at 1,000 segments.** A segment entry is about 330 bytes of JSON (two 26-character
ULIDs, two keys of up to 64 base64 characters, seven integers of up to 20 digits, field
names). 1,000 segments therefore give roughly 330 KB, plus under 2 KB of fixed fields and
history. This refines 00 §4.1's "kilobytes at a thousand segments": it is hundreds of
kilobytes, still one GET on wake. The leveled shape keeps a 64 GB database near 1,000
segments; a typical tenant (100 MB, G2) has fewer than 10, about 5 KB. Not measured; a
compact binary manifest is an open item if the wake drill shows the parse on the path.

### 4.4 Retention

The writer sets `retention.window_ms` from the plan (the CP passes it at open) and, at each
manifest write, `pitr_horizon_ts = HLC(now − window_ms)` (the 48-bit physical prefix with
counter 0) and `keep_manifests_after_ts = pitr_horizon_ts`. A manifest is **retained** if
its `max_commit_ts > keep_manifests_after_ts`, or it is the newest manifest with
`max_commit_ts <= keep_manifests_after_ts` (the base for a restore exactly at the horizon),
or it is the current one. Retention never shrinks below what a descendant's branch point
needs: a manifest named as `parent_manifest` by a live descendant is retained while that
descendant exists (the CP tree, §8, answers this).

### 4.5 Fencing

At open, after the CP grants the lease (00 §4.6), the daemon GETs `CURRENT`, remembers its
ETag, GETs the body, and checks that `epoch` in `CURRENT` is lower than its own lease epoch
(equal would mean a second holder of the same epoch; it refuses to open). Its first act is
a manifest write under its own epoch (`seq = 1`, same segments, **unchanged**
`ledger_position`: 03 §5.2 step 8 leaves the recovered tail in the memtable, not yet
flushed; the write is 03 §5.2 step 10), which moves `CURRENT` to the new epoch and fences
every older writer. On every later write it uses the ETag returned by its own previous
`CURRENT` PUT. `seq` counts every manifest body written under the epoch, so the first
flush after open writes `seq = 2`.

**A database that has never been opened has no `CURRENT`.** The GET returns not-found,
and the daemon writes an initial manifest (`manifest/<epoch>.1.json`: no levels, no
segments, `ledger_position = {epoch, lsn: 0}`, `max_commit_ts = 0`, `schema_version = 0`,
`durability` set from the peer set the lease named, no `branch`) and then PUTs `CURRENT`
with `PutMode::Create`, which succeeds only if no `CURRENT` exists. Two daemons racing for
the first open with different epochs therefore cannot both win: the loser's create fails,
it re-reads `CURRENT`, and the ordinary rule applies (a higher epoch there means it was
never the holder; it closes). The initial manifest **is** the first write under the
opener's epoch: on a fresh database 03 §5.2 step 10 is skipped, because the fence already
carries the epoch, and the next body written is `seq = 2`. Only after `CURRENT` exists does
03's recovery run (it finds an empty tail) and 05's bootstrap write the first transaction. Rejected: letting the CP
create the first manifest at `CREATE DATABASE` time, because the CP then needs bucket
credentials and a second code path for a write the daemon already knows how to make.

**A `412 Precondition Failed` on `CURRENT` means the lease is lost**: the daemon stops
accepting writes, aborts in-flight transactions (04 §6.2: the committing session gets
`08007`, every other session `57P01`), drops its memtable without flushing, and closes the database; the
segments it wrote become orphans for the sweep. It never retries with a fresh ETag, since
a fresh ETag would overwrite the new holder's commit. Transport errors whose outcome is
unknown are resolved by a GET of `CURRENT`: if it names the manifest just written, the PUT
succeeded and the returned ETag is adopted; otherwise the same rule applies.

## 5. Flush

A flush turns the active memtable into an L0 segment:

1. **Freeze.** The commit path swaps in a new empty memtable; the frozen one becomes
   read-only and stays visible to readers through the merge iterator. Every memtable
   tracks `max_lsn`, the highest lsn of the batches applied to it; the frozen memtable's
   `max_lsn` at the cut, or the previous manifest's `ledger_position.lsn` if that is
   higher, is `flushed_lsn` (the floor keeps the position non-decreasing when a forced
   flush finds an empty memtable, or after recovery, whose `Noop` is never applied to the
   memtable, 03 §5.2); its highest `commit_ts` bounds the new
   `max_commit_ts`. The flush never waits for the ledger quorum: the manifest PUT itself
   makes the memtable's contents durable, so `flushed_lsn` may exceed what peers have
   acknowledged (only `synchronous_commit = off` commits can be in that gap). This is what
   lets the forced flush at degraded entry run when no quorum exists (03 §8). Rejected:
   waiting for the quorum to acknowledge everything in the memtable, which deadlocks
   exactly at degraded entry.
2. **Build and PUT** the segment (multipart above 8 MiB) under `seg/<ulid>.sst`.
   Statistics, bloom and properties are computed in the same pass.
3. **Manifest write**: prepend the segment to L0, set `ledger_position = {epoch,
   flushed_lsn}`, `max_commit_ts`, `schema_version`, and commit with §4.1.
4. **Release**: drop the frozen memtable and report `flushed_lsn` to the ledger (03), which
   truncates at or below it. In degraded mode the log objects now covered by
   `ledger_position` become deletable by the sweep (§9.4); the flush deletes nothing.

Flushes of one database are sequential; a flush running when the next trigger fires
delays it (the memtable grows; the memory budget, 00 §6, applies back-pressure on the
writer token if it overruns). Triggers: *target* 64 MB or 30 s (00 §4.2), plus **forced
flushes** at every durability transition (03), at the idle transition (00 §5, together with
the hot set, §7), on memory pressure (00 §6: "flushed and shrunk, never killed"), and before
a graceful lease hand-off. A forced flush of an empty memtable still writes a manifest when
its durability or hot fields change.

**Crash between steps 2 and 3** leaves a segment no manifest names; it is an orphan, the
recovering writer replays the ledger from the old `ledger_position` and flushes again into
a new segment, and the sweep deletes the orphan after the grace period. **Crash after
step 3, before step 4**, leaves ledger records already in a segment; the recovering writer
starts replay above the manifest's `ledger_position`, so they are not applied twice.

## 6. Block cache

The cache is two tiers sharing one key: `(segment ULID, block offset)`. Because ULIDs are
global, a parent and its branches on one node share cached blocks of shared segments.

### 6.1 RAM tier

Each open database owns a RAM tier with the D14 budget (*default* 32 MB, 256 MB on Pro),
charged to the database's memory account (00 §6). It holds **decompressed, verified**
blocks (data, index and filter partitions), charged at decompressed size plus a fixed
per-entry overhead.

Admission and eviction: **segmented LRU** (*default*), with a probationary segment of 20 %
and a protected segment of 80 %. A new block enters probation; a second hit promotes it to
protected; eviction takes from the probationary tail first. Blocks fetched by scan
readahead (§6.3) enter probation and are never promoted by the scan that fetched them, so a
large scan cannot flush the point-read working set. Rejected as the default:
**W-TinyLFU** (moka-style; better hit ratio in published traces, but a frequency sketch and
window per database, multiplied by 1,000 open databases, against a 2 MB per-tenant overhead
target, and a policy whose behaviour the drills must first show is needed); **plain LRU**
(one sequential scan evicts everything); **LRU with a ghost list (ARC/2Q-shaped)** (closer
to SLRU's benefit with more bookkeeping). The policy sits behind one trait so the drill can
swap it.

When a database goes idle its RAM tier is dropped (after the hot set is recorded, §7);
blocks remain in the NVMe tier.

### 6.2 NVMe tier

One per node, shared by all databases, sized at *default* 80 % of the cache volume. The
remaining 20 % is the node's **spill reservation** on `DiskRoot::Spill` (06 §11.1,
*default* `<cache-dir>/spill`; filesystem headroom is taken out of the 80 %, not the
reservation); the reservation is shared by 04's
transaction spill, `CREATE INDEX` backfill and `WITH HOLD` cursors, and a transaction whose
spill would exceed the free reservation fails with `53100` *disk_full* (04 §3.5). It stores blocks **as they are in
the bucket** (compressed payload plus trailer), so the xxh3 in the trailer is **rechecked
on every read**; a mismatch deletes the entry, increments a corruption counter and
refetches from the bucket.

Directory layout, under `env.disk(DiskRoot::Cache)` (06 §11.1):

```
<cache_root>/v1/<ulid[0..2]>/<ulid>.blk     sparse file, block bytes at their segment offset
<cache_root>/v1/<ulid[0..2]>/<ulid>.map     append-only (offset u64, len u32) records
```

A per-segment sparse file avoids one file per block (an 800 GB tier holds about 25 million
32 KiB blocks, too many inodes). The `.map` sidecar records present blocks; it is replayed
at daemon start and trusted only as a hint, since the checksum recheck catches a block
whose bytes are missing or stale. Eviction of one block punches a hole
(`DiskFile::punch_hole(offset, len)`, `fallocate(PUNCH_HOLE)` on Linux) and appends a tombstone record to the map;
where hole punching is unavailable, eviction falls back to whole-segment files. The map is
compacted when its tombstones exceed half its records.

Eviction is LRU **with per-database fair share**: every database (open or idle) with
blocks on the tier has a share `tier_size × w_db / Σ w`, with weight `w` from the plan
(1 by default, the Pro multiplier otherwise). Under pressure the victim is the LRU block of
the database furthest above its share; a database under its share is never evicted for
another's benefit. A shared segment's blocks are charged to the database that fetched them
first. When a database's NVMe bytes reach zero it is eligible for the cold transition
(00 §5: "→ cold on cache pressure").

Rejected: one LMDB/RocksDB-style embedded store (the brief excludes embedded databases),
and file per block (inodes, directory size).

### 6.3 Miss path

On a RAM miss the NVMe tier is consulted; on an NVMe miss the block is fetched with a
**ranged GET** (`bytes=offset..offset+size+9`). Concurrent misses on the same key share one
in-flight request (a single-flight map). Adjacent misses are **coalesced**: when a reader
asks for blocks whose byte ranges are contiguous or separated by less than *default*
64 KiB, one GET covers them. Readahead is *default* **4 blocks for scans** (the next four
blocks of the same index partition are requested with the current one) and **0 for point
reads**. Fetched blocks are written to NVMe asynchronously and inserted in RAM after
decompression and checksum verification.

### 6.4 Metrics

Exposed per database over the control socket and as Prometheus series (00 §6): RAM bytes
and budget, NVMe bytes and fair share, hits and misses per tier, GET count and bytes,
coalesced GETs, corruption count, and **per-segment hit ratio** (an exponentially decayed
hits/(hits+misses), half-life *default* 60 s) which the planner reads as D15's cache-miss
term. Node-wide: NVMe tier bytes, eviction rate, in-flight GETs.

## 7. The hot set

At the idle transition, before the RAM tier is dropped, the database walks the RAM tier's
recency order (protected segment first, most recent first) and records blocks until
*default* 8 MB of on-bucket bytes. It writes `tenants/<db>/hot/<epoch>.<seq>.json`, where
`<epoch>.<seq>` equals the manifest about to be written:

```json
{ "format": 1, "manifest": "manifest/00000000000000000007.00000000000000000042.json",
  "blocks": [ { "db": "01JA7Q...", "segment": "01JA9ZB0...", "offset": 1048576, "length": 32777 } ] }
```

Then the idle flush writes the manifest with `hot` pointing to it. A hot object written but
not committed is an orphan for the sweep.

On **wake** (00 §5), after the manifest GET, the daemon GETs the hot object, discards
entries whose segment the manifest no longer names, coalesces adjacent entries (§6.3), and
issues the GETs with *target* 16 in parallel into the NVMe and RAM tiers; the database opens
for queries without waiting for the prefetch to finish (a query racing it joins the
in-flight request). 16 is a target to measure in the wake drill alongside the G2 latency.
If a listed segment no longer exists (a 404 because GC removed it after a later manifest
dropped it, or the hot object predates a compaction), the entry is skipped and a counter
increments; a missing hot object is treated as an empty set. Neither is an error: the hot
set is an optimisation and never consulted for correctness.

## 8. Branches

A branch (P2 for the product surface; the format is fixed now) is a new `DbId` whose
**first manifest** copies the parent manifest's `levels` with every `Segment.db` naming the
owning prefix (the parent's, or further up for inherited segments), sets
`branch = {parent_db, parent_manifest, parent_ledger_position}` to the parent's chosen
manifest and its ledger position, `ledger_position = {branch epoch, 0}` on a fresh LSN
space of its own, `prev = null`, and copies `max_commit_ts`, `schema_version` and
retention. Creating it is one manifest write plus one CP row: O(1) in database size (G6).
A branch taken at the parent's current state rather than a past manifest first forces a
parent flush so that the unflushed tail is in segments.

The branch's own flushes and compaction outputs go under `tenants/<branch>/seg/`.
Compaction that rewrites inherited segments produces own segments and drops the inherited
references, so a branch gradually stops depending on its parent.

The **CP records the branch tree**: for each child, `(parent_db, parent_manifest,
created_at)`, kept until the child is deleted. A dropped database whose segments are still
referenced by descendants stays in the tree as a tombstoned node so its prefix keeps being
swept (§9).

## 9. Segment GC across branches

### 9.1 The sweep

For a database X:

1. **List** `tenants/X/seg/` and record each object's `LastModified` as reported by the
   store (the store's clock, not a node's).
2. **Referenced set** = every `Segment` with `db = X` named by (a) every **retained**
   manifest of X (§4.4, found by listing `tenants/X/manifest/`, a maintenance-path LIST
   only), and (b) every retained manifest of every **descendant** of X, transitively, with
   the descendant list asked from the CP. The brief names "the current manifest of every
   descendant"; retained manifests are a superset and are required, because a descendant's
   PITR inside its own retention may restore a manifest that still names X's segments.
3. **Delete** every listed segment not in the referenced set and whose `LastModified` is
   older than the **grace period** (*default* 1 h). Deletions are idempotent.

There are no reference counters in the bucket (00 §4.7, brief).

### 9.2 Who runs it

The daemon holding X's lease runs the sweep for X on the maintenance class, *default* once
per hour and after a compaction that dropped at least *default* 256 MB of inputs, with its
LIST and DELETE traffic charged to X's compaction token bucket. For a **cold** database,
the CP nominates a node (the least loaded in the region) to run the sweep without opening
the database; no lease is taken, since the sweep writes nothing but deletes. The CP
schedules cold sweeps only while retention or descendants can still free bytes.

### 9.3 Races and why the grace period closes them

**In-flight flush or compaction.** A segment is PUT before the manifest naming it (§5,
§3.6). Between the two it is listed but unreferenced. Every such window is shorter than the
grace period: flush and compaction are bounded by the 64 MB segment size, the 1 GB
compaction split and the rate budget, and the daemon **aborts** any flush or compaction
whose wall time exceeds half the grace period (it discards its outputs and retries). A
segment younger than the grace period is never deleted, so the manifest swap cannot lose
it. A wake on another node during a cold sweep is the same case: its new segments are
young; it cannot resurrect old ones, because compaction only removes references.

**Readers on an old manifest.** Snapshots on the node may still read compaction inputs
that the current manifest dropped (D3). Those inputs are named by retained manifests as
long as the reader's snapshot is above the PITR horizon, which the GC watermark guarantees
(04 never lets the horizon pass an active snapshot). Additionally, the lease-holding daemon
adds to the referenced set every segment pinned by an open `SegmentReader`.

**A descendant created during the sweep.** It copies the parent's current manifest, whose
segments are all referenced by (a) already.

### 9.4 Other objects

The same sweep deletes, beyond retention and the grace period: manifest bodies not
retained (never `CURRENT`'s target, never a live `parent_manifest`); orphan manifest bodies
from failed `CURRENT` swaps; hot objects not named by any retained manifest; and degraded
log objects `tenants/X/log/<epoch>.<from>.<to>` whose `to` LSN is at or below the current
manifest's `ledger_position.lsn` and whose `LastModified` is older than the grace period
(03 owns their format and does not delete them itself). The earlier extra condition, "that
no retained manifest lists in `log_objects`", went with that field (§4.3). In P2, `vec/` graphs follow their
segments.

Dropped tables and indexes need nothing from the sweep directly: their bytes leave the
bucket when compaction (§3.4) rewrites the segments that held them and those inputs fall
out of retention; a `dropped_ranges` entry only leaves the manifest once no segment overlaps
its prefix, so a retained manifest that still names such a segment also still carries the
mask that hides it.

**Never deleted:** a segment, manifest or hot object referenced by any manifest inside the
retention of X or of any descendant of X, and anything younger than the grace period.

## 10. PITR at the storage level

PITR is P2 as a product surface; the mechanics follow from §4.4 and §8. Restoring database
X to time T (a `CommitTs` from the wall-clock instant, brief):

1. **Pick** the newest retained manifest M of X with `max_commit_ts <= T`. If T is below
   `pitr_horizon_ts`, the request is refused (outside retention).
2. **Branch** from M (§8), recording `as_of_ts = T` in `branch`.
3. **Replay** the versions with `commit_ts` in `(M.max_commit_ts, T]` into the branch and
   flush. 04 and 03 own the replay. Because the ledger is truncated at `flushed_lsn` (03),
   those versions are normally no longer in the ledger; storage supplies them as a merge
   iterator over the segments of the **next** retained manifest after M (whose L0 contains
   everything flushed after M, and whose compactions kept every version above the
   watermark, §3.4), filtered by `commit_ts <= T`, then the unflushed ledger tail or
   degraded log objects for whatever lies beyond the last flush.

The result is a new database whose first own manifest is consistent at T. The parent is not
modified; the branch shares M's segments.

## 11. Rust type sketches

Sketches, not code; names fix the contract between `colibri-segment`, `colibri-manifest`
and their callers.

```rust
pub struct SegmentId(pub Ulid);                       // seg/<ulid>.sst
pub struct SegmentRef { pub db: DbId, pub id: SegmentId }

#[derive(Copy, Clone)]
pub struct BlockHandle { pub offset: u64, pub size: u32 } // size excludes 9-byte trailer

pub struct SegmentMeta {                               // the manifest's Segment entry
    pub r: SegmentRef, pub min_key: Bytes, pub max_key: Bytes,
    pub min_ts: CommitTs, pub max_ts: CommitTs,
    pub bytes: u64, pub meta_offset: u64, pub rows: u64, pub tombstones: u64,
}

pub struct SegmentReader { /* footer, top index, properties; Arc-shared per node */ }
impl SegmentReader {
    pub async fn open(meta: &SegmentMeta, cache: &BlockCache, db: DbId) -> Result<Self>;
    pub async fn get(&self, user_key: &[u8], snap: SnapshotTs) -> Result<Option<Version>>;
    pub fn iter(&self, range: KeyRange, readahead: u8) -> SegmentIter;
    pub fn may_contain(&self, user_key: &[u8]) -> MaybeFuture<bool>;   // bloom
    pub fn statistics(&self) -> MaybeFuture<Arc<SegmentStats>>;
}

pub struct SegmentBuilder { /* restart interval, block size, bloom bits, stats sinks */ }
impl SegmentBuilder {
    /// `layout`: column ids, types and the dropped-below-watermark set, supplied by
    /// colibri-db (06 §1.1), so segments and compaction never read the catalog.
    pub fn new(id: SegmentId, level: u8, cfg: BuildConfig, layout: ColumnLayout) -> Self;
    pub fn add(&mut self, key: &[u8], value: &[u8]) -> Result<()>;    // strictly increasing
    pub async fn finish(self, store: &dyn ObjectStore, db: DbId) -> Result<SegmentMeta>;
}

pub struct Manifest {                                  // serde mirror of §4.2
    pub format: u32, pub db_id: DbId, pub epoch: Epoch, pub seq: u64,
    pub prev: Option<String>, pub ledger_position: LedgerPos,
    pub max_commit_ts: CommitTs, pub schema_version: u64, pub catalog_format_version: u32,
    pub durability: Durability, pub levels: Vec<Level>,
    pub hot: Option<String>, pub retention: Retention,
    pub branch: Option<BranchOf>,
    pub dropped_ranges: Vec<DroppedRange>,
}
pub struct DroppedRange {
    pub prefix: DroppedPrefix, pub dropped_at_ts: CommitTs, pub strict: bool,
}
pub enum DroppedPrefix { Table(TableId), Index(TableId, IndexId) } // 8 or 16 hex digits

/// Implemented by segment iterators here and by the memtable in colibri-txn (06 §1.1).
pub trait VersionSource: Send + Sync {
    fn iter(&self, range: KeyRange) -> Box<dyn VersionIter + '_>;  // encoded key + ts order
    fn get(&self, user_key: &[u8], snap: SnapshotTs) -> MaybeFuture<Option<Version>>;
}
pub fn merge<'a>(sources: &'a [&'a dyn VersionSource], range: KeyRange) -> MergeIter<'a>;

pub struct ManifestStore { db: DbId, epoch: Epoch, current_etag: Option<ETag> }
impl ManifestStore {
    pub async fn load_current(&mut self) -> Result<(Manifest, ETag)>;
    /// Create the body, then CAS CURRENT. Err(Fenced) => lease lost, close the database.
    pub async fn put_if_match(&mut self, m: &Manifest) -> Result<(), ManifestError>;
    pub async fn list_retained(&self) -> Result<Vec<String>>;   // GC and PITR only
}
pub enum ManifestError { Fenced, Conflict, Io(EnvError) }

pub struct BlockCache { /* per-db SLRU RAM tiers + node NVMe tier + single-flight map */ }
impl BlockCache {
    pub async fn get(&self, db: DbId, seg: &SegmentRef, h: BlockHandle, hint: ReadHint)
        -> Result<Arc<Block>>;
    pub fn hot_list(&self, db: DbId, max_bytes: u64) -> Vec<HotEntry>;
    pub async fn prefetch(&self, db: DbId, entries: &[HotEntry], parallel: usize);
    pub fn drop_ram_tier(&self, db: DbId);
    pub fn stats(&self, db: DbId) -> CacheStats;
}
```

The crate calls 06's `Env` (06 §11) and defines no environment trait of its own. Object
store: `ObjectStore::get_range` (the ranged GET of §1.1, §6.3), `get`, `put` with
`PutMode::{Create, Update, Overwrite}`, `put_multipart` (parts *default* 8 MiB, §3.6 and
§5), `list` (with `last_modified`, §9.1) and `delete`. Local disk: the NVMe tier opens
`env.disk(DiskRoot::Cache)` and uses `DiskFile::read_at`, `write_at`, `append`, `sync` and
`punch_hole(offset, len)`, plus `Disk::remove`; `Rng` supplies ULID randomness and the
`Env` clock the decayed hit ratio. It never touches `std::fs` or tokio's I/O traits
(`futures::io` where streams appear).

## 12. Test plan

All tests are public (D18); seeds for every failure found go to the private corpus.

| Test | Kind | What it proves |
|---|---|---|
| Segment round trip | property test | Random sorted key/value sets (including empty values, tombstones, single entries over 32 KiB, keys sharing long prefixes) built and read back by point get and full and bounded scans give the input exactly; at restart intervals 1, 16, 128 and block sizes 4–64 KiB |
| Corruption detection | property test | Flipping any single byte of a segment makes open or the read of the affected block fail with a checksum error, never a wrong answer; truncating the object fails at open |
| Bloom rate | unit | Measured false-positive rate at 10 bits per key over 1 M absent keys within 0.5–1.2 % (checks the implementation against the 0.82 % computation) |
| Merge and visibility | model test | A `BTreeMap` model of versions versus memtable plus segments across levels at random snapshots; tombstones hide older versions; equal keys from frozen memtable and segment dedupe |
| Compaction GC | model test | After compaction at watermark W, every read at any snapshot ≥ W returns what it returned before; tombstones survive while a lower level holds the key; a table dropped at D stays readable at a snapshot below D until the watermark reaches D, then its versions are discarded |
| Statistics merge | unit | HLL union equals the sketch of the union; merged histogram quantiles within one bucket width of the exact quantiles on synthetic distributions |
| Manifest fencing | simulation (06) | Two daemons with epochs 7 and 8 interleave flushes under seeded delays, lost responses and crashes; exactly one sequence of `CURRENT` values exists, the epoch-7 daemon closes on its first 412, no acknowledged flush of epoch 8 is lost, and an unknown-outcome PUT is resolved by the GET rule |
| GC with branches | simulation | Parent, child, grandchild; flush and compaction PUTs delayed up to the abort bound while sweeps run on the lease holder and on a nominated node; invariant: every segment named by any retained manifest of any database in the tree exists at every step |
| Orphans | simulation | Crashes between segment PUT and manifest PUT, and between body PUT and `CURRENT` PUT, leave only objects the next sweep after the grace period removes |
| Cache accounting | unit and sim | RAM bytes per database never exceed budget; NVMe fair share respected under two competing databases; a scan does not evict the protected segment; corrupt NVMe bytes are detected and refetched; single-flight issues one GET for N concurrent misses |
| Hot set | sim | Wake with a hot object naming deleted segments opens and counts skips; without a hot object opens |
| Store conformance | integration | `put_if_match`, create and ranged GET against MinIO, S3 and R2 (00 §11 risk 4) before each store is supported |

## 13. What this document does not decide

- **Block size, bloom bits per key, restart interval, index partition size, segment size,
  L0 trigger, size ratio, compaction split.** All *defaults* until the P0 wake, density and
  compaction drills. The format carries them implicitly, so changing them changes no reader.
- **Rate budget values** (8 MB/s per tenant, 50 % of disk node-wide) are D14 targets.
- **`CURRENT` versus listing.** Chosen as `CURRENT` (§4.1); revisited only if the store
  conformance suite finds a supported store whose conditional PUT on a small hot object is
  unreliable, in which case the fallback is listing with reverse-sortable names and a
  different fence.
- **Cache admission policy.** SLRU is the default; W-TinyLFU is evaluated only if the
  density drill's hit ratios fall short.
- **Manifest encoding.** JSON at hundreds of kilobytes for large databases; a binary or
  delta-encoded manifest only if the wake drill shows its parse or GET on the path.
- **Hot-set size and wake parallelism** (8 MB, 16 GETs) are targets for the wake drill.
- **Vector graph file format** (`vec/<ulid>.hnsw`) and the vector compaction term are P2.
- **Filter type** beyond bloom (ribbon, xor) and per-column min/max zone maps per block are
  later format versions.
- **Branch and PITR product surface** (P2), including retention windows per plan and how
  the CP schedules cold sweeps at scale.
