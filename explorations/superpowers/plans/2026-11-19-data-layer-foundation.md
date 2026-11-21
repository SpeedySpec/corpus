# Data layer foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The general data-access layer under `wpdb`: store contracts, the `sqlparser`-based statement model, the
Eloquent-style builder with partials, the rewrite-pass chain, the engine registry, the unit of work with its identity
map, and the MySQL engine, proven by a shared SQL Store conformance suite against a real MySQL.

**Architecture:** Contracts in `speedyspec-data` (no implementations); the parser, builder and registry in
`speedyspec-data-query`; the unit of work in `speedyspec-data-uow`; the MySQL engine (Abstract Factory: store,
dialect, introspector) in `speedyspec-data-mysql`; the conformance suite in `speedyspec-data-conformance`. Every SQL
statement is parsed; there is no pass-through path. All access is blocking and single-threaded (`Rc`, `RefCell`).

**Tech Stack:** Rust 1.99, edition 2024; `sqlparser` (latest release, `visitor` feature), `mysql` (blocking
driver), `indexmap`, `thiserror` is not used (hand-written `Display`, as the Hook API crates do).

**Spec:** `docs/superpowers/specs/2026-11-19-database-layer-design.md`. Plan index (repository, worktree, shared
names): `docs/superpowers/plans/2026-11-19-data-and-options-ports-index.md`.

## Global Constraints

- Work in a new worktree of `SpeedySpec/speedyspec-server` branched from `origin/main`, branch
  `feat/data-layer-foundation`. New crates go under `crates/`; the workspace's `members = ["crates/*"]` needs no edit.
- Each crate's `Cargo.toml` uses `edition.workspace = true`, `rust-version.workspace = true`,
  `license.workspace = true`, version `0.1.0`, and a one-line `description`.
- Contracts never share a crate with their implementations. No crate named `-domain`, `-app`, `-infra` or `-core`.
- No WordPress, PHP or prefix knowledge anywhere in this plan's crates.
- Every statement reaches an engine only as a parsed `Statement`. No method takes SQL text and executes it.
- A construct a dialect cannot express fails with `DbError::Unsupported { construct, dialect }`; never a silent
  rewrite to something different.
- Entities never load, save or query themselves.
- Patterns used here: Query Object (`Statement`), Builder (`Query`), Interpreter (parsing), Visitor and Chain of
  Responsibility (`PassChain`), Abstract Factory (`Engine`), Strategy (`Dialect`), Registry (`EngineRegistry`),
  Unit of Work and Identity Map (`speedyspec-data-uow`), Adapter (`MySqlStore`).
- Tests that need MySQL are `#[ignore = "needs MySQL: set SPEEDYSPEC_TEST_MYSQL_URL"]`; start MySQL with
  `docker run -d --name speedyspec-mysql -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=speedyspec_test -p 33306:3306 mysql:8.4`
  and run with `SPEEDYSPEC_TEST_MYSQL_URL=mysql://root:password@127.0.0.1:33306/speedyspec_test`.

## Review Focus

- **SQL `sqlparser` cannot parse** (e.g. a MySQL-only hint): the caller gets `DbError::Parse` with a line and column,
  never a panic, on every engine. Test in Task 3.
- **Duplicate column names in a result** (`SELECT a.id, b.id`): `Row::get("id")` returns the last one, as mysqli's
  associative fetch does. Test in Task 1.
- **An `UPDATE` that sets a value to what it already is**: `rows_affected` is 0 on MySQL (changed rows, not matched
  rows). Test in the conformance suite, Task 8.
