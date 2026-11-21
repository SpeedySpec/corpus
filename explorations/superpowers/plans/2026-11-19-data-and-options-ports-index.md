# Database layer and options port: plan index

Specs: `docs/superpowers/specs/2026-11-19-database-layer-design.md`,
`docs/superpowers/specs/2026-11-19-options-api-port-design.md` (both approved 2026-11-19).

## Where the work happens

All crates go in the **SpeedySpec server repository** (`git@github.com:SpeedySpec/speedyspec-server.git`), whose
`main` holds `crates/speedyspec-php-runtime` and the Hook API crates. Each plan runs in **its own new worktree of that
repository, branched from `origin/main`** (or from the previous plan's merged branch), named `feat/<plan-slug>`.
Never work in `packages/speedyspec-server-runtime-gate/` or `packages/speedyspec-server/`: those are other sessions'
checkouts. The workspace's `members = ["crates/*"]` picks new crates up without editing `Cargo.toml`.

Changes to `crates/speedyspec-php-runtime/src/worker.rs` (creating `$wpdb`, reading `wp-config.php` at worker
start, refusing drop-ins, hosting the ReDB owner) are **handed to the runtime's owner**: each plan that needs one
ends with a task that writes the request as a note in the shared notes folder
(`/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/`),
and adds nothing to `worker.rs`. Changes each plan needs in wordpress-develop (emptying `class-wpdb.php`, removing
ported functions from `option.php`, `cache.php`) are listed in those notes too, not done by the plans.

## Plans, in order

| # | Plan | Spec part | Needs |
|---|---|---|---|
| 1 | `2026-11-19-data-layer-foundation.md` | DB design: contracts, parser, builder, partials, passes, registry, unit of work, MySQL engine (step 1, first half) | — |
| 2 | `2026-11-19-wpdb-port.md` | DB design: `speedyspec-wp-db`, `speedyspec-wp-db-php`, `compat/wp-db` (step 1, second half) | 1 |
| 3 | `2026-11-19-caching-store-and-object-cache.md` | DB design step 2; options design phase 1; `speedyspec-php-serialize` | 1 |
| 4 | `2026-11-19-options-api-port.md` | Options design phase 2 | 2, 3 |
| 5 | `2026-11-19-sqlite-engine.md` | DB design step 3: neutral passes, SQLite dialect, emulation, corpus, cross-engine replay | 1, 2 |
| 6 | `2026-11-19-postgres-engine.md` | DB design step 4 | 5 |
| 7 | `2026-11-19-nosql-and-direct-stores.md` | DB design step 5: Valkey, ReDB owner process, files, memory stores, `KvCache`, `FileCache` | 1, 3 |
| 8 | `2026-11-19-persistent-cache-transients-network-options.md` | Options design phases 3 and 4 | 4, 7 |

Plans 2 and 3 can run in parallel; so can 5 and 7.

## Cross-plan interfaces

These names are fixed here so plans written in parallel agree. Each owning plan defines the full signatures; others
use them as written.

**Plan 1, `speedyspec-data`** (contracts): `DbValue`, `Row`, `Rows`, `ColumnInfo`, `ExecResult`, `DbError`,
`Statement`, `EmittedSql`, `Prepared`, `EmulatedOp`, `DialectKind`, `Dialect`, `Capabilities`, `UpsertForm`,
`SqlStore`, `Introspector`, `ColumnDef`, `IndexDef`, `NoSqlStore` (with `put_if_absent`), `DirectStore`, `Health`, `EngineKind`,
`EngineHandle`, `Engine`, `EngineConfig`, `EngineFactory`, `EngineLookup`, `Identified`, `Repository`,
`PendingChange`, `ChangeKind`, `UnitOfWork`, `CommitReport`, `CommitError`.
**`speedyspec-data-query`**: `parse_one`, `parse_many`, `Query`, `Partial`, `Clause`, `RewritePass`, `PassChain`,
`EngineRegistry`. **`speedyspec-data-uow`**: `DefaultUnitOfWork`, `IdentityMap`. **`speedyspec-data-mysql`**:
`MySqlEngineFactory`, `MySqlStore`, `MySqlDialect`. **`speedyspec-data-conformance`**: `sql_store_suite`,
`nosql_store_suite`, `direct_store_suite`.

**Plan 2, `speedyspec-wp-db`**: `WpDb` (the port's API, the handler of every command and query), `WpdbState` (port),
`MemoryWpdbState`, `TableNames` (`TableNames::name(&self, logical: &str) -> String`, global vs per-site tables,
`set_blog_id`), `Prepare`, `PlaceholderEscape`, `CharsetRules`, `FieldFormats`, `OutputShape` (`Object`, `ObjectK`,
`ArrayA`, `ArrayN`), `WpDbError`. Commands `Query`, `Insert`, `Replace`, `Update`, `Delete`, … and queries
`GetVar`, `GetRow`, `GetCol`, `GetResults`, `PrepareSql`, … as the DB design lists, keys
`SpeedySpec\WpDb\<Name>Handler`. Other ported APIs send statements with
`WpDb::query_statement(&self, stmt: Statement) -> Result<QueryOutcome, WpDbError>` (fires `query` through the Hook
API's empty-hook fast path, counts, logs). **`speedyspec-wp-db-php`**: `WpDbArea` (the native `wpdb` class and
`$wpdb`).

**Plan 3, `speedyspec-cache`**: `CachingStore` (bytes; `get`, `get_many`, `set`, `add`, `replace`, `delete`,
`increment`, `decrement`, `flush`, `flush_group`, all `(group: &str, key: &str, ..)`, expiry as
`Option<Duration>`), `CacheError`. Implementations `MemoryCache`, `SqlCache`. **`speedyspec-php-serialize`**:
`PhpValue` (its own enum with an `Object { class, properties }` variant, since the Hook API's `Value` has none),
`serialize`, `unserialize`, `maybe_serialize`, `maybe_unserialize`, `is_serialized`, conversions to and from
`speedyspec_hooks::Value`. **`speedyspec-wp-object-cache`**: `ObjectCache` (the API: `get`, `get_multiple`, `set`,
`add`, `replace`, `delete`, `incr`, `decr`, `flush`, `flush_group`, `flush_runtime`, `add_global_groups`,
`add_non_persistent_groups`, `switch_to_blog`, `supports`), its values `CacheValue` (`Php(speedyspec_hooks::Value)`
first level, bytes second). **`speedyspec-wp-object-cache-php`**: `ObjectCacheArea`.

**Plan 4, `speedyspec-wp-options`**: `OptionsApi`, `Option`, `OptionName`, `Autoload`, `SerializedValue`,
`OptionRepository`, `AutoloadPolicy`, ports `Installing`, `Multisite`, `Sanitizing`, `Deprecations`,
`ProtectedOptions`, and `OptionsApi::hooks_fired_by_get(&self, name: &OptionName) -> Vec<HookName>`.
**`speedyspec-wp-options-sql`**: `SqlOptionRepository`, `OptionMapper`. **`-memory`**: `MemoryOptionRepository`.
**`-php`**: `OptionsArea`.

## Shared test infrastructure

- MySQL for tests: `docker run -d --name speedyspec-mysql -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=speedyspec_test -p 33306:3306 mysql:8.4`,
  and `SPEEDYSPEC_TEST_MYSQL_URL=mysql://root:password@127.0.0.1:33306/speedyspec_test`. Tests that need it are
  `#[ignore = "needs MySQL: set SPEEDYSPEC_TEST_MYSQL_URL"]` and run with `-- --ignored`. PostgreSQL and Valkey follow
  the same shape in their plans (`SPEEDYSPEC_TEST_POSTGRES_URL`, `SPEEDYSPEC_TEST_VALKEY_URL`).
- Recorded compatibility uses `speedyspec-compat` as the Hook API does (`compat/<area>/scenarios.json`, fixture
  `wordpress-<version>.json`, `breaks.json`, `source.json`); recording `wp-db` and `options` needs the MySQL above.
- After Linux container test runs, delete that run's `target-linux*/` folder or Docker target volume.

## Reconciliation

The plans were drafted in parallel and then checked against each other. Where a plan's "Assumed from" list differs
from what the owning plan defines, the plan carries a "Reconciled" block that wins. The changes made to owning plans
during that check: `NoSqlStore::put_if_absent` (plan 1), `WpDb::table_name` (plan 2), `ObjectCacheArea::api`
(plan 3). Plan 8 adds `EngineRegistry::routed` to plan 1's registry.
