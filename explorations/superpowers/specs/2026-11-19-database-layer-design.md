# Database layer design

Date: 2026-11-19
Status: approved 2026-11-19

Step 1 of the port order in `2026-11-19-native-plugin-features-design.md`, first half. The options port
(`2026-11-19-options-api-port-design.md`) is the second half and builds on this one.

## Intent

Port WordPress's database layer (`wpdb`) to Rust, and put a general data-access layer under it, so that:

- Rust code in the runtime (the firewall, the response cache, native responders, and every later port) reads and
  writes data without entering PHP;
- PHP plugins and themes keep using `$wpdb` unchanged, through a native `wpdb` class over the Rust port;
- a site can keep WordPress's tables on MySQL, PostgreSQL or SQLite, because every statement, including raw
  MySQL-flavoured SQL from plugins, is parsed and re-emitted in the target engine's dialect;
- other kinds of storage (ReDB, Valkey, files, any system with its own API) serve the caching and the Rust-owned
  stores the native features need.

The why, from the native features design: minimize how much PHP is in flight per request. Features cannot run in Rust
until the data they need is reachable from Rust through a proper port.

## Goals

- **Three kinds of store, each its own contract,** so code always knows which kind it holds:
  - **SQL Store**: SQL statements, the query builder, transactions. Holds WordPress's tables. MySQL, PostgreSQL and
    SQLite.
  - **NoSQL Store**: keyed reads and writes, ranges, transactions where the engine has them. ReDB and Valkey.
  - **Direct Store**: a storage system reached through its own API, neither SQL nor NoSQL (files first). Any system,
    one adapter each.
- **A Caching Store** above them: a higher-level contract (get, set, delete, expiry, groups) whose implementations
  sit on any of the three kinds.
- **Repositories above everything:** a repository contract per aggregate, with one implementation per store kind it
  supports. Rust callers use the repository and never learn which store is underneath.
- **Engines in the manner of SQLAlchemy:** each database is reached through an engine that knows its dialect and its
  optimizations. Several engines are configured at once; each repository is routed to one.
- **A query builder in the manner of Eloquent's** over `sqlparser-rs`'s syntax tree. Raw SQL is parsed into a builder
  instance and changed with partials (adding or replacing clauses and fragments).
- **Every SQL statement goes through the parser,** on every engine, MySQL included (project owner's decision). There
  is no pass-through path.
- **A unit of work** groups changes and commits them together.
- **Plain objects only:** entities are plain Rust structs and plain PHP objects that know nothing about storage. Data
  Mappers and Repositories load and save them.
- **`wpdb` ported in full:** every public method and property, its hooks (`query`, `incompatible_sql_modes`,
  `log_query_custom_data`, `pre_get_table_charset`, `pre_get_col_charset`) and its error behaviour, recorded against
  WordPress and replayed against the port.

## Non-goals

- Active Record, or Eloquent's models. "In the manner of Eloquent" is the builder's style only.
- Compatibility with Laravel or SQLAlchemy APIs. They are style references; SpeedySpec is not Laravel.
- WordPress's tables on ReDB, Valkey or a Direct Store (project owner's decision: WordPress's tables live on SQL
  engines). Raw SQL and the builder target SQL Stores only.
