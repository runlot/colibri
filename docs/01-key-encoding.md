# 01 — Key encoding, versions and row values

> Kind: **design**. Status: **pre-implementation** (2026-10-11). Fills the gap left by
> docs/00-overview.md §4.1 (the key layout sentence of D1), §4.3, §4.4 and §7.2 for phase
> P0. It owns the crate `colibri-encoding`: the key space, the order-preserving encoding of
> every phase-1 type, the timestamp suffix, the row value format, tombstones, secondary
> index keys, the hidden `rowid`, and the key shape for non-`C` collations (D20).
> Depended on by 02 (segments store and compare these keys; bloom filters hash the user
> key), 03 (a `TxnBatch` payload is a list of the pairs defined here), 04 (write buffer,
> unique checks, sequences), and 05 (catalog rows, `attnum`, dropped columns, the OID
> ranges reserved here). It depends on 04 §sequences for `rowid` allocation and on 05 for
> the fixed catalog OIDs.

## 1. The key space

### 1.1 Layout

One database is one ordered KV space. Every encoded key has three parts:

```
 0        4        8                                   n-8            n
 ┌────────┬────────┬──────────────────────────────────┬──────────────┐
 │TableId │IndexId │ user key (tuple encoding, §2)    │ !CommitTs    │
 │u32 BE  │u32 BE  │ self-delimiting elements         │ u64 BE, NOT  │
 └────────┴────────┴──────────────────────────────────┴──────────────┘
 └──── prefix ─────┘
 └──────────────── UserKey (what the bloom filter hashes) ─────────────┘ minus the suffix
```

`TableId` is the relation's OID and `IndexId` is the index's OID, both big-endian so that
byte order equals numeric order and every relation and every index is one contiguous
range. `IndexId = 0` is the primary (row) store of the table; the primary-key index that
PostgreSQL would create as a separate relation is reported in `pg_index` under its own OID
(05) but its entries *are* the row store, so that OID never appears as an `IndexId` in a
key. The user key is the tuple encoding (§2) of the key columns. The suffix is the
**bitwise NOT** of the 8-byte big-endian `CommitTs`; it is fixed width, so a decoder splits
the suffix off by length and never has to parse the user key to find it.

In this document *encoded key* means all three parts, *user key* means prefix plus tuple
(everything but the suffix), and *tuple* means the part after the prefix.

### 1.2 Why newest first

`!ts` reverses timestamp order, so for one user key the versions are laid out newest
first. Two consumers depend on that.

**Point lookup at a snapshot.** A reader at `snapshot_ts = S` seeks to
`user_key || !S`. Every version with `commit_ts <= S` has `!commit_ts >= !S` and lies at or
after the seek position; every version newer than `S` lies before it. The first entry at or
after the seek position whose user key matches is therefore the visible version, found in
one seek with no backward step. With ascending timestamps the reader would have to seek
past the newest visible version and step back, which block-based segment iterators (02)
do badly.

**The merge across memtable and segments.** 00 §4.3 resolves a key through the memtable
and then segments newest to oldest. A merging iterator ordered by the full encoded key
yields, per user key, versions in descending `commit_ts` regardless of which source holds
them, so the visibility rule is "the first version with `commit_ts <= S`; skip the rest of
the key" for scans too, and compaction (02) keeps the newest version at or below the GC
watermark plus everything above it by reading each key's run in one pass. Ties across
sources cannot occur: a `(user key, commit_ts)` pair is written by one commit only.

Rejected: ascending timestamps with a reverse iterator (doubles the iterator code in 02
and makes point lookups two-step), and putting the timestamp in the value with one key per
row (that is an update-in-place store, which D1 rules out).

### 1.3 The ordering invariant

For two versions with user keys `u1 != u2` and any timestamps, `u1 < u2` (bytewise) implies
`encode(u1, t1) < encode(u2, t2)`. This holds only if no user key is a proper prefix of
another user key in the same index, because otherwise the shorter key's suffix bytes would
be compared against the longer key's tuple bytes. The tuple encoding guarantees this:
every element is self-delimiting (§2), every index has a fixed element sequence, and the
one case with a variable element count (the NULL rule of unique indexes, §5.2) is decided
by the elements before the variable part, so two keys that agree on the shared elements
agree on the count. `colibrid` asserts the invariant in debug builds on every memtable
insert, and the property tests of §10 check it.

A consequence used by 02 and 04: all versions of one user key are contiguous, and a range
`[lo, hi)` of user keys is the range of encoded keys `[lo, hi)` as well (no suffix needed on
the bounds, since an encoded key `lo || !ts` sorts after `lo`).

### 1.4 Reserved ranges

| Range | Meaning | Owner |
|---|---|---|
| `TableId = 0` | Reserved; never written in P0 (kept for database-level metadata if a later phase needs a key outside any relation) | 01 |
| `TableId` 1 – 16383 | Internal catalog relations and other fixed system objects (00 §4.1 "catalog root") | 05 fixes the table |
| `TableId` ≥ 16384 | User relations and sequences, allocated from `next_oid` | 05 |
| `IndexId = 0` | Primary row store | 01 |
| `IndexId = 0xFFFF_FFFF` | Overflow chunks of large column values (§4.4) | 01 |
| `IndexId` `0xFFFF_FF00` – `0xFFFF_FFFE` | Reserved per-relation side ranges for later phases (e.g. P2 vector index metadata) | 01 |

