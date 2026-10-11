# 05 — The catalog: internal relations, OIDs, schema versions and the pg_catalog projection

> Kind: **design**. Status: **pre-implementation** (2026-10-11). Fills the gap left by
> docs/00-overview.md D4 (own upper half, own catalog), §7.2 (types) and §7.3 (catalog and
> introspection) for phase P0. Depends on 01 (row value format, column-id tagging, key
> prefixes) and 04 (write buffer, writer token, transactional DDL, sequences, GC
> watermark); both are written in parallel, so this document cites their topics rather
> than their section numbers. Depended on by 02 (the manifest's `dropped_ranges` field,
> §7.3 here, and per-table statistics, §8.2), 04 (`schema_version` bump at DDL commit) and
> 06 (`colibri-catalog` crate contents). Phases other than P0 are marked where they occur.

## 1. Principles

**One internal catalog, stored as ordinary rows.** Colibri's catalog is a set of internal
relations (`c_*`) with their own column sets, chosen for what the engine needs rather than
for what PostgreSQL happens to store. Each lives in the database's single KV space under
a fixed OID below 16384 (§2), its rows encoded with 01's row value format, written through
04's write buffer like any user row, versioned by `commit_ts` like any user row, and
compacted like any user row. Nothing about the catalog is special to storage: DDL is a
transaction that writes catalog rows, so transactional DDL (04), PITR and branches (00
§4.7) apply to the schema for free. Rejected: a catalog blob in the manifest (00 §4.1
names a "catalog root"; making it a blob would put every DDL on the manifest's conditional
PUT path and break transactional DDL), and PostgreSQL's own catalog layouts stored
verbatim (they carry heap-specific columns, `relfilenode`, `relfrozenxid`, `attcacheoff`,
that have no meaning here, and would bind our storage to PostgreSQL's catalog evolution).

The "catalog root" of 00 §4.1 is therefore not a pointer to anything: it is the
`schema_version` and `catalog_format_version` copied from `c_meta` into the manifest at
each flush (02 owns the field), so a tool can read a manifest and know which catalog
generation its segments carry without opening the database.

**`pg_catalog` and `information_schema` are projections, never stored.** Every
`pg_catalog.*` and `information_schema.*` relation in §8 is a virtual table whose rows are
computed on scan from the in-memory `CatalogSnapshot` (§5) the statement is bound to. No
KV key ever carries a PostgreSQL catalog OID as its prefix. The compatibility surface is
**PostgreSQL 18's column set** for each projected view: column names, order, types and
type modifiers match PG 18 so that `SELECT *` and positional access in ORM introspection
code behave identically; values are derived where we have the information and constant
where we do not (§8.4 lists the constants). Rejected: materialising `pg_class` and friends
as real tables kept in sync by triggers (two sources of truth; every DDL writes twice).

**What "listing what exists" means for `pg_proc`.** 00 §7.3 says `pg_proc` lists what
exists. Built-in functions, operators and types are not catalog rows: they are static
tables compiled into the binary (§9), keyed by PostgreSQL 18's OIDs, and `pg_proc`,
`pg_operator` and `pg_type` scans concatenate the static registry with the user rows from
`c_proc` and (P2) `c_type`. A function appears in `pg_proc` if and only if the analyzer
can call it; a PostgreSQL built-in that Colibri does not implement is absent, so
introspection never advertises a function that then fails with `42883`.

## 2. The internal relations

### 2.1 OID block

The internal relations and their catalog indexes take the fixed block **9000–9099**.
Relations use 9000–9049, catalog indexes 9050–9099. The block was chosen so that no
internal OID coincides with any OID that PostgreSQL 18 hands out at bootstrap: PG's
hand-assigned OIDs (1–9999) leave 8000–9999 for in-development patches and ship nothing
there in a release, while the range 1000–1099 suggested in the brief collides with PG's
array types (`_bool` is 1000, `_int4` 1007, `_text` 1009). An internal OID never appears
in any projection, but a unique number makes every log line, key dump and `regclass`
error unambiguous. Rejected: 1000–1099 (the collision above), and 10000–16383 (PG's
genbki and initdb ranges, also occupied).

| OID | Relation | Holds | Primary key (KV user key) |
|---|---|---|---|
| 9000 | `c_meta` | singleton: OID counter, versions | `(id int2)`, always `1` |
| 9001 | `c_database` | the one database row | `(oid)` |
| 9002 | `c_namespace` | schemas | `(oid)` |
| 9003 | `c_class` | tables, indexes, sequences, views | `(oid)` |
| 9004 | `c_attribute` | columns | `(rel_oid, attnum)` |
| 9005 | `c_type` | user types (P2) | `(oid)` |
| 9006 | `c_index` | index definitions | `(index_oid)` |
| 9007 | `c_constraint` | pk, unique, check, fk, not null | `(oid)` |
| 9008 | `c_sequence` | sequence parameters | `(seq_oid)` |
| 9009 | `c_proc` | user functions | `(oid)` |
| 9010 | `c_role` | roles | `(oid)` |
| 9011 | `c_auth` | SCRAM verifiers | `(role_oid)` |
| 9012 | `c_role_member` | role membership | `(role_oid, member_oid)` |
| 9013 | `c_privilege` | grants | `(object_kind, object_oid, sub_id, grantee_oid, grantor_oid)` |
| 9014 | `c_depend` | dependency edges | `(dep_kind, dep_oid, dep_sub, ref_kind, ref_oid, ref_sub)` |
| 9015 | `c_setting` | `ALTER DATABASE/ROLE ... SET` | `(role_oid, name)` |
| 9016 | `c_description` | `COMMENT ON` | `(object_kind, object_oid, sub_id)` |

| OID | Catalog index | On | Key columns | Unique |
|---|---|---|---|---|
| 9050 | `c_namespace_name` | `c_namespace` | `(name)` | yes |
| 9051 | `c_class_name` | `c_class` | `(namespace_oid, name)` | yes |
| 9052 | `c_attribute_name` | `c_attribute` | `(rel_oid, name)` | yes, live columns only (§6.3) |
| 9053 | `c_type_name` | `c_type` | `(namespace_oid, name)` | yes |
| 9054 | `c_index_table` | `c_index` | `(table_oid)` | no |
| 9055 | `c_constraint_rel` | `c_constraint` | `(rel_oid)` | no |
| 9056 | `c_constraint_name` | `c_constraint` | `(namespace_oid, name)` | no (PG allows duplicates across tables) |
| 9057 | `c_proc_name` | `c_proc` | `(namespace_oid, name, arg_types)` | yes |
| 9058 | `c_role_name` | `c_role` | `(name)` | yes |
| 9059 | `c_depend_ref` | `c_depend` | `(ref_kind, ref_oid, ref_sub)` | no |

Catalog keys follow 01 exactly (`IndexId = 0` for rows, the catalog index's OID for an
index), and catalog indexes are maintained by the same code as user secondary indexes.
Since the catalog is loaded whole into memory (§5), they serve the writer's uniqueness
checks (04) and rebuilding a historical snapshot without a full scan (§5.2). Column types
below are PostgreSQL names; `oid` columns use 01's `oid` tag (`0x17`, 4 bytes unsigned).

### 2.2 `c_meta`, `c_database`, `c_namespace`

`c_meta` holds `next_oid` (oid, next OID to hand out, from 16384), `schema_version` (int8,
incremented by every committed DDL transaction) and `catalog_format_version` (int4, layout
generation of the `c_*` relations, §4.2).

| `c_database` column | Type | Meaning |
|---|---|---|
| `oid`, `name`, `owner_oid`; `encoding`, `collation`, `ctype` | oid, text, oid; text | `oid` constant 5 (§11), name given by the CP; always `UTF8`, `C`, `C` (D20) |
| `db_id` | bytea | the 16-byte `DbId`, for cross-checking a branch's lineage |
| `conn_limit` | int4 | `-1`; stored, not enforced in P0 |

`c_namespace` is `(oid, name, owner_oid)`: `public` is 2200 (PG's value), user schemas take
OIDs from the counter. `pg_catalog` (11), `information_schema` and `pg_toast` (99) are static namespaces in the
binary, not rows (§4.1).

### 2.3 `c_class`

| Column | Type | Meaning |
|---|---|---|
| `oid` | oid | the `TableId` (or `IndexId`) used in KV prefixes |
| `namespace_oid`, `name`, `owner_oid` | oid, text, oid | |
| `kind` | char | `r` table, `i` index, `S` sequence, `v` view (PG's `relkind` letters) |
| `pk_index_oid`, `rowid_seq_oid` | oid | primary key index; hidden sequence feeding the hidden `rowid int8` column (01, 04, §2.8); each `0` if none |
| `persistence`, `truncated_at` | char, int8 | `p` (`t` for temporary tables, §2.11); `CommitTs` of the last `TRUNCATE`, `0` if never (§7.4) |
| `options` | text[] | `WITH (...)` reloptions as `key=value`, stored and reported, none interpreted in P0 |
| `view_query`, `view_check` | bytea, char | views only: serialized analyzed query (§6.5); `WITH CHECK OPTION` `n`/`l`/`c` |
| `row_version` | int8 | `CommitTs` of the last DDL touching this relation (§5.4) |

An index's OID is its `IndexId`: its KV prefix is `(table_oid, index_oid)`, the table's
rows are under `(table_oid, 0)`.

### 2.4 `c_attribute`

| Column | Type | Meaning |
|---|---|---|
| `rel_oid`, `attnum` | oid, int2 | stable column id, 1-based; the column id tag in 01's row value |
| `name` | text | for a dropped column: `........pg.dropped.N........` as PG writes it |
| `type_oid`, `typmod` | oid, int4 | PG type OID (user types P2); PG's `atttypmod` encoding (`varchar(n)` = n + 4, `-1` none) |
| `ndims`, `not_null` | int2, bool | array dimensions are reporting only, as in PG |
| `default_expr`, `default_oid` | bytea, oid | serialized analyzed expression (§6.5), empty if none; `pg_attrdef.oid` of it, `0` if none (§8.3) |
| `missing_value`, `has_missing`; `identity`, `generated` | bytea, bool; char, char | value for rows written before the column existed (§6.2); identity `''`/`a` ALWAYS/`d` BY DEFAULT; `''` (stored generated `s` is P2) |
| `is_dropped`, `is_hidden` | bool | `is_hidden`: the `rowid` column; never in `SELECT *` or any projection |
| `collation_oid` | oid | `950` (`C`) for collatable types, `0` otherwise (D20) |

The hidden `rowid` takes `attnum` `-2`, outside the user range, so that adding it to a
table never shifts user column numbers; PostgreSQL's system columns use negative numbers
the same way. It is excluded from every projection in §8.

### 2.5 `c_type`

Built-in types are static (§9). `c_type` exists in P0 with its key and its name index so
that the format does not change when user types arrive, but P0 rejects `CREATE TYPE` and
`CREATE DOMAIN` with `0A000`. Columns fixed now: `oid`, `namespace_oid`, `name`,
`owner_oid`, `kind` (`e` enum, `d` domain, `c` composite, `r` range), `array_oid` (the
implicit array type's OID, allocated alongside), `base_type_oid` and `base_typmod`
(domains), `not_null` and `default_expr` (domains), `enum_labels` (text[] in sort order,
with `enum_sort` float4[] for `ALTER TYPE ... ADD VALUE ... BEFORE`), `comp_rel_oid`
(composites: a `c_class` row of kind `c`, P2). The contents are P2 (00 §7.2).

### 2.6 `c_index`

| Column | Type | Meaning |
|---|---|---|
| `index_oid`, `table_oid` | oid | |
| `key_attnums`, `key_exprs` | int2[], bytea | key columns in order; `0` marks an expression column whose serialized expression is in `key_exprs` (P2, 01) |
| `include_attnums` | int2[] | `INCLUDE` columns, stored in the index value (01) |
| `directions`, `nulls_first` | bool[], bool[] | per key column: DESC; NULLS FIRST (defaults follow PG) |
| `is_unique`, `nulls_not_distinct`, `is_primary`; `predicate` | bool; bytea | partial index `WHERE`, serialized, empty if none |
| `is_valid` | bool | false while a `CREATE INDEX` backfill is in flight (§6.6), and false after a collation version change until `REINDEX` |
| `method`, `collation_versions` | text, text[] | `btree` in P0, `hnsw` P2; per key column: the version string of the collation whose sort key 01 encoded into the index key (empty string for `C`); recorded at build time |

A non-default `nulls_first` is encoded by 01 in the key bytes, so the flag is a format
input, not a planner hint. `collation_versions` exists for the same reason: 01 puts a
collation's sort key into the index key (D20), and a sort key is only as stable as the
collation version that produced it. On open, the daemon compares each index's recorded
versions with the versions the binary provides; a mismatch sets `is_valid = false`, the
planner stops using the index, `pg_index.indisvalid` reports it, and `REINDEX` rebuilds
it and records the new versions. This is PostgreSQL's `pg_depend.refobjversion` warning
turned into a hard rule, because here a stale sort key is a wrong answer rather than a
wrong order. In P0 every entry is the empty string, since only `C` exists.

### 2.7 `c_constraint`

| Column | Type | Meaning |
|---|---|---|
| `oid`, `namespace_oid`, `name`, `kind` | oid, oid, text, char | `p` primary key, `u` unique, `c` check, `f` foreign key, `n` not null (PG 18 catalogs NOT NULL constraints) |
| `rel_oid`, `attnums`, `index_oid` | oid, int2[], oid | constrained table and columns; backing index for `p`/`u`, referenced unique index for `f` |
| `check_expr`; `ref_rel_oid`, `ref_attnums` | bytea; oid, int2[] | `c` only; `f` only |
| `on_update`, `on_delete`, `match_type` | char | `a` no action, `r` restrict, `c` cascade, `n` set null, `d` set default; match `s` simple, `f` full |
| `deferrable`, `deferred`, `validated` | bool | stored; deferral semantics P1; `validated` false for `NOT VALID` |

**Foreign keys are stored and reported in P0 and enforced in P1.** P0 accepts
`REFERENCES` and `FOREIGN KEY`, writes the row, the `c_depend` edge and reports it in
`pg_constraint` and `information_schema.referential_constraints`, so introspection of a
migrated schema is exact; it does not check inserts or run actions. The ledger reason is
`unsupported` until P1, and the server emits a `NOTICE` on each `CREATE` so the gap is not
silent. Check constraints and NOT NULL are enforced in P0; unique and primary key are
enforced by 04's unique-check path.

### 2.8 `c_sequence`

| Column | Type | Meaning |
|---|---|---|
| `seq_oid`, `type_oid`; `start`, `increment`, `min`, `max` | oid; int8 | the sequence's `c_class` OID and `int2`/`int4`/`int8` |
| `cache`, `cycle` | int8, bool | stored and reported; 04's `SeqReserve` reserves `max(32, cache)` values |
| `owned_by_rel`, `owned_by_attnum`, `is_rowid` | oid, int2, bool | `OWNED BY`, identity and serial columns; `is_rowid`: the hidden per-table `rowid` sequence, excluded from projections |

The **live counter** is not in `c_sequence`. It is a separate row under the sequence's
own OID, key `prefix(seq_oid, 0) || ()` (empty user key), value `{last_value int8,
is_called bool, reserved_upto int8}` at `attnum` 1, 2 and 3 in 01's row format v1,
written by 04's `SeqReserve` outside transactions.
Splitting it keeps the transactional parameters (rolled back with an aborted `ALTER
SEQUENCE`) apart from the non-transactional counter (never rolled back), exactly the
split PostgreSQL has between `pg_sequence` and the sequence relation's own tuple.

A table without a primary key gets a hidden sequence (`is_rowid = true`, kind `S`, not in
any projection) created in the same DDL transaction; this is the per-table sequence 04
draws `rowid` values from. Rejected: a counter in `c_class` (it would make every insert a
catalog write and bump nothing useful).

### 2.9 `c_proc`

| Column | Type | Meaning |
|---|---|---|
| `oid`, `namespace_oid`, `name`, `owner_oid` | | |
| `language` | text | `sql` from P1; `plpgsql` in P2 |
| `arg_types`, `arg_names`, `arg_modes` | oid[], text[], char[] | modes `i`, `o`, `b`, `v` (empty if all `i`) |
| `arg_defaults` | bytea | serialized default expressions for trailing args |
| `return_type`, `returns_set`, `volatility`, `strict`, `security_definer` | oid, bool, char, bool, bool | volatility `i`, `s`, `v`; `security_definer` meaningful once privileges are enforced (P1) |
| `body_text` | text | the source as written (`prosrc`) |
| `body_ast` | bytea | for `sql` with `BEGIN ATOMIC`: serialized analyzed body (§6.5); empty for string bodies, which are analyzed at call time as PG does |

The `c_proc` relation, its key and name index exist from P0 so the format is fixed; `CREATE
FUNCTION ... LANGUAGE sql` is **P1** (00 §7.5, §12) and P0 rejects it with `0A000`, as it
does PL/pgSQL until P2.

### 2.10 `c_role`, `c_auth`, `c_role_member`

| `c_role` column | Type | Meaning |
|---|---|---|
| `oid`, `name` | oid, text | the bootstrap owner is 10 (PG's bootstrap superuser OID) |
| `superuser`, `inherit`, `create_role`, `create_db`, `can_login`, `bypass_rls` | bool | PG's role attributes; `create_db` is reported but `CREATE DATABASE` is a CP operation (§11) |
| `conn_limit`, `valid_until` | int4, timestamptz | limit stored, enforced P1; `valid_until` checked at SCRAM authentication in P0 |

`c_auth` is `(role_oid, scram_verifier text)` in PG's exact text form
`SCRAM-SHA-256$<iter>:<salt>$<StoredKey>:<ServerKey>`. `c_role_member` is `(role_oid,
member_oid, grantor_oid, admin_option, inherit_option, set_option)`, PG 16+'s options.

The verifier is in its own relation so that no projection of `c_role` can leak it;
`pg_authid` is not projected (§8.3).

### 2.11 `c_privilege`, `c_depend`, `c_setting`, `c_description`

| `c_privilege` column | Type | Meaning |
|---|---|---|
| `object_kind`, `object_oid`, `sub_id` | char, oid, int2 | kinds `r` relation, `s` sequence, `n` namespace, `f` function, `d` database, `t` type; `sub_id` = column for column privileges, else 0 |
| `grantee_oid`, `grantor_oid` | oid | grantee `0` = `PUBLIC` |
| `privileges`, `grant_option` | int4 | bitmasks in PG's `ACL_*` bit order (`a` INSERT = bit 0 … `m` MAINTAIN, PG 17+) |

The absence of any row for an object means the owner's default privileges, exactly as a
NULL `relacl` means in PostgreSQL; the first `GRANT` or `REVOKE` materialises the owner's
row, again as PostgreSQL does, so `relacl` text matches after any sequence of statements.

| `c_depend` column | Type | Meaning |
|---|---|---|
| `dep_kind`, `dep_oid`, `dep_sub` | char, oid, int2 | the dependent object (kind letters as in `c_privilege`, plus `i` index, `c` constraint, `v` view) |
| `ref_kind`, `ref_oid`, `ref_sub` | char, oid, int2 | the object depended upon |
| `deptype` | char | `n` normal, `a` auto, `i` internal (PG's letters, §7) |

`c_setting` is `(role_oid, name, value)`: `role_oid` `0` for `ALTER DATABASE ... SET`, else
the role; the GUC name and text value are validated against the static GUC registry at
write. `c_description` is `(object_kind, object_oid, sub_id, description)`, deleted with the
object.

Temporary tables (`persistence = 't'`) are P1. When they arrive they will not be KV rows
at all but a per-session overlay; the column is fixed now so the projection can report it.

## 3. OID allocation

`c_meta.next_oid` starts at **16384** and is advanced inside the DDL transaction that
needs OIDs: the transaction reads it through its write buffer, takes as many values as it
creates objects, and writes the new value back. DDL holds the writer token from its first
catalog write (04), so no other transaction can read the same counter value; if the
transaction aborts, the counter write is discarded with the buffer and the values are
handed out again to the next DDL. This is the only sense in which a value is "reused":
an OID that was ever **committed** is never issued again, because `next_oid` only grows
and a dropped object's OID is not returned to any pool. Since `TableId` and `IndexId` are
OIDs, never-reuse is what lets a dropped table's KV prefix be reclaimed lazily (§7.3)
without any risk that a new table writes under the same prefix. The 32-bit space allows
about 4.29 × 10⁹ objects per database; wraparound raises `54000` (program limit
exceeded) rather than wrapping, since a database that creates that many objects is a
defect report, not a workload.

One counter serves every kind of object, as PostgreSQL's does.

**Reserved values.** Two bands are never issued, and the allocator checks both as an
invariant even though reaching them would take billions of objects. `TableId 0` is
reserved by 01. `0xFFFF_FF00..=0xFFFF_FFFF` is 01's side-range band: `0xFFFF_FFFF` is the
overflow-chunk index id and the rest are reserved for later side ranges. Because
`IndexId` values are OIDs, an index whose OID fell in that band would collide with the
overflow chunks of every table, so `next_oid` stops at `0xFFFF_FEFF` and the wraparound
error above fires there.

`pg_class.oid`, `pg_type.oid`, `pg_proc.oid`, `pg_namespace.oid`, `pg_constraint.oid`
and `pg_roles.oid` are these OIDs verbatim. An ORM or driver that caches a table's OID or
a user type's OID (pgjdbc and Npgsql cache type OIDs; Prisma reads `pg_class.oid` during
introspection) keeps working across restarts, wakes, node moves and branches, because the
OID is a stored value, not something re-assigned at open.

**`regclass` and friends.** `'name'::regclass` resolves the possibly-qualified name with
the session's `search_path` against the statement's `CatalogSnapshot`, covering user
relations and the virtual `pg_catalog`/`information_schema` relations (which carry PG
18's OIDs, e.g. `'pg_class'::regclass = 1259`). `regclass` output maps an OID back to a
name, quoted and schema-qualified only when not visible on the `search_path`, as PG does.
Internal `c_*` relations are invisible to name resolution: `'c_class'::regclass` raises
`42P01`, and `9003::regclass` prints `9003`, as PG prints an OID with no `pg_class` row.
`regtype`, `regproc`, `regprocedure`, `regnamespace` and `regrole` resolve the same way
against the static registries plus the catalog rows. They are phase-1 types for this
purpose because every ORM's introspection query casts through them.

## 4. Bootstrap and catalog format

### 4.1 What is static and what is written

The internal relations describe themselves in the binary, not in rows. Their column ids,
types and keys are a static table in `colibri-catalog`; storage code needs nothing more
to decode a catalog row, and a self-describing catalog would only add a bootstrapping
loop (to read `c_attribute` you would need `c_attribute`'s descriptor). Also static: the
`pg_catalog` (11), `information_schema` and `pg_toast` (99) namespaces, the built-in type,
function, operator, cast and access-method registries (§9), the predefined roles
(`pg_database_owner`, `pg_read_all_data` and the rest of PG 18's list, with their PG OIDs),
and the GUC registry behind `pg_settings`. Rejected: writing rows for the internal
relations as PostgreSQL does at `initdb` (it would make the binary's descriptors and the
stored rows two sources of truth that a format migration must keep equal).

What is written is the per-database state. `CREATE DATABASE` is a CP operation (§11); the
daemon that takes the first lease on the new `DbId` runs the **bootstrap routine** before
accepting any session, and the routine writes one transaction, which becomes the first
`TxnBatch` in the database's ledger:

| Row | Content |
|---|---|
| `c_meta` | `next_oid = 16384`, `schema_version = 1`, `catalog_format_version = CURRENT` |
| `c_database` | `oid 5`, name from the CP request, `owner_oid 10`, `UTF8`, `C`, `C`, `db_id` |
| `c_namespace` | `public`, oid 2200, `owner_oid` = 6171 (`pg_database_owner`, PG 15+ behaviour) |
| `c_role` | oid 10, name from the CP request (the tenant's owner role), `superuser`, `can_login` |
| `c_auth` | the owner's SCRAM verifier if the CP request supplied one; none for trusted-peer only |
| `c_privilege` | `public` namespace: `USAGE` to `PUBLIC`, `CREATE` to owner (PG 15 default) |

Emptiness is `manifest.schema_version == 0` (02 §4.5 writes that initial manifest) and no
committed record in the ledger tail; bootstrap runs after 03's recovery and before any
session. A bootstrap that dies before its batch commits leaves exactly that state, so the
next lease holder runs it again. A database that has a manifest and ledger records but no `c_meta` row is corrupt
and refuses to open (`XX001`).

### 4.2 `catalog_format_version` and migration on open

`catalog_format_version` is an integer, starting at 1, that names the layout of every
`c_*` relation. A binary knows its `CURRENT` and the migration steps from every earlier
version, written as functions `fn(prev_catalog_rows) -> Vec<CatalogWrite>`. On open:

1. If the stored version equals `CURRENT`, proceed.
2. If it is lower, the writer (it holds the lease and opens the database exclusively)
   takes the writer token before admitting sessions and applies the steps in order as
   **one** DDL-like transaction: one ledger batch, one `schema_version` bump, one commit.
   A crash midway leaves the old version, and the next open repeats the whole migration.
3. If it is higher, the binary refuses to open with a clear error to the CP: catalog
   migration is **forward-only**, and an older binary never reads a newer catalog.

Because a migration is a transaction it is in the ledger, and a PITR restore to a point
before it brings back the old format, which the next open migrates again. Branches
migrate independently on their own first open. Rejected: migrating lazily per row on read
(every reader path would carry every historic layout forever), and migrating in a
separate offline tool (a cold database must be openable by any node at any version, G1
and G2).

## 5. The in-memory `CatalogSnapshot`

### 5.1 Shape and identity

A `CatalogSnapshot` is an immutable value behind an `Arc`: every catalog row of one
committed state, decoded into typed structs (§12) and indexed by OID and by
`(namespace, name)`, plus the resolved views of the static registries. Its identity is
`(schema_version, commit_ts)`: the `schema_version` from `c_meta` and the `commit_ts` of
the DDL that produced it. It is built in full at open (the catalog of a typical
application is kilobytes to a few megabytes; *not measured*) and, at each DDL commit,
derived from its predecessor by applying the committed transaction's catalog delta. The
delta is already decoded in memory (the transaction built it, §5.3), so derivation costs
a copy of the touched maps, not a rescan; an implementation may use persistent maps to
make it cheaper, which is an optimisation, not a contract. Memory is charged to the
database's budget (00 §6).

### 5.2 Which snapshot a statement binds against

The analyzer binds a statement against the catalog **as of the statement's own snapshot
timestamp**: the newest committed `CatalogSnapshot` whose `commit_ts <= snapshot_ts`.
The database keeps a short history of snapshots back to the one current at 04's GC
watermark (the watermark is the minimum of the active snapshots and the PITR horizon, so
this covers every active statement and the layout compaction asks for, §6.3), retired as
the watermark advances. A
REPEATABLE READ transaction that began before an `ADD COLUMN` therefore keeps seeing the
old column list for its whole life, which is consistent with the data it reads. In the
rare case where a snapshot older than the retained history is needed (PITR-style
`AS OF` reads, P2), the snapshot is rebuilt from the catalog rows at that timestamp,
which MVCC keeps.

This differs from PostgreSQL, where catalog lookups use a fresh catalog snapshot and
table locks stop concurrent DDL; Colibri has no table locks in P0. Rejected: binding to
the latest catalog (a REPEATABLE READ reader would decode old rows against a new
descriptor). The one observable consequence is handled at write time: a transaction whose snapshot predates a committed
DDL on a relation it then writes to raises `40001` at the write, because its bound
descriptor (columns, defaults, constraints) is stale. 04's first-write rule already
raises `40001` for stale keys; this extends it to stale descriptors, detected by
comparing the bound `Relation::row_version` (§2.3) with the newest one. READ COMMITTED
statements never hit it, because a writing statement takes the token before its
snapshot (04). Ledger reason `intended`.

### 5.3 A transaction sees its own uncommitted DDL

Choice: **re-derive a private snapshot per DDL statement**. When a transaction executes
DDL, the `CatalogWriter` (§12) records its changes in the write buffer (that is what makes
the DDL transactional) and also in a decoded `CatalogDelta` held in the transaction
state. After each DDL statement the transaction derives a private `CatalogSnapshot` =
bound committed snapshot + delta, and every later statement in the transaction binds
against that private snapshot until COMMIT or ROLLBACK. Savepoints keep a stack of
deltas, so `ROLLBACK TO SAVEPOINT` restores both the buffer (04's undo log) and the
private snapshot. At COMMIT the same delta derives the next shared snapshot (§5.1), and
`schema_version` is the committed `c_meta` value plus one.

Rejected: an overlay consulted on every lookup (every name resolution in every statement
of the transaction pays a second probe, and the overlay's merge rules duplicate the
snapshot's own indexing); and re-reading catalog rows through the write buffer at each
statement (correct but a full decode per statement). Derivation costs one copy per DDL
statement, and DDL statements are rare and already hold the token.

### 5.4 Plan invalidation

A prepared statement's plan records the `schema_version` it was planned at and the set of
OIDs it depends on with their `row_version`. At `Bind`/`Execute`, if the session's current
snapshot has the same `schema_version`, the plan is used as is. Otherwise each dependency's
`row_version` is compared; if none changed the plan is reused and re-stamped (a DDL on an
unrelated table does not re-plan), and if any changed or disappeared the statement is
**re-analyzed and re-planned silently**. If the re-planned result row description (column
count, names or type OIDs) differs from the one sent at `Parse`/`Describe`, the execute
fails with `0A000` `cached plan must not change result type`, PostgreSQL's error for the
same case; otherwise the client never notices. A dropped dependency surfaces as whatever
error the re-analysis raises (`42P01` and so on), as in PostgreSQL. Search-path changes
also force re-analysis (PG does the same since 9.3).

## 6. Columns

### 6.1 `attnum` is stable

A column's `attnum` is assigned once at `CREATE TABLE` or `ADD COLUMN` (max existing
attnum + 1, counting dropped columns) and never changes or is reused. It is the column-id
tag in 01's row value, so a row value written under any past schema decodes under any
later one: unknown ids are dropped columns, missing ids are columns added later.

### 6.2 `ADD COLUMN`

`ADD COLUMN` with no default, a NULL default, or a default that is constant after
analysis (a literal, an immutable or stable expression evaluated once at DDL time —
`now()` is stable and is evaluated once, as PG 11+ does) is **catalog-only**. The DDL
evaluates the default once, stores the expression in `default_expr` for future inserts and
the evaluated datum in `missing_value` with `has_missing = true`. On read, a row value
that lacks the column's id yields `missing_value` (or NULL without `has_missing`). This is
the "missing value" PostgreSQL 11 added (`attmissingval`), with the same meaning.

This requires one property of 01's row format, stated here as a dependency: a row value
must distinguish *the column did not exist when this row was written* from *the column is
NULL*. Either NULLs are encoded explicitly for every live column, or the row carries a
null bitmap over the attnums live at write time; whichever 01 chooses, an insert after
`ADD COLUMN x DEFAULT 5` with an explicit `NULL` must read back as NULL, not 5.

`missing_value` is retained until the table has been fully rewritten (a compaction that
rewrites every row with the column present, or a later full rewrite); clearing it is an
optimisation, not required in P0. `ALTER COLUMN ... SET DEFAULT` changes `default_expr`
only and never `missing_value`, as in PG.

`ADD COLUMN` with a **volatile** default (`random()`, `nextval()`, `gen_random_uuid()`,
`clock_timestamp()`) or with an identity or stored generated column **rewrites** the
table: under the token, inside the DDL transaction, every live row is read at the
transaction's snapshot and rewritten with the new column into the write buffer. The
rewrite's size is bounded by 04's buffer spill; a table too large for the spill limit
fails the DDL with `54000` and the documented workaround (add the column nullable, backfill
in batches, then `SET NOT NULL`).

### 6.3 `DROP COLUMN`

`DROP COLUMN` sets `is_dropped`, renames the column to PG's `........pg.dropped.N........`
placeholder, clears `not_null`, `default_expr` and `missing_value`, and removes the
column's index entries in `c_attribute_name` (so the name is free for a new column). It is
catalog-only. Readers skip the id; writers stop emitting it; compaction strips dropped ids
from the values it rewrites. Compaction never reads the catalog (06 §1.1): `colibri-db`
builds a `ColumnLayout` from the catalog snapshot at the compaction's GC watermark and
hands it to the `SegmentBuilder` (02 §3.4), and that layout names only the column ids
dropped at or below the watermark, so an older snapshot never loses a column it can still
see. Dropping a column that is part of an index or
constraint drops those with it under `CASCADE`, or fails under `RESTRICT` (§7). Dropping a
primary key column requires dropping the primary key first, as in PG.

### 6.4 `ALTER COLUMN TYPE` and `RENAME`

A type change is **catalog-only** when the cast is binary-compatible in both the value and
the key encoding: `varchar(n)` to `varchar(m)` with m ≥ n, `varchar` to `text`, `text` to
`varchar` without a length, `numeric(p,s)` to unconstrained `numeric`, and widening a
`varchar` length in an index key (01 encodes all of these as UTF-8 bytes or the same
numeric encoding). Every other change rewrites the table and every index containing the
column, under the token and the spill bound, exactly as volatile `ADD COLUMN`. `RENAME`
of a table, column, index, constraint, schema or sequence is catalog-only: it updates the
row and its name index entry. Stored expressions refer to columns by `attnum` (§6.5), so
renames never touch them.

### 6.5 Stored expressions

Defaults, check constraints, partial-index predicates, view queries and `BEGIN ATOMIC`
function bodies are stored as **serialized analyzed trees**: names already resolved to
OIDs and attnums, operators to PG operator OIDs, with a leading `expr_format_version`
byte. `colibri-catalog` stores the bytes opaquely; `colibri-sql` owns the encoding and
the deparser behind `pg_get_expr` and friends (§9.2). Rejected: storing SQL text and
re-analyzing on load, because the meaning of a stored default would then depend on the
loading session's `search_path` and on renames since — the reason PostgreSQL stores
`nodeToString` trees. The format is versioned through `catalog_format_version`: a change
to the tree encoding is a catalog migration (§4.2).

### 6.6 `CREATE INDEX`

`CREATE INDEX` in P0 writes the `c_class` and `c_index` rows with `is_valid = false`,
backfills index entries for every live row in the same transaction (bounded by the spill
like a rewrite), sets `is_valid = true` and commits. The planner ignores invalid indexes;
in P0 an invalid index is never visible outside its own transaction, but the flag is
there for `CREATE INDEX CONCURRENTLY` (P2, built as a background backfill under the
maintenance class). `CONCURRENTLY` is accepted in P0 and runs the non-concurrent path with
a `NOTICE`; ledger reason `intended`.

## 7. Dependencies and drop

### 7.1 Edges

`c_depend` holds one row per edge. The edges written in P0:

| Dependent → referenced | `deptype` | Written when |
|---|---|---|
| index → table (its columns) | `a` | `CREATE INDEX`, or `i` for a constraint's backing index (→ the constraint) |
| constraint → table / columns | `a` | any constraint |
| fk constraint → referenced table / columns and its unique index | `n` | `REFERENCES` |
| sequence → column | `a` (`OWNED BY`, serial) or `i` (identity) | `serial`, identity, `ALTER SEQUENCE OWNED BY` |
| column default → sequence | `n` | a default calling `nextval('s'::regclass)` |
| view → each table, column and function it references | `n` | `CREATE VIEW` |
| function → argument and return types | `n` | `CREATE FUNCTION` (user types P2) |

### 7.2 RESTRICT and CASCADE

`DROP` computes the closure of objects reachable through `c_depend_ref` from the target.
Edges of type `a` and `i` are always followed (an index goes with its table without
`CASCADE`). If any `n` edge is reached, `RESTRICT` (the default) fails with `2BP01` and a
`DETAIL` listing the dependents in PostgreSQL's wording; `CASCADE` drops the whole
closure and emits PG's `NOTICE: drop cascades to ...` lines. Every dropped object's
catalog rows, its `c_depend` rows in both directions, its `c_privilege` and
`c_description` rows are deleted (tombstoned) in the DDL transaction. `DROP ROLE` fails
with `2BP01` while the role owns objects or holds grants, as in PG; `REASSIGN OWNED` and
`DROP OWNED` are P1.

### 7.3 What happens to a dropped table's KV range

Choice: **a `dropped_ranges` entry in the manifest, used by compaction to discard keys
without tombstones.** `DROP TABLE` (and `DROP INDEX`) commits catalog changes only; no
data key is touched in the transaction. The committed snapshot no longer contains the
relation, so no new statement can reach the prefix. At the next manifest rewrite after
the drop commits, flush or compaction, the writer adds to the manifest from the pending
list the commit filled (02 §3.4 owns the field and the timing; this is what 05 needs):

`dropped_ranges` entries are `{prefix, dropped_at_ts, strict}` (02 §4.2). `prefix` is
`TableId` alone (8 hex digits: the whole table, every index and the overflow side range
`IndexId = 0xFFFF_FFFF` included) for `DROP TABLE` and `TRUNCATE`, or `TableId || IndexId`
(16 hex digits) for `DROP INDEX`; `dropped_at_ts` is the DROP's `CommitTs`; `strict` is
`false` except for `TRUNCATE`'s `< dropped_at_ts` rule (§7.4).

Compaction, when its watermark (04) is at or above `dropped_at_ts`, discards every
version under the prefix with `commit_ts <= dropped_at_ts` while writing its output, and
a segment wholly inside a dropped prefix is removed from the level without being read. An
entry is removed from the manifest once no segment in any level overlaps its prefix.
Readers at a snapshot before the drop still bind a catalog that contains the table and
still find its keys, because compaction waits for the watermark; PITR inside retention
likewise. The flush derives the entries from the catalog deltas of the transactions it
contains, so ledger replay on recovery reconstructs them without extra records.

Rejected: a tombstone per key in the DROP transaction (a 10 GB table becomes a 10 GB
write through the buffer and the ledger), and a background range delete on the
maintenance class writing tombstones in bulk (the same bytes later, and the tombstones
then wait for their own compaction). The manifest entry costs nothing at drop time. It is
safe only because OIDs are never reused (§3).

### 7.4 TRUNCATE

`TRUNCATE` uses the same mechanism with a read-time rule. It sets the table's
`c_class.truncated_at` to the transaction's `commit_ts` (and the same on each of its
indexes), and the next manifest rewrite adds one `dropped_ranges` entry whose `prefix` is the
`TableId` alone (covering every index and the overflow side range), `dropped_at_ts = T` and
`strict = true` (discard `commit_ts < T`). Readers whose bound catalog carries `truncated_at = T` ignore every
version under the table's prefixes with `commit_ts < T`; every earlier commit has a
smaller timestamp (the HLC is monotone), while rows inserted after the truncate in the
same transaction carry exactly `T` and later ones more, so both stay visible. Since the
commit timestamp is not known until COMMIT, the `c_class` write carries 04 §3's
`stamp_offsets` at `truncated_at`'s bytes, and the commit path writes `commit_ts` there; inside the
truncating transaction the private snapshot (§5.3) hides the committed rows and shows only
the buffer. `RESTART IDENTITY` resets owned sequences through the live
counter row. Rejected: deleting row by row (the cost of `DELETE FROM t`, which is what
users call `TRUNCATE` to avoid).

## 8. The `pg_catalog` projection

### 8.1 Mechanics

Each projected view is a `VirtualTable` with a static column list (PG 18's names, types
and order) and a scan function `fn(&CatalogSnapshot, &Session) -> RowIter`. The planner
treats them as small relations with exact row counts (it knows them from the snapshot)
and supports equality filters on `oid` and name columns by direct lookup, which is what
introspection queries do most. Each is listed in `pg_class` with PG 18's own OID and
relkind (`r` for catalogs such as `pg_class`, `v` for views such as `pg_roles`), so
`'pg_class'::regclass` and `\d pg_class` match.

### 8.2 `pg_class` and `pg_attribute` in full

`pg_class` (PG 18 column order):

| Column | Source |
|---|---|
| `oid`, `relname`, `relnamespace`, `relowner`, `relpersistence`, `relkind` | `c_class` `oid`, `name`, `namespace_oid`, `owner_oid`, `persistence`, `kind` |
| `reltype` | `0` until composite row types (P2), `intended` |
| `relam` | `403` (btree) for indexes, `2` (heap) for tables, `0` otherwise |
| `relfilenode` | `c_class.oid` (there is no file; `intended`) |
| `relpages`, `reltuples` | `ceil(bytes / 8192)` and `rows` from 02 §1.5's per-`(table_id, index_id)` properties for the relation's own prefix only (`index_id = 0` for a table, its own id for an index), summed over the manifest's segments; `-1` tuples if none |
| `relhasindex`, `relnatts`, `relchecks` | any `c_index` row; non-hidden `c_attribute` rows incl. dropped; `c_constraint` kind `c` |
| `relhasrules` | `false` (views are not rules here; `\d+` of a view uses `pg_get_viewdef`) |
| `relhastriggers`, `relrowsecurity`, `relforcerowsecurity` | `false` in P0 (`relhastriggers` `true` for fk-bearing tables from P1 to match PG's `\d`; RLS P3) |
| `relacl` | `c_privilege` rows rendered as `aclitem[]`, NULL when none (§2.11); `reloptions` = `c_class.options`, NULL when empty |
| all other columns | PG's value for a plain unpartitioned heap table: `reloftype`, `reltablespace`, `relallvisible`, `relallfrozen` (PG 18), `reltoastrelid`, `relrewrite`, `relfrozenxid`, `relminmxid` 0 (no XIDs, D1); `relisshared`, `relhassubclass`, `relispartition` false; `relispopulated` true; `relreplident` `d`; `relpartbound` NULL |

The rowid sequence (`is_rowid`) and all `c_*` relations are absent. The virtual
`pg_catalog` and `information_schema` relations are present with PG's OIDs and
`relnamespace` 11 or the `information_schema` OID.

`pg_attribute`:

| Column | Source |
|---|---|
| `attrelid`, `attname` | `c_attribute.rel_oid`, `name` |
| `atttypid` | `type_oid` (`0` for dropped columns, as PG) |
| `attlen`, `attbyval`, `attalign`, `attstorage` | from the type registry (PG's values per type) |
| `attnum` | `attnum`; system columns (`ctid`, `xmin`, …) are **not** listed, as PG lists no `ctid` row here |
| `atttypmod`, `attndims`, `attnotnull`, `atthasmissing`, `attidentity`, `attgenerated`, `attisdropped`, `attcollation` | `typmod`, `ndims`, `not_null`, `has_missing`, `identity`, `generated`, `is_dropped`, `collation_oid` |
| `atthasdef` | `default_expr` non-empty |
| `attstattarget` | NULL (PG 17+ default) |
| `attacl` | column-level `c_privilege` rows, NULL when none |
| `attmissingval` | `missing_value` rendered as a one-element `anyarray`, NULL if none |
| all other columns | constants: `attcacheoff` -1, `attcompression` `'\0'`, `attislocal` true, `attinhcount` 0, `attoptions`/`attfdwoptions` NULL |

### 8.3 The other views, non-trivial columns only

| View | Derived columns worth naming; everything else is PG's constant default |
|---|---|
| `pg_namespace` | static rows 11, 99, `information_schema` + `c_namespace`; `nspacl` from `c_privilege` |
| `pg_type` | static registry ∪ `c_type` (P2); `typrelid` 0 for tables until P2; `typarray`, `typelem`, `typinput`/`typoutput` regproc names from the registry |
| `pg_index` | `indexrelid`, `indrelid`, `indnatts` (key + include), `indnkeyatts`, `indisunique`, `indnullsnotdistinct`, `indisprimary`, `indisvalid`, `indisready` = `is_valid`, `indkey` (`int2vector` of key then include attnums), `indoption` (bit 0 DESC, bit 1 NULLS FIRST per key column), `indcollation`, `indclass` (PG's default btree opclass OID per type), `indpred` via `pg_get_expr`-compatible text, `indexprs` NULL in P0 |
| `pg_constraint` | `contype`, `conrelid`, `conindid`, `conkey`, `confrelid`, `confkey`, `confupdtype`, `confdeltype`, `confmatchtype`, `condeferrable`, `condeferred`, `convalidated`, `conbin` (rendered for `pg_get_constraintdef`), `conislocal` true; `conpfeqop`/`conppeqop`/`conffeqop` from the registry's equality operators |
| `pg_attrdef` | one row per column with `default_expr`; `oid` = `c_attribute.default_oid`; `adrelid`, `adnum`; `adbin` deparsable by `pg_get_expr` |
| `pg_proc` | static registry ∪ `c_proc`; `prosrc`, `proargtypes` (`oidvector`), `proallargtypes`, `proargmodes`, `proargnames`, `prokind` `f`/`a`/`w` from the registry |
| `pg_database` | one row (§11): `oid` 5, `datname`, `datdba`, `encoding` 6 (UTF8), `datlocprovider` `c`, `datcollate`/`datctype` `C`, `datallowconn` true, `datacl` from `c_privilege` |
| `pg_roles`, `pg_auth_members` | `c_role` ∪ static predefined roles, `rolpassword` always `********` as PG's view; `c_role_member` |
| `pg_settings` | static GUC registry, overlaid by `c_setting` (database, then role), then the session's `SET`; `source` says which; `context`, `vartype`, `min_val`, `max_val`, `enumvals` from the registry |
| `pg_description` | `c_description`; `classoid` is the projected catalog's OID (1259 for relations, 1255 for functions …) |
| `pg_depend` | `c_depend` mapped to PG's `classid`/`objid`/`objsubid` form; static built-in objects have no rows (PG has `p` pins; PG 15+ dropped them) |
| `pg_am`, `pg_tablespace` | static: `heap` (2), `btree` (403), `hnsw` from P2; `pg_default` (1663), `pg_global` (1664) |
| `pg_sequence`, `information_schema.sequences` | `c_sequence` minus rowid sequences; the latter's numeric columns as text, as PG |
| `information_schema.schemata`, `.tables` | `c_namespace` ∪ the static three, `catalog_name` = database name; `c_class` kinds `r`/`v` + virtual views, `table_type` `BASE TABLE` / `VIEW` |
| `information_schema.columns` | live `c_attribute` rows; `data_type`, `udt_name`, `character_maximum_length`, `numeric_precision`/`_scale` from `type_oid` + `typmod` exactly as PG's view definitions compute them; `column_default` = `pg_get_expr` text; `is_identity`, `identity_*` from `identity` and `c_sequence` |
| `information_schema.table_constraints` | `c_constraint` kinds `p`, `u`, `c`, `f`, and NOT NULL as `CHECK` rows as PG's view emits them |
| `information_schema.key_column_usage` | one row per (constraint, column) for `p`, `u`, `f`; `position_in_unique_constraint` for `f` |
| `information_schema.referential_constraints`, `.views` | `c_constraint` kind `f`, `update_rule`/`delete_rule` spelled out; `c_class` kind `v`, `view_definition` = `pg_get_viewdef` |

`pg_authid` is not projected: it contains `rolpassword`, which a non-superuser cannot
read in PG either. `pg_shadow` is not projected. Ledger reason `unsupported`.

### 8.4 Deliberate constants

The constants in §8.2's last rows and these below are `intended` in the compatibility
ledger and may differ from a real PostgreSQL 18; the replay test (§13) masks those columns
rather than excluding whole queries. Beyond §8.2: `relpages`/`reltuples` are segment
estimates, not `ANALYZE` results; `relhasrules` is false for views; `ctid` is not a
`pg_attribute` row (as in PG) but `SELECT ctid` works and returns 01 §6's synthesised value;
other system columns (`xmin`, …) are absent and fail with `42703`;
`pg_database.oid` is 5 and `datfrozenxid`/`datminmxid` 0; the `information_schema`
namespace OID (initdb-order dependent in PG) is the constant recorded from PG 18 with the
fixtures; `relacl` of the virtual catalogs is NULL.

## 9. Built-in functions, operators and types

### 9.1 The static registry

`colibri-catalog` contains static tables `BUILTIN_TYPES`, `BUILTIN_FUNCTIONS`,
`BUILTIN_OPERATORS`, `BUILTIN_CASTS` and `BUILTIN_OPCLASSES`, keyed by **PostgreSQL 18's
OIDs** (`int4` = 23, `int4pl` = 177, `text_eq` operator = 98, …), covering the phase-1 type
surface of 00 §7.2 and the functions over it that Colibri implements. Each function entry
names its implementation (a Rust function pointer in `colibri-sql`, linked through a
registration trait so the catalog crate does not depend on the executor), its
volatility, strictness and argument and return OIDs. Using PG's OIDs means the binary
format OIDs a driver sees in `RowDescription`, the `pg_proc` rows ORMs read and
`::regproc` output all agree with PostgreSQL.

The tables are generated by a script from PG 18's `src/include/catalog/pg_*.dat` files,
filtered to the implemented set, and the generated Rust file is **checked in**; the script
does not run in the build (D6's reproducibility; the `.dat` files are PostgreSQL-licensed
and only the script reads them). A CI check fails if an entry's OID, name or signature
differs from the recorded PG 18 values. Rejected: hand-written tables (OID typos are
silent compatibility defects), and build-time generation (another input to the hermetic
build).

### 9.2 Functions the catalog layer itself provides

| Function | Behaviour |
|---|---|
| `format_type(oid, int4)` | PG's spelling for every built-in type and typmod (`character varying(20)`, `numeric(10,2)`, `timestamp with time zone`, `integer[]`); user types P2 |
| `pg_get_expr(pg_node_tree, oid [, bool])` | deparses a stored tree (§6.5); `pg_node_tree` values in projections are an opaque text form that only this function reads, as in PG |
| `pg_get_indexdef(oid [, int, bool])` | `CREATE [UNIQUE] INDEX name ON schema.table USING btree (cols [DESC] [NULLS ...]) [INCLUDE (...)] [WHERE ...]` in PG's exact layout |
| `pg_get_constraintdef(oid [, bool])` | `PRIMARY KEY (...)`, `FOREIGN KEY (...) REFERENCES t(...) ON DELETE ...`, `CHECK (...)` with PG's parenthesisation |
| `pg_get_viewdef(oid [, bool])` | deparse of the stored view query, PG's pretty layout |
| `pg_get_serial_sequence(text, text)` | from `c_sequence.owned_by_*` |
| `obj_description(oid, name)`, `col_description(oid, int)`, `shobj_description` | from `c_description` |
| `current_schema()`, `current_schemas(bool)` | first existing schema on `search_path`; with `true`, `pg_catalog` prepended |
| `current_database()` | `c_database.name` |
| `version()` | `PostgreSQL 18.0 (Colibri <version>) on <target-triple>, compiled by rustc <version>, 64-bit` — PG-shaped so version parsers (`/PostgreSQL (\d+)\.(\d+)/`) work, with the Colibri mark of 00 §13 and D17's `unofficial` mark when applicable |
| `pg_table_is_visible(oid)`, `pg_type_is_visible`, `pg_function_is_visible` | against `search_path` |
| `has_table_privilege`, `has_schema_privilege`, `has_column_privilege`, `has_function_privilege`, `has_database_privilege`, `has_sequence_privilege` | **return `true` in P0** for any existing object and raise PG's errors for missing ones; real checks in P1 (§10) |
| `pg_relation_size` and the other size functions | sums of 02 §1.5's per-prefix `bytes` over the manifest (estimates): `pg_relation_size(t)` is the `index_id = 0` prefix alone, as PG's main fork excludes TOAST; `pg_table_size` adds the overflow range `0xFFFF_FFFF`; `pg_indexes_size` sums the table's index prefixes; `pg_total_relation_size` is all of the table's prefixes |

## 10. Roles, authentication and privileges

P0 has roles with the attributes of §2.10, SCRAM-SHA-256 verifiers in `c_auth`, `LOGIN`
and `SUPERUSER`, and membership in `c_role_member`. `CREATE ROLE`, `ALTER ROLE`,
`DROP ROLE`, `GRANT role TO role`, `ALTER ROLE ... PASSWORD` (the client sends the
verifier or the server computes one with 4096 iterations, PG's default) are implemented.
The wire layer's SCRAM exchange (00 §3, v3 wire) reads the verifier from the current
committed `CatalogSnapshot`; authentication never reads the KV store directly. Roles are
**per database**: a tenant's database holds its own roles, whereas PostgreSQL's are
cluster-wide. With one database per cluster (§11) the difference is not observable.

The trusted-peer handshake of D11 (`SO_PEERCRED` plus a per-worker token) produces a
**role name** chosen by the binding; the session runs as the `c_role` with that name,
which must exist and have `can_login`, or the connection fails with `28000`. The
bootstrap owner role (§4.1) is the default the binding uses. No SCRAM verifier is needed
for a role used only through trusted peer.

`GRANT` and `REVOKE` on tables, columns, sequences, schemas, functions and the database
are parsed, validated and **stored** in `c_privilege` in P0, and reported through
`relacl`, `nspacl`, `proacl`, `datacl`, `attacl`, `information_schema.table_privileges`
and `has_*_privilege` (which returns `true`). They are **not enforced in P0**: every
role that can log in can read and write everything in the database. In Runlot's
deployment every session of a tenant runs as the tenant's owner role, so P0 loses nothing
there; an on-prem or external user who creates a restricted role would be surprised, and
the product surface says so. The orchestrator adds the access-control item to 00 §14.

**Enforcement plan (P1).** An ACL check at analyze time: for each relation in the bound
statement, the required privilege bits (SELECT on read columns, INSERT/UPDATE on written
columns, DELETE, TRUNCATE, REFERENCES for fk creation, USAGE on sequences and schemas,
EXECUTE on functions) are checked against the effective privileges of the current role
(direct grants, `PUBLIC`, and roles reached through `c_role_member` with `inherit`),
computed once per `(role, schema_version)` and cached in the snapshot. Superusers and
owners short-circuit. A failed check raises `42501` with PG's message text. Because the
check runs at analyze time, a cached plan is re-checked when the role or the
`schema_version` changes. Default privileges (`ALTER DEFAULT PRIVILEGES`) are P2;
row-level security (`CREATE POLICY`) is P3.

## 11. One database per tenant

`CREATE DATABASE` is a **control-plane operation**: it allocates a new `DbId`, the CP
records it, and the first lease holder runs bootstrap (§4.1). It is not a catalog row in
another database, and there is no cluster catalog spanning databases. Over the wire,
`CREATE DATABASE` and `DROP DATABASE` raise `0A000` with a hint naming the CP operation;
`ALTER DATABASE <current> SET` and `RENAME` of the current database work (rename is
reported back to the CP through the control socket, P3).

`pg_database` therefore lists **exactly one row**, the current database; `psql \l` shows
one database; `\c other` fails with `3D000` unless the other is a different tenant
database reachable through the same front (a new connection, not a catalog lookup).
Ledger reason `intended`: a Colibri database is PostgreSQL's database-and-cluster in one,
and pretending otherwise (listing `postgres`, `template0`, `template1`) misleads tools
that then try to connect to them.

## 12. Rust type sketches

Sketches, not code; they fix the contract between `colibri-catalog`, `colibri-sql`,
`colibri-txn` and `colibri-db`. Maps keyed by OID are `OidMap<T>` (a `BTreeMap`); hash maps are
`DetHashMap` (06 §2.3), never `std::collections::HashMap`; name maps mirror §2.1's indexes.

```rust
pub struct CatalogSnapshot {
    pub schema_version: u64, pub commit_ts: CommitTs, pub next_oid: Oid,
    pub database: Database,
    namespaces: OidMap<Namespace>,     relations: OidMap<Arc<Relation>>,
    functions: OidMap<Arc<Function>>,  roles: OidMap<Role>,
    rel_by_name: DetHashMap<(Oid, String), Oid>,   // plus ns, fn, role name maps
    privileges: Vec<Privilege>, depends: DependGraph,
    settings: Vec<Setting>, descriptions: DetHashMap<ObjRef, String>,
}
impl CatalogSnapshot {
    pub fn relation(&self, oid: Oid) -> Option<&Arc<Relation>>;
    pub fn resolve_relation(&self, search_path: &[Oid], n: &QualName) -> Result<Oid, SqlError>;
    pub fn apply(&self, delta: &CatalogDelta, commit_ts: CommitTs) -> CatalogSnapshot; // §5.1, §5.3
}

pub struct Relation {
    pub oid: Oid, pub namespace: Oid, pub name: String, pub kind: RelKind, // Table|Index|Sequence|View
    pub owner: Oid, pub columns: Vec<Column>,            // by attnum, dropped and hidden included
    pub pk_index: Option<Oid>, pub rowid_seq: Option<Oid>,
    pub indexes: Vec<Arc<Index>>, pub constraints: Vec<Constraint>,
    pub sequence: Option<Sequence>,                      // kind == Sequence
    pub truncated_at: Option<CommitTs>, pub row_version: CommitTs,
    pub view_query: Option<StoredExpr>, pub options: Vec<(String, String)>,
}
pub struct Column {
    pub attnum: i16, pub name: String, pub ty: Oid, pub typmod: i32, pub not_null: bool,
    pub default: Option<(Oid, StoredExpr)>, pub missing: Option<Datum>,
    pub identity: Option<IdentityKind>, pub dropped: bool, pub hidden: bool, pub collation: Oid,
}
pub struct Index {
    pub oid: Oid, pub table: Oid, pub keys: Vec<IndexKey>, // { attnum, desc, nulls_first }
    pub include: Vec<i16>, pub unique: bool, pub nulls_not_distinct: bool,
    pub primary: bool, pub predicate: Option<StoredExpr>, pub valid: bool,
}
pub enum Constraint {
    PrimaryKey { oid: Oid, name: String, cols: Vec<i16>, index: Oid },
    Unique     { oid: Oid, name: String, cols: Vec<i16>, index: Oid },
    Check      { oid: Oid, name: String, expr: StoredExpr, validated: bool },
    NotNull    { oid: Oid, name: String, col: i16 },
    ForeignKey { oid: Oid, name: String, cols: Vec<i16>, ref_rel: Oid, ref_cols: Vec<i16>,
                 on_update: FkAction, on_delete: FkAction, match_full: bool }, // enforced P1
}
pub struct Sequence {   // parameters only; the live counter is 04's row under prefix(oid, 0)
    pub ty: Oid, pub start: i64, pub increment: i64, pub min: i64, pub max: i64,
    pub cache: i64, pub cycle: bool, pub owned_by: Option<(Oid, i16)>, pub is_rowid: bool,
}
pub struct Role { pub oid: Oid, pub name: String, pub attrs: RoleAttrs,
                  pub scram: Option<ScramVerifier>, pub member_of: Vec<Membership> }
pub struct StoredExpr(pub Bytes);   // opaque here; colibri-sql encodes and decodes (§6.5)

/// DDL mutation API: every call writes catalog rows and their catalog-index entries
/// into the transaction's write buffer and records the decoded change in the
/// transaction's CatalogDelta (§5.3); the first call marks schema_version for bump.
pub struct CatalogWriter<'t> { txn: &'t mut TxnWrite, delta: &'t mut CatalogDelta,
                               base: Arc<CatalogSnapshot> }
impl CatalogWriter<'_> {
    pub fn alloc_oid(&mut self) -> Result<Oid, SqlError>;           // c_meta.next_oid (§3)
    pub fn create_relation(&mut self, d: RelationDef) -> Result<Oid, SqlError>;
    pub fn add_column(&mut self, rel: Oid, c: ColumnDef, missing: Option<Datum>) -> Result<i16, SqlError>;
    pub fn alter_column(&mut self, rel: Oid, attnum: i16, ch: ColumnChange) -> Result<(), SqlError>;
    pub fn drop_column(&mut self, rel: Oid, attnum: i16) -> Result<(), SqlError>;
    pub fn create_index(&mut self, d: IndexDef) -> Result<Oid, SqlError>;
    pub fn add_constraint(&mut self, rel: Oid, d: ConstraintDef) -> Result<Oid, SqlError>;
    pub fn rename(&mut self, obj: ObjRef, name: &str) -> Result<(), SqlError>;   // and grant,
    // create_role, set_comment: same shape
    pub fn drop(&mut self, obj: ObjRef, b: DropBehavior) -> Result<Vec<ObjRef>, SqlError>; // §7.2
    pub fn truncate(&mut self, rel: Oid) -> Result<(), SqlError>;   // §7.4
}

pub struct BuiltinType {     // PG 18 OIDs throughout; generated, checked in (§9.1)
    pub oid: u32, pub name: &'static str, pub array_oid: u32, pub elem_oid: u32,
    pub len: i16, pub by_val: bool, pub align: u8, pub storage: u8, pub category: u8,
    pub collatable: bool, pub io: [u32; 4],              // input, output, recv, send
}
pub struct BuiltinFunction {
    pub oid: u32, pub name: &'static str, pub args: &'static [u32], pub ret: u32,
    pub retset: bool, pub kind: ProKind, pub volatility: Volatility, pub strict: bool,
    pub imp: FnImplId,       // resolved through colibri-sql's registration table
}
pub static BUILTIN_TYPES: &[BuiltinType] = &[/* generated */];
pub static BUILTIN_FUNCTIONS: &[BuiltinFunction] = &[/* generated */];
```

## 13. Test plan

**Introspection replay against a recorded PostgreSQL 18.** A recorder runs a fixed set of
schemas (a dozen tables with every phase-1 type, composite and partial indexes, every
constraint kind, identity and serial columns, a view, comments, grants) against a real
PostgreSQL 18 with `--no-locale`, captures the exact introspection queries each tool
sends and their result sets, and checks them into the public harness tree (D18). The
replay runs the same schema DDL against `colibrid`, sends the same queries and compares
result sets with the §8.4 column mask, every remaining difference carrying a ledger
reason. The query sets are:

| Set | Phase it gates |
|---|---|
| `psql` `\d`, `\d <table>`, `\d+ <table>`, `\dt`, `\di`, `\ds`, `\dv`, `\dn`, `\du`, `\l`, `\df` | **P0** (`\d`, `\dt`, `\di`, `\ds` are the P0 exit set; the rest are recorded and tracked) |
| Prisma `db pull` introspection queries | P1 exit criterion (00 §12) |
| Drizzle `introspect` queries | P1 exit criterion |
| pgjdbc / Npgsql type-loading queries (`pg_type` joins) | P1 driver rows |

**OID stability across restart, wake and branch.** Create objects, record every OID from
`pg_class`, `pg_type` (user types P2), `pg_constraint`, `pg_proc`; close the database,
evict caches, reopen on another simulated node (06's simulation `Env`), and compare; repeat
after a ledger-only recovery (crash before flush) and on a branch. A property test drops
and creates objects in random order and asserts no committed OID is issued twice,
including across aborted DDL and crashes mid-DDL.

**DDL-in-transaction visibility.** Within one transaction: `CREATE TABLE`, insert, `ALTER
TABLE ADD COLUMN`, select, `SAVEPOINT`, `DROP COLUMN`, `ROLLBACK TO`, select, `COMMIT`; a
concurrent session sees nothing until COMMIT and then sees the final state in one step;
a REPEATABLE READ session started before the COMMIT keeps the old schema and gets
`40001` if it writes to the table. A prepared statement over the table re-plans silently
after `ADD COLUMN` when the result type is unchanged (explicit column list) and fails with
`0A000` for `SELECT *`.

**Missing value and rewrite.** `ADD COLUMN ... DEFAULT 5` on a table whose segments were
flushed before the DDL returns 5 for old rows; an explicit NULL inserted afterwards returns
NULL; after a full compaction the results are unchanged. `ADD COLUMN ... DEFAULT random()`
gives every row a distinct value and fails with `54000` above the spill bound.

**Drop and truncate reclaim.** Drop a table, flush, and assert the manifest carries the
`dropped_ranges` entry; hold a snapshot from before the drop and read the table's rows
through it; release it, compact, and assert no segment overlaps the prefix and the entry
is gone. The same for `TRUNCATE` with inserts after it in the same transaction.

**Catalog format migration.** A fixture database written by a test binary with
`catalog_format_version = 1` and a synthetic version-2 migration step: open with the
version-2 binary, assert one migration transaction in the ledger and the new version in
`c_meta` and the manifest; crash inside the migration under simulation and assert the
next open repeats it; open the migrated database with the version-1 binary and assert it
refuses.

**Bootstrap.** A crash at each bootstrap step under simulation; the next lease opens with
exactly one bootstrap batch committed.

## 14. What this document does not decide

- **User types**: enums, domains, composite types and their `pg_type` rows, row types per
  table (`reltype`). Row format fixed in §2.5; contents P2.
- **PL/pgSQL** function rows, triggers and `pg_trigger`. P2.
- **Default privileges** (`ALTER DEFAULT PRIVILEGES`, `pg_default_acl`). P2.
  **Enforcement** of privileges is P1 (§10); RLS and `pg_policy` are P3.
- **Event triggers**, `pg_event_trigger`. Not scheduled.
- **`pg_stat_*` and `pg_statio_*` views**, `pg_stat_activity`, `pg_locks`. They project
  runtime counters (00 §6), not the catalog; a later document decides their sources.
- **Foreign-key enforcement and actions**, `DEFERRABLE` semantics. P1 (§2.7).
- **Temporary tables** and their session-local catalog overlay. P1 (§2.11).
- **Expression indexes** and `CREATE INDEX CONCURRENTLY` as a real background build. Both
  P2 (§6.6, 01 §5.4).
- **Non-`C` collations** in `c_attribute.collation_oid` and `pg_collation`. Later phase
  (D20); the column is fixed now.
- **Catalog snapshot cost**: the memory per open database and the derivation time per
  DDL are *not measured*; P0's density drill (G3) measures them and decides whether
  persistent maps are needed.
- The exact **`information_schema` namespace OID** (recorded from PG 18, §8.4).
