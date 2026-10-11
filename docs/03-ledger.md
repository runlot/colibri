# 03 — The ledger: append, fencing, recovery, truncation and degraded mode

> Kind: **design**. Status: **pre-implementation** (2026-10-11). Fills the gap left by
> docs/00-overview.md D7, §4.2 step 2, §4.5 and §4.6 for phase P0: the record format, the
> peer's on-disk layout and group fsync, the append and acknowledgement protocol, epoch
> fencing, recovery on a lease change, truncation, the three durability modes and the
> transitions between them, and single-node mode. It depends on **02** (the manifest's
> `ledger_position`, `durability` field and interval history, and the forced flush) and on
> **04** (which writes go into a transaction's batch and where in the commit path the
> ledger is called); this document owns the byte layouts of the `TxnBatch` and
> `SeqReserve` payloads (§2.2), which 04 cites. **06** owns the `Env` trait whose transport
> and disk subset this crate (`colibri-ledger`) uses, and the simulation harness §12 runs
> under. 00 §4.1 draws the shared peer-wide log of §2.

## 1. Roles and placement

### 1.1 Peers

A ledger peer is `colibrid --role ledger` (D7, 06 §9.1): the same binary as the engine daemon, run
with a role flag that starts only the peer service, the local log store and the metrics
endpoint. A region runs **three** peers by default, on three different nodes and, where the
provider has them, three failure domains. A peer serves every database whose lease names
it; it has no notion of which databases exist beyond the records it holds.

A node may run both roles in one process (`--role node,ledger`; the engine-only role is `--role node`). The two share nothing
but the process: the peer's log store is a separate directory, and the engine reaches its
co-located peer through the same `Transport` as a remote one, over a loopback channel
(§3.3). This keeps one code path and lets the simulation treat co-location as a network
property.

### 1.2 The peer set is part of the lease

The CP's lease grant (00 §4.6, D13) is `(db-id, epoch, holder, peers)` where `peers` is an
ordered set of one or three peer addresses, `{peer_a, peer_b, peer_c}`. The writer never
chooses peers itself and never adds or removes one during an epoch. A database's peer set
is therefore a function of its epoch, and every statement in this document about "a
majority" means a majority of the set named by the epoch being discussed.

The CP chooses the set at placement: three healthy peers in the database's region, least
loaded by held bytes (§10). It records the set in the lease row so the next writer, after
any crash, is told which peers might hold the tail.

### 1.3 Membership change

A peer set changes only by **a new lease epoch with a new set**. The algorithm, run by the
CP and the new writer:

1. The CP decides to move the database from set `S_old` (epoch `e`) to `S_new`. If the
   database is open, the CP asks the current holder to **flush** (00 §4.2 step 3, 02's
   forced flush): the holder freezes its memtable, writes the segment, writes the manifest
   with `ledger_position = (e, L)` where `L` is the memtable's `max_lsn` at the cut (§8;
   it may exceed the last acknowledged lsn under `synchronous_commit = off`), and reports
   `flushed_lsn = L` (§6).
2. The holder releases the lease (or the CP revokes it if the holder is unreachable).
3. The CP grants epoch `e+1` with `S_new` to the new holder (possibly the same daemon).
   The grant also carries `S_old` as `prev_peers` with `prev_epoch = e`.
4. The new writer reads the manifest. If `ledger_position.lsn` is at or above the last
   lsn the old set holds — which it is when step 1 completed — the old set holds nothing
   above the manifest and recovery (§5) runs against `S_new` only, finding empty tails, and the
   database opens with the next lsn `L+1`.
5. If step 1 did not complete (holder crashed, flush failed), the new writer first runs
   §5 recovery against **`S_old`** with epoch `e+1` (bumping `S_old` fences the old
   holder), replays the old tail into its memtable, immediately forces a flush and a
   manifest write that advances `ledger_position` past the old tail, and only then bumps
   `S_new` and opens. The old set's records are then below the flushed position and are
   truncated normally.

Why this is safe: at every instant the unflushed committed records of the database lie in
exactly one peer set, the one of the latest epoch that appended. Step 5 never appends to
`S_new` until the manifest proves that everything from `S_old` is in the bucket, so no
recovery ever needs to consult two sets at once; and bumping `S_old` to `e+1` before
reading it means the old holder can no longer gain an acknowledgement there (§4). The cost
is one forced flush per membership change, which is cheap because membership changes are
rare (their frequency is open, §13). Rejected: joint consensus (Raft's two-majority
transition), because it needs per-record knowledge of two configurations and gives nothing
a flush does not already give in a system whose truth is the bucket.

### 1.4 Single node

On single-node on-prem the lease's set is `{local}`: one peer, the local daemon, majority
1 of 1. Every rule below holds with `n = 1`; §9 covers what changes on the surface.

## 2. On-peer storage

### 2.1 Choice: one peer-wide shared log, not a file per database

The brief's starting point is per-database append-only files. The cost of that layout is
in the fsync: a group-commit window in which batches for `N` databases arrive touches `N`
files and needs `N` `fdatasync` calls (or a `syncfs`, which also flushes unrelated files).
A peer serving thousands of databases with a handful active per millisecond would turn
every window into tens of fsyncs on the same device. Neon's safekeeper uses a file per
timeline because its timelines are few and large; Colibri's are many and small (G3).

**Decision: a peer writes one shared append-only log for all databases**, under
`<data-dir>/ledger/log/<segment-seq>.log`, with records of every database interleaved, and
keeps a per-database in-memory index. One window costs one `fdatasync` of one file
regardless of how many databases contributed. Rejected: per-database files (N fsyncs per
window), and per-database files with `syncfs` (flushes the page cache of the whole
filesystem, so latency depends on unrelated writers; not portable to the simulation `Env`).