Because `IndexId` is an OID, the OID allocator of 05 must never hand out a value in
`0xFFFF_FF00 – 0xFFFF_FFFF`; it skips the band. At one OID per DDL object and no reuse this
costs 256 OIDs of a 32-bit space.

## 2. The order-preserving tuple encoding

### 2.1 Principles

A tuple is the concatenation of its elements. Each element starts with a **tag byte** that
names its type class (the type is also known from the catalog; the tag makes keys
self-describing for debugging tools, for fuzzing, and for the NULL rule of §5.2), followed
by a payload whose bytewise order equals PostgreSQL's btree order for that type under its
default operator class. Comparing two tuples of the same index bytewise therefore compares
them element by element, which is what a multi-column btree does.

Tags live in `0x10 – 0x4F`. Two bytes outside that band are the NULL markers: `0x01` for a
NULL that sorts **first** and `0xFE` for one that sorts **last**. `0x00` and `0xFF` are never
the first byte of an element.

**DESC and NULLS FIRST/LAST.** For a column declared `DESC`, every byte of the non-NULL
element (tag and payload, terminators included) is complemented (`b → 0xFF ^ b`). The NULL
marker is never complemented; it is chosen from the column's NULLS option: `0xFE` for NULLS
LAST and `0x01` for NULLS FIRST, with PostgreSQL's defaults (ASC → NULLS LAST, DESC →
NULLS FIRST). Complemented tags lie in `0xB0 – 0xEF`, so the first byte of an element always
tells NULL from non-NULL in both directions, and NULLS FIRST/LAST is independent of
ASC/DESC exactly as in PostgreSQL. Complementing preserves self-delimitation because the
decoder knows the column's direction from the index definition and complements before
decoding.

Rejected: complementing NULL markers together with the value (ties NULLS placement to
direction, which PostgreSQL does not), and a separate "direction byte" per element (one byte
per element for information the catalog already has).

### 2.2 Variable-length bytes

`text`, `varchar`, `bpchar`, `bytea` and the collated forms of §3 use the FoundationDB /
CockroachDB escape: each `0x00` in the payload is written `0x00 0xFF`, and the payload ends
with the terminator `0x00 0x01`. Under ASC, a string sorts before every extension of it
(`0x00 0x01` < `0x00 0xFF` < any byte `>= 0x01`), and an embedded NUL sorts before every
other byte at that position, which is memcmp order. Under DESC the complemented terminator
is `0xFF 0xFE` and the complemented escape `0xFF 0x00`, and the order reverses correctly.
PostgreSQL `text` cannot contain NUL, so for text the escape never fires; it is kept for
`bytea` and for uniformity.

### 2.3 The type table

All multi-byte integers are big-endian. "Flip" means XOR of the sign bit (`0x80` on the
first byte), which maps two's-complement order onto unsigned order.

| PG type (OID) | Tag | Payload (ASC) | Notes |
|---|---|---|---|
| NULL | `0x01` / `0xFE` | none | Marker replaces tag; §2.1 |
| `bool` (16) | `0x10` false, `0x11` true | none | false < true, as PG |
| `int2` (21) | `0x12` | 2 bytes, flip | |
| `int4` (23) | `0x13` | 4 bytes, flip | |
| `int8` (20) | `0x14` | 8 bytes, flip | Also the hidden `rowid` (§6) |
| `oid` (26) | `0x17` | 4 bytes unsigned, no flip | Catalog keys (05) |
| `float4` (700) | `0x15` | 4 bytes, IEEE fold | §2.4 |
| `float8` (701) | `0x16` | 8 bytes, IEEE fold | §2.4 |
| `numeric` (1700) | `0x18` | class byte, exponent, base-100 digits | §2.5; lossy (scale) |
| `text` (25), `varchar` (1043), `name` (19) | `0x20` | escaped UTF-8 bytes | `C` collation only in P0 |
| `bpchar` (1042) | `0x23` | escaped bytes, trailing spaces removed | PG compares `char(n)` ignoring trailing blanks; lossy |
| `bytea` (17) | `0x21` | escaped bytes | memcmp, as PG |
| `uuid` (2950) | `0x22` | 16 raw bytes | PG's `uuid_cmp` is memcmp |
| collated text (§3) | `0x24`, `0x25` | sort key (+ tie-breaker) | Later phase; format fixed now |
| `date` (1082) | `0x30` | 4 bytes, flip (days since 2000-01-01) | `-infinity`/`infinity` are `INT32_MIN`/`MAX`, sort at the ends as PG |
| `time` (1083) | `0x31` | 8 bytes, flip (µs since midnight) | |
| `timetz` (1266) | `0x32` | 8 bytes flip of `time_us + zone_s × 10^6`, then 4 bytes flip of `zone_s` | PG's `timetz_cmp_internal`: UTC-adjusted time, then zone |
| `timestamp` (1114) | `0x33` | 8 bytes, flip (µs since 2000-01-01) | infinities are `INT64_MIN`/`MAX` |
| `timestamptz` (1184) | `0x34` | 8 bytes, flip (UTC µs) | same |
| `interval` (1186) | `0x35` | 16 bytes, flip of the 128-bit span | §2.6; lossy |
| arrays of the above | `0x40` | element list | §2.7 |