- An async data layer. Access is blocking for now (project owner's decision); see Approaches.
- Two-phase commit across engines.
- Porting the APIs that sit on `wpdb` (options, posts, terms, users). Options has its own design; the others come
  later.
- Database drop-ins (`db.php`) such as HyperDB or the SQLite Database Integration plugin. They replace `wpdb`, which
  the native class now is; a site that installs one gets an error naming the drop-in (designer's call).

## Anti-goals

- A second copy of `wpdb`'s logic, one in PHP and one in Rust. `prepare()`, escaping and the insert/update helpers
  exist once, in Rust.
- A statement that reaches an engine without having been parsed.
- Silent mistranslation: a MySQL construct a dialect cannot express must fail with an error naming the construct, not
  run as something different.
- Entities that load, save or query themselves.
- Storage, PHP or WordPress types in the data-access contracts. WordPress knowledge lives in `speedyspec-wp-db` and
  above; the general layer below knows only data.

## Constraints

- Everything in the native features design's Constraints, notably: stock WordPress never runs; PHP is embedded only
  for what remains; ports follow the Hook API's pattern (commands and queries on the CQRS buses, PHP twins via
  `#[php_port]`, recorded compatibility in `speedyspec-compat`); features work per replica, using `[cluster]`'s
  shared state when configured.
- **Runtime workers are separate processes,** each serving one request at a time
  (`2026-11-19-speedyspec-php-runtime-and-admin-gate-design.md`). Connections are per worker.
- The Hook API is single-threaded (`Rc`), and so is everything that fires hooks from here.
- **Crate naming follows the Hook API port** (`2026-11-18-speedyspec-php-foundation-design.md`): one crate per area
  holding its core and ports, one crate per adapter technology, contracts never in the same crate as their
  implementations, and no crate named `-domain`, `-app`, `-infra` or `-core`.
- Rust 1.99, edition 2024.
- **Where the crates live** (designer's call). In the workspace that holds the runtime (`speedyspec-php-runtime`),
  beside the Hook API's crates: today that is `packages/speedyspec-server-runtime-gate/`, which another session owns,
  and which the runtime design moves the PHP crates into. Each crate is plain and self-contained, so it moves with that
  workspace if it is merged back. The general layer is named `speedyspec-data*`, not `speedyspec-db*`, because the
  CLI workspace already has an unrelated `speedyspec-db` and copies crates between workspaces.
- **Wiring into the worker is handed off.** Creating `$wpdb` at request start, holding connections for the worker's
  lifetime and refusing `db.php` and `object-cache.php` drop-ins happen in `speedyspec-php-runtime`'s worker. This
  design says what the worker must do; the runtime's owner makes the change.
- The design principles in `.claude/rules/design-principles.md`.

## Approaches considered

### The statement model behind the builder and raw SQL

1. **`sqlparser-rs`'s syntax tree as the model.** The builder wraps it; parsing uses its MySQL dialect; re-emitting
   rewrites the tree per target dialect. Covers the most SQL soonest; the contracts depend on a third-party crate's
   types.
2. **Our own statement model, with the parser as an adapter.** The purest dependency direction, but running arbitrary
   plugin SQL means our model must grow to everything MySQL accepts, rebuilding most of `sqlparser-rs`.

**Chosen: 1**, by the project owner ("it exists and should handle the AST"). The dependency is accepted the way a
standard library is: the syntax tree is a value type, and nothing in it touches storage.

**No MySQL fast path** (project owner's decision). The designer proposed sending raw SQL through unparsed on MySQL
when no partial changed it; the project owner chose that every statement is parsed, so every statement has a tree
that Rust can inspect and change, on every engine.

### Blocking or async access

1. **Blocking drivers.** Fits one-request-per-worker, the single-threaded Hook API and PHP's blocking `$wpdb` calls,
   with no async runtime in between.
2. **Async drivers** (`sqlx` and similar), with a `block_on` on every call from PHP and the hooks.

**Chosen: 1**, by the project owner: "it will be slower, but it has to work with PHP." Async can arrive later as
separate adapters behind the same contracts.

### How PHP's `$wpdb` reaches Rust

1. **A native `wpdb` class,** the way `WP_Hook` became native: every method and property is a shim over the Rust
   port, and `class-wpdb.php` is deleted once the recorded compatibility tests pass.
2. **A PHP `wpdb` subclass as a `db.php` drop-in** passing only the query to Rust, leaving `prepare()` and escaping
   in PHP.

**Chosen: 1**, by the project owner. Option 2 would keep two copies of `wpdb`'s logic.

### Where ReDB fits

The designer asked whether ReDB might hold a whole site. The project owner chose separate store kinds instead:
WordPress's tables on the SQL Store, ReDB (and Valkey) as NoSQL Stores for caching and Rust-owned data, and a Direct
Store for systems with their own API. That removes "raw SQL on ReDB" as a problem: it never happens.

## Design

### Layers and crates

Dependencies point inward. The general data-access layer knows nothing of WordPress; `wpdb` sits on it; PHP sits on
`wpdb`.

```
  PHP plugins, themes            Rust features and ports
        |                               |
  speedyspec-wp-db-php  ---->  speedyspec-wp-db (wpdb port: commands, queries)
   (native wpdb class)                  |
                                        v
              speedyspec-data (contracts: stores, engine, dialect, repository, mapper, unit of work, statement model)
                 ^           ^              ^                ^
                 |           |              |                |
   speedyspec-data-query  speedyspec-data-uow   engine adapters   speedyspec-cache (Caching Store contract)
   (parser, builder,    (unit of work,      -mysql -postgres        ^
    partials, passes)    identity map)      -sqlite -redb      cache implementations
                                            -valkey -file      -sql -kv -file -memory
                                            -memory
```

| Crate | Holds | Depends on |
|---|---|---|
| `speedyspec-data` | Contracts only: `SqlStore`, `NoSqlStore`, `DirectStore`, `Engine`, `Connection`, `Dialect`, `Repository`, `DataMapper`, `UnitOfWork`, `IdentityMap`; the statement model (a re-export of `sqlparser`'s AST plus our `Statement` wrapper); `Row`, `DbValue`, `DbError`. | `sqlparser` |
| `speedyspec-data-query` | The parser front end (MySQL dialect), the query builder and partials, the neutral rewrite passes, the engine registry. | `speedyspec-data` |
| `speedyspec-data-uow` | The default unit of work and identity map. | `speedyspec-data` |
| `speedyspec-data-mysql`, `-postgres`, `-sqlite` | SQL engines: connection, dialect emitter, rewrite passes, type mapping, schema introspection, optimizations. | `speedyspec-data`, `speedyspec-data-query`, the driver |
| `speedyspec-data-redb`, `-valkey` | NoSQL engines. | `speedyspec-data`, the driver |
| `speedyspec-data-file` | The first Direct Store: a directory of files. | `speedyspec-data` |
| `speedyspec-data-memory` | In-memory NoSQL and Direct Stores, for tests. Tests that need SQL use SQLite in memory. | `speedyspec-data` |
| `speedyspec-cache` | The Caching Store contract. | none |
| `speedyspec-cache-sql`, `-kv`, `-file`, `-memory` | Caching Store implementations: over any `SqlStore`, over any `NoSqlStore`, over files, in memory. | `speedyspec-cache`, `speedyspec-data` |
| `speedyspec-wp-db` | The `wpdb` port: its functions as commands and queries, its domain services, its outbound ports. | `speedyspec-data`, `speedyspec-data-query`, `speedyspec-wp-hooks`, `speedyspec-cqrs`, `speedyspec-container` |
| `speedyspec-wp-db-php` | The native `wpdb` class and the `$wpdb` global. | `speedyspec-wp-db`, `speedyspec-php`, `speedyspec-php-ext` |

Crate names are the designer's call within the naming rule; the builder may adjust them.

### Patterns, and where they are used

| Pattern | Where |
|---|---|
| **Repository** (PoEAA), one per aggregate | Every Rust-side data access above the stores: options (next design), and later posts, terms, users, firewall counters, redirect rules, the 404 log. One implementation per store kind supported. |
| **Data Mapper** (PoEAA) | Between rows (or keyed records) and plain entities, one per aggregate and store kind. Entities have no storage code. |
| **Unit of Work** (PoEAA) | `speedyspec-data-uow`: tracks new, changed and removed entities per request, and commits them per engine in one transaction each. |
| **Identity Map** (PoEAA) | Inside each unit of work: one in-memory entity per identity, so two loads of the same row in a request give the same entity. |
| **Query Object** (PoEAA) | `Statement`: a query as an object, built by the builder or parsed from SQL, changed with partials, emitted per dialect. |
| **Builder** (GoF) | The Eloquent-style query builder over `Statement`. |
| **Interpreter** (GoF) | Parsing SQL into the syntax tree (`sqlparser`'s parser with its MySQL dialect). |
| **Visitor** (GoF) | Rewrite passes over the tree (`sqlparser`'s `VisitorMut`): MySQL-to-neutral passes, then each dialect's passes, then emission. |
| **Chain of Responsibility** (GoF) | The ordered list of rewrite passes a statement goes through; each pass handles the constructs it knows and passes the tree on. |
| **Abstract Factory** (GoF) | `Engine`: each engine kind makes its family of collaborators (connection, dialect, type mapper, schema introspector) that must match each other. |
| **Strategy** (GoF) | `Dialect` (how a tree becomes that engine's SQL), engine optimizations, and Caching Store backends. |
| **Registry** (PoEAA) | The engine registry: named engines and the routing of repositories to them. |
| **Gateway** (PoEAA) | `wpdb` itself, as WordPress uses it: one object that hides the database behind methods. The port keeps that face for PHP. |
| **Facade** (GoF) | The native `wpdb` class over the `speedyspec-wp-db` commands and queries. |
| **Adapter** (GoF) | Each driver behind its store contract; PHP twins behind Rust ports (`#[php_port]`). |
| **Decorator** (GoF) | Query logging (`SAVEQUERIES`, `log_query_custom_data`) and timing wrapped around a connection; a request-lifetime cache over a persistent Caching Store. |
| **Serialized LOB** (PoEAA) | WordPress's PHP-serialized values in `option_value`, `meta_value` and the like. The mappers decode them; the column stays opaque to SQL. |
| **Observer** | Through the Hook API: `query` and the other `wpdb` hooks. |
| **Command / CQRS** | Every `wpdb` method is a command or query on the buses, as in the Hook API port. |

Patterns not listed are not used. Lazy Load, Table Inheritance and the like have no problem to solve here yet.

### Store contracts

All blocking. Each contract is a trait in `speedyspec-data`. **The general store contracts have no PHP twins**
(designer's call): PHP cannot hold a `Statement`, the contracts crate must not depend on PHP types, and no use needs
a PHP-written SQL engine. Twins exist only for the WordPress-level ports a PHP replacement makes sense for
(`WpdbState` here, `OptionRepository` in the options design), hand-written in their `-php` crates as the Hook API's
`HookStorage` adapter is.

- **`SqlStore`**: `execute(&Statement) -> Result<ExecResult>` (rows affected, insert id), `query(&Statement) ->
  Result<Rows>`, `begin`, `commit`, `rollback`, `savepoint`, `dialect()`, `capabilities()`, `introspect()` (tables,
  columns, indexes, charsets, for the `SHOW` and `information_schema` emulation below). Statements only: there is no
  `execute_str`. A caller with SQL text parses it first.
- **`NoSqlStore`**: named keyspaces (tables in ReDB, key prefixes in Valkey); `get`, `get_many`, `put`, `put_many`,
  `delete`, `range` (ordered engines; Valkey answers with `Unsupported` or a scan), `increment` (atomic, for counters),
  `expire` where the engine has expiry, and `transaction(|tx| ...)` where it has transactions. Keys and values are
  bytes; mappers encode.
- **`DirectStore`**: what any storage system can share and no more: `connect`, `health`, `close`, `kind()`. A Direct
  Store has no generic query surface. Repositories written for one system (files, later object storage or a remote
  service) take that system's own client from its adapter crate. This is the "direct API" path: the contract marks the
  kind and owns the lifecycle; the per-system repository does the work.
- **`Capabilities`** on every store say what it can do (transactions, savepoints, ordered ranges, expiry, atomic
  increment, `RETURNING`, upsert form), so the unit of work and the cache implementations choose their strategy
  without knowing the engine by name.

### Engines and the registry

- An **`Engine`** is a named, configured database: its kind (MySQL, PostgreSQL, SQLite, ReDB, Valkey, file, memory),
  its connection settings, and its factory for the connection, dialect, type mapper and introspector.
- **Several at once.** The registry holds every configured engine by name. `default` is the engine holding
  WordPress's tables. Routing maps a repository or a store use ("options", "object-cache", "page-cache",
  "firewall") to an engine name; anything not routed uses `default`.
- **Configuration.** Rust needs the credentials before any PHP request runs. `DB_NAME`, `DB_USER`, `DB_PASSWORD`,
  `DB_HOST`, `DB_CHARSET`, `DB_COLLATE` and `$table_prefix` define `default`; further engines are named entries with
  a URL (`mysql://`, `postgres://`, `sqlite:///path`, `redb:///path`, `valkey://`, `file:///path`). Where the
  `default` values come from (designer's call, approved with the design): **each worker runs
  `wp-config.php` once at startup** in the embedded PHP, with `ABSPATH` pointing at a stub whose `wp-settings.php` is
  empty (the way the CLI's maintenance design already reads it), and keeps the constants and `$table_prefix`. Sites
  and the CLI's setup need no change. Per request, `wp-config.php` runs as usual and defines its own constants; the
  native class defines none. Further engines come from the site configuration. The exact keys are left to the
  builder.
- **Connections are per worker and live across requests.** At request end the connection is reset: an open
  transaction is rolled back (and logged), session variables set by the request are restored, `FOUND_ROWS` state is
  cleared. A broken connection is reopened on next use, as `wpdb::check_connection()` does.
- **Engine optimizations** (Strategy, per engine), for example: MySQL keeps `SQL_CALC_FOUND_ROWS` natively and uses
  multi-row `INSERT`; PostgreSQL uses `ON CONFLICT` for upserts and `RETURNING` for insert ids; SQLite sets
  `journal_mode=WAL` and `busy_timeout`, and batches `get_many` into one `IN` query. Optimizations never change
  results.

### The statement pipeline

Every SQL statement, from the builder or from raw text, follows one path:

```
 SQL text (from $wpdb or Rust)
   -> `query` filter (PHP and Rust callbacks get the string, as in WordPress)
   -> parse (sqlparser, MySQL dialect)                     -- Interpreter
   -> Statement (the Query Object; partials may change it)
   -> neutral passes  (MySQL-isms the target lacks)        -- Visitor, chained
   -> dialect passes  (the target engine's own forms)      -- Visitor, chained
   -> emit            (the target dialect's SQL text)      -- Strategy
   -> execute on the routed SqlStore
   -> rows mapped back to wpdb's shapes (MySQL semantics)
```

- **Parse failures fail on every engine, MySQL included.** A statement `sqlparser` cannot parse makes `$wpdb->query()`
  return false and set `last_error` to a message naming the parser error, as a MySQL syntax error would. This is a
  documented break (`compat/wp-db/breaks.json`): SQL that MySQL accepts but `sqlparser` does not. Gaps found are fixed
  by extending the parser (upstream first, a vendored dialect patch if needed).
- **`last_query` keeps the text** as it was after the `query` filter, not the emitted text, so plugins that read it
  see what they sent. The emitted text is in the query log.
- **Two entry points, one path** (designer's call). Ported WordPress APIs (options, and later posts, terms, users)
  send their statements through the `wpdb` port's `Query` command, because in WordPress every one of their queries
  passes through `$wpdb->query()`: it fires `query`, is counted in `num_queries` and is logged with `SAVEQUERIES`, and
  query loggers and counters depend on that. The `query` filter goes through the Hook API's empty-hook fast path: with
  no callback on `query` it is only counted, and the builder's tree is used as is. Only when a callback is registered
  is the tree emitted as MySQL text, filtered, and parsed again. Rust-owned stores that WordPress never had (firewall
  counters, redirect rules, the response cache) use `SqlStore` directly, outside `wpdb`, and fire no `wpdb` hooks.
- **A new Rust hook on the tree**, `speedyspec_query_statement` (a filter whose value is the `Statement`), fires after
  parsing, for Rust callers that want to change statements structurally. It is Rust-only: PHP cannot hold a tree, and
  PHP keeps the `query` string filter. If a PHP callback is registered on `query`, the string is filtered first and
  parsed after, so the tree always reflects what PHP left.

### The query builder and partials

The builder is Eloquent-flavoured in style and Rust in shape:

```rust
let stmt = Query::table("options")
    .select(["option_name", "option_value"])
    .where_in("autoload", ["yes", "on", "auto-on", "auto"])
    .order_by("option_id")
    .build();

// Raw SQL into the same builder, then changed with partials:
let mut q = Query::parse("SELECT * FROM wp_posts WHERE post_status = 'publish'")?;
q.and_where(Partial::expr("post_type = 'page'")?)
 .replace(Clause::OrderBy, Partial::order_by("post_date DESC")?)
 .limit(10);
```

- **Partials** are parsed fragments of SQL (an expression, a join, an order-by list, a projection), parsed with the
  same MySQL dialect, or built by the builder. They add to or replace one clause of a statement: `and_where`,
  `or_where`, `replace(Clause::Where | OrderBy | GroupBy | Limit | Projection, ..)`, `join`, `remove(Clause::..)`.
- **Bindings** are values, never interpolated text, until emission; the dialect decides placeholders or literals.
  `wpdb::prepare()`'s output is still a string (WordPress's contract), and it is parsed like any other SQL.
- **Table names** in the builder are logical (`options`); the WordPress layer resolves them to prefixed names
  (`wp_options`, `wp_2_options`) through a table-name resolver, so the general layer has no idea of prefixes.
- Every statement kind is covered: `SELECT` (with joins, subqueries, unions), `INSERT` (with upsert), `UPDATE`,
  `DELETE`, `REPLACE`, and the DDL `dbDelta()` sends (`CREATE TABLE`, `ALTER TABLE`, `CREATE INDEX`).

### Translating MySQL for PostgreSQL and SQLite

WordPress and its plugins write MySQL. The neutral and dialect passes cover, at least (the full list grows from the
corpus, see Testing):

- **Lexical:** backtick identifiers, `"`-strings, `\`-escapes in strings, `LIMIT offset, count`, `<=>`, `REGEXP`,
  `||` as logical OR, `BINARY` comparisons.
- **Statements:** `INSERT ... ON DUPLICATE KEY UPDATE` and `VALUES(col)` (to `ON CONFLICT ... DO UPDATE` and
  `excluded.col`), `REPLACE INTO`, `INSERT IGNORE`, `UPDATE`/`DELETE` with `LIMIT` and `ORDER BY`, multi-table
  `DELETE`, `SQL_CALC_FOUND_ROWS` with `SELECT FOUND_ROWS()` (emulated: the pass runs the count of the same query
  without its limit and keeps it on the connection).
- **Functions:** `NOW()`, `CURDATE()`, `DATE_FORMAT`, `DATE_ADD`/`DATE_SUB` with `INTERVAL`, `UNIX_TIMESTAMP`,
  `FROM_UNIXTIME`, `CONCAT`, `CONCAT_WS`, `IF`, `IFNULL`, `FIELD`, `RAND`, `CHAR_LENGTH`, `LEFT`, `SUBSTRING_INDEX`,
  `GREATEST`, `LEAST`, `CAST(... AS SIGNED|UNSIGNED|CHAR)`, `MATCH ... AGAINST` (to the engine's full-text search,
  or an error naming it).
- **Administrative statements,** answered from the store's `introspect()` instead of passed through: `SHOW TABLES
  [LIKE]`, `SHOW [FULL] COLUMNS FROM`, `DESCRIBE`, `SHOW INDEX FROM`, `SHOW CREATE TABLE`, `SHOW VARIABLES [LIKE]`,
  `SHOW TABLE STATUS`, `SELECT ... FROM information_schema.tables|columns`, `SET NAMES`, `SET SESSION sql_mode`,
  `SELECT VERSION()`, `SELECT DATABASE()`.
- **DDL from `dbDelta()`:** MySQL column types to the target's (`bigint(20) unsigned` to `bigint`, `longtext` to
  `text`, `datetime` default `'0000-00-00 00:00:00'` kept as text on SQLite and mapped on PostgreSQL),
  `AUTO_INCREMENT`, `KEY` inside `CREATE TABLE` (split into `CREATE INDEX`), index prefix lengths (dropped where
  unsupported), `ENGINE=`, `CHARSET=`, `COLLATE=` (dropped).
- **Results with MySQL semantics,** through each engine's type mapper: every column value comes back as a string or
  null, as `mysqli` gives them to `wpdb`; `rows_affected` counts changed rows (MySQL's default), not matched rows;
  `insert_id` comes from `LAST_INSERT_ID()`, `RETURNING` or `last_insert_rowid()`; case-insensitive comparison follows
  WordPress's default collation (`utf8mb4_unicode_520_ci`) as nearly as the engine allows, with the differences listed
  as breaks.

A construct a pass cannot express fails with `DbError::Unsupported { construct, dialect }`; `$wpdb` turns that into
`last_error`. Prior art the builder should read: the SQLite Database Integration plugin's MySQL-to-SQLite translator,
as a list of what plugins actually send.

### The `wpdb` port (`speedyspec-wp-db`)

The port is `wpdb` with WordPress's behaviour, over a `SqlStore` routed as `default`.

- **Commands and queries** follow the foundation's rules. Queries: `Prepare`, `EscLike`, `Escape`, `GetVar`, `GetRow`,
  `GetCol`, `GetResults`, `GetColInfo`, `GetTableCharset`, `GetColCharset`, `GetColLength`, `GetBlogPrefix`, `Tables`,
  `HasCap`, `DbVersion`, `DbServerInfo`, `SupportsCollation`, `GetCharsetCollate`. Commands: `Query` (it returns rows
  or a count: a command carrying a result, the exception this area lists, since raw SQL may write), `Insert`,
  `Replace`, `Update`, `Delete`, `SetPrefix`, `SetBlogId`, `SetCharset`, `SetSqlMode`, `Flush`, `Select`, `Close`,
  `CheckConnection`. The `get_*` methods with a query argument run it first, as WordPress does, and so are commands
  too.
- **Request state** (`last_query`, `last_result`, `last_error`, `num_rows`, `rows_affected`, `insert_id`,
  `num_queries`, `queries`, `col_info`, `func_call`, error switches) lives in a `WpdbState` outbound port, request
  lifetime, with a memory adapter (the default) and a PHP twin.
- **Domain services:** `PlaceholderEscape` and `Prepare` (WordPress's exact `prepare()` rules: `%s`, `%d`, `%f`,
  `%i`, numbered and padded forms, the placeholder escape string, the `_doing_it_wrong` cases), `CharsetRules`
  (`strip_invalid_text` and the column-length and charset checks), `TableNames` (prefixes, `$wpdb->posts` and the
  rest, global vs per-site tables, `set_blog_id`), `FieldFormats` (`process_fields` and the format arrays).
- **Hooks fire through the Rust Hook API**, unchanged names and arguments: `query`, `incompatible_sql_modes`,
  `log_query_custom_data`, `pre_get_table_charset`, `pre_get_col_charset`.
- **Rust callers** use the same commands and queries, or go one level down to the builder and `SqlStore` when they
  want structure instead of text. Both share the worker's connection, so a transaction opened by either is seen by
  both.

### The native `wpdb` class (`speedyspec-wp-db-php`)

- `wpdb` is a native class with WordPress's methods, signatures, defaults and public properties. Table-name properties
  (`$wpdb->posts`, `$wpdb->options`, `$wpdb->prefix`, `$wpdb->base_prefix`, `$wpdb->blogid`) and the request state
  properties read and write through the Rust port. `__get`, `__set`, `__isset`, `__unset` behave as WordPress's do
  for the protected properties they expose.
- `$wpdb` is created at request start by the runtime from the default engine, before `wp-settings.php` would call
  `require_wp_db()`; `require_wp_db()` finds it set and does nothing. A `db.php` drop-in is refused with an error
  naming it.
- Results are PHP arrays and `stdClass` objects, built from the Rust rows at the boundary, in `OBJECT`, `OBJECT_K`,
  `ARRAY_A` and `ARRAY_N` shapes.
- `$wpdb->dbh` is not a `mysqli` object. Code that reaches into `dbh` (calling `mysqli_*` on it) breaks; that is a
  documented break.
- `class-wpdb.php` (`packages/wp-query/wp-query-legacy/classes/class-wpdb.php`) is deleted once the replayer and
  WordPress's own database tests pass, as `plugin.php` was.

### Unit of work and identity map

- A **unit of work** belongs to one request (a PHP request, or one native-path request handled in Rust). Repositories
  register new, changed and removed entities with it; nothing is written until `commit`.
- **Commit** groups changes by engine, opens one transaction per engine (where the engine has transactions), applies
  inserts, then updates, then deletes, in registration order within each, and commits each engine in the configured
  order. There is no two-phase commit: if a later engine's commit fails after an earlier one succeeded, the unit of
  work reports which engines committed (`CommitError::Partial`) and the caller decides. Engines without transactions
  apply changes in order and say so in their capabilities.
- The **identity map** is per unit of work, keyed by aggregate type and identity.
- `$wpdb` from PHP stays autocommit, as in WordPress: each `query()` runs at once. A unit of work in Rust shares the
  worker's connection, so a raw `START TRANSACTION` from PHP and a unit of work never interleave silently: opening a
  unit-of-work transaction while PHP holds one is an error naming both.

### The Caching Store

- **Contract** (`speedyspec-cache`): `get`, `get_many`, `set` (with optional expiry), `add` (only if absent),
  `replace`, `delete`, `increment`, `decrement`, `flush`, `flush_group`, all under a group and key, with values as
  bytes. Groups exist because WordPress's object cache has them.
- **Implementations**, each built on a lower store (Strategy): `SqlCache` over any `SqlStore` (one cache table,
  emitted per dialect through the builder), `KvCache` over any `NoSqlStore` (ReDB, Valkey), `FileCache` over the
  file Direct Store, `MemoryCache` for tests and for request-lifetime caching.
- The object cache port (in the options design) and the page cache, firewall counters and transients use it.
  The native features design's "file, Redis and database" rule reads here as file, Valkey and database, plus ReDB.

### Errors and logging

- `DbError` covers connection, parse (with position), unsupported construct (with dialect), constraint, deadlock,
  and engine errors (with the engine's code and message). `wpdb` maps each to `last_error` and `print_error()`
  exactly as WordPress words MySQL's errors where a recorded scenario shows the wording; otherwise the engine's own.
- With `SAVEQUERIES`, the query log records the original text, the emitted text, the engine name and the time, and
  `log_query_custom_data` fires as in WordPress.

## Testing

- **Recorded compatibility** (`speedyspec-compat`): a `wp-db` catalogue of scenarios (every `prepare()` form, every
  helper, every result shape, errors, charset stripping, prefixes and `set_blog_id`) recorded against WordPress's
  `wpdb` on MySQL and replayed against the port on MySQL, exactly.
- **Cross-engine equivalence:** the same catalogue replayed on PostgreSQL and SQLite must give the MySQL recording's
  results, except where `compat/wp-db/breaks.json` lists a difference (collation edge cases, `MATCH ... AGAINST`).
- **The SQL corpus:** every statement WordPress's own PHPUnit suite sends (logged through the `query` filter), plus
  statements from a list of popular plugins, collected into a corpus. Every statement in it must parse; every one must
  translate for PostgreSQL and SQLite, or be listed as unsupported with its construct.
- **WordPress's own tests:** `tests/phpunit/tests/db.php`, `tests/phpunit/tests/dbdelta.php` and `tests/phpunit/tests/db/`
  run against the native class on each SQL engine.
- **Store contract tests:** one shared suite per contract (`SqlStore`, `NoSqlStore`, `DirectStore`, `CachingStore`,
  `UnitOfWork`) that every implementation runs, the memory ones included.

## Order of work

1. Contracts, the parser front end, the builder and partials, the MySQL engine, the unit of work, `speedyspec-wp-db`
   and the native class; recorded compatibility on MySQL. This unblocks options.
2. `MemoryCache` and `SqlCache`, so the object cache and options can land.
3. SQLite: dialect passes, introspection, the corpus and cross-engine replays.
4. PostgreSQL, the same way.
5. NoSQL and Direct Stores: Valkey, ReDB, files; `KvCache` and `FileCache`. These unblock the shared-state choices in
   the native features design.

### ReDB across processes

ReDB locks its file to one process at a time, and runtime workers are separate processes in several pools (blog,
front-end API, admin API) that need the same stores. Designer's call, approved with the design: **one owner
process per site** inside the runtime (beside the background job runner, supervised the same way) opens the ReDB
file, and workers reach it through a `NoSqlStore` adapter over a local Unix socket (`speedyspec-data-redb` holds both
the owner and the client adapter). This is runtime-internal, not a channel to the server, so it keeps the runtime
design's rule. Each call costs one local round trip; Valkey remains the faster shared choice. With `[cluster]`, ReDB is
per replica, as files are. The alternatives were a private ReDB file per worker (nothing shared, so no `alloptions`
or counters) and ReDB only in single-process setups (CLI, tests).

## Left to the builder

- Crate names within the naming rule; driver crates (`mysql`, `postgres`, `rusqlite`, `redb`, `redis` for Valkey).
- Configuration keys for engines and routing.
- The cache table's schema for `SqlCache`, and key layout for `KvCache`.
- How parse gaps are patched (upstream contribution or a vendored dialect), as long as no statement skips the parser.
- The exact wording of errors where no recording fixes it.