The shared log is split into **segments** that rotate at a size (*default* 64 MiB) or an
age (*default* 60 s), whichever comes first. A segment is never rewritten; it is deleted as
a whole when no record in it is still needed (§6.2).

### 2.2 Record framing

Every record on disk and on the wire has the same frame. All integers are little-endian
(peers and writers are the same binary; little-endian matches every supported target and
needs no swap on x86-64 or aarch64).

```
 offset size field
 0      4    len         u32, bytes from offset 8 to end of payload
 4      4    crc32c      over bytes [8, 8+len)  (Castagnoli, hardware on both arches)
 8      1    version     record format version, 1 in P0
 9      1    kind        0 = TxnBatch, 1 = SeqReserve, 2 = Noop, 3 = Bump,
                         4 = Discard, 5 = Checkpoint      (3..5 are peer-local, §2.3)
 10     2    flags       bit 0: payload compressed (always 0 in P0, §13)
 12     16   db_id       ULID
 28     8    epoch       epoch that originally appended this record
 36     8    lsn         u64
 44     8    commit_ts   HLC, 0 for peer-local kinds
 52     ..   payload     kind-specific; TxnBatch and SeqReserve below
```

Header size is 52 bytes. A record (header plus payload) is capped at *default* **16 MiB**,
so one record always fits 06's 16 MiB transport frame (06 §11.5). Rejected: the earlier
64 MiB cap, because a record larger than the frame could not be sent unsplit.

**`TxnBatch` payload.** A fixed 17-byte header, then `n_entries` entries:

```
 offset size field
 0      8    commit_ts   u64, duplicate of the record header (self-describing payload)
 8      2    frag_index  u16, 0 for the first fragment of a transaction
 10     1    frag_final  0 or 1; 1 on the transaction's last fragment
 11     4    n_entries   u32
 15     2    flags       bit 0: contains catalog rows (05); other bits 0 in P0
 17     ..   entries     n_entries × (key_len varint, key, value_len varint, value)
```

Keys are in 01's encoded form **without** the timestamp suffix;
apply derives the suffix from `commit_ts`. A value is the encoded version 04 writes into
the memtable (a tombstone included). Varints are unsigned LEB128.

**Fragments.** A transaction whose batch exceeds the record cap is split into fragments:
records with **consecutive lsns** sharing one `commit_ts`, `frag_index` counting from 0,
`frag_final = 1` on the last; a transaction that fits is one record with `frag_index = 0,
frag_final = 1`. No other transaction's record is interleaved (lsns are assigned under the
commit token, 04). The writer acknowledges a transaction only after its **final**
fragment's quorum ack, and replay and recovery apply a run only when its final fragment is
present (§5.2 steps 5 and 8).

**`SeqReserve` payload.**

```
 offset size field
 0      4    seq_oid        u32, the sequence's oid (05 §2.8)
 4      8    reserved_upto  i64
 12     1    is_called      0 or 1
```

`Noop` has an empty payload.

A torn tail is detected on restart: the peer scans the newest segment, and the first
record whose `len` runs past end of file or whose crc fails ends the log; the bytes after
it are truncated. This is safe because a record is acknowledged only after the
`DiskFile::sync()` (an `fdatasync`) that covered all its bytes (§3.4).

### 2.3 Peer-local records

Three kinds never leave the peer:

| Kind | Payload | Purpose |
|---|---|---|
| `Bump` | `{new_epoch}` | Durable record that the peer adopted an epoch for `db_id` (§4.1) |
| `Discard` | `{from_lsn}` | Records of `db_id` with `lsn >= from_lsn` written before this record are void (§5.3) |
| `Checkpoint` | `{epoch, last_epoch, last_lsn, flushed_lsn}` | First records of every new segment, one per database still held, so a database's state survives deletion of the segments that held its `Bump` |

### 2.4 In-memory index

For each database the peer holds:

```rust
// Sketch.
struct DbIndex {
    promised_epoch: Epoch,          // highest epoch adopted (Bump)
    last: Option<(Epoch, Lsn)>,     // (record epoch, lsn) of the last live record
    flushed_lsn: Lsn,               // reported by the writer (§6)
    entries: BTreeMap<Lsn, Loc>,    // lsn -> (segment, offset, record epoch, crc)
}
```

The index is rebuilt at start by scanning the retained segments in order and applying each
record (`Discard` truncates `entries` from `from_lsn`). With the flush age bound (§6.3) a
peer holds tens of seconds of writes, so the scan is bounded by peer disk read speed over
at most the backpressure threshold (§6.4); the start time is *not measured*.

### 2.5 Group fsync

The peer's write loop accepts appends from every connection into one buffer, and closes a
window either after *target* 1 ms or when the buffer exceeds *default* 1 MiB. It then
writes the buffer to the current segment, calls `sync()` once, and sends every
`AppendAck` whose records were in the window. Records are acknowledged only after that
fsync returns success; an fsync error makes the peer **stop** (fail-stop, exit non-zero),
because after a failed fsync the kernel's page-cache state is unspecified (the PostgreSQL
"fsyncgate" lesson). A stopped peer is one fewer ack; the quorum covers it.

The 1 ms window is a *target* that trades a per-commit latency floor for fewer fsyncs; its
value is open (§13).

## 3. The append protocol

### 3.1 Messages