### 2.4 Floats

PostgreSQL orders floats as: `-Infinity` < negatives < `-0 = +0` < positives < `Infinity` <
`NaN`, and all NaNs are equal. The key payload is computed from the IEEE bits `b`:
normalise `-0.0` to `+0.0` and every NaN to the single encoding `0xFF…FF`; otherwise, if
the sign bit is clear, set it (`b | 0x8000…`), and if it is set, complement all bits
(`!b`). Positive values then sort above negative ones, negative magnitudes reverse, and
the NaN encoding sorts above `+Infinity` (`0xFFF0…` after folding). `-0.0` normalisation
makes the key lossy for that one value; §4.3's composite rule keeps the original in the
value.

### 2.5 Numeric

PostgreSQL 18 `numeric` has `NaN`, `+Infinity`, `-Infinity`, and finite values with up to
131072 digits before the point and 16383 after. Equality ignores display scale (`1.0 =
1.00`), so the key encodes the *value* canonically and the scale is kept in the value
(§4.3). Order: `-Infinity` < negatives < zero < positives < `+Infinity` < `NaN`.

After the tag, one **class byte**:

| Class | Meaning | Rest of payload |
|---|---|---|
| `0x02` | `-Infinity` | none |
| `0x03` | negative finite | complemented exponent and digits |
| `0x04` | zero | none |
| `0x05` | positive finite | exponent and digits |
| `0x06` | `+Infinity` | none |
| `0x07` | `NaN` | none |

A finite nonzero magnitude is written `0.d1 d2 … dk × 100^E` with base-100 digits
`d1 != 0`, `dk != 0` (trailing zero digits stripped; leading ones removed by normalisation),
so the representation is unique. For a positive value the payload is `E` as a 4-byte
flipped `i32` (the base-100 exponent reaches ±65544, beyond `i16`), then each digit as the
byte `2·d + 1` (range `0x01 – 0xC7`, never `0x00`), then the terminator `0x00`. For equal
exponents the digit bytes compare as the digits do, and a value that is a digit-prefix of
another is smaller and ends first, which the `0x00` terminator sorts first. A larger
exponent is a larger magnitude because `d1 != 0`. For a negative value every byte after
the class byte (exponent, digits, terminator) is complemented, which reverses magnitude
order; the complemented terminator `0xFF` is greater than any complemented digit byte
(`>= 0x38`), so a longer negative magnitude with the same prefix (a smaller number) sorts
first, as required. Example: `12.5` → `18 05 80 00 00 01 19 65 00` (`E = 1`, digits 12, 50).

This is CockroachDB's decimal key encoding in shape (class, exponent, base-100 mantissa
with `2d+1` bytes); the differences are the 4-byte exponent and the infinity classes PG 18
needs. Rejected: base-10000 digits matching PG's storage (two bytes per digit pair group
with no sentinel-free byte range, more complex terminator), and a varint exponent
(saves three bytes per value at the cost of an order-preserving varint, not worth it for P0).

### 2.6 Interval

PostgreSQL's `interval_cmp_value` compares intervals by one 128-bit span:
`(month × 30 + day) × 86400·10^6 + time_us`, so `'1 mon'` equals `'30 days'` and they are
the same index key. The key payload is that span as a 16-byte flipped two's-complement
integer. PG 18's infinite intervals (all three fields at `INT_MAX`/`INT64_MAX`, or at the
minima) produce the largest and smallest spans and sort at the ends. Because many
`(month, day, time)` triples share one span, `interval` is lossy and the original triple is
kept in the value (§4.3). The "comparable triple" form some engines use (months, days, µs
compared lexicographically) was rejected: it does not equal PostgreSQL's order
(`'1 mon' > '31 days'` lexicographically, but `<` in PG).

### 2.7 Arrays

PostgreSQL's `array_cmp` compares element by element (a NULL element sorts after any
non-NULL one), then by number of items, then by number of dimensions, then per dimension
by length and lower bound. The key payload after tag `0x40` is: each element in flattened
order as `0x02 || element encoding` for a non-NULL element or `0x03` for a NULL element,
then the terminator `0x01`, then `ndims` as one byte, then per dimension the length and the
lower bound, each as a 4-byte flipped `i32`. Elements are always encoded ASC; a DESC array
column complements the whole payload. The terminator `0x01` sorts before `0x02`, so a
shorter array whose items are a prefix of a longer one sorts first, which is PG's item-count
rule for the prefix case; the trailing dimension block settles arrays with equal flattened
items. An array whose element type is lossy (numeric, interval, bpchar, float) is lossy as a
whole.

### 2.8 What is a valid index key in P0

Valid btree key column types in P0 (primary key, secondary index, unique constraint):
`bool`, `int2`, `int4`, `int8`, `oid`, `float4`, `float8`, `numeric`, `text`, `varchar`,
`name`, `bpchar`, `bytea`, `uuid`, `date`, `time`, `timetz`, `timestamp`, `timestamptz`,
`interval`, and arrays of these.

