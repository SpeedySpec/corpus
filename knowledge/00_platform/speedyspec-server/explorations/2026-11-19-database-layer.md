# Exploration: The database layer and the `wpdb` port

**Status:** built in part
**Created:** 2026-11-19
**Author:** Claude, for the project owner
**Context:** Port WordPress's `wpdb` to Rust over a general data-access layer that runs WordPress's MySQL on MySQL, PostgreSQL and SQLite, with NoSQL, file and caching stores beside it.
**Approved by:** the project owner on 2026-11-19
**Socialized with:** not socialized

---

## Problem Statement

Native features in the runtime (the firewall first, then the response cache) could not run in Rust, because the data
they need was only reachable through PHP's `$wpdb` and `mysqli`. Every read meant entering PHP. Plugins and themes also
send raw MySQL-flavoured SQL through `$wpdb`, so any new data layer had to keep that working unchanged. And the
project wanted sites to keep WordPress's tables on databases other than MySQL, which raw MySQL from plugins makes hard.

## Current State

- Only the Hook API had been ported (`speedyspec-wp-hooks` and its PHP crate, speedyspec-server #4 and #5), with the
  CQRS buses, the container and recorded compatibility (`speedyspec-compat`).
- Database access was WordPress's own `class-wpdb.php` over `mysqli`, running in the embedded PHP.
- The runtime's workers are separate processes, each serving one request at a time, with a single-threaded Hook API.

## Proposed Solution

### Architecture

```
  PHP plugins, themes                   Rust features and ports
        |                                       |
  speedyspec-wp-db-php  ------------>  speedyspec-wp-db (wpdb as commands and queries)
  (native wpdb class, $wpdb)                    |
                                                v
        speedyspec-data (contracts: SqlStore, NoSqlStore, DirectStore, Engine, Dialect,
                         Repository, UnitOfWork, Statement over sqlparser's AST)
           ^              ^               ^                       ^
   -data-query       -data-uow      engines: -mysql            speedyspec-cache (Caching Store)
   (parser, builder, (unit of work,  -sqlite -postgres          ^
    partials, passes, identity map)  -valkey -redb -file        -memory -sql -kv -file
    engine registry)                 -memory
                     -data-translate (passes shared by SQLite and PostgreSQL)
```

Every SQL statement, from the builder or from raw text, takes one path: the `query` filter, parse with MySQL's
dialect, the statement object (partials may change it), the neutral passes, the target dialect's passes, emission,
execution, and rows mapped back to MySQL's shapes.

### Data Model

- `Statement`: `sqlparser`'s syntax tree plus bound parameters. Placeholders are numbered before rewriting, so passes
  can move and repeat expressions without disturbing the parameters.
- `DbValue` (`Null`, `Int`, `UInt`, `Float`, `Text`, `Bytes`) with a MySQL text form; values reach PHP as strings or
  null, as `mysqli` gives them.
- On SQLite and PostgreSQL, a MySQL schema catalogue in three `_speedyspec_*` tables (Metadata Mapping) keeps the
  MySQL types, collations and index prefix lengths that the engines cannot hold.

### Interfaces

- `SqlStore`: `execute(&Statement)`, `query(&Statement)`, transactions and savepoints, `introspect()`, `reset()`. No
  method takes SQL text.
- `NoSqlStore`: keyspaces of byte keys and values with `get`, `get_many`, `put`, `put_if_absent`, `delete`, `range`,
  `increment`, `clear`.
- `DirectStore`: only lifecycle (`connect`, `health`, `close`, `kind`); each system's repository uses its own client.
- `CachingStore`: `get`, `get_many`, `set`, `add`, `replace`, `delete`, `increment`, `decrement`, `flush`,
  `flush_group`, all by group and key.
- `WpDb::query_statement(Statement)`: the one entry ported APIs use, so `query` fires and is counted as in WordPress.

## Design Decisions

1. **`sqlparser`'s syntax tree is the statement model.** The project owner chose it: it already handles the AST. A
   home-grown model would have had to grow to everything MySQL accepts. Recorded in [ADR-009](../../adr/009-database-port-requirements.md).
   *Trade-off:* the contracts depend on a third-party crate's types.
2. **Every statement is parsed, on every engine, MySQL included.** The designer proposed a MySQL fast path for
   unchanged raw SQL; the project owner refused it, so every statement has a tree Rust can inspect.
   *Trade-off:* SQL MySQL accepts but the parser rejects fails everywhere; such gaps are a documented break.
3. **No silent mistranslation.** A construct a dialect cannot express fails with an error naming the construct and
   dialect. *Trade-off:* some plugin SQL fails on SQLite or PostgreSQL instead of running approximately.
4. **Blocking drivers.** The project owner: "it will be slower, but it has to work with PHP." It fits one request per
   worker and the single-threaded Hook API. *Trade-off:* no async I/O; async can come later as separate adapters.
5. **A native `wpdb` class, not a `db.php` drop-in.** The project owner chose it so `prepare()`, escaping and the
   helpers exist once, in Rust ([ADR-014](../../adr/014-port-wordpress-to-rust-and-delete-its-php.md)).
   *Trade-off:* `$wpdb->dbh` is no longer a `mysqli` object; code calling `mysqli_*` on it breaks. Database drop-ins
   are refused with an error naming the file.
6. **Three store kinds, WordPress's tables on SQL only.** The designer asked whether ReDB could hold a whole site; the
   project owner chose separate kinds instead. *Trade-off:* raw SQL and the builder target SQL stores only.
7. **One ReDB owner process per site.** ReDB locks its file to one process, so an owner serves workers over a private
   Unix socket (designer's call, approved). *Trade-off:* one local round trip per call; Valkey stays the faster shared
   choice ([ADR-007](../../adr/007-shared-stores-file-redis-database.md), Redis read as Valkey).
8. **No two-phase commit.** The unit of work commits one transaction per engine and reports a partial commit.
   *Trade-off:* the caller decides what to do after a partial failure.
9. **Every `wpdb` method a command or query** on the CQRS buses, with request state behind a `WpdbState` port that has
   a PHP twin ([ADR-015](../../adr/015-ports-and-adapters-with-cqrs-use-cases.md),
   [ADR-017](../../adr/017-php-twins-and-the-override-flag.md)). `Query` is a command carrying a result.
   *Trade-off:* 31 message types to keep in step with WordPress.
10. **The store contracts have no PHP twins** (designer's call): PHP cannot hold a statement, and no one needs a
    PHP-written SQL engine. *Trade-off:* PHP can replace `WpdbState`, not a store.
11. **PostgreSQL text is case-insensitive through an ICU nondeterministic collation, not `citext`.** It is case- and
    accent-insensitive like MySQL's default collation and works in unique indexes. *Trade-off:* PostgreSQL refuses
    `LIKE` on such collations, so `LIKE` becomes `ILIKE`, which stays accent-sensitive.
12. **MySQL types live in one shared catalogue for both non-MySQL engines.** The PostgreSQL plan first stored them in
    column comments; reconciliation moved it onto the SQLite plan's catalogue, so `DESCRIBE` and `dbDelta` round-trip
    the same way on both. *Trade-off:* extra tables to keep in step with every DDL statement.

## Implementation Phases

1. **Foundation.** Contracts, parser, builder and partials, pass chain, engine registry, unit of work, MySQL engine,
   SQL store conformance suite. *Delivers:* data access from Rust on MySQL. Shipped in speedyspec-server #12.
2. **`wpdb` port.** `speedyspec-wp-db` and the native class; the `compat/wp-db` catalogue recorded against WordPress
   7.1.5; WordPress's own database tests against the native class. *Delivers:* `$wpdb` served by Rust. Shipped in #13.
3. **Caching Store.** Contract, conformance suite, `MemoryCache`, `SqlCache` (with the object cache, covered in the
   options exploration). *Delivers:* a cache any store can back. Shipped in #14.
4. **SQLite.** Shared translation crate, SQLite passes and store, the SQL corpus, cross-engine replay.
   *Delivers:* WordPress's tables on SQLite. Shipped in #16.
5. **PostgreSQL.** Its passes, mysqli-shaped values, support functions, store. *Delivers:* WordPress's tables on
   PostgreSQL 16+. Shipped in #17.
6. **NoSQL and Direct stores.** Memory, Valkey, ReDB with its owner, files; `KvCache` and `FileCache`.
   *Delivers:* shared state choices for native features. Shipped in #18.
7. **Follow-ups.** Purge tests on all engines (#22), CI and test-flake fixes (#23, #26), priority deferred findings
   (#25).
8. **Runtime wiring.** Read `wp-config.php` once per worker, build the engine registry, create `$wpdb` per request,
   reset connections at request end, refuse drop-ins, supervise the ReDB owner. Handed to the runtime's owner as
   notes. *Delivers:* sites actually served by the port. Not shipped: `worker.rs` on main does not reference it.

## Fleet Impact

- **speedyspec-server:** about twenty new crates under `crates/` (confirmed by the merged PRs).
- **PHP runtime (`speedyspec-php-runtime`):** needs the worker changes in phase 8; not made yet (checked `worker.rs`
  on main).
- **wordpress-develop fork:** `class-wpdb.php` is to be emptied once the port serves sites; it is still present in
  full.
- **CLI:** the PostgreSQL support SQL is public so the CLI can pre-install it. Whether the CLI uses it: unverified.
- **Blog, front-end API, admin API pools:** all reach the same stores once wired; the ReDB owner serves every pool.
  Unverified until wiring lands.

## Open Questions

1. **When does the runtime wire the port in?** Settled by the runtime's owner landing the worker changes.
2. **When is `class-wpdb.php` deleted?** Settled once sites run on the native class ([ADR-021](../../adr/021-recorded-compatibility.md)).
3. **How are parser gaps patched**, upstream or with a vendored dialect? Left to the builder; settled by the first
   gap found in plugin SQL.
4. **Full-text search** (`MATCH ... AGAINST`) is refused on SQLite and PostgreSQL. Settled by wiring each engine's
   full-text search.
5. **TLS to PostgreSQL** is not supported. Settled by a later change to the PostgreSQL engine.
6. **Schema changes made outside SpeedySpec** reach PostgreSQL workers only after a restart. Settled by a
   cross-process invalidation that covers external DDL.
7. The Minor review findings are listed in the [deferred findings](../../../../deferred/speedyspec-server/README.md).

## Spike Findings

**Settles:** whether `delete_expired_transients()`'s MySQL-only multi-table `DELETE a, b FROM ...` needed work on the
other engines.
**Built:** the purge tests run on MySQL, SQLite and PostgreSQL (#22).
**Established:** no change was needed: SQLite's DML pass and PostgreSQL's multi-delete pass already translate it, and
only expired value-and-timeout pairs are removed.
**Does not establish:** that every multi-table `DELETE` plugins send translates.
**Consequence:** the tests stay as a guard; no code changed.

## Success Criteria

- [x] MySQL, SQLite and PostgreSQL pass the shared SQL store conformance suite (each engine's conformance test).
- [x] The `wp-db` catalogue replays exactly on MySQL against WordPress 7.1.5's recording (`compat/wp-db`).
- [x] The same catalogue replays on SQLite and PostgreSQL; PostgreSQL needed no engine breaks (#17).
- [x] WordPress's database PHPUnit tests pass against the native class; on PostgreSQL apart from 20 listed tests
  (non-Unicode charset conversion and spatial types).
- [x] Every statement in the SQL corpus (core tests, core schema, ten popular plugins) translates for SQLite, and
  translates and prepares on PostgreSQL, or is listed as unsupported.
- [x] Valkey, ReDB, file and memory stores pass the NoSQL and Direct store suites; `KvCache` and `FileCache` pass the
  Caching Store suite.
- [ ] Sites are served through the native `wpdb` (needs the runtime wiring).

---

## Non-Goals

- **Active Record or Eloquent models** — "in the manner of Eloquent" is the builder's style only.
- **Laravel or SQLAlchemy API compatibility** — they are style references.
- **WordPress's tables on ReDB, Valkey or files** — the project owner kept them on SQL engines.
- **An async data layer** — blocking for now.
- **Two-phase commit across engines** — partial commits are reported instead.
- **Porting options, posts, terms or users here** — options has its own design; the others come later.
- **Database drop-ins such as HyperDB** — they replace `wpdb`, which is now native.

## Alternatives Considered

### Option: Own statement model, parser as an adapter
- **Pros:** the purest dependency direction.
- **Cons:** it would have to grow to all MySQL plugins send, rebuilding most of `sqlparser`.
- **Verdict:** rejected by the project owner.

### Option: MySQL fast path for unparsed SQL
- **Pros:** less work per statement on MySQL.
- **Cons:** statements without a tree that Rust can inspect or change.
- **Verdict:** rejected by the project owner.

### Option: Async drivers with `block_on`
- **Pros:** the common Rust ecosystem choice.
- **Cons:** an async runtime between PHP's blocking calls and a single-threaded Hook API.
- **Verdict:** rejected; may return later as separate adapters.

### Option: A PHP `wpdb` subclass as a `db.php` drop-in
- **Pros:** smaller change; only the query passes to Rust.
- **Cons:** two copies of `wpdb`'s logic.
- **Verdict:** rejected by the project owner.

### Option: ReDB per worker, or only in single-process setups
- **Pros:** no owner process.
- **Cons:** nothing shared between workers, so no shared `alloptions` or counters.
- **Verdict:** rejected in favour of one owner per site.

## Dependencies

- The Hook API port, the CQRS buses, the container and `speedyspec-compat` (speedyspec-server #4, #5).
- The embedded PHP runtime and its worker model (speedyspec-server #6).
- [ADR-003](../../adr/003-native-features-wait-for-api-ports.md): the database layer is step 1 of the port order.

## References

- [ADR-003](../../adr/003-native-features-wait-for-api-ports.md), [ADR-007](../../adr/007-shared-stores-file-redis-database.md), [ADR-009](../../adr/009-database-port-requirements.md), [ADR-014](../../adr/014-port-wordpress-to-rust-and-delete-its-php.md), [ADR-015](../../adr/015-ports-and-adapters-with-cqrs-use-cases.md), [ADR-017](../../adr/017-php-twins-and-the-override-flag.md), [ADR-021](../../adr/021-recorded-compatibility.md)
- [Options API port exploration](2026-11-19-options-api-port.md)
- [Deferred findings](../../../../deferred/speedyspec-server/README.md)
- [SpeedySpec/speedyspec-server](https://github.com/SpeedySpec/speedyspec-server): [#12](https://github.com/SpeedySpec/speedyspec-server/pull/12), [#13](https://github.com/SpeedySpec/speedyspec-server/pull/13), [#14](https://github.com/SpeedySpec/speedyspec-server/pull/14), [#16](https://github.com/SpeedySpec/speedyspec-server/pull/16), [#17](https://github.com/SpeedySpec/speedyspec-server/pull/17), [#18](https://github.com/SpeedySpec/speedyspec-server/pull/18), [#22](https://github.com/SpeedySpec/speedyspec-server/pull/22), [#23](https://github.com/SpeedySpec/speedyspec-server/pull/23), [#25](https://github.com/SpeedySpec/speedyspec-server/pull/25), [#26](https://github.com/SpeedySpec/speedyspec-server/pull/26)
- Source spec: `2026-11-19-database-layer-design.md`
- Source plans: `2026-11-19-data-and-options-ports-index.md`, `2026-11-19-data-layer-foundation.md`, `2026-11-19-wpdb-port.md`, `2026-11-19-caching-store-and-object-cache.md`, `2026-11-19-sqlite-engine.md`, `2026-11-19-postgres-engine.md`, `2026-11-19-nosql-and-direct-stores.md`