| Message | Direction | Fields |
|---|---|---|
| `Append` | writer → peer | `db, epoch, from_lsn, records[]` — records with lsn `from_lsn ..` contiguous |
| `AppendAck` | peer → writer | `db, epoch, upto_lsn` — cumulative: every record `<= upto_lsn` is durable |
| `Bump` | writer → peer | `db, epoch` |
| `BumpAck` | peer → writer | `db, epoch, last_epoch, last_lsn, flushed_lsn` |
| `ReadTail` | writer → peer | `db, epoch, from_lsn` |
| `Tail` | peer → writer | `db, epoch, records[], end` — streamed in chunks, `end` on the last |
| `Truncate` | writer → peer | `db, epoch, upto_lsn` — records `<= upto_lsn` are in the bucket |
| `Fenced` | peer → writer | `db, promised_epoch` — request epoch is stale |
| `Full` | peer → writer | `db, disk_fill_pct` — backpressure (§6.4) |
| `Error` | peer → writer | `db, code, detail` — protocol violation, peer fault |

`Append` carries a field `truncate_above: bool`, set only by recovery (§5.3): it tells the
peer that its records of `db` at `>= from_lsn` may be replaced. Ordinary appends set it to
false, and a peer that receives an ordinary `Append` whose `from_lsn` is not exactly
`last_lsn + 1` (or a re-send, §3.6) answers `Error{gap}`.

### 3.2 Encoding

A frame on the connection is `u32 length || u8 message_type || body`, with bodies in a
fixed little-endian layout and records embedded in their §2.2 framing unchanged, so a peer
writes received record bytes to disk without re-encoding and the crc computed by the writer
is the crc stored. Decision: a hand-written binary format in `colibri-ledger`, about a
dozen message types. Rejected: protobuf/prost (a build-time code generator and a
dependency for a protocol whose two ends are always the same binary; it would also
re-encode record bytes), and serde with bincode or similar (a format whose layout is a
library's choice, not ours, on the one path that must stay stable across rolling upgrades).
A connection opens with `Hello{protocol_version, node_id}`; mismatched major versions are
refused.

### 3.3 Transport

TCP between nodes, with **mutual TLS using the node certificates Runlot's node PKI already
issues**: a peer accepts a connection only from a certificate whose node identity the CP
lists as engine or ledger in the region, and the writer verifies the peer's certificate
against the address in the lease. This is a dependency on Runlot's node PKI (issue,
rotation, revocation) that this document does not design; on-prem without Runlot supplies
certificates from its own CA through the same configuration. In single-node mode and for a
co-located peer the transport is an in-process channel (no socket, no TLS) behind the same
`Transport` trait (§11.4).

One connection per (writer daemon, peer) pair carries every database's traffic, so a node
with 1,000 open databases holds three connections, not 3,000.

### 3.4 Pipelining and the quorum rule

The writer assigns lsns in commit order and may have many `Append`s in flight per peer
(*default* window 64 batches or 8 MiB). Peers process a connection's messages in order.
Acks are cumulative: `AppendAck{upto_lsn}` covers everything below.

For each peer the writer keeps `acked[peer]`. The database's **committed lsn** is the
largest `L` such that a majority of the set has `acked >= L` — 2 of 3, or 1 of 1. When the
committed lsn advances, every transaction whose last record is `<= L` completes its commit
in lsn order: 04 applies it to the memtable, advances `last_committed_ts` and acknowledges
the client. Commit completions never reorder, even if a later batch was acked first by
some peer, because the committed lsn is a prefix.

A co-located peer counts like any other: it is one of the three, reached over the
loopback channel, and its ack is an fsync on the local disk.

### 3.5 Timeout and retry

Each in-flight append to a peer has a deadline (*default* 500 ms). On timeout or
connection loss the writer reconnects with exponential backoff (*default* 10 ms doubling to
1 s, jittered by `Env` randomness) and re-sends everything above `acked[peer]`. A
database whose committed lsn has not advanced for *default* 2 s while appends are pending
enters degraded mode (§7.2). A slow peer never blocks commits while the other two answer.

### 3.6 Idempotency

Re-sent records are normal. A peer receiving a record at an lsn it already holds compares
`(record epoch, crc32c)` with its stored entry: equal means the record is already durable,
and the peer acknowledges without writing. Unequal under an ordinary `Append` is
impossible under the protocol (only one writer per epoch assigns a given lsn) and the peer
answers `Error{conflict}`; under recovery's `truncate_above` it is the divergence case
(§5.3).

## 4. Epoch fencing

### 4.1 The ledger fence