Not valid in P0: `json` (PostgreSQL itself has no btree operator class for it and raises
`42704`; Colibri raises the same), `jsonb` (PostgreSQL has a btree class with its own
ordering; Colibri raises `0A000 feature_not_supported` and the `pg_regress` ledger carries
it as `unsupported` until a later phase defines the order-preserving form), `vector`
(btree on vectors is a pgvector curiosity; HNSW is the index, 00 §4.4), and every phase-2
type. Expression keys are phase 2 (§5.4). The maximum encoded user key length is *default*
4 KiB; a longer key raises `54000 program_limit_exceeded`, as PostgreSQL does for an index
row above its btree maximum (PG's limit is about 2.7 KB; ours differs because our
encoding differs, and the ledger records the diff).

## 3. Non-`C` collations (D20), fixed now

P0 has the `C` collation only, and text keys are their UTF-8 bytes (tag `0x20`). When
ICU4X collations arrive, a text column under a non-`C` collation is encoded as follows, so
that no key format changes then.

**Deterministic collations** (PostgreSQL's default for ICU collations; equality is byte
equality): tag `0x24`, then the escaped (§2.2) collation **sort key**, then the escaped
original UTF-8 bytes as a **tie-breaker**. PostgreSQL's deterministic collations compare
with the collator and break ties with `strcmp`; the two-part key reproduces exactly that
order, and two distinct strings never produce the same key. The value need not carry the
original, since the tie-breaker holds it, so the type is lossless.

**Nondeterministic collations** (`CREATE COLLATION … deterministic = false`, where
`'a' = 'A'` may hold): tag `0x25`, then the escaped sort key only. Strings equal under the
collation produce the same key, which is what a unique index under that collation must
reject and what an equality lookup must find; the original is stored in the value under
the composite rule (§4.3). Rejected: a tie-breaker here too, because it would make
collation-equal strings distinct keys and break uniqueness and equality lookups.

**Collation versions.** A sort key is only comparable to sort keys from the same collator
version. The catalog (05) records the collation version used by each index, as
PostgreSQL's `pg_collation.collversion` does; on a version change the index is marked
invalid for writes and reads until `REINDEX`, and the version is shown in a warning the way
PostgreSQL shows a collation version mismatch. Rejected: storing a version byte per key
(costs bytes forever for an event that happens at upgrade only) and storing originals only
with comparison at read time (gives up memcomparable keys, which 02's block search and
bloom filters require).

Whether ICU4X exposes sort-key generation in the form needed is not verified here; if it
does not, the phase that adds collations must provide one before it can ship, and this
format stands.

## 4. Version values and the row format

### 4.1 Versions and tombstones

A version's value starts with one **kind byte**:

| Byte | Kind |
|---|---|
| `0x00` | Tombstone. The value is exactly this one byte |
| `0x01` | Row format v1 (row store) or index-entry format v1 (§5) |
| `0x02 – 0x7F` | Reserved for later format versions |
| `0x80 – 0xFF` | Reserved |

A reader that meets an unknown kind byte fails the read with `XX001 data_corrupted` and the
key in the message; it never guesses. A format change adds a kind; old versions keep
decoding with the old reader until compaction rewrites them (02), so there is no
flag-day migration.

### 4.2 Row format v1

```
 ┌──────┬──────────────┬───────────────────────────────────────────────┐
 │ 0x01 │ max_attnum   │ entry*  (ascending attnum, to end of value)    │
 │ kind │ varint       │                                               │
 └──────┴──────────────┴───────────────────────────────────────────────┘
 entry:
 ┌──────────────┬──────────────────────────┬──────────────────────────┐
 │ attnum delta │ len << 1 | overflow bit  │ len bytes                │
 │ varint ≥ 1   │ varint                   │ stored form (§4.3) or    │
 │              │                          │ overflow pointer (§4.4)  │
 └──────────────┴──────────────────────────┴──────────────────────────┘
```

Varints are unsigned LEB128. Entries are sorted by `attnum` and each carries the delta from
the previous entry's `attnum` (the first delta is from 0), so most deltas are one byte.

**NULL is an absent entry.** **Missing is distinguished from NULL by `max_attnum`**: the
writer records the table's highest `attnum` at the time the row version was written. On
read, a column with `attnum <= max_attnum` and no entry is NULL; a column with
`attnum > max_attnum` was added after the row was written and takes the catalog's missing
value (05; PostgreSQL's `attmissingval`, which is NULL when the column was added without a
default). This is PostgreSQL's own rule (a heap tuple's `natts` against `attmissingval`),
and it makes `ADD COLUMN` with a NULL or constant default catalog-only. **DROP COLUMN** is
catalog-only: readers skip entries whose `attnum` the catalog marks `attisdropped`, and
compaction (02) omits an entry only when its column was dropped at or below the GC
watermark (04; 05 §6.3 owns the rule), because a snapshot older than the drop still reads
the column. Until then the entry stays in the rewritten segment. A volatile default (`ADD COLUMN … DEFAULT random()`)
is not catalog-only and rewrites the table, as in PostgreSQL.

**Key columns are not repeated** in the value when their key encoding is lossless; the
reader decodes them from the key. A lossy key column (§4.3) has an entry with its original
stored form. Rejected: a fixed column bitmap plus offsets (PostgreSQL's heap layout:
compact for narrow tables but forces a rewrite or a schema-versioned layout on every
`DROP COLUMN`), and a schema-version stamp per row with positional columns (needs every
historical layout kept to decode old rows; attnum tags make each row self-describing).

### 4.3 Stored forms