- **A transaction left open by a request** (PHP's raw `START TRANSACTION` with no `COMMIT`): `reset()` rolls it back,
  and a unit of work refuses to start over it. Tests in Tasks 7 and 8.
- **A request for an engine of the wrong kind** (`sql("cache")` when `cache` is ReDB): `DbError::WrongKind`, naming
  the engine. Test in Task 6.

---

### Task 1: `speedyspec-data` values, rows, errors and the statement model

**Files:**
- Create: `crates/speedyspec-data/Cargo.toml`, `crates/speedyspec-data/src/lib.rs`, `src/value.rs`, `src/row.rs`,
  `src/error.rs`, `src/statement.rs`
- Test: `crates/speedyspec-data/tests/values.rs`

**Interfaces:**
- Produces:
  - `pub use sqlparser::ast;`
  - `pub enum DbValue { Null, Int(i64), UInt(u64), Float(f64), Text(String), Bytes(Vec<u8>) }` with
    `fn to_mysql_text(&self) -> Option<Vec<u8>>` (None for `Null`; integers in decimal; floats in Rust's shortest
    round-trip `{}` form; text as UTF-8 bytes; bytes as is) and `From` impls for `i64`, `u64`, `f64`, `&str`,
    `String`, `Vec<u8>`, `Option<T>`.
  - `pub struct ColumnInfo { pub name: String, pub org_name: String, pub table: String, pub org_table: String,
    pub sql_type: String, pub length: u64, pub max_length: u64, pub nullable: bool, pub flags: u32,
    pub decimals: u8, pub charset: u16 }` (mysqli's field info, which `wpdb::get_col_info` exposes).
  - `pub struct Row` with `Row::new(columns: Rc<[String]>, values: Vec<DbValue>) -> Row`, `columns(&self) ->
    &[String]`, `values(&self) -> &[DbValue]`, `get(&self, column: &str) -> Option<&DbValue>` (last matching
    column), `get_index(&self, i: usize) -> Option<&DbValue>`.
  - `pub struct Rows { pub columns: Vec<ColumnInfo>, pub rows: Vec<Row> }`.
  - `pub struct ExecResult { pub rows_affected: u64, pub insert_id: Option<u64> }`.
  - `pub enum DialectKind { MySql, Postgres, Sqlite }` (`Display`: `mysql`, `postgres`, `sqlite`).
  - `pub enum DbError { Connection { engine: String, message: String }, Parse { message: String, line: u64,
    column: u64 }, Unsupported { construct: String, dialect: DialectKind }, Constraint { engine: String, message:
    String }, Deadlock { engine: String, message: String }, Engine { engine: String, code: Option<String>, message:
    String }, Transaction(String), Config(String), WrongKind { engine: String, expected: &'static str },
    NoEngine(String) }`, implementing `Display` and `std::error::Error`.
  - `pub enum StatementKind { Select, Insert, Replace, Update, Delete, Ddl, Show, Set, Transaction(TxControl),
    Other }`, `pub enum TxControl { Begin, Commit, Rollback, Savepoint, Release, RollbackTo }`.
  - `pub struct Statement` with `Statement::new(tree: ast::Statement, params: Vec<DbValue>) -> Statement`,
    `tree(&self) -> &ast::Statement`, `tree_mut(&mut self) -> &mut ast::Statement`, `params(&self) -> &[DbValue]`,
    `params_mut(&mut self) -> &mut Vec<DbValue>`, `kind(&self) -> StatementKind`, `into_parts(self) ->
    (ast::Statement, Vec<DbValue>)`. Placeholders in the tree are `ast::Value::Placeholder("?")`, one per param, in
    order.
  - `pub struct EmittedSql { pub sql: String, pub params: Vec<DbValue> }`.

- [ ] **Step 1: Write the failing tests** in `tests/values.rs`:
  `to_mysql_text_matches_mysqli` (`DbValue::Null` → `None`; `Int(-5)` → `b"-5"`; `UInt(18446744073709551615)` →
  `b"18446744073709551615"`; `Float(0.1)` → `b"0.1"`; `Text("é")` → `"é".as_bytes()`);
  `row_get_returns_last_duplicate_column` (columns `["id", "id"]`, values `[Int(1), Int(2)]` → `get("id") ==
  Some(&Int(2))`, `get_index(0) == Some(&Int(1))`, `get("missing") == None`);
  `statement_kind_from_tree` (build trees with `sqlparser::parser::Parser::parse_sql(&MySqlDialect {}, ..)` for
  `SELECT 1`, `INSERT INTO t VALUES (1)`, `REPLACE INTO t VALUES (1)`, `UPDATE t SET a = 1`, `DELETE FROM t`,
  `CREATE TABLE t (a int)`, `SHOW TABLES`, `SET NAMES utf8mb4`, `START TRANSACTION`, `COMMIT` and assert the kinds
  `Select, Insert, Replace, Update, Delete, Ddl, Show, Set, Transaction(Begin), Transaction(Commit)`);
  `db_error_display_names_engine` (`Engine { engine: "default", code: Some("1062"), message: "Duplicate" }`
  displays a string containing `default`, `1062` and `Duplicate`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data` — Expected: FAIL (crate and items missing).
- [ ] **Step 3: Implement** the items above. `StatementKind` maps `ast::Statement` variants; an `Insert` whose
  `replace_into` is set is `Replace`; `CREATE`, `ALTER`, `DROP`, `TRUNCATE` are `Ddl`; every `Show*` variant is
  `Show`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data && git commit -m "feat(data): values, rows, errors and the statement model"`

### Task 2: `speedyspec-data` store, engine and unit-of-work contracts

**Files:**
- Create: `crates/speedyspec-data/src/dialect.rs`, `src/store.rs`, `src/engine.rs`, `src/uow.rs`; modify
  `src/lib.rs`
- Test: `crates/speedyspec-data/tests/contracts.rs`

**Interfaces:**
- Consumes: Task 1.
- Produces (all object-safe; all methods `&self`):
  - `pub enum UpsertForm { OnDuplicateKey, OnConflict, None }`; `pub struct Capabilities { pub transactions:
    bool, pub savepoints: bool, pub ordered_ranges: bool, pub expiry: bool, pub atomic_increment: bool, pub
    returning: bool, pub upsert: UpsertForm }`.
  - `pub enum EmulatedOp { ShowTables { like: Option<String> }, ShowColumns { table: String, full: bool, like:
    Option<String> }, ShowIndex { table: String }, ShowCreateTable { table: String }, ShowVariables { like:
    Option<String> }, ShowTableStatus { like: Option<String> }, FoundRows, SetSession { name: String, value:
    String }, Noop }`; `pub enum Prepared { Sql(Vec<EmittedSql>), Emulated(EmulatedOp) }`.
  - `pub trait Dialect { fn kind(&self) -> DialectKind; fn prepare(&self, stmt: &Statement) -> Result<Prepared,
    DbError>; }`
  - `pub struct ColumnDef { pub name: String, pub sql_type: String, pub nullable: bool, pub key: String, pub
    default: Option<String>, pub extra: String, pub collation: Option<String> }` (the fields of MySQL's
    `SHOW FULL COLUMNS`, types written MySQL's way); `pub struct IndexDef { pub name: String, pub unique: bool,
    pub columns: Vec<(String, Option<u32>)> }`.
  - `pub trait Introspector { fn server_version(&self) -> Result<String, DbError>; fn tables(&self, like:
    Option<&str>) -> Result<Vec<String>, DbError>; fn columns(&self, table: &str) -> Result<Vec<ColumnDef>,
    DbError>; fn indexes(&self, table: &str) -> Result<Vec<IndexDef>, DbError>; fn table_collation(&self, table:
    &str) -> Result<Option<String>, DbError>; }`
  - `pub trait SqlStore { fn engine(&self) -> &str; fn dialect(&self) -> &dyn Dialect; fn capabilities(&self) ->
    Capabilities; fn execute(&self, stmt: &Statement) -> Result<ExecResult, DbError>; fn query(&self, stmt:
    &Statement) -> Result<Rows, DbError>; fn begin(&self) -> Result<(), DbError>; fn commit(&self) -> Result<(),
    DbError>; fn rollback(&self) -> Result<(), DbError>; fn savepoint(&self, name: &str) -> Result<(), DbError>;
    fn release_savepoint(&self, name: &str) -> Result<(), DbError>; fn rollback_to_savepoint(&self, name: &str) ->
    Result<(), DbError>; fn in_transaction(&self) -> bool; fn introspect(&self) -> &dyn Introspector; fn
    reset(&self) -> Result<(), DbError>; }` — `in_transaction` also reflects transaction-control statements run
    through `execute`; `reset` rolls back an open transaction and restores the connection's session state.
  - `pub trait NoSqlStore { fn engine(&self) -> &str; fn capabilities(&self) -> Capabilities; fn get(&self,
    keyspace: &str, key: &[u8]) -> Result<Option<Vec<u8>>, DbError>; fn get_many(&self, keyspace: &str, keys:
    &[&[u8]]) -> Result<Vec<Option<Vec<u8>>>, DbError>; fn put(&self, keyspace: &str, key: &[u8], value: &[u8],
    ttl: Option<Duration>) -> Result<(), DbError>; fn put_if_absent(&self, keyspace: &str, key: &[u8], value: &[u8],
    ttl: Option<Duration>) -> Result<bool, DbError> (atomic: true when written, false when the key existed); fn
    delete(&self, keyspace: &str, key: &[u8]) -> Result<bool,
    DbError>; fn range(&self, keyspace: &str, start: Bound<&[u8]>, end: Bound<&[u8]>, limit: Option<usize>) ->
    Result<Vec<(Vec<u8>, Vec<u8>)>, DbError>; fn increment(&self, keyspace: &str, key: &[u8], by: i64) ->
    Result<i64, DbError>; fn clear(&self, keyspace: &str) -> Result<(), DbError>; }` (implemented in plan 7).
  - `pub struct Health { pub ok: bool, pub detail: String }`; `pub trait DirectStore { fn engine(&self) -> &str;
    fn kind(&self) -> &'static str; fn connect(&self) -> Result<(), DbError>; fn health(&self) -> Result<Health,
    DbError>; fn close(&self) -> Result<(), DbError>; fn as_any(&self) -> &dyn Any; }` (`as_any` is how a
    per-system repository reaches its system's client).
  - `pub enum EngineKind { MySql, Postgres, Sqlite, Redb, Valkey, File, Memory }`; `pub enum EngineHandle {
    Sql(Rc<dyn SqlStore>), NoSql(Rc<dyn NoSqlStore>), Direct(Rc<dyn DirectStore>) }` with `as_sql`, `as_nosql`,
    `as_direct` returning `Option<&Rc<..>>`; `pub struct EngineConfig { pub name: String, pub url: String, pub
    options: BTreeMap<String, String> }`.
  - `pub trait Engine { fn name(&self) -> &str; fn kind(&self) -> EngineKind; fn open(&self) ->
    Result<EngineHandle, DbError>; }`; `pub trait EngineFactory { fn scheme(&self) -> &'static str; fn
    create(&self, config: EngineConfig) -> Result<Rc<dyn Engine>, DbError>; }`; `pub trait EngineLookup { fn
    handle(&self, engine: &str) -> Result<EngineHandle, DbError>; fn commit_order(&self) -> Vec<String>; }`.
  - `pub trait Identified { type Id: Clone + Eq + Hash + Debug + 'static; fn identity(&self) -> Self::Id; }`;
    `pub trait Repository<E: Identified> { fn find(&self, id: &E::Id) -> Result<Option<E>, DbError>; fn add(&self,
    entity: E) -> Result<(), DbError>; fn update(&self, entity: E) -> Result<(), DbError>; fn remove(&self, id:
    &E::Id) -> Result<(), DbError>; }` (aggregate repositories add their own methods beside these).
  - `pub enum ChangeKind { New, Dirty, Removed }`; `pub trait PendingChange { fn engine(&self) -> &str; fn
    kind(&self) -> ChangeKind; fn apply(&self, handle: &EngineHandle) -> Result<(), DbError>; }`;
    `pub struct CommitReport { pub committed: Vec<String> }`; `pub enum CommitError { Failed { engine: String,
    error: DbError }, Partial { committed: Vec<String>, failed: String, error: DbError } }` (`Display`, `Error`);
    `pub trait UnitOfWork { fn register(&self, change: Box<dyn PendingChange>); fn after_commit(&self, callback:
    Box<dyn FnOnce()>); fn commit(&self) -> Result<CommitReport, CommitError>; fn discard(&self); fn is_empty(&self)
    -> bool; }`.

- [ ] **Step 1: Write the failing test** `contracts_are_object_safe` in `tests/contracts.rs`: a function taking
  `&dyn SqlStore`, `&dyn NoSqlStore`, `&dyn DirectStore`, `&dyn Dialect`, `&dyn Introspector`, `&dyn Engine`,
  `&dyn EngineFactory`, `&dyn EngineLookup`, `&dyn UnitOfWork`, `&dyn PendingChange` compiles; and
  `engine_handle_accessors` (`EngineHandle::Sql(..)` from a trivial test double gives `as_sql().is_some()` and
  `as_nosql().is_none()`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data --test contracts` — Expected: FAIL (items missing).
- [ ] **Step 3: Implement** the traits and types above. Doc comments state each contract's rule in one sentence
  (for `SqlStore`: "Statements only; a caller with SQL text parses it first").
- [ ] **Step 4: Run** `cargo test -p speedyspec-data` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data): store, engine and unit-of-work contracts"` (add new files first).

### Task 3: The parser front end (`speedyspec-data-query`)

**Files:**
- Create: `crates/speedyspec-data-query/Cargo.toml`, `src/lib.rs`, `src/parse.rs`
- Test: `crates/speedyspec-data-query/tests/parse.rs`

**Interfaces:**
- Consumes: `Statement`, `DbError` (Task 1).
- Produces: `pub fn parse_one(sql: &str) -> Result<Statement, DbError>` (exactly one statement; an empty string or
  only a comment is `Parse { message: "empty query", .. }`; more than one is `Parse { message: "multiple statements
  in one query", .. }`, as mysqli without multi-statements rejects them); `pub fn parse_many(sql: &str) ->
  Result<Vec<Statement>, DbError>` (for `dbDelta`-style batches). Both use `sqlparser::dialect::MySqlDialect` and
  produce statements with no params.

- [ ] **Step 1: Write the failing tests:** `parses_wordpress_core_queries` (each of these parses and keeps its kind:
  ``SELECT option_name, option_value FROM wp_options WHERE autoload IN ('yes','on','auto-on','auto')``,
  ``INSERT INTO `wp_options` (`option_name`, `option_value`, `autoload`) VALUES ('a', 'b', 'auto') ON DUPLICATE KEY UPDATE `option_name` = VALUES(`option_name`), `option_value` = VALUES(`option_value`), `autoload` = VALUES(`autoload`)``,
  ``SELECT SQL_CALC_FOUND_ROWS wp_posts.ID FROM wp_posts WHERE 1=1 ORDER BY wp_posts.post_date DESC LIMIT 0, 10``,
  ``SELECT FOUND_ROWS()``, ``SHOW FULL COLUMNS FROM `wp_posts` ``, ``SET NAMES 'utf8mb4' COLLATE 'utf8mb4_unicode_520_ci'``);
  `parse_error_has_position` (`parse_one("SELECT * FROM WHERE")` is `Err(Parse { line: 1, column, .. })` with
  `column > 0`); `rejects_empty_and_multiple` (`""`, `"-- c"`, `"SELECT 1; SELECT 2"`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-query --test parse` — Expected: FAIL.
- [ ] **Step 3: Implement** `parse_one` and `parse_many`. The line and column come from `sqlparser`'s error
  location when the error carries one, otherwise from its message (`Line: N, Column: M`); `0, 0` when neither has
  it. If a core query above does not parse with the pinned `sqlparser`, stop and report it (the spec's rule: fix the
  parser, never bypass it).
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(data-query): parse SQL into statements with MySQL's dialect"`

### Task 4: The query builder

**Files:**
- Create: `crates/speedyspec-data-query/src/builder.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-data-query/tests/builder.rs`

**Interfaces:**
- Consumes: Tasks 1 and 3.
- Produces: `pub struct Query` (holds a `Statement` and the first error met) with constructors
  `Query::table(table: &str) -> Query` (`SELECT *`), `Query::insert_into(table: &str, columns: &[&str]) -> Query`,
  `Query::replace_into(table: &str, columns: &[&str]) -> Query`, `Query::update(table: &str) -> Query`,
  `Query::delete_from(table: &str) -> Query`, `Query::from_statement(stmt: Statement) -> Query`; modifiers (all
  `&mut self -> &mut Query`): `select(&[&str])`, `where_eq(&str, impl Into<DbValue>)`, `where_in(&str, impl
  IntoIterator<Item = impl Into<DbValue>>)`, `order_by(&str, Order)`, `limit(u64)`, `offset(u64)`, `values(Vec<DbValue>)`
  (one row; call again for more), `on_duplicate_update(&[&str])` (emits `col = VALUES(col)` for each, as core's
  `add_option` writes it), `set(&str, impl Into<DbValue>)`; and `build(&self) -> Result<Statement, DbError>`
  (clones; returns the first error a modifier met, e.g. `set` on a `SELECT` is `Unsupported { construct: "SET on
  SELECT", dialect: MySql }`). `pub enum Order { Asc, Desc }`. Values become `?` placeholders with params, in order.
  Identifiers are emitted unquoted unless they need quoting.

- [ ] **Step 1: Write the failing tests** (compare `build()?.tree().to_string()` and `params()`):
  `select_with_where_in` (`Query::table("wp_options").select(&["option_name","option_value"]).where_in("autoload",
  ["yes","on"]).order_by("option_id", Order::Asc)` → `SELECT option_name, option_value FROM wp_options WHERE
  autoload IN (?, ?) ORDER BY option_id ASC`, params `[Text("yes"), Text("on")]`);
  `insert_on_duplicate` (`Query::insert_into("wp_options", &["option_name","option_value","autoload"]).values(vec![
  "a".into(),"b".into(),"auto".into()]).on_duplicate_update(&["option_name","option_value","autoload"])` → text
  containing `ON DUPLICATE KEY UPDATE option_name = VALUES(option_name)`, three params);
  `update_and_delete` (`UPDATE wp_options SET option_value = ? WHERE option_name = ?`; `DELETE FROM wp_options WHERE
  option_name = ?`); `misuse_is_reported_at_build` (`Query::table("t").set("a", 1i64).build()` is
  `Err(Unsupported { .. })`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-query --test builder` — Expected: FAIL.
- [ ] **Step 3: Implement** by constructing `ast` nodes directly (not by formatting text and parsing it).
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(data-query): an Eloquent-style query builder over the syntax tree"`

### Task 5: Raw SQL into the builder, and partials

**Files:**
- Create: `crates/speedyspec-data-query/src/partial.rs`; modify `src/builder.rs`, `src/lib.rs`
- Test: `crates/speedyspec-data-query/tests/partials.rs`

**Interfaces:**
- Consumes: Tasks 3 and 4.
- Produces: `Query::parse(sql: &str) -> Result<Query, DbError>`; `pub struct Partial` with `Partial::expr(sql:
  &str)`, `Partial::order_by(sql: &str)`, `Partial::group_by(sql: &str)`, `Partial::projection(sql: &str)`,
  `Partial::join(sql: &str)`, each `-> Result<Partial, DbError>` and parsed with the MySQL dialect;
  `pub enum Clause { Projection, Joins, Where, GroupBy, Having, OrderBy, Limit }`; modifiers on `Query`:
  `and_where(Partial)`, `or_where(Partial)` (wrap the existing condition in parentheses), `join(Partial)`,
  `replace(Clause, Partial)`, `remove(Clause)`. They work on `SELECT`, and on `UPDATE`/`DELETE` where the clause
  exists there (`Where`, `OrderBy`, `Limit`); elsewhere they record `Unsupported`.

- [ ] **Step 1: Write the failing tests:** `spec_example` (`Query::parse("SELECT * FROM wp_posts WHERE post_status =
  'publish'")?`, then `.and_where(Partial::expr("post_type = 'page'")?).replace(Clause::OrderBy,
  Partial::order_by("post_date DESC")?).limit(10)` → `SELECT * FROM wp_posts WHERE post_status = 'publish' AND
  post_type = 'page' ORDER BY post_date DESC LIMIT 10`); `or_where_parenthesises` (`WHERE a = 1 AND b = 2` plus
  `or_where(c = 3)` → `WHERE (a = 1 AND b = 2) OR c = 3`); `remove_clause` (removing `OrderBy` and `Limit` from a
  parsed query drops both); `bad_partial_is_parse_error` (`Partial::expr("AND AND")` is `Err(Parse { .. })`);
  `partial_on_insert_is_unsupported`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-query --test partials` — Expected: FAIL.
- [ ] **Step 3: Implement.** `Partial::expr` uses `sqlparser`'s `Parser::parse_expr`; the others parse a minimal
  wrapping `SELECT` and lift the clause out.
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(data-query): parse raw SQL into the builder and change it with partials"`

### Task 6: Rewrite passes and the engine registry

**Files:**
- Create: `crates/speedyspec-data-query/src/passes.rs`, `src/registry.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-data-query/tests/passes.rs`, `tests/registry.rs`

**Interfaces:**
- Consumes: Task 2.
- Produces:
  - `pub struct PassContext<'a> { pub target: DialectKind, pub introspector: Option<&'a dyn Introspector> }`;
    `pub enum Rewrite { Continue, Emulate(EmulatedOp), Split(Vec<Statement>) }`; `pub trait RewritePass { fn
    name(&self) -> &'static str; fn rewrite(&self, stmt: &mut Statement, cx: &PassContext) -> Result<Rewrite,
    DbError>; }`; `pub enum Rewritten { Statements(Vec<Statement>), Emulated(EmulatedOp) }`;
    `pub struct PassChain` with `new() -> PassChain`, `push(&mut self, pass: Box<dyn RewritePass>)`, `run(&self,
    stmt: Statement, cx: &PassContext) -> Result<Rewritten, DbError>`. Passes run in order; `Split` sends each new
    statement through the passes after the splitting one; `Emulate` stops the chain.
  - `pub struct EngineRegistry` with `new() -> EngineRegistry`, `register_factory(&self, factory: Rc<dyn
    EngineFactory>)`, `configure(&self, config: EngineConfig) -> Result<(), DbError>` (scheme is the URL's text
    before `://`; unknown scheme or duplicate name is `Config`), `route(&self, use_name: &str, engine: &str) ->
    Result<(), DbError>` (`NoEngine` if not configured), `engine_for(&self, use_name: &str) -> String` (the route,
    else `"default"`), `sql(&self, use_name: &str) -> Result<Rc<dyn SqlStore>, DbError>`, `nosql(..)`,
    `direct(..)` (`WrongKind` naming the engine when the kind differs), `reset_all(&self) -> Result<(), DbError>`
    (calls `reset` on every opened SQL store; the runtime calls it at request end), and `impl EngineLookup`
    (`handle` opens an engine once and keeps the handle for the worker's life; `commit_order` is configuration
    order).

- [ ] **Step 1: Write the failing tests:** `passes_run_in_order_and_split` (two recording passes; the first splits
  one statement into two; the second sees both; result `Statements` of length 2); `emulate_stops_chain`;
  `registry_routes_and_opens_once` (a test factory with scheme `fake` counting `open` calls: configure `default`
  and `cache`, route `object-cache` → `cache`; `sql("object-cache")` twice opens `cache` once; `engine_for("x") ==
  "default"`); `registry_wrong_kind` (a `fake` engine returning `EngineHandle::Direct` → `sql(..)` is `WrongKind {
  engine: "cache", expected: "sql" }`); `registry_rejects_unknown_scheme_and_route`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-query` — Expected: FAIL.
- [ ] **Step 3: Implement** with `RefCell` and `indexmap::IndexMap` for configuration order.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-query` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(data-query): rewrite-pass chain and engine registry"`

### Task 7: The unit of work and identity map (`speedyspec-data-uow`)

**Files:**
- Create: `crates/speedyspec-data-uow/Cargo.toml`, `src/lib.rs`, `src/identity_map.rs`, `src/unit_of_work.rs`
- Test: `crates/speedyspec-data-uow/tests/unit_of_work.rs`

**Interfaces:**
- Consumes: Task 2 contracts.
- Produces: `pub struct IdentityMap` with `get<E: Identified + 'static>(&self, id: &E::Id) ->
  Option<Rc<RefCell<E>>>`, `insert<E: Identified + 'static>(&self, entity: E) -> Rc<RefCell<E>>` (returns the
  existing entry if present), `remove<E: Identified + 'static>(&self, id: &E::Id)`, `clear(&self)`;
  `pub struct DefaultUnitOfWork` with `new(lookup: Rc<dyn EngineLookup>) -> DefaultUnitOfWork`,
  `identity_map(&self) -> &IdentityMap`, and `impl UnitOfWork`.
- Commit rules (from the spec): group changes by engine; engines in `commit_order()`, then any others in first-
  registration order; per SQL engine with transactions: if `in_transaction()` is already true, fail with
  `Failed { engine, error: Transaction("a transaction opened outside the unit of work is open on <engine>") }`
  before touching anything; `begin`, apply `New`, then `Dirty`, then `Removed` (registration order within each),
  `commit`; on error `rollback` that engine and return `Failed` if no engine committed yet, else `Partial`. Engines
  without transactions (or non-SQL handles) apply in the same order without `begin`. `after_commit` callbacks run,
  in registration order, only when every engine committed. Commit and failure both clear the pending changes, the
  callbacks and the identity map.

- [ ] **Step 1: Write the failing tests** (a recording fake `SqlStore` and `EngineLookup` in the test file):
  `applies_new_dirty_removed_in_order`; `one_transaction_per_engine_in_commit_order` (two engines, log shows
  `begin a, .., commit a, begin b, .., commit b`); `failure_on_first_engine_is_failed_and_rolls_back`;
  `failure_on_second_engine_is_partial` (`Partial { committed: ["a"], failed: "b", .. }`, no callback ran);
  `refuses_open_outside_transaction`; `callbacks_after_success_only`; `identity_map_returns_same_entity`
  (two `insert`s of the same identity give `Rc::ptr_eq` handles).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-uow` — Expected: FAIL.
- [ ] **Step 3: Implement.** `IdentityMap` keys by `(TypeId, Box<dyn Any>)` through a per-type `HashMap<E::Id, Rc<RefCell<E>>>`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-uow` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(data-uow): unit of work with identity map"`

### Task 8: The SQL Store conformance suite (`speedyspec-data-conformance`)

**Files:**
- Create: `crates/speedyspec-data-conformance/Cargo.toml`, `src/lib.rs`, `src/sql.rs`

**Interfaces:**
- Consumes: Tasks 2–5.
- Produces: `pub fn sql_store_suite(store: &dyn SqlStore, scratch_table: &str)`, panicking with the failing check's
  name. Plan 7 adds `nosql_store_suite` and `direct_store_suite` here.
- Checks, in order, each building statements with `Query` or `parse_one` (never text sent directly):
  1. `create`: `CREATE TABLE <t> (id bigint(20) unsigned NOT NULL AUTO_INCREMENT, name varchar(191) NOT NULL
     DEFAULT '', value longtext NOT NULL, PRIMARY KEY (id), UNIQUE KEY name (name)) DEFAULT CHARSET=utf8mb4`
     (dropping it first if it exists).
  2. `insert_id`: inserting `('a', '1')` gives `rows_affected == 1`, `insert_id == Some(1)`.
  3. `select_params`: `SELECT name, value FROM <t> WHERE name = ?` with `a` returns one row whose values'
     `to_mysql_text()` are `b"a"`, `b"1"`, and `columns[0].name == "name"`.
  4. `changed_rows`: `UPDATE <t> SET value = '1' WHERE name = 'a'` gives `rows_affected == 0`; with `'2'`, `1`.
  5. `upsert` (when `capabilities().upsert != None`): the builder's `on_duplicate_update` insert of `('a','3')`
     leaves one row with value `3`.
  6. `rollback`: `begin`, insert `('b','x')`, `rollback`; `b` is absent; `in_transaction()` false after.
  7. `savepoint`: `begin`, insert `c`, `savepoint("s1")`, insert `d`, `rollback_to_savepoint("s1")`, `commit`: `c`
     present, `d` absent.
  8. `raw_transaction_tracked`: `execute(parse_one("START TRANSACTION"))` makes `in_transaction()` true;
     `reset()` makes it false and the row inserted inside it is gone.
  9. `introspect`: `tables(Some(<t>))` contains `<t>`; `columns(<t>)` names `id, name, value` in order with `id`'s
     `key == "PRI"`; `indexes(<t>)` has a unique `name` index.
  10. `parse_error_surfaces`: `parse_one("SELEC 1")` is `Err(Parse { .. })` (documents the no-bypass rule).
  11. `drop`: `DROP TABLE <t>`.

- [ ] **Step 1: Implement** `sql_store_suite` as above (this crate is a test library: its own test is Task 9's run).
- [ ] **Step 2: Run** `cargo build -p speedyspec-data-conformance` — Expected: builds.
- [ ] **Step 3: Commit** — `git commit -m "feat(data-conformance): the SQL Store conformance suite"`

### Task 9: The MySQL engine (`speedyspec-data-mysql`)

**Files:**
- Create: `crates/speedyspec-data-mysql/Cargo.toml`, `src/lib.rs`, `src/dialect.rs`, `src/store.rs`,
  `src/introspect.rs`, `src/factory.rs`
- Test: `crates/speedyspec-data-mysql/tests/dialect.rs`, `tests/conformance.rs`

**Interfaces:**
- Consumes: Tasks 1–8.
- Produces: `pub struct MySqlDialect` (`impl Dialect`, `kind() == MySql`); `pub struct MySqlStore` (`impl SqlStore`,
  engine name from its config); `pub struct MySqlEngineFactory` (`scheme() == "mysql"`; URL
  `mysql://user:password@host[:port]/database`, options `socket`, `charset` (default `utf8mb4`), `collate`
  (default empty: server default)).
- Rules:
  - `MySqlDialect::prepare` runs an empty `PassChain` (MySQL is the source dialect) and emits
    `stmt.tree().to_string()` with `?` placeholders: always `Prepared::Sql`, never emulated.
  - With params the store uses the binary protocol; without, the text protocol, mapping `mysql::Value::Bytes` to
    `DbValue::Text` when valid UTF-8, else `Bytes`; dates and times to `Text` in MySQL's own format.
  - `ColumnInfo` comes from the driver's column metadata; `max_length` is the longest value's byte length in the
    result, as mysqli computes it.
  - `ExecResult.insert_id` is `None` when the driver reports 0. The connection never sets `CLIENT_FOUND_ROWS`, so
    `rows_affected` counts changed rows.
  - Connection init runs `SET NAMES <charset>[ COLLATE <collate>]`. `reset()` rolls back an open transaction,
    resets the connection (`COM_RESET_CONNECTION`), and runs the init statements again.
  - `in_transaction` follows `begin`/`commit`/`rollback` and `Transaction(..)` statements run through `execute`.
  - Driver errors map to `DbError`: 1062 and other integrity codes → `Constraint`; 1213 → `Deadlock`; connection
    loss → `Connection`; the rest → `Engine { code: Some(<number>), .. }`.
  - `MySqlIntrospector` answers from `SHOW TABLES LIKE`, `SHOW FULL COLUMNS`, `SHOW INDEX`, `information_schema`
    and `SELECT VERSION()`, each built through `parse_one`.

- [ ] **Step 1: Write the failing tests:** `tests/dialect.rs` (no MySQL needed): `emits_parsed_sql`
  (`prepare(&parse_one("SELECT `a` FROM `t` LIMIT 0, 10")?)` gives one `EmittedSql` whose text, parsed again,
  equals the original tree); `placeholders_stay_question_marks`. `tests/conformance.rs` (ignored without MySQL):
  `mysql_conformance` (build the store from `SPEEDYSPEC_TEST_MYSQL_URL` through `MySqlEngineFactory` and an
  `EngineRegistry`, then `sql_store_suite(&*store, "speedyspec_conformance")`); `reconnects_after_kill` (run `KILL
  CONNECTION_ID()` from a second connection, then a query succeeds after one `Connection` error, as
  `check_connection` expects).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-mysql` and, with MySQL up, `cargo test -p speedyspec-data-mysql
  -- --ignored` — Expected: FAIL.
- [ ] **Step 3: Implement** the dialect, store, introspector and factory per the rules above.
- [ ] **Step 4: Run** both commands — Expected: PASS, the conformance suite printing no failed check.
- [ ] **Step 5: Run** `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --check` — Expected: clean.
- [ ] **Step 6: Commit** — `git commit -m "feat(data-mysql): the MySQL engine"`