For each database a peer keeps `promised_epoch`, the highest epoch it has adopted. It
adopts an epoch on `Bump{e}` with `e > promised_epoch`, or on the first `Append`/`ReadTail`
carrying `e > promised_epoch` (the brief's "adopt on first contact"), by writing a `Bump`
record and **fsyncing it before replying**. A message carrying `e < promised_epoch` is
answered with `Fenced{promised_epoch}` and has no effect. A `Bump{e}` with
`e == promised_epoch` is answered again with the same `BumpAck` (idempotent).

In practice the writer always bumps before it appends (§5), so adoption on first `Append`
only matters for a peer that was unreachable during the bump and comes back during the
epoch; it then joins with an empty or stale tail and is caught up by the writer re-sending
from that peer's `last_lsn + 1` (the writer learns it from the `Error{gap}` reply carrying
the peer's `last_lsn`).

A writer that receives `Fenced` from any peer **closes the database immediately**: it
fails every pending commit with the SQLSTATEs 04 §6.2 assigns (`08007` for the committing
session, `57P01` for the others), drops the memtable, stops flushing and tells the CP. It does not
retry: a higher epoch exists, so its lease is gone.

### 4.2 The two fences together

The manifest fence (00 §4.6, 02): every manifest write is a conditional PUT against the
version the writer last read or wrote, and the manifest carries its epoch. A new writer's
first act after recovery is a manifest write (§5.2 step 10) that bumps the manifest
version, so any older writer's next conditional PUT fails.

**Claim.** A writer `W` holding epoch `e`, after a writer `W'` has completed recovery in
epoch `e' > e`, can neither acknowledge a commit nor publish a manifest.

*Sketch.* `W'`'s recovery bumped a majority `M'` of the set to `e'` before reading tails.
For `W` to acknowledge a commit it needs a majority `M` of fsync acks in epoch `e`; `M`
and `M'` intersect in some peer `p` (if the epoch changed the peer set, §1.3 bumps the
old set too). `p` has `promised_epoch >= e' > e`, so `p` answers `Fenced` to `W`'s
append. Hence `W` gets at most a minority of acks after `W'` bumped, and a commit acked
*before* the bump was on a majority, hence is in `M'`'s tails and is recovered by `W'`
(§5.4). For the manifest: `W'` wrote a manifest after reading the current one, so the
bucket's version is not the one `W` holds, and `W`'s conditional PUT fails. A record `W`
appended to a minority after the bump is never acknowledged; recovery may keep or drop it
(§5.4) and in either case no client was told it committed.

The ledger fence protects acknowledgements; the manifest fence protects the bucket. Either
alone is insufficient: without the ledger fence a stale writer could acknowledge commits
that the new writer never sees; without the manifest fence a stale writer's flush could
overwrite the new writer's manifest.

## 5. Recovery on lease acquisition

### 5.1 Inputs

The new writer holds `(db, e, peers)` from the CP and has read the current manifest `m`
with `m.ledger_position = (e_m, P)` and `m.durability`. `m` always exists at recovery,
because 02 §4.5 writes an initial manifest before recovery when there is no `CURRENT`; on
a fresh database `P = 0` and the tail is empty. If `m.durability` is
`bucket-direct`, recovery is §7.4 instead and the peers are not read for the tail (they
are bumped, so a stale writer is fenced, and must hold nothing above `P` for the
database; a peer that does is answered with `Truncate{P'}` after the bucket replay).

### 5.2 Algorithm

```
writer (epoch e)                         peer_a          peer_b          peer_c
  | Bump{db,e} ------------------------>  |               |               |
  | Bump{db,e} ---------------------------------------->  |               |
  | Bump{db,e} -------------------------------------------------------->  |
  |  <------------- BumpAck{last=(3,120)} |               |               |
  |  <------------------------------- BumpAck{last=(3,118)}               |
  |      (majority reached; c may answer later or never)                  |
  | donor = argmax (last_epoch,last_lsn) = peer_a                         |
  | ReadTail{db,e,from=P+1} ------------> |                               |
  |  <-------------------- Tail{P+1..120} |                               |
  | verify crc of every record, contiguity, lsn > P                       |
  | Append{db,e,from=P+1 or first divergent, truncate_above} --> b (and c)|
  |  <---------------------------------- AppendAck{upto=120}  |           |
  | Append{db,e,from=121,[Noop]} --> a, b, c                              |
  |  <-- AppendAck{121} from a majority                                   |
  | replay P+1..120 into memtable; HLC = max(wall, m.max_ts, tail ts)+1   |
  | manifest PUT (epoch e, same segments, ledger_position unchanged)      |
  | open                                                                  |
```

Precisely:

1. Send `Bump{db, e}` to every peer in the set. Wait until a majority answer `BumpAck`.
   Any `Fenced` reply means a higher epoch exists: abort and report to the CP. Peers that
   answer later are handled as stragglers (step 7).
2. From the replies take `(last_epoch, last_lsn)` per peer. The **donor** is the replier
   with the highest pair in lexicographic order; ties are broken by peer order in the
   lease (any tied donor is equivalent, §5.4).
3. If the donor's `last_lsn <= P`, the tail is empty: skip to step 8.
4. `ReadTail{db, e, from = P+1}` from the donor. Verify each record's crc, that lsns are
   contiguous from `P+1`, that `db_id` matches, and that record epochs never decrease
   along the tail. Any failure: the donor is excluded and step 2 is repeated with the
   remaining repliers; if no majority-backed donor is left, recovery **fails closed**
   (the database stays closed and the CP is told; §5.5).
5. **Drop an incomplete trailing run.** Let `donor_last` be the donor's last lsn. If the
   tail ends in `TxnBatch` fragments without their `frag_final` record, lower
   `donor_last` to the lsn before that run's first fragment; the run is neither copied
   nor replayed, and every peer holding it (the donor included) gets step 6's `Append`
   with `d <= donor_last + 1`, whose `Discard` voids it. This is correct because the
   writer acknowledges a transaction only after its final fragment's quorum ack, so a run
   without its final fragment was never acknowledged to any client.
6. For each replying peer, find the first lsn `d > P` at which its entry differs from the
   donor's tail up to `donor_last` by `(record epoch, crc)`, is missing, or lies above
   `donor_last` (the peer reports its entries in reply to a `ReadTail` of its own; the
   writer compares). Send `Append{db, e, from_lsn = d, records = donor[d..=donor_last],
   truncate_above = true}`. The peer writes `Discard{from_lsn = d}` if it held records
   `>= d`, then the records, fsyncs and acks. Copied records keep their **original**
   `epoch`, `lsn` and `commit_ts`; only the message is in epoch `e`.
7. Append one `Noop` record at `lsn = donor_last + 1` in epoch `e` to every peer, and wait
   for a majority ack. This record is what makes the copied tail committed under epoch
   `e` (§5.4) and gives every later recovery a peer whose last record epoch is `e`.
   Stragglers that answer `Bump` late are caught up by the same steps against the donor
   before they are counted toward later acks.
8. Replay the tail `P+1 ..= donor_last` into the memtable in lsn order, applying each
   `TxnBatch` at its recorded `commit_ts` and each `SeqReserve` to the sequence counters
   (04). Fragmented transactions are applied as one run when their final fragment is
   reached.
9. Seed the HLC to `max(wall clock, m.max_commit_ts, max commit_ts in the tail) + 1`
   (brief, *Identifiers and timestamps*). The next lsn is `donor_last + 2`.
10. Write the manifest (conditional PUT against `m`) with `epoch = e`, unchanged segments
   and `ledger_position`, so the manifest fence moves to `e` (§4.2). If the PUT fails
   because the version moved, another writer exists: close and report. Skipped on a fresh
   database, where the initial manifest 02 §4.5 just created already carries `e`.
11. Open: admit sessions.

### 5.3 Divergent records

A peer may hold records above the committed prefix that the donor does not have or has
differently: appended by an earlier writer to a minority before it lost its lease. Step 5
replaces them. The peer never deletes bytes in place; `Discard` voids them in the index and
the segment is reclaimed by rotation (§6.2).

### 5.4 Correctness argument

Write `C` for the set of records any client was told committed (with `synchronous_commit
= on`). A record is in `C` only if a majority acked it in its epoch.

*Every record in `C` above `P` is on the donor.* Let `r` be such a record, appended in
epoch `e_r` at lsn `l_r`, acked by majority `M_r`. Every later epoch's recovery bumped a
majority, which intersects `M_r`; by induction over epochs (each recovery picks a donor
whose log contains `r` and copies it, before appending anything in its own epoch), every
peer that holds a record from an epoch `> e_r` also holds `r` at `l_r`, and peers of `M_r`
hold `r` itself. Our bump majority `M` intersects `M_r` in some peer `q`. The donor's last
`(epoch, lsn)` is `>=` `q`'s, which is `>= (e_r, l_r)`. If the donor's last epoch equals
`e_r`, its log is that epoch's single writer's sequence and has length `>= l_r`, so it
contains `r`. If it is greater, the donor holds a record from a later epoch and so, by the
induction, holds `r`. This is the Raft log-matching and leader-completeness argument in
Neon's safekeeper form; the `Noop` of step 7 plays the role of Raft's "commit only an entry
of the current term": a copied tail from an earlier epoch is never considered committed by
virtue of being on a majority alone, only by the new epoch's `Noop` above it.

*Records above the committed prefix.* The donor may hold records that were never acked to
any client (appended to a minority, then the writer died). Keeping them is safe: no client
was told they failed either — the client saw a connection error, the outcome PostgreSQL
gives for a commit interrupted by a crash, and the outcome is "committed" or "not", either
of which is a legal answer. Dropping them would be equally safe. **Decision: keep them**,
as Neon's safekeepers do: the donor's whole tail is copied and becomes committed with the
`Noop`. It is simpler (no second boundary to find), and it is the only choice that does not
need to know which records were acked, which no peer knows.

*Replayed state equals committed state.* The tail is replayed above `P`, and everything
`<= P` is in the manifest's segments (02's flush invariant), so after step 8 the memtable
plus segments hold every record of `C`, plus possibly kept unacked records, in lsn order.