The stored form of a column value is **not** order-preserving; it is chosen for size and
for zero-translation output. Where PostgreSQL's binary send format is a plain fixed layout,
it is reused, so a binary-format result column is a byte copy of the stored form.

| Type | Stored form | Lossy as key? |
|---|---|---|
| `bool` | 1 byte (send format) | no |
| `int2/4/8`, `oid` | big-endian, 2/4/8/4 bytes (send format) | no |
| `float4/8` | IEEE big-endian (send format) | yes (`-0`) |
| `numeric` | PG send format: `ndigits, weight, sign, dscale` as `i16`, then base-10000 digits as `i16` | yes (scale) |
| `text`, `varchar`, `name`, `json` | UTF-8 bytes (send format) | no (`C`) |
| `bpchar` | UTF-8 bytes as padded (send format) | yes (trailing spaces) |
| `bytea` | raw bytes (send format) | no |
| `uuid` | 16 bytes (send format) | no |
| `date`, `time`, `timestamp`, `timestamptz` | big-endian `i32`/`i64` (send format) | no |
| `timetz` | `i64` µs + `i32` zone (send format) | no |
| `interval` | `i64` µs, `i32` days, `i32` months (send format) | yes (span) |
| `jsonb` | one sub-format byte, then: `1` = PG send format (version byte 1 plus canonical text) | not a key |
| arrays | PG array send format (ndim, has-null flag, element OID, dims and lower bounds, length-prefixed elements) | if the element is |
| `vector` | pgvector send format (`u16` dims, `u16` unused, `f32` big-endian each) | not a key |

`jsonb` in P0 is stored as its canonical text and parsed on read. That is slow for
operator-heavy workloads; a binary tree form is a later phase and becomes sub-format `2`
without a row-format change. Rejected for P0: PostgreSQL's on-disk varlena forms
(alignment padding, platform-endian, TOAST headers we do not have), and our own compact
integer varints (saves bytes on small integers but makes binary output a translation).

**The composite rule.** A key column whose type is marked lossy above always also has its
original stored form in the value (the row value for primary-key columns, the index-entry
value for secondary-index columns, §5). The rule is per type, not per value, so a decoder
never has to inspect a key to know whether to look in the value. This is CockroachDB's
"composite column" treatment, and it covers collated text in §3 by the same mechanism.

### 4.4 Large values: overflow chunks

A value larger than a segment block (*default* 32 KiB, 02) would make blocks large and
scans slow. Large column values therefore move out of line, in the shape of TOAST:

- A column's stored form above the **inline threshold** (*default* 8 KiB, a quarter of a
  block so a block still holds several rows) is written out of line.
