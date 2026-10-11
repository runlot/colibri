# Colibri

**A PostgreSQL-dialect database engine built for idle-zero, instant wake, and threaded
throughput.**

Colibri is a single-writer, multi-reader engine whose source of truth is object storage,
whose compute is a per-node daemon that opens and closes tenant databases the way a
browser opens tabs, and whose PostgreSQL compatibility is a measured ledger against
external yardsticks rather than a property inherited from vendored code.

It is the engine behind Runlot's `env.db`. Colibri keeps PostgreSQL's grammar, type
OIDs, wire protocol and catalog shapes, and owns everything else: storage, transactions,
catalog, planner and executor. The design is in [docs/](docs/README.md).

## The name

*Colibri* is the hummingbird (French, Spanish, Portuguese; 콜리브리, コリブリ). Three facts
about the bird are the project's three goals:

| The bird | The goal |
|---|---|
| The smallest bird; a hummingbird weighs a few grams | **Density.** Open tenants per node bounded by cache RAM, not by processes or slots |
| At night it enters *torpor*: heart rate and metabolism fall to near zero, and it costs almost nothing to stay alive | **Idle converges to the bucket.** A cold tenant holds nothing on any node; its marginal cost is the bytes it keeps in object storage |
| It leaves torpor and is at full wingbeat within seconds | **Instant wake.** From cold to first query in one lease, one manifest and one prefetch |

The fourth goal, threaded throughput, is the wingbeat itself: readers on every core, one
writer per database, many databases' writers sharing the cores.

The name is one word, pronounceable in English, Korean and Japanese, and carries no
existing database or data-infrastructure project. Alternatives considered and the reason
each lost are in [docs/00-overview.md §13](docs/00-overview.md#13-name).

What the name binds:

| Thing | Name |
|---|---|
| Daemon binary and its roles | `colibrid`, `colibrid --role ledger` |
| Crate workspace prefix | `colibri-*` |
| Environment prefix | `COLIBRI_*` |
| Control socket | `/run/colibri/control.sock` |

## Licence

Colibri Source License 1.0 (`LICENSE`). Source is available. Production use is free for
individuals, educational and non-profit organisations, and organisations with annual
revenue under USD 10 million; organisations at or above that threshold need a commercial
licence. Regardless of revenue, offering Colibri to third parties as a database service
needs one. Each version converts to Apache-2.0 four years after its first public release.
Commercial licences: licensing@runlot.io. The reasoning is D17 in the design document;
contributions need the CLA described in `CONTRIBUTING.md`.