### 5.5 Edge cases

**A peer with a stale epoch.** A peer down during epoch `e-1` holds an older
`promised_epoch` and an older tail. It is bumped like the others; its `(last_epoch,
last_lsn)` is lower, so it is never the donor over a peer that saw epoch `e-1`; step 6
overwrites its divergent suffix. If it answers only after the majority, it is a straggler.

**Same `(epoch, lsn)`, different crc.** Impossible under the protocol: an lsn in an epoch
is assigned once by the single writer of that epoch, records are crc'd by the writer
before sending, and a peer verifies the crc before writing. Seeing it means a bug, a disk
fault that passed crc, or two writers in one epoch (a CP bug). The writer **fails
closed**: recovery aborts, the database stays closed, the event is logged with both
records' bytes and raised as a page to the operator; no automatic choice is made.

**The donor dies mid-copy.** If it dies during `ReadTail`, the writer has a partial
verified tail; it discards it and repeats from step 1 (bumping is idempotent). If it dies
after the read, the writer already holds the full tail in memory and continues copying to
the others; if fewer than a majority are now reachable, step 7 cannot complete and the
writer waits or enters degraded recovery (§7.2: force flush is possible because the tail is
in memory, replayed into a memtable that is flushed to a segment and a manifest written
with `durability: bucket-direct`). A writer that crashes mid-copy leaves peers with partial
copies; the next recovery treats them as ordinary tails, all of whose records are prefixes
of the donor's at their original `(epoch, lsn)`, so the argument of §5.4 is unchanged.

**No majority answers `Bump`.** Recovery cannot read a trustworthy tail. The database does
not open (it does **not** enter degraded mode, because the unflushed tail may be on the
unreachable peers). The CP is told; the wait is bounded by the CP's lease policy (§13).

## 6. Truncation

### 6.1 Reporting

After a flush's manifest PUT succeeds with `ledger_position = (e, F)`, the writer sends
`Truncate{db, e, upto_lsn = F}` to every peer of the set. The message is advisory and
idempotent; it is re-sent with the next flush if lost. A peer records `flushed_lsn = F` in
its index; if `F` is above its `last_lsn` for the database (records it never received were
made durable by the manifest, §8), it also sets `last_lsn = F`, so the writer's next
ordinary `Append` at `F + 1` passes the gap check of §3.1. The index entry is persisted lazily (at the next fsync window, in the next `Checkpoint`); losing it
in a crash only delays reclamation, because recovery reads only above the manifest's
position.