- If the row value is still above the **row target** (*default* 16 KiB) after that, the
  largest remaining columns go out of line until it fits or none above *default* 2 KiB
  remains (PostgreSQL's `toast_tuple_target` shape).
- The maximum stored size of one column value is 1 GiB − 1 byte, PostgreSQL's varlena
  limit; above it the write fails with `54000`.

Out-of-line values are written in **chunks** of *default* 28 KiB (so a chunk plus its key
fits one 32 KiB block) under the reserved index id:

```
key   = prefix(TableId, 0xFFFF_FFFF) || pk tuple || int4(attnum) || int4(chunk_no) || !commit_ts
value = 0x01 || chunk bytes
```

The entry in the row value has the overflow bit set and holds a pointer:
`total_len` (varint), `chunk_count` (varint), `chunk_ts` (`u64` BE), `checksum` (xxh3-64 of
the whole value). `chunk_ts = 0` means "the commit timestamp of this row version"; the
writer does not know its commit timestamp until commit (04), so a new value is always
written with `0`, and the chunks are written in the same batch at the same timestamp. An
`UPDATE` that does not change the column copies the pointer with `chunk_ts` set to the
timestamp of the version it read, so the chunks are not rewritten. A reader fetches chunk
`i` as the newest version of its chunk key with `commit_ts <= chunk_ts` (resolving `0` to
the row version's own timestamp) and verifies the checksum.

Lifecycle: rewriting the column writes new chunks and tombstones chunk numbers beyond the
new count; making the value inline again or deleting the row tombstones all its chunks. The
GC rule of 04 (keep the newest version at or below the watermark plus all above) is then
sufficient: any retained row version references a `chunk_ts` no newer than itself, and a
chunk version newer than that `chunk_ts` but not newer than the watermark could only have
been written by a newer row version that is also retained instead. Rejected: a
per-value id with chunks keyed by id (needs reference tracking to collect orphans, which
the PK-keyed form gets from ordinary MVCC GC), and no overflow at all with a hard row limit
(CockroachDB's choice; PostgreSQL applications store multi-megabyte `text` and `bytea`
routinely, and `pg_regress` does).

Keys are never out of line; the 4 KiB key limit of §2.8 applies.

## 5. Secondary index entries

### 5.1 Non-unique indexes

```
key   = prefix(TableId, IndexId) || index columns || pk columns || !commit_ts
value = 0x01 || pk_len varint (= 0) || body
```

The PK columns make every entry unique and give the row's key for the fetch: the row key is
`prefix(TableId, 0) || pk columns`, a byte copy of the suffix of the index tuple. `body` is
a row-format v1 body without the kind byte (`max_attnum` then entries), keyed by **table**
`attnum`, carrying the `INCLUDE` columns (P0 supports covering indexes) and the originals of
lossy index columns (§4.3). An empty body is `max_attnum = 0` and no entries, so the
smallest index value is three bytes. A delete or an update that changes an indexed
column writes a tombstone for the old entry's key and a new entry; an update that changes
only `INCLUDE` columns writes a new version of the same key.

### 5.2 Unique indexes

For a UNIQUE index the indexed columns alone must be unique, so the key omits the PK:

```
key   = prefix(TableId, IndexId) || index columns || !commit_ts
value = 0x01 || pk_len varint || pk tuple bytes || body
```

A uniqueness check (04) is then one point lookup of the newest version of the key, and
the PK is read from the value. **NULLs are distinct** by default in PostgreSQL: two rows
with `(NULL)` do not conflict. So when **any** indexed column is NULL, the PK columns are
appended to the key and `pk_len` is 0, which makes the keys distinct exactly when the rows
are; this is what CockroachDB does. With `NULLS NOT DISTINCT` (PostgreSQL 15+) the PK is
never appended and NULL keys conflict. The decoder knows whether the PK suffix is present
from the NULL markers it has already read, which is what keeps the encoding prefix-free
(§1.3). An update that changes the PK but not the indexed columns writes a new version of
the same index key with a new value. Rejected: always appending the PK (the uniqueness
check becomes a prefix scan, and an index-only point lookup of a unique key a scan), and
keeping NULL rows out of the index (breaks `IS NULL` index scans, which PostgreSQL serves).

The primary key itself is the row store (§1.1) and needs no separate index.

### 5.3 Partial indexes

A partial index (`WHERE predicate`) has the same key and value shape and contains entries
only for rows whose new version satisfies the predicate. Maintenance evaluates the
predicate on the old and the new row: true → false writes a tombstone, false → true a new
entry, true → true the usual update. P0 supports partial indexes with immutable predicates
over the table's columns, which is what PostgreSQL allows.

### 5.4 Expression indexes

Phase 2. The key element will be the tuple encoding of the expression's result type, the
value carries the result's stored form when the type is lossy, and the expression's
immutability is checked as PostgreSQL does. Nothing in this format needs to change; P0
raises `0A000` for `CREATE INDEX … ((expr))`.

### 5.5 Per-column ordering

Each index column carries its own `ASC`/`DESC` and `NULLS FIRST`/`LAST` (§2.1). The PK
columns appended to a secondary index key are always encoded ASC with the PK's own
column options, since they are a tie-breaker and the row store's key, not part of the
index's declared order.

## 6. The hidden `rowid` and `ctid`

A table with no primary key gets a hidden key column `rowid int8`, allocated from a
per-table sequence (04 §sequences) at insert. It has no `attnum` among the user columns
(05 gives it a negative system `attnum`, as PostgreSQL's system columns have), it is the
sole element of the row store's tuple, it is never stored in the value, and it is appended
to secondary-index keys like any PK. It does not appear in `SELECT *`. A later
`ALTER TABLE … ADD PRIMARY KEY` rewrites the table (as in PostgreSQL for a new PK index
build, but here the row keys change), which is acceptable at DDL time.

**`ctid`** is answered with a synthetic `tid` (block `u32`, offset `u16`):

- For a `rowid` table: `block = rowid >> 16`, `offset = (rowid & 0xFFFF) + 1` (offset 0 is
  invalid in a `tid`). The mapping is invertible for `rowid < 2^48`, so `WHERE ctid = '(b,o)'`
  plans as a point lookup on the row store. `pg_regress` uses this shape on tables without
  a primary key, which is why the choice is made here.
- For a table with a primary key: the `tid` is derived from the first 48 bits of
  `xxh3-64(pk tuple)`. `WHERE ctid = …` is evaluated as a filter over a scan (correct, not
  fast). Two rows can collide; at one million rows the birthday probability is on the
  order of 10^-3, so `WHERE ctid = …` on a PK table may in rare cases match two rows. That
  limitation is documented and carried in the ledger.

Two deliberate differences from PostgreSQL go into the `pg_regress` ledger as `intended`:
an `UPDATE` does not change a row's `ctid` (PostgreSQL's does, and some tests print it), and
`ctid` values do not reflect physical placement. Rejected: no `ctid` at all (breaks
`pg_regress` files and tools that select it), and a per-scan ordinal (unstable between two
statements, so `WHERE ctid = <value read before>` would fail).

## 7. Sequences and catalog rows

A sequence's state is one row in the row store of its own OID with an empty tuple:
`key = prefix(seq_oid, 0) || !commit_ts`. The value is row format v1 with the live-counter
columns 05 §2.8 defines: `{last_value: i64, is_called: bool, reserved_upto: i64}`; the
`log_cnt` of PostgreSQL is not stored, because the ahead-reservation (04 §8.2,
`max(32, CACHE)` values) is the ledger's `SeqReserve` record instead. Applying a
`SeqReserve { seq_oid, reserved_upto, is_called }` (03 §2.2, 04 §8) writes a new version of
this row at the record's own `commit_ts` carrying those values; on wake the in-memory
counter resumes from it, so `nextval` starts at `reserved_upto + increment` when
`is_called`. Sequence rows are therefore MVCC versions like any
other, and the GC rule keeps only the newest below the watermark. The semantics
(non-transactional `nextval`, gaps) are 04's.

Catalog rows are ordinary rows in the internal catalog relations under the fixed OIDs of
§1.4, with the primary keys 05 defines, encoded by the same tuple encoding (catalog keys
mostly use `oid` and `name` elements) and the same row format. DDL writes them through the
transaction write buffer (04) and they are versioned and compacted like user data.

## 8. Worked example

Schema (OIDs as 05 would allocate them from 16384; the PK's `pg_index` OID 16385 never
appears in a key, §1.1):

```sql
CREATE TABLE users (id int4 PRIMARY KEY, email text, score numeric(6,2));  -- 16384
CREATE UNIQUE INDEX users_email ON users (email);                          -- 16386
CREATE TABLE notes (body text);                                            -- 16388, no PK → rowid
```

Three single-row transactions commit at HLC `CommitTs` values with physical part
2026-10-11T00:00:00Z (`1791676800000` ms) and logical counters 0, 1, 2:

| Txn | `CommitTs` | `!CommitTs` |
|---|---|---|
| 1 | `01 A1 28 42 5C 00 00 00` | `FE 5E D7 BD A3 FF FF FF` |
| 2 | `01 A1 28 42 5C 00 00 01` | `FE 5E D7 BD A3 FF FF FE` |
| 3 | `01 A1 28 42 5C 00 00 02` | `FE 5E D7 BD A3 FF FF FD` |

**Txn 1:** `INSERT INTO users VALUES (1, 'a@x', 12.50)`.

```
row key    00 00 40 00  00 00 00 00 | 13 80 00 00 01 | FE 5E D7 BD A3 FF FF FF
            TableId      IndexId 0    int4 1           !ts
row value  01 | 03 | 02 06 61 40 78 | 01 18 00 02 00 00 00 00 00 02 00 0C 13 88
           kind max  email:Δ2 len3   score:Δ1 len12, numeric send format:
                attnum "a@x"         ndigits 2, weight 0, sign +, dscale 2, digits 12, 5000

index key  00 00 40 00  00 00 40 02 | 20 61 40 78 00 01 | FE 5E D7 BD A3 FF FF FF
                         IndexId     text "a@x" + terminator
index val  01 | 05 | 13 80 00 00 01 | 00
           kind pk_len pk tuple       empty body (max_attnum 0)
```

`id` is lossless and not repeated in the row value. The unique index has no NULL column,
so the PK is in the value.

**Txn 2:** `INSERT INTO users VALUES (2, NULL, 0)`.

```
row key    00 00 40 00 00 00 00 00 | 13 80 00 00 02 | FE 5E D7 BD A3 FF FF FE
row value  01 | 03 | 03 10 00 00 00 00 00 00 00 00
           email absent (attnum 2 <= max_attnum 3 → NULL); score: Δ3 len 8, numeric 0
index key  00 00 40 00 00 00 40 02 | FE | 13 80 00 00 02 | FE 5E D7 BD A3 FF FF FE
                                     NULL (ASC, NULLS LAST) then the appended PK
index val  01 | 00 | 00
```

**Txn 3:** `INSERT INTO notes VALUES ('hi')`, `rowid` 1 from the table's sequence.

```
row key    00 00 40 04 00 00 00 00 | 14 80 00 00 00 00 00 00 01 | FE 5E D7 BD A3 FF FF FD
                                     int8 rowid 1
row value  01 | 01 | 01 04 68 69
ctid       (0,2)
```

A later `DELETE FROM users WHERE id = 2` at `CommitTs` `…03` writes the row key with suffix
`FE 5E D7 BD A3 FF FF FC` and value `00`, and the index key `…FE 13 80 00 00 02` with the
same suffix and value `00`. In the merged order the tombstones sort immediately before the
versions they shadow (§1.2).

## 9. Rust type sketches

Sketches of the `colibri-encoding` contract, not code.

```rust
/// prefix || tuple || !commit_ts. Ord is bytewise.
pub struct EncodedKey(Box<[u8]>);

/// prefix || tuple, no timestamp. What the write buffer (04) orders by and the bloom hashes.
pub struct UserKey(Box<[u8]>);

impl EncodedKey {
    pub fn new(user: &UserKey, ts: CommitTs) -> Self;
    pub fn user_key(&self) -> &[u8];            // all but the last 8 bytes
    pub fn commit_ts(&self) -> CommitTs;        // !BE of the last 8 bytes
    pub fn seek_at(user: &UserKey, snapshot: SnapshotTs) -> Self; // §1.2
}

pub struct Prefix { pub table: TableId, pub index: IndexId }
pub const PRIMARY: IndexId = IndexId(0);
pub const OVERFLOW: IndexId = IndexId(0xFFFF_FFFF);

pub enum Version<'a> {
    Tombstone,                    // value == [0x00]
    Row(RowValue<'a>),            // kind 0x01 in a row store
    IndexEntry(IndexValue<'a>),   // kind 0x01 in a secondary index
}

pub struct RowValue<'a> {
    pub max_attnum: u16,
    entries: &'a [u8],            // lazily iterated, ascending attnum
}
pub enum Stored<'a> { Inline(&'a [u8]), Overflow(OverflowPtr) }
pub struct OverflowPtr { pub total_len: u64, pub chunk_count: u32, pub chunk_ts: u64, pub checksum: u64 }

pub struct IndexValue<'a> { pub pk: Option<&'a [u8]>, pub body: RowValue<'a> }

#[derive(Clone, Copy)]
pub struct KeyColumn { pub ty: KeyType, pub desc: bool, pub nulls_first: bool }

pub fn encode_tuple(cols: &[KeyColumn], vals: &[Datum], out: &mut Vec<u8>) -> Result<(), EncodeError>;
pub fn decode_tuple(cols: &[KeyColumn], bytes: &[u8]) -> Result<(Vec<Datum>, usize), DecodeError>;
pub fn encode_row(max_attnum: u16, cols: &[(AttNum, &Datum)], out: &mut Vec<u8>) -> Result<(), EncodeError>;

/// Phase-1 values. Owned for simplicity in the sketch; the executor may use borrowed forms.
pub enum Datum {
    Null,
    Bool(bool), Int2(i16), Int4(i32), Int8(i64), Oid(u32),
    Float4(f32), Float8(f64),
    Numeric(Numeric),             // PG semantics incl. NaN, ±Infinity, dscale
    Text(String),                 // text, varchar, name
    Bpchar(String),
    Bytea(Vec<u8>),
    Uuid([u8; 16]),
    Date(i32), Time(i64), TimeTz { us: i64, zone_s: i32 },
    Timestamp(i64), TimestampTz(i64),
    Interval { months: i32, days: i32, us: i64 },
    Json(String), Jsonb(Jsonb),
    Array(Box<Array>),            // element OID, dims, lower bounds, Vec<Datum>
    Vector(Vec<f32>),
}
```

Decoding never panics on any input: every function returns an error with the byte offset.
`decode_tuple` returns the consumed length so a caller can find where the PK suffix starts.

## 10. Test plan for `colibri-encoding`

**Ordering equals PostgreSQL's.** Two oracles. The first is a pure-Rust reference
comparator per type, written directly from the PostgreSQL 18 comparison functions named
above (`float8_cmp_internal`, `cmp_numerics`, `interval_cmp_value`, `timetz_cmp_internal`,
`array_cmp`, `bpcharcmp`, `uuid_cmp`), used by `proptest`: for random pairs of values and
random column options (ASC/DESC, NULLS FIRST/LAST, 1–4 columns),
`sign(memcmp(encode(a), encode(b))) == sign(reference_cmp(a, b))`. Generators bias toward
edges: NaN, ±Infinity, `-0.0`, numeric with trailing zeros and huge exponents, empty
strings, embedded `0x00` in `bytea`, `'1 mon'` versus `'30 days'`, arrays with NULL elements
and different lower bounds. The second oracle is PostgreSQL 18 itself, in a CI job: a
seeded generator inserts the same random values into PostgreSQL and asks for
`ORDER BY … COLLATE "C"`, and the encoded order must match row for row. The reference
comparator keeps the unit tests hermetic; the PostgreSQL job catches a reference that is
wrong in the same way as the encoder.

**Round trip.** For every lossless type, `decode(encode(v)) == v`; for lossy types,
`decode(encode(v))` equals `v` under PostgreSQL equality and the composite value restores
the original exactly. Row format: `decode_row(encode_row(r)) == r` with random attnum sets,
`max_attnum` semantics, and overflow on and off.

**Invariants.** Prefix-freeness (§1.3) over random tuples of the same index, including the
unique-index NULL rule; for random version sets, the seek of §1.2 returns the version the
naive "max `commit_ts <= S`" scan returns; the DESC complement of every element never
starts with `0x01` or `0xFE`.

**Fuzz.** `cargo fuzz` targets for `decode_tuple`, `decode_row` and the overflow pointer:
no panic, no unbounded allocation (lengths are checked against the remaining input), and
`encode(decode(x)) == x` whenever `decode(x)` succeeds (canonical encodings only; a
non-canonical input such as a numeric with a trailing zero digit must be rejected). The
fuzz corpus seeds are public; crashes found become regression tests (D18).

**Golden bytes.** The examples of §8 are a test: the hex is asserted byte for byte, so a
format change cannot slip in without this document changing too.

## 11. What this document does not decide

- **The values of the thresholds**: inline threshold 8 KiB, row target 16 KiB, chunk size
  28 KiB, key limit 4 KiB are *defaults* chosen from the 32 KiB block *default* of 02; none
  is measured. P0's scan and point-lookup benchmarks replace them.
- **An order-preserving `jsonb` key** and a binary `jsonb` stored form (sub-format `2`);
  later phase. Until then `jsonb` btree indexes are `unsupported` in the ledger.
- **Expression index keys** (phase 2, §5.4) and keys for phase-2 types (inet/cidr, ranges,
  composites, enums, domains). Enums will need their sort-order float, not their OID.
- **Whether ICU4X can produce sort keys** as §3 requires, and the cost of the
  tie-breaker's doubled key size for deterministic collations; decided when collations are
  scheduled.
- **Compression of keys** beyond 02's prefix compression (for instance a dictionary for the
  repeated 8-byte prefix); measured in P0.
- **`ctid` collision behaviour on PK tables** beyond documenting it; revisited only if a
  ledger row or a customer depends on it.
- **The catalog's column sets and OIDs**, including the sequence-state columns: 05.
- **Version GC, unique-check timing and deferrable constraints**: 04. **Segment block
  layout and bloom construction** over these keys: 02.
