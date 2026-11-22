# speedyspec-server: deferred findings

Repository: [SpeedySpec/speedyspec-server](https://github.com/SpeedySpec/speedyspec-server).

## Where these came from

The data-and-options port to Rust ran as eight plans, merged as #12 to #19 in November 2026. A fresh reviewer read
each plan's whole branch. Findings graded Critical or Important were fixed before merge. These are the Minor ones,
deferred.

Seven of the original 72 were fixed afterwards in #25 (SpeedySpec/speedyspec-server#25), so they are not listed
here:
- `SqlCache`'s duplicate `add` inside a PostgreSQL transaction (not a bug: the store already uses savepoints).
- SQLite writing to the file on every open.
- SQLite refusing MySQL's `LIMIT o, 18446744073709551615`.
- `FileCache` locks inside the group directory, and reads that removed expired files without the lock.
- `KvCache` increment and decrement races.
- A failed `settings_put` being lost silently.
- No cross-process test of transients on the persistent object cache.

One more is left out: it concerned the wording of a review ledger that has since been deleted, not the code.

The kses port (#50, #52, #53, #58) had its own final review. Its six Minor findings, and the speed goals it has
not yet met, are in [kses.md](kses.md).

Recorded 2026-11-21. Each finding was written at review time; where later code changed part of one, the text says
what is still true.

## Index

| File | Area | Findings |
|---|---|---|
| [data-layer.md](data-layer.md) | `speedyspec-data`, `-data-query`, `-data-uow`, `-data-mysql` | 9 |
| [wpdb.md](wpdb.md) | `speedyspec-wp-db`, `-wp-db-php` | 11 |
| [cache-and-object-cache.md](cache-and-object-cache.md) | `speedyspec-cache-*`, `-php-serialize`, `-wp-object-cache*` | 11 |
| [options-api.md](options-api.md) | `speedyspec-wp-options*` | 10 |
| [sqlite-engine.md](sqlite-engine.md) | `speedyspec-data-sqlite`, `-data-translate` | 6 |
| [postgres-engine.md](postgres-engine.md) | `speedyspec-data-postgres` | 6 |
| [nosql-and-direct-stores.md](nosql-and-direct-stores.md) | `speedyspec-data-valkey`, `-data-redb`, `-data-file`, `-cache-file` | 5 |
| [persistent-cache-transients-network-options.md](persistent-cache-transients-network-options.md) | `speedyspec-wp-options*`, `-wp-object-cache` | 7 |
| [kses.md](kses.md) | `speedyspec-wp-kses*`, `-wp-html-syntax` | 6, plus 2 speed goals |