### 6.2 Rotation and segment deletion

A shared segment is deletable when, for every database with records in it, all those
records have `lsn <= flushed_lsn` or are voided by a `Discard`. Each segment keeps, in the
index, the highest lsn it holds per database; the check is a scan of that small map. At
rotation the new segment starts with one `Checkpoint` per database still held, so deleting
old segments never loses a `promised_epoch`. A database whose last record was flushed and
whose writer closed is dropped from the index once its records are all reclaimable, and
its `promised_epoch` survives in the next checkpoint for *default* 7 days so that a stale
writer that wakes late is still fenced; after that the manifest fence alone stops it.

**Consequence of the shared log.** One slow database pins every segment that contains any
of its unflushed records. The bound is the flush age (§6.3): a database cannot hold
unflushed records older than its age budget plus one flush duration, so segments older
than that become deletable. A writer that dies without flushing pins segments until its
database's next recovery copies and flushes; the CP's lease policy (§13) bounds that.

### 6.3 What bounds peer disk

Peer disk holds the sum over databases of unflushed bytes, plus at most one segment's
rotation slack, plus voided bytes not yet rotated. The flush triggers (02, *target* 64 MB
or 30 s) bound each open database's share; the 30 s age bound is what makes the total "the
last few tens of seconds of writes across the region" rather than a function of tenant
count.

### 6.4 Backpressure

When a peer's data volume passes *default* 80 % full, it answers new `Append`s with
`Full{disk_fill_pct}` (it still accepts `Bump`, `ReadTail`, `Truncate`, and appends for a
database whose `from_lsn` continues an in-progress recovery). A writer that receives `Full`
treats that peer as not acking and **forces a flush** of the database, which lets the peer
truncate. If two peers are full, the database's commits stall; after the 2 s degraded
threshold it enters `bucket-direct` (§7). At *default* 95 % a peer stops accepting all
appends. These thresholds are defaults, not measurements.

## 7. Durability modes and transitions

### 7.1 The three modes

| Mode | Where unflushed committed records live | Commit latency floor | Set by |
|---|---|---|---|
| `quorum` | 2 of 3 ledger peers, fsync'd | network RTT + one peer fsync | normal operation |
| `bucket-direct` | the bucket, `tenants/<db>/log/` objects | one object-store PUT (*not measured*) | fewer than 2 peers ack |
| `single` | one local fsync'd file | one local fsync | lease names a set of one |

The manifest's `durability` field (02 §4.2) holds the current mode and an interval
history: `durability: {mode, history: [{mode, from_lsn, from_ts, to_lsn?, to_ts?}]}`,
newest last, with `to_lsn`/`to_ts` absent on the open interval. The invariant (brief, *Memtable
and flush*): **at any instant, the unflushed records of a database are in exactly one
place**, named by the manifest's current mode.

### 7.2 Entering `bucket-direct`

Trigger: appends pending and the committed lsn has not advanced for *default* 2 s (fewer
than two peers acking). The writer:

1. Stops taking new commits into the ledger path (new commits queue behind the token).
2. **Forces a flush.** The memtable contains every transaction applied to it: synchronous
   commits after their quorum ack, and `synchronous_commit = off` commits before theirs
   (§8). The flush does **not** wait for the quorum (there is none); its
   `ledger_position.lsn` is the memtable's `max_lsn` at the cut, floored at the previous
   manifest's position (02 §5), and the manifest PUT itself makes those records durable. Batches sent but not acknowledged **and not
   yet applied to the memtable** (synchronous commits waiting for their ack) are held
   aside.
3. Writes the flushed segment and the manifest with `durability = bucket-direct`, the
   new interval starting at `from_lsn = F + 1` where `F` is the flushed lsn, and
   `ledger_position = (e, F)`.
4. After that manifest PUT, re-submits the held-aside batches, in lsn order and with
   their **original** lsns and `commit_ts`, through the bucket path: they become the first
   log object(s), and their sessions are acknowledged as each PUT succeeds. `off` commits
   already in the memtable need nothing: the flush covered them.
5. From then on, commits are group-committed: batches arriving within a window (*default*
   10 ms) are written as one object `tenants/<db>/log/<epoch>.<from-lsn>.<to-lsn>` with
   `PutMode::Create` (so a stale writer cannot overwrite one), each record in §2.2
   framing. A commit is acknowledged after the PUT succeeds.

Held-aside batches may also exist on a minority of peers. That is harmless: the manifest
now says `bucket-direct` from `F+1`, so recovery never reads the peers for those lsns
(§7.4), and the next flush moves `ledger_position` past them, after which `Truncate` lets
the peers drop them. Conversely a peer may later receive a re-sent `Append` whose lsn is at
or below a `Truncate` it already processed (an `off` record flushed before its quorum ack):
it acknowledges without writing, the same idempotent answer as §3.6.

### 7.3 Leaving `bucket-direct`

While degraded, the writer probes the set with `Bump{db, e}` every *default* 1 s. When two
peers answer: force a flush (every log object's records are in the memtable, so the flush
covers them), write the manifest with `durability = quorum`, the interval closed at
`to = (e, F')`, and `ledger_position = (e, F')`; then resume ledger appends at `F' + 1`,
the peers having been bumped. The ledger does **not** delete log objects: 02's sweep
(§9.4) deletes `log/` objects whose `to-lsn` is at or below the current manifest's
`ledger_position.lsn` and whose `LastModified` is older than the grace period, and the
manifest keeps no list of them (recovery finds them by LIST, §7.4). Rejected: deleting
them in the flush's cleanup, because one sweep with a grace period already handles crash
leftovers and a second deletion path adds nothing.

### 7.4 Recovery when the manifest says `bucket-direct`

List `tenants/<db>/log/` for objects whose `from-lsn > P` (names sort by epoch then lsn;
the listing is filtered by parsing), verify each object's records (crc, contiguity),
replay them in lsn order, seed the HLC from their maximum `commit_ts`, and continue in
`bucket-direct` — or, if a peer majority answers `Bump`, immediately leave it (§7.3). A
gap in the lsn sequence across objects is impossible (each object is acknowledged only
after its PUT, in order) and fails closed.

### 7.5 Why transitions only at flush boundaries

Because each transition is a flush followed by a manifest write that names the new mode,
the records above `ledger_position` were all written under one mode, and recovery reads
exactly one source: the peers or the log objects (or the local file), never a merge of
two. Rejected: switching mid-stream with a per-record "which source" map, which would make
recovery merge two partially-written sources whose failure behaviours differ (a peer's
acked records vs. a PUT's atomic object), for the saving of one flush per transition.

How long a database may stay degraded before the CP moves it to a node with a healthy
quorum is a CP policy and open (00 §14).

## 8. `synchronous_commit = off`

With `off` (per session, 00 §3.1), the commit path hands the batch to the `LedgerClient`
(which assigns its lsn and starts the appends), then applies it to the memtable, advances
`last_committed_ts`, releases the token and acknowledges the client — without waiting for
the quorum ack. What still holds: lsn order equals commit order, so the ledger's committed
prefix is always a prefix of commit order. A crash loses at most a suffix of `off`
commits, never one in the middle; and an `on` commit from another session waits for its
own lsn's quorum, which by the prefix rule implies every earlier `off` commit is durable
too. What is lost: a crash of the writer node and a minority of peers loses `off` commits
not yet on a majority; in the worst case (all appends failing and the database then
closing before a flush) the loss is bounded by the flush interval, which is the recovery
point 00 states.

One wrinkle: an `off` commit is in the memtable before it is durable, so a flush can put a
not-yet-acked record into a segment. Decision: **the flush never waits for the quorum.**
The manifest PUT makes the memtable's contents durable by itself, so the flush's
`ledger_position.lsn` is the memtable's `max_lsn` at the cut, floored at the previous
manifest's position so it never moves backwards (02 §5), even when that
exceeds the committed lsn; a peer that later receives an append at or below a processed
`Truncate` acknowledges it without writing (§7.2). Rejected: waiting for the committed lsn
to reach the memtable's highest lsn before the manifest PUT, because it deadlocks on
degraded entry (§7.2), whose forced flush runs precisely because no quorum exists.

## 9. Single-node mode

The lease's set is `{local}` and the peer is the engine daemon's own in-process peer
service writing the §2 log under its data directory. Majority is 1 of 1: an append is
acknowledged after the local `sync()`. Recovery is §5 with one peer. The manifest's
mode is `durability: single` for the whole history, and the product surface (00 §4.2 last
paragraph, §11 item 3; `docs/onprem-install.md`) states plainly that a lost disk loses
commits since the last flush (*target* at most 30 s). Degraded mode does not apply: with
the only peer down, the database is down. Moving a single-node database to a three-peer
set is a membership change (§1.3) and so a flush plus a new epoch.

## 10. Observability

Exact counters and histograms, exposed as 00 §6 requires (control socket and Prometheus).

| Scope | Series | Kind |
|---|---|---|
| per database (writer) | `ledger_lag_bytes`, `ledger_lag_ms` (oldest unflushed record age) | gauge |
| | `ledger_acks_outstanding` (batches appended not yet committed) | gauge |
| | `ledger_commit_wait_seconds` (append to quorum ack) | histogram |
| | `ledger_durability_mode` (0 quorum, 1 bucket-direct, 2 single) | gauge |
| | `ledger_degraded_intervals_total`, `ledger_degraded_seconds_total` | counter |
| | `ledger_fenced_total`, `ledger_recoveries_total`, `ledger_recovery_seconds` | counter, histogram |
| | `ledger_peer_acked_lsn{peer}` | gauge |
| per peer | `peer_bytes_held`, `peer_databases_held`, `peer_segments` | gauge |
| | `peer_fsync_seconds`, `peer_window_records` | histogram |
| | `peer_disk_fill_ratio`, `peer_full_rejections_total` | gauge, counter |
| | `peer_fenced_replies_total`, `peer_crc_failures_total` | counter |

## 11. Rust type sketches

Sketches, not code; names are fixed, signatures may change in implementation.

### 11.1 Record

```rust
pub enum RecordKind { TxnBatch, SeqReserve, Noop }   // + peer-local Bump, Discard, Checkpoint

pub struct LedgerRecord {
    pub db_id: DbId,
    pub epoch: Epoch,          // original appending epoch
    pub lsn: Lsn,
    pub commit_ts: CommitTs,
    pub kind: RecordKind,
    pub payload: Bytes,        // TxnBatch / SeqReserve layouts: §2.2
    pub crc32c: u32,           // over header-after-crc and payload
}
```

### 11.2 Client (in the engine daemon)

```rust
pub struct Lease { pub db: DbId, pub epoch: Epoch, pub peers: Vec<PeerAddr> }

pub struct Recovered {
    pub tail: Vec<LedgerRecord>,      // above ledger_position, lsn order
    pub next_lsn: Lsn,
    pub max_commit_ts: CommitTs,
    pub mode: DurabilityMode,
}

// Async methods return `BoxFuture` by hand (06 §2.1); `async fn` is written here for
// brevity only.
pub trait LedgerClient: Send + Sync {
    /// Run §5 (or §7.4) and leave the client ready to append.
    async fn recover(&self, lease: &Lease, manifest_pos: LedgerPos,
                     mode: DurabilityMode) -> Result<Recovered, LedgerError>;
    /// Assign the next lsn(s) and start replication; the handle resolves at
    /// quorum ack (or log-object PUT in bucket-direct).
    fn append(&self, kind: RecordKind, commit_ts: CommitTs, payload: Bytes)
        -> Result<AppendHandle, LedgerError>;
    /// After a successful manifest PUT.
    fn report_flushed(&self, upto: Lsn);
    fn committed_lsn(&self) -> Lsn;
    fn mode(&self) -> DurabilityMode;
}

pub enum LedgerError { Fenced { epoch: Epoch }, NoQuorum, Corrupt(String), Closed }
```

`append` is synchronous up to lsn assignment so the commit path (04) fixes order without an
await; the `AppendHandle` is a future of `Result<Lsn, LedgerError>`. Mode transitions are
driven by the client, which asks the database (through a callback that 04 and 02 provide)
to force a flush and write the manifest.

### 11.3 Peer service

```rust
pub trait LedgerPeer: Send + Sync {   // same convention as §11.2
    async fn bump(&self, db: DbId, epoch: Epoch) -> Result<BumpAck, PeerError>;
    async fn append(&self, req: Append) -> Result<AppendAck, PeerError>;
    async fn read_tail(&self, db: DbId, epoch: Epoch, from: Lsn)
        -> Result<TailStream, PeerError>;
    async fn truncate(&self, db: DbId, epoch: Epoch, upto: Lsn) -> Result<(), PeerError>;
}
pub enum PeerError { Fenced { promised: Epoch }, Full { fill_pct: u8 },
                     Gap { last_lsn: Lsn }, Conflict, Io }
```

### 11.4 What the crate needs from `Env` (06 owns `Env`)

```text
// The subset of 06 §11's Env this crate calls; not a separate trait.
env.clock()          // Clock: timeouts, backoff, HLC wall part
env.rng()            // Rng: backoff jitter
env.spawn()          // Spawn: peer write loop, per-connection tasks
env.net()            // Net::connect / listen -> ByteStream (futures::io AsyncRead +
                     //   AsyncWrite); mTLS or loopback; Framed<S> for §3.2 frames
env.disk(DiskRoot::Ledger)   // Disk / DiskFile: append, sync(), read_at, list,
                             //   remove, truncate (torn tail), punch_hole
env.object_store()   // get, get_range, put (PutMode::Create), list, delete:
                     //   bucket-direct log objects
```

The simulation `Env` implements `Net` with its partitionable network and `Disk` with its
crashable disk, including losing un-`sync()`ed bytes on crash.

## 12. Test plan under the simulation harness

Every test below runs under the deterministic simulation `Env` (06, D18) with a seed; a
failing seed is a reproduction and is committed to the private corpus with the defect it
found (D18). Each run checks the **property**: every commit acknowledged to a client with
`synchronous_commit = on` is present, at its `commit_ts`, in the database state after any
later successful open; no two commits share an lsn; replayed state equals the state an
oracle builds from the acknowledged commit log.

| Scenario | What is injected |
|---|---|
| Partition sweep | For each protocol step (§3 append, §5 steps 1–11, §7.2 steps 1–5, §7.3), partition every subset of `{writer, a, b, c}` at that step; heal after a random delay |
| Writer crash between ack and apply | Crash after the quorum ack, before the memtable apply and client ack; the recovering writer must replay the batch (the client may or may not have seen success) |
| Two writers, consecutive epochs | Writer `e` keeps running while writer `e+1` recovers; check `e` gets `Fenced` and its manifest PUT fails, and nothing `e` acked is lost |
| Divergent tails | Writer appends to one peer only, crashes; next writer recovers with that peer in and out of its majority |
| Degraded enter/leave with crashes | Crash the writer at each step of §7.2 and §7.3; crash peers while degraded; recover from each intermediate manifest |
| Peer disk faults | Lose un-fsynced bytes, torn last record, fsync error (peer must stop) |
| Duplicated and reordered messages | Duplicate every `Append`, delay `AppendAck`s past the timeout, reorder across connections |
| Backpressure | Fill a peer past 80 % and 95 %; check forced flushes and degraded entry |
| Membership change | Change the set with and without a completed flush (§1.3 step 5) and crash at each step |
| Single node | Crash with un-fsynced writes; recover with the set of one |
| Crc conflict | Corrupt a record such that two peers disagree at one `(epoch, lsn)`; recovery must fail closed |

Unit tests outside the simulation cover framing round-trip, crc, torn-tail scan and index
rebuild on fixed byte inputs.

## 13. What this document does not decide

- The **values** of the group fsync window (*target* 1 ms), the bucket-direct group window
  (*default* 10 ms), the degraded threshold (*default* 2 s), timeouts, the pipelining
  window, segment rotation size and age, and the 80 %/95 % disk thresholds. P0's ledger
  benchmarks and the fencing drill replace them.
- How often the CP changes a database's peer set, and how it chooses peers beyond "least
  held bytes"; P3 with the CP.
- How long a database may run degraded, or wait without a peer majority at recovery,
  before the CP moves it (00 §14).
- **Cross-region ledgers** (a peer set spanning regions, for regional failure); not in any
  phase yet.
- Whether peers **compress** records (the `flags` bit is reserved); decided from measured
  peer bandwidth and disk.
- PITR's use of ledger records beyond the flushed position (00 §4.7) — whether truncated
  records are first archived to the bucket — is P2 with branches and PITR.
- Rolling upgrades across record `version` and protocol major versions.
- The node PKI (issue, rotation, revocation) the transport relies on, which is Runlot's.
