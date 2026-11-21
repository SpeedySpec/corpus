# PostgreSQL engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `speedyspec-data-postgres`: a PostgreSQL engine whose dialect turns WordPress's MySQL into PostgreSQL and
whose store gives back MySQL's results (strings as mysqli gives them, changed-rows counts, insert ids, zero dates,
case-insensitive text), proven by the SQL Store suite, the SQL corpus and the `wp-db` replay on a real PostgreSQL.

**Architecture:** One adapter crate (Abstract Factory: `PostgresEngine` makes the store, dialect and introspector that
belong together). `PostgresDialect` (Strategy) runs plan 5's neutral pass chain, then its own passes (Visitor, Chain
of Responsibility), then emits `$n` SQL; a crate-internal `PgPlan` tells the store how to count each emitted
statement. `PostgresStore` (Adapter) drives the blocking `postgres` client; `PostgresIntrospector` answers in MySQL's
spelling from `pg_catalog`.

**Tech Stack:** Rust 1.99, edition 2024; `postgres` (latest 0.19.x, blocking), `bytes`, `indexmap`; `sqlparser` as
pinned by plan 1. PostgreSQL 16 or newer with ICU; tests on `postgres:17`.

**Spec:** `docs/superpowers/specs/2026-11-19-database-layer-design.md` (sections "Translating MySQL for PostgreSQL
and SQLite", "Engines and the registry", "Testing"). Plan index: `docs/superpowers/plans/2026-11-19-data-and-options-ports-index.md`.
Plan 1: `2026-11-19-data-layer-foundation.md`. Plan 5: `2026-11-19-sqlite-engine.md`.

## Global Constraints

- Work in a new worktree of `SpeedySpec/speedyspec-server`, branched from plan 5's merged branch (or `origin/main`
  once it holds plan 5), branch `feat/postgres-engine`. Never touch `packages/speedyspec-server-runtime-gate/`,
  `packages/speedyspec-server/` or `crates/speedyspec-php-runtime/src/worker.rs`.
- `crates/speedyspec-data-postgres/Cargo.toml`: `edition.workspace = true`, `rust-version.workspace = true`,
  `license.workspace = true`, version `0.1.0`, one-line `description`. Depends on `speedyspec-data`,
  `speedyspec-data-query`, plan 5's neutral-pass crate, `postgres`, `bytes`, `indexmap`; dev-depends on
  `speedyspec-data-conformance`.
- Caller statements reach PostgreSQL only through `PostgresDialect`. The adapter's own fixed SQL (session setup,
  support objects, catalog queries) is adapter code, not caller SQL, and may be plain text with `$n` parameters.
- A construct with no faithful PostgreSQL form fails with `DbError::Unsupported { construct, dialect: Postgres }`,
  naming it. Never a silent different meaning. Dropping an index (`FULLTEXT KEY`) is allowed: indexes never change
  results.
- Server floor: `server_version_num >= 160000` (for `any_value`), ICU available. Below it, connect fails with
  `DbError::Connection { message: "PostgreSQL 16 or newer is required (server is <version>)" }`.
- Session time zone is always `UTC`; `UNIX_TIMESTAMP`, `FROM_UNIXTIME` and `timestamptz` values use it.
- Patterns: Abstract Factory (`PostgresEngine`), Strategy (`PostgresDialect`), Visitor + Chain of Responsibility (the
  `Pg*` passes after the neutral chain), Adapter (`PostgresStore`, `PgParam`, `PgCell`). `SchemaCache` is a plain
  map, not a pattern. Nothing else.
- Live tests are `#[ignore = "needs PostgreSQL: set SPEEDYSPEC_TEST_POSTGRES_URL"]`. Start PostgreSQL with
  `docker run -d --name speedyspec-postgres -e POSTGRES_PASSWORD=password -e POSTGRES_DB=speedyspec_test -p 35432:5432 postgres:17`
  and run with `SPEEDYSPEC_TEST_POSTGRES_URL=postgres://postgres:password@127.0.0.1:35432/speedyspec_test cargo test -p speedyspec-data-postgres -- --ignored`.
  Each live test uses its own schema (engine option `schema=t_<test_name>`), dropped and recreated at its start.
- After Linux container test runs, delete that run's `target-linux*/` folder or Docker target volume.

## Assumed from plan 5

**Reconciled after plan 5 was written (these win over the roles below and over any conflicting decision in this
plan):**

- The neutral passes live in plan 5's crate **`speedyspec-data-translate`**: `neutral_chain() -> PassChain`,
  `number_placeholders(stmt)` (placeholders become `?1`, `?2`, … before rewriting; PostgreSQL's emitter turns
  `?n` into `$n`), `take_found_rows(stmt) -> Option<Statement>` (the found-rows protocol),
  `answer_emulated(op, introspector, session: &mut SessionVars, found_rows: u64) -> Result<Rows, DbError>` (the
  emulated-op executor), `SessionVars::mysql_defaults(database, version)`.
- **MySQL types are kept in plan 5's shared catalogue, not in column comments.** Use `catalogue_ddl()`,
  `catalogue_table`, `catalogue_writes`, `read_catalogue`, `catalogue_tables`, `is_catalogue_table` and
  `index_object_name` (`"<table>__<index>"`, the same naming this plan chose). Drop this plan's `column_comment` /
  `parse_comment` and the `COMMENT ON COLUMN` statements; MySQL `COMMENT` text goes into the catalogue as plan 5
  stores it. One mechanism serves both engines, so `DESCRIBE` and `dbDelta` round-trip identically.
- The corpus runner is plan 5's `check(entries, dialect, plan)` with `CorpusEntry`/`UnsupportedEntry`; per-engine
  differences go under `engine_breaks.postgres` in `compat/wp-db/breaks.json`, beside plan 5's `engine_breaks.sqlite`;
  the replayer is `WpDbScenarioRunner::for_engine(EngineConfig)`.


Bind each role to plan 5's actual name; do not re-implement any of them here.

- **The neutral pass chain constructor** (returns the neutral passes for a target `DialectKind`), which
  `PostgresDialect` runs first. Assumed outputs: backticks removed, `LIMIT a, b` as `LIMIT b OFFSET a`, `<=>` as
  `IS NOT DISTINCT FROM`, `||` as `OR`, `ON DUPLICATE KEY UPDATE c = VALUES(c)` as `ON CONFLICT [target] DO UPDATE
  SET c = excluded.c`, `INSERT IGNORE` as `ON CONFLICT DO NOTHING`, `KEY`/`UNIQUE KEY` inside `CREATE TABLE` split
  into `CREATE [UNIQUE] INDEX`, `ENGINE=`/`CHARSET=` table options dropped, `IFNULL` as `COALESCE`.
- **The found-rows protocol**: how the neutral pass marks the count statement for `SQL_CALC_FOUND_ROWS`, and the
  per-connection holder the store fills and `EmulatedOp::FoundRows` reads.
- **The emulated-op executor**: `(EmulatedOp, &dyn Introspector, session state) -> Rows` in MySQL's shape, for `SHOW`,
  `DESCRIBE`, `information_schema`, `SET NAMES`, `SET SESSION`, `VERSION()`, `DATABASE()`.
- **The MySQL-compatible version prefix** reported by `Introspector::server_version` (default here: `8.4.0`).
- **The SQL corpus** (statement files and WordPress core schema DDL), **the corpus runner** (translate every
  statement with a given `Dialect`, compare failures against a per-dialect expected-unsupported file), and **the
  cross-engine `wp-db` replayer** (replay `compat/wp-db` against an engine URL, allowing per-engine entries in
  `compat/wp-db/breaks.json`).

**Overlap rule:** if plan 5's neutral chain already rewrites a construct this plan's passes handle (for example
multi-table `DELETE` or `UPDATE ... LIMIT`) into a form PostgreSQL accepts, delete that part of the `Pg*` pass and set
the dialect test's expected text to what the chain now emits; keep the live tests unchanged.

## Decisions

- **Case-insensitive text: an ICU nondeterministic collation, not `citext`.** The store creates
  `speedyspec_ci (provider = icu, locale = 'und-u-ks-level1', deterministic = false)`. Level 1 is UCA primary
  strength, case- and accent-insensitive, as `utf8mb4_unicode_520_ci` is, for `=`, `IN`, `ORDER BY`, `GROUP BY`,
  `DISTINCT` and unique indexes (so `Admin` and `admin` collide in `user_login`, as on MySQL). `citext` is
  accent-sensitive, needs an extension, and loses its behaviour through any text function. Text columns whose MySQL
  collation ends `_bin` get `"C"`; `_cs` gets `"und-x-icu"`; every other text column gets `speedyspec_ci`. PostgreSQL
  17 refuses `LIKE` and regular expressions on nondeterministic collations, so those become `ILIKE`/`~*` on the
  operand recollated to `"und-x-icu"` (case-insensitive, accent-sensitive: a listed break).
- **Identifiers: quoted, stored case kept, references resolved case-insensitively.** DDL keeps each name as written,
  double-quoted. In DML every column reference is resolved against the tables in scope (`SchemaCache`) and emitted
  with its stored spelling; a plain column reference written in another case gets `AS "<as written>"`, as MySQL
  names it; unknown names are kept as written (PostgreSQL then reports the missing column, as MySQL would). Table
  names stay as written (MySQL on Linux is case-sensitive for tables). So `SELECT *` gives `ID`, `SELECT id` gives
  `id`, and reserved words (`key`, `value`, `order`) work as column names.
- **Expression result names** are pinned before any pass runs: every unaliased, non-column projection item gets
  `AS "<sqlparser's MySQL display of the original expression>"` (`COUNT(*)`), so `ARRAY_A` keys match MySQL except for
  spacing (a listed break).
- **Zero dates:** `datetime` is `timestamp(0)`, and `'0000-00-00 00:00:00'` (`'0000-00-00'` for `date`) is stored as
  `-infinity`, which sorts before every date as MySQL's zero date does. Literals and params compared with or assigned
  to a date/datetime column are rewritten; `-infinity` reads back as the zero string; `+infinity` as
  `9999-12-31 23:59:59`.
- **Changed rows:** every `UPDATE` gets `AND (<set columns>) IS DISTINCT FROM (<set expressions>)` (text columns
  compared under `COLLATE "C"`, so a case-only change still counts and is saved); upserts get the same guard in
  `DO UPDATE ... WHERE` and `RETURNING ("xmax" = 0)` so the store counts 1 per insert and 2 per changed update, as
  MySQL does.
- **Insert ids:** `INSERT` into a table with an `auto_increment` column gets `RETURNING "<col>" AS "__speedyspec_id"`;
  `insert_id` is the smallest returned id. An insert that supplies that column explicitly is followed by a hidden
  `setval` statement so the next generated id is past it, as MySQL's counter is.
- **Loose `GROUP BY`:** non-aggregated projection, `HAVING` and `ORDER BY` columns that are neither grouped nor
  functionally dependent on a grouped primary key are wrapped in `any_value(..)` (MySQL's own "any row's value"),
  `SELECT *` under `GROUP BY` is expanded first; select aliases in `HAVING` are replaced by their expressions.
- **Parameters** are bound in PostgreSQL's text format (`PgParam`, `ToSql::encode_format` = `Text`), so the server
  coerces `'5'` into `bigint` the way MySQL coerces; results are decoded from binary by type OID (`PgCell`).
- **MySQL-lenient casts and numeric comparison** use support functions the store installs once per schema
  (`support.sql`): `speedyspec_signed(text) bigint`, `speedyspec_decimal(text) numeric`, `speedyspec_double(text)
  double precision` (leading numeric prefix, else 0; NULL stays NULL), `speedyspec_date(text) date` and
  `speedyspec_datetime(text) timestamp(0)` (invalid → NULL). A text column compared with a numeric literal or numeric
  param is wrapped in `speedyspec_double(..)` (WordPress's `b.option_value < 1700000000`).
- **Index names** are per schema in PostgreSQL, per table in MySQL: index `<i>` of table `<t>` is named `<t>__<i>`,
  plus `__p<len>_<len>..` when any column has a prefix length (`0` for none); the introspector strips both back. A
  name over 63 bytes is `Unsupported`.
- **MySQL types survive in comments:** each column gets `COMMENT ON COLUMN .. IS` with lines `mysql_type=<as
  written, lowercased>`, `collation=<mysql collation>` and `comment=<MySQL COMMENT>`; the table comment holds
  `collation=` and `comment=`. The introspector reports these, displayed as MySQL 8.4 does (integer display widths
  dropped except `tinyint(1)`), so `dbDelta` sees no difference and alters nothing.
- **Statement-level rollback:** inside a transaction each caller statement runs under `SAVEPOINT speedyspec_stmt`
  and is rolled back to it on error, so one failed statement does not abort the transaction (MySQL's behaviour).
  Plans of several statements run atomically (their own `BEGIN`/`COMMIT` outside a transaction).
- **Schema cache freshness:** DDL run through any `PostgresStore` sends `NOTIFY speedyspec_schema`; every connection
  `LISTEN`s and drops its cache on `reset()` when a notice arrived, and drops it on SQLSTATE `42703`/`42P01`. DDL run
  outside SpeedySpec is seen after a worker restart (documented).
- **TLS** is out of this plan: option `sslmode` accepts only `disable`; anything else is `DbError::Config("TLS for
  postgres is not supported yet")`.

## Review Focus

- **A case-only edit** (`UPDATE wp_options SET option_value = 'hello'` over `'Hello'`): saved, `rows_affected == 1`.
  Tests in Tasks 4 and 8.
- **WordPress's expired-transient sweep** (multi-table `DELETE a, b ... b.option_value < <int>`): runs and deletes both
  rows. Test in Task 8.
- **Mixed-case columns** (`ID`): `SELECT *`, `SELECT id`, `SELECT p.ID` each name the column as MySQL does. Tests in
  Tasks 1 and 8.
- **Two plugin tables declaring the same index name** (`KEY name (name)`): both `CREATE TABLE`s succeed and `SHOW
  INDEX` reports `name` for each. Test in Task 7.
- **A failing statement inside a PHP-opened `START TRANSACTION`**: later statements run and `COMMIT` keeps them. Test
  in Task 8.

---

### Task 1: Crate, dialect skeleton, identifiers and placeholders

**Files:**
- Create: `crates/speedyspec-data-postgres/Cargo.toml`, `src/lib.rs`, `src/dialect.rs`, `src/plan.rs`,
  `src/result_names.rs`, `src/schema.rs`, `src/passes/mod.rs`, `src/passes/identifiers.rs`, `src/passes/placeholders.rs`
- Test: `tests/common/mod.rs`, `tests/identifiers.rs`

**Interfaces:**
- Consumes: plan 1 `Dialect`, `Statement`, `Prepared`, `EmittedSql`, `RewritePass`, `PassChain`, `PassContext`,
  `Rewrite`, `Introspector`, `ColumnDef`, `IndexDef`, `parse_one`; plan 5's neutral pass chain constructor.
- Produces:
  - `pub struct PostgresDialect` with `new(introspector: Option<Rc<dyn Introspector>>) -> PostgresDialect`,
    `plan(&self, stmt: &Statement) -> Result<PgPlan, DbError>`, and `impl Dialect` (`kind() == Postgres`;
    `prepare` = `plan` with the counts dropped). `plan` clones the statement, runs `result_names::pin`, then the
    neutral chain, then the `Pg*` passes in this order: `PgDdl`, `PgMultiDelete`, `PgDmlLimit`, `PgReplace`,
    `PgUpsert`, `PgInsertId`, `PgChangedRows`, `PgLooseGroupBy`, `PgExpressions`, `PgCollation`, `PgZeroDates`,
    `PgIdentifiers`, `PgPlaceholders` (each task adds its pass to the chain; unwritten ones are absent).
  - `pub enum PgPlan { Sql(Vec<PgEmitted>), Emulated(EmulatedOp) }`, `pub struct PgEmitted { pub sql: EmittedSql,
    pub count: RowCount }`, `pub enum RowCount { Rows, Affected, Returning, Upsert, CountColumn, Hidden }`, and the
    marker aliases `ID_ALIAS = "__speedyspec_id"`, `INSERTED_ALIAS = "__speedyspec_inserted"`, `COUNT_ALIAS =
    "__speedyspec_count"`, `SEQUENCE_ALIAS = "__speedyspec_sequence"`. `plan` derives `count` from the final
    statement: `SELECT` with `SEQUENCE_ALIAS` → `Hidden`, with `COUNT_ALIAS` → `CountColumn`, other `SELECT`/`SHOW`
    → `Rows`; `RETURNING` with `INSERTED_ALIAS` → `Upsert`, with `ID_ALIAS` → `Returning`; else `Affected`.
  - `pub(crate) struct SchemaCache` in `schema.rs`: `new(introspector: Option<Rc<dyn Introspector>>)`,
    `columns(&self, table: &str) -> Option<Rc<[ColumnDef]>>`, `indexes(&self, table: &str) ->
    Option<Rc<[IndexDef]>>` (filled from the introspector on first use; `None` without one or for an unknown table),
    `resolve(&self, table: &str, written: &str) -> Option<String>` (stored spelling, case-insensitive),
    `clear(&self)`. `PostgresDialect` owns one `Rc<SchemaCache>` and hands a clone to each `Pg*` pass it constructs;
    `PostgresDialect::invalidate_schema(&self)` clears it.
  - `tests/common/mod.rs`: `pub fn assert_pg(d: &PostgresDialect, mysql: &str, params: Vec<DbValue>, expected:
    &[&str]) -> Vec<PgEmitted>` (parses each emitted SQL and each expected string with `sqlparser`'s
    `PostgreSqlDialect`, lowercases every unquoted identifier in both trees, asserts equality and the count of
    statements); `pub fn wp_dialect() -> PostgresDialect` over `FakeIntrospector` holding `wp_options` (`option_id
    bigint(20) unsigned` auto_increment PRI, `option_name varchar(191)` UNI, `option_value longtext`, `autoload
    varchar(20)` MUL), `wp_posts` (`ID bigint(20) unsigned` auto_increment PRI, `post_date datetime`, `post_date_gmt
    datetime`, `post_title text`, `post_status varchar(20)`, `post_type varchar(20)`, `menu_order int(11)`),
    `wp_postmeta` (`meta_id` auto PRI, `post_id bigint(20) unsigned`, `meta_key varchar(255)` indexed `(meta_key,
    191)`, `meta_value longtext`) and `t_two_unique` (`id bigint` auto PRI, `name varchar(50)` UNI, `n int`); `pub
    fn pg_url() -> String`.

- [ ] **Step 1: Write the failing tests** in `tests/identifiers.rs` (each `assert_pg(&wp_dialect(), mysql, params, &[pg])`):
  - `quotes_and_keeps_stored_case`: ``SELECT `option_value` FROM `wp_options` WHERE `option_name` = 'siteurl' LIMIT 1``
    → `SELECT "option_value" FROM "wp_options" WHERE "option_name" = 'siteurl' LIMIT 1`.
  - `resolves_case_and_names_as_written`: `SELECT id, post_title FROM wp_posts WHERE Post_Status = 'publish'` →
    `SELECT "ID" AS "id", "post_title" FROM "wp_posts" WHERE "post_status" = 'publish'`; `SELECT ID FROM wp_posts` →
    `SELECT "ID" FROM "wp_posts"`; `SELECT * FROM wp_posts` → `SELECT * FROM "wp_posts"`.
  - `qualified_and_aliased`: `SELECT p.id FROM wp_posts AS p ORDER BY P.post_date DESC` →
    `SELECT "p"."ID" AS "id" FROM "wp_posts" AS "p" ORDER BY "p"."post_date" DESC`.
  - `select_alias_reference`: `SELECT option_name AS Name FROM wp_options ORDER BY name` →
    `SELECT "option_name" AS "Name" FROM "wp_options" ORDER BY "Name"`.
  - `unknown_column_kept`: `SELECT nope FROM wp_options` → `SELECT "nope" FROM "wp_options"`.
  - `expression_names_pinned`: `SELECT COUNT(*) FROM wp_posts` → `SELECT COUNT(*) AS "COUNT(*)" FROM "wp_posts"`.
  - `numbered_placeholders`: `SELECT option_value FROM wp_options WHERE option_name IN (?, ?) AND autoload = ? LIMIT 2, 5`
    with three params → `SELECT "option_value" FROM "wp_options" WHERE "option_name" IN ($1, $2) AND "autoload" = $3
    LIMIT 5 OFFSET 2`, params unchanged.
  - `plan_counts`: the `SELECT` above plans as `RowCount::Rows`; `DELETE FROM wp_options WHERE option_name = 'a'` as
    `Affected`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test identifiers` — Expected: FAIL (crate missing).
- [ ] **Step 3: Implement** the crate, `PostgresDialect`, `PgPlan`, `SchemaCache`, `result_names::pin`,
  `PgIdentifiers` (quotes table, column, alias and CTE identifiers with `"`; never function names; resolves columns
  through the tables in the statement's `FROM`/`JOIN`/target, then select aliases) and `PgPlaceholders` (each
  `Value::Placeholder("?")` in tree order becomes `$1`, `$2`, ..).
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data-postgres && git commit -m "feat(data-postgres): dialect skeleton, identifiers and placeholders"`

### Task 2: DDL: types, identity, indexes and recorded MySQL types (`PgDdl`)

**Files:**
- Create: `src/types.rs`, `src/passes/ddl.rs`; modify `src/dialect.rs`
- Test: `tests/ddl.rs`

**Interfaces:**
- Consumes: Task 1.
- Produces: `pub(crate) fn pg_type(mysql: &ast::DataType) -> Result<String, DbError>` and `pub(crate) fn
  mysql_display(mysql_type: &str) -> String` (MySQL 8.4's `SHOW COLUMNS` spelling) in `types.rs`; `pub(crate) fn
  index_name(table: &str, index: &str, columns: &[(String, Option<u32>)]) -> Result<String, DbError>` and
  `pub(crate) fn parse_index_name(table: &str, pg_name: &str) -> Option<(String, Vec<Option<u32>>)>`; the column
  comment format `pub(crate) fn column_comment(mysql_type: &str, collation: Option<&str>, comment: Option<&str>) ->
  String` / `parse_comment(text: &str) -> BTreeMap<String, String>`.
- Type map: `tinyint(n)`, `smallint`, `tinyint unsigned` → `smallint`; `mediumint`, `int`, `integer`, `smallint
  unsigned` → `integer`; `int unsigned`, `bigint`, `bigint unsigned` → `bigint`; `decimal(p,s)` → `numeric(p,s)`;
  `float` → `real`; `double` → `double precision`; `char(n)` → `char(n)`; `varchar(n)` → `varchar(n)`;
  `tinytext|text|mediumtext|longtext` → `text`; `*blob`, `binary(n)`, `varbinary(n)` → `bytea`; `datetime`,
  `timestamp` → `timestamp(0)`; `date`, `time` → `date`, `time(0)`; `year` → `smallint`; `enum(..)` →
  `varchar(<longest>)` plus `CHECK ("<col>" IN (..))`; `json` → `text`; anything else → `Unsupported`.
  `AUTO_INCREMENT` → `GENERATED BY DEFAULT AS IDENTITY`; `DEFAULT CURRENT_TIMESTAMP` → `DEFAULT LOCALTIMESTAMP(0)`;
  `ON UPDATE CURRENT_TIMESTAMP` → `Unsupported`; zero-date defaults → `'-infinity'`; `CHARACTER SET` dropped;
  collation per the Decisions; `FULLTEXT KEY` dropped; `TRUNCATE` → `TRUNCATE TABLE .. RESTART IDENTITY`; `CREATE
  TABLE IF NOT EXISTS` makes its split indexes `IF NOT EXISTS`.

- [ ] **Step 1: Write the failing tests** in `tests/ddl.rs`:
  - `wp_options_create` — MySQL `CREATE TABLE wp_options ( option_id bigint(20) unsigned NOT NULL auto_increment,
    option_name varchar(191) NOT NULL default '', option_value longtext NOT NULL, autoload varchar(20) NOT NULL default
    'yes', PRIMARY KEY  (option_id), UNIQUE KEY option_name (option_name), KEY autoload (autoload) ) DEFAULT CHARACTER
    SET utf8mb4 COLLATE utf8mb4_unicode_520_ci` → these statements, in order:
    `CREATE TABLE "wp_options" ("option_id" bigint NOT NULL GENERATED BY DEFAULT AS IDENTITY, "option_name" varchar(191) COLLATE "speedyspec_ci" NOT NULL DEFAULT '', "option_value" text COLLATE "speedyspec_ci" NOT NULL, "autoload" varchar(20) COLLATE "speedyspec_ci" NOT NULL DEFAULT 'yes', PRIMARY KEY ("option_id"))`,
    `CREATE UNIQUE INDEX "wp_options__option_name" ON "wp_options" ("option_name")`,
    `CREATE INDEX "wp_options__autoload" ON "wp_options" ("autoload")`,
    `COMMENT ON TABLE "wp_options" IS 'collation=utf8mb4_unicode_520_ci'`,
    then one `COMMENT ON COLUMN "wp_options"."<col>" IS '<column_comment>'` per column, the first being
    `COMMENT ON COLUMN "wp_options"."option_id" IS 'mysql_type=bigint(20) unsigned'` and the second
    `COMMENT ON COLUMN "wp_options"."option_name" IS 'mysql_type=varchar(191)\ncollation=utf8mb4_unicode_520_ci'`
    (a real newline).
  - `prefix_index_name` — `CREATE TABLE wp_postmeta (meta_id bigint(20) unsigned NOT NULL auto_increment, meta_key
    varchar(255) default NULL, PRIMARY KEY (meta_id), KEY meta_key (meta_key(191)))` emits
    `CREATE INDEX "wp_postmeta__meta_key__p191" ON "wp_postmeta" ("meta_key")`.
  - `type_table` — one `CREATE TABLE t (a tinyint(1), b int(11) unsigned, c decimal(10,2), d double, e datetime NOT
    NULL default '0000-00-00 00:00:00', f date, g blob, h enum('a','bb') NOT NULL default 'a', i varchar(32) COLLATE
    utf8mb4_bin, j timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP)` whose first statement is
    `CREATE TABLE "t" ("a" smallint, "b" bigint, "c" numeric(10,2), "d" double precision, "e" timestamp(0) NOT NULL DEFAULT '-infinity', "f" date, "g" bytea, "h" varchar(2) COLLATE "speedyspec_ci" NOT NULL DEFAULT 'a' CHECK ("h" IN ('a', 'bb')), "i" varchar(32) COLLATE "C", "j" timestamp(0) NOT NULL DEFAULT LOCALTIMESTAMP(0))`.
  - `alter_forms` — `ALTER TABLE wp_options ADD COLUMN foo varchar(20) NOT NULL default ''` →
    `ALTER TABLE "wp_options" ADD COLUMN "foo" varchar(20) COLLATE "speedyspec_ci" NOT NULL DEFAULT ''` and its
    comment; `ALTER TABLE wp_posts CHANGE COLUMN post_title post_title longtext NOT NULL` →
    `ALTER TABLE "wp_posts" ALTER COLUMN "post_title" TYPE text COLLATE "speedyspec_ci", ALTER COLUMN "post_title" SET NOT NULL, ALTER COLUMN "post_title" DROP DEFAULT`
    and its comment; `CHANGE COLUMN a b ..` with different names emits `ALTER TABLE .. RENAME COLUMN "a" TO "b"`
    first; `ALTER TABLE wp_postmeta ADD KEY meta_key (meta_key(191))` →
    `CREATE INDEX "wp_postmeta__meta_key__p191" ON "wp_postmeta" ("meta_key")`; `ALTER TABLE wp_postmeta DROP INDEX
    meta_key` → `DROP INDEX "wp_postmeta__meta_key__p191"` (name rebuilt from the introspected index).
  - `truncate_resets_identity` — `TRUNCATE TABLE wp_options` → `TRUNCATE TABLE "wp_options" RESTART IDENTITY`.
  - `ddl_refusals` — `ON UPDATE CURRENT_TIMESTAMP`, `CHANGE COLUMN .. auto_increment` on an existing column, and an
    index whose generated name exceeds 63 bytes are each `Err(Unsupported { construct, .. })` with `construct`
    naming it; a `FULLTEXT KEY` is dropped (no statement mentions it).
  - `mysql_display_matches_84` — `mysql_display("bigint(20) unsigned") == "bigint unsigned"`, `"int(11)"` → `"int"`,
    `"tinyint(1)"` → `"tinyint(1)"`, `"VARCHAR(191)"` → `"varchar(191)"`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test ddl` — Expected: FAIL.
- [ ] **Step 3: Implement** `types.rs` and `PgDdl` per the map above; `PgDdl` returns `Rewrite::Split` when one
  MySQL statement becomes several. Comment text is a `SingleQuotedString` value (the emitter escapes it).
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-postgres): DDL translation with recorded MySQL types"` (add new files first).

### Task 3: Expressions, collation and zero dates (`PgExpressions`, `PgCollation`, `PgZeroDates`)

**Files:**
- Create: `src/passes/functions.rs`, `src/passes/expressions.rs`, `src/passes/collation.rs`,
  `src/passes/zero_dates.rs`; modify `src/dialect.rs`
- Test: `tests/expressions.rs`

**Interfaces:**
- Consumes: Tasks 1–2 (`SchemaCache` for column types).
- Produces: the three passes. `PgExpressions` rewrites functions (table below), MySQL truthiness (a non-boolean
  expression used as a condition in `WHERE`, `HAVING`, `ON`, `CASE WHEN`, `IF`, `AND`/`OR`/`NOT` operands: integer
  literal → `true`/`false`, else `<e>::numeric <> 0`), `a DIV b` → `trunc(a / b)::bigint`, `ISNULL(x)` → `x IS NULL`,
  `CAST`s, and text-versus-number comparisons. A cast's non-atomic operand is wrapped in `Expr::Nested`; casts are
  built as `CastKind::DoubleColon`.

| MySQL | PostgreSQL |
|---|---|
| `NOW()`, `CURRENT_TIMESTAMP`, `CURDATE()`, `CURTIME()` | `LOCALTIMESTAMP(0)`, `LOCALTIMESTAMP(0)`, `CURRENT_DATE`, `LOCALTIME(0)` |
| `UTC_TIMESTAMP()` | `(now() AT TIME ZONE 'UTC')::timestamp(0)` |
| `UNIX_TIMESTAMP()`, `UNIX_TIMESTAMP(x)` | `floor(extract(epoch FROM now()))::bigint`, `floor(extract(epoch FROM x))::bigint` |
| `FROM_UNIXTIME(n)` | `(to_timestamp(n) AT TIME ZONE 'UTC')::timestamp(0)` |
| `DATE_ADD(d, INTERVAL n u)`, `DATE_SUB(..)`, `d + INTERVAL n u` | `(d + n * INTERVAL '1 u')`, `(d - n * INTERVAL '1 u')` |
| `YEAR/MONTH/DAY/DAYOFMONTH/HOUR/MINUTE/SECOND(d)` | `extract(<FIELD> FROM d)::integer` |
| `DAYOFWEEK(d)` | `(extract(DOW FROM d)::integer + 1)` |
| `WEEK(d)`, `WEEK(d, 0)` | `floor((extract(DOY FROM d) + 6 - extract(DOW FROM d)) / 7)::integer` |
| `WEEK(d, 1)` | `CASE WHEN extract(ISOYEAR FROM d) < extract(YEAR FROM d) THEN 0 WHEN extract(ISOYEAR FROM d) > extract(YEAR FROM d) THEN 53 ELSE extract(WEEK FROM d)::integer END` |
| `WEEK(d, 3)`; other modes | `extract(WEEK FROM d)::integer`; `Unsupported("WEEK mode <n>")` |
| `DATE_FORMAT(d, '<fmt>')` | `to_char(d, '<fmt>')` with `%Y YYYY, %y YY, %m MM, %c FMMM, %d DD, %e FMDD, %H HH24, %h HH12, %i MI, %s SS, %p AM, %j DDD, %M FMMonth, %b Mon, %W FMDay, %a Dy, %% %`; other text double-quoted; other specifiers or a non-literal format → `Unsupported` |
| `CONCAT(a, b, ..)` | `(a || b || ..)`, each operand that is not a string literal cast `::text` (NULL in, NULL out, as MySQL) |
| `CONCAT_WS`, `LEFT`, `RIGHT`, `REPLACE`, `TRIM`, `CHAR_LENGTH`, `MD5`, `COALESCE` | unchanged |
| `GROUP_CONCAT([DISTINCT] x [ORDER BY o] [SEPARATOR s])` | `string_agg([DISTINCT] x::text, s [ORDER BY o])`, `s` default `','` |
| `IF(c, a, b)` | `CASE WHEN <truthy c> THEN a ELSE b END` |
| `FIELD(x, a, b, ..)` | `CASE x WHEN a THEN 1 WHEN b THEN 2 .. ELSE 0 END` |
| `GREATEST(a, b, ..)`, `LEAST(..)` | `CASE WHEN a IS NULL OR b IS NULL .. THEN NULL ELSE GREATEST(a, b, ..) END` |
| `RAND()`; `RAND(seed)` | `random()`; `Unsupported` |
| `LENGTH(x)`, `LCASE`, `UCASE`, `LOCATE(s, x)`, `INSTR(x, s)` | `octet_length(x)`, `lower`, `upper`, `position(s IN x)`, `strpos(x, s)` |
| `SUBSTRING_INDEX(s, d, 1)`, `(s, d, -1)`, `(s, d, k>1)`, `(s, d, -k)` | `split_part(s, d, 1)`, `split_part(s, d, -1)`, `array_to_string((string_to_array(s, d))[1:k], d)`, `array_to_string((string_to_array(s, d))[greatest(cardinality(string_to_array(s, d)) - k + 1, 1):], d)`; non-literal count → `Unsupported` |
| `CAST(x AS SIGNED|UNSIGNED)`, `DECIMAL(p,s)`, `CHAR`, `DATE`, `DATETIME`, `BINARY` | `speedyspec_signed(x::text)`, `speedyspec_decimal(x::text)::numeric(p,s)`, `x::text`, `speedyspec_date(x::text)`, `speedyspec_datetime(x::text)`, `convert_to(x::text, 'UTF8')` |
| `LAST_INSERT_ID()`, `DATABASE()`, `UUID()` | `lastval()`, `current_database()`, `gen_random_uuid()::text` |
| `MATCH (..) AGAINST (..)` | `Unsupported("MATCH ... AGAINST")` |
| text column `<op>` numeric literal or `Int`/`UInt`/`Float` param | `speedyspec_double(<col>) <op> <n>` |

- `PgCollation`: `x LIKE p` → `x COLLATE "und-x-icu" ILIKE p` (`NOT LIKE` → `NOT ILIKE`); `x REGEXP p`, `RLIKE` →
  `x COLLATE "und-x-icu" ~* p` (`NOT REGEXP` → `!~*`); `BINARY x` and `x COLLATE BINARY` (from the neutral chain) →
  `x COLLATE "C"`, and then `LIKE`/`REGEXP` on it stay case-sensitive (`LIKE`, `~`); `ESCAPE` kept.
- `PgZeroDates`: in comparisons (`=`, `<>`, `<`, `>`, `<=`, `>=`, `IN`, `BETWEEN`, `IS [NOT] DISTINCT FROM`),
  `SET` assignments and `INSERT` values whose column is `datetime`/`timestamp`/`date` per `SchemaCache`, the string
  `'0000-00-00 00:00:00'` or `'0000-00-00'` (literal or `Text` param) becomes `'-infinity'`.

- [ ] **Step 1: Write the failing tests** in `tests/expressions.rs`, one `assert_pg` per table row above that has a
  fixed output (with the `FROM "wp_posts"`/`"wp_options"` context), plus:
  - `date_sub_now`: `SELECT ID FROM wp_posts WHERE post_date > DATE_SUB(NOW(), INTERVAL 7 DAY)` →
    `SELECT "ID" FROM "wp_posts" WHERE "post_date" > (LOCALTIMESTAMP(0) - 7 * INTERVAL '1 day')`.
  - `date_format_named`: `SELECT DATE_FORMAT(post_date, '%Y-%m-%d') AS d FROM wp_posts` →
    `SELECT to_char("post_date", 'YYYY-MM-DD') AS "d" FROM "wp_posts"`; `'%U'` → `Err(Unsupported)`.
  - `meta_cast`: `SELECT post_id FROM wp_postmeta WHERE CAST(meta_value AS SIGNED) > 5` →
    `SELECT "post_id" FROM "wp_postmeta" WHERE speedyspec_signed("meta_value"::text) > 5`.
  - `text_number_compare`: `SELECT option_name FROM wp_options WHERE option_value < 1700000000` →
    `SELECT "option_name" FROM "wp_options" WHERE speedyspec_double("option_value") < 1700000000`; the same with `?`
    and param `Int(1700000000)` → `.. speedyspec_double("option_value") < $1`; with param `Text("1")` unchanged.
  - `truthiness`: `SELECT ID FROM wp_posts WHERE 1 AND menu_order` →
    `SELECT "ID" FROM "wp_posts" WHERE true AND "menu_order"::numeric <> 0`.
  - `like_and_regexp`: `SELECT option_name FROM wp_options WHERE option_name LIKE '\\_transient\\_%' AND option_name NOT REGEXP '^_site'`
    → `SELECT "option_name" FROM "wp_options" WHERE "option_name" COLLATE "und-x-icu" ILIKE '\_transient\_%' AND "option_name" COLLATE "und-x-icu" !~* '^_site'`;
    `.. WHERE BINARY option_name LIKE 'A%'` → `.. WHERE "option_name" COLLATE "C" LIKE 'A%'`.
  - `zero_dates`: `SELECT ID FROM wp_posts WHERE post_date_gmt = '0000-00-00 00:00:00'` →
    `SELECT "ID" FROM "wp_posts" WHERE "post_date_gmt" = '-infinity'`; with `?` and `Text("0000-00-00 00:00:00")`
    the emitted params are `[Text("-infinity")]`; `SELECT ID FROM wp_posts WHERE post_title = '0000-00-00 00:00:00'`
    is unchanged apart from quoting.
  - `match_against_unsupported`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test expressions` — Expected: FAIL.
- [ ] **Step 3: Implement** the three passes; `functions.rs` holds the function table and `DATE_FORMAT`/`WEEK`
  builders, `expressions.rs` the truthiness, cast and comparison rules.
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-postgres): MySQL functions, collation and zero dates for PostgreSQL"`

### Task 4: Writes with MySQL's counts (`PgUpsert`, `PgReplace`, `PgInsertId`, `PgChangedRows`, `PgDmlLimit`, `PgMultiDelete`)

**Files:**
- Create: `src/passes/writes.rs`, `src/passes/deletes.rs`; modify `src/dialect.rs`
- Test: `tests/writes.rs`

**Interfaces:**
- Consumes: Tasks 1–3; `SchemaCache` columns (`extra == "auto_increment"`, types) and indexes (unique ones).
- Produces the passes; rules:
  - `PgUpsert` on `ON CONFLICT .. DO UPDATE`: the conflict target is the unique index (primary key included) whose
    columns are all among the inserted columns; none → the clause is removed (MySQL just inserts); several →
    `Unsupported("ON DUPLICATE KEY UPDATE with several unique keys among the inserted columns: <a>, <b>")`. Adds
    `WHERE (<target table>.<set col>, ..) IS DISTINCT FROM (<set expr>, ..)` (text columns under `COLLATE "C"`) and
    `RETURNING .. ("xmax" = 0) AS "__speedyspec_inserted"`. `ON CONFLICT DO NOTHING` keeps no target.
  - `PgReplace` (`REPLACE INTO .. VALUES`): `Split` into `DELETE FROM t WHERE (<u1 cols> = <row values>) OR ..` over
    every unique index fully covered, for every row, then the `INSERT` (no covered unique index → the `INSERT`
    alone). Params of the `DELETE` are copies. `REPLACE .. SELECT` → `Unsupported`.
  - `PgInsertId`: `INSERT` into a table with an `auto_increment` column adds `RETURNING "<col>" AS
    "__speedyspec_id"`; if the insert names that column, `Split` adds
    `SELECT setval(pg_get_serial_sequence('"<t>"', '<col>'), GREATEST((SELECT COALESCE(MAX("<col>"), 0) FROM "<t>"), COALESCE(pg_sequence_last_value(pg_get_serial_sequence('"<t>"', '<col>')), 0), 1)) AS "__speedyspec_sequence"`.
  - `PgChangedRows`: `UPDATE` gets `AND (<set cols>) IS DISTINCT FROM (<set exprs>)` (a single column written as a
    scalar comparison; text columns under `COLLATE "C"`), the original `WHERE` parenthesised; param copies for the
    guard are appended after the existing params.
  - `PgDmlLimit`: `UPDATE`/`DELETE` with `ORDER BY` or `LIMIT` → `.. WHERE "ctid" IN (SELECT "ctid" FROM t WHERE
    <where> ORDER BY .. LIMIT n)`.
  - `PgMultiDelete`: `DELETE a, b FROM <from> WHERE <w>` → one statement: `WITH "__speedyspec_doomed" AS (SELECT
    "a"."ctid" AS "c0", "b"."ctid" AS "c1" FROM <from> WHERE <w>)`, then one data-modifying CTE per distinct target
    table deleting `"ctid" IN (SELECT "cN" .. UNION SELECT "cM" ..)` with `RETURNING 1`, then `SELECT count(*) AS
    "__speedyspec_count"` over their union. If the pinned `sqlparser` cannot hold a data-modifying CTE, stop and
    report it.

- [ ] **Step 1: Write the failing tests** in `tests/writes.rs`:
  - `add_option_upsert`: `INSERT INTO wp_options (option_name, option_value, autoload) VALUES ('a', 'b', 'auto') ON DUPLICATE KEY UPDATE option_name = VALUES(option_name), option_value = VALUES(option_value), autoload = VALUES(autoload)`
    → `INSERT INTO "wp_options" ("option_name", "option_value", "autoload") VALUES ('a', 'b', 'auto') ON CONFLICT ("option_name") DO UPDATE SET "option_name" = "excluded"."option_name", "option_value" = "excluded"."option_value", "autoload" = "excluded"."autoload" WHERE ("wp_options"."option_name" COLLATE "C", "wp_options"."option_value" COLLATE "C", "wp_options"."autoload" COLLATE "C") IS DISTINCT FROM ("excluded"."option_name" COLLATE "C", "excluded"."option_value" COLLATE "C", "excluded"."autoload" COLLATE "C") RETURNING "option_id" AS "__speedyspec_id", ("xmax" = 0) AS "__speedyspec_inserted"`,
    planned as `RowCount::Upsert`.
  - `upsert_without_unique_is_insert`: `INSERT INTO wp_postmeta (post_id, meta_key, meta_value) VALUES (1, 'k', 'v') ON DUPLICATE KEY UPDATE meta_value = VALUES(meta_value)`
    → `INSERT INTO "wp_postmeta" ("post_id", "meta_key", "meta_value") VALUES (1, 'k', 'v') RETURNING "meta_id" AS "__speedyspec_id"`.
  - `upsert_two_uniques_unsupported`: `INSERT INTO t_two_unique (id, name, n) VALUES (1, 'a', 1) ON DUPLICATE KEY UPDATE n = n + 1`
    → `Err(Unsupported { construct })` with `construct` containing `PRIMARY` and `name`.
  - `insert_ignore`: `INSERT IGNORE INTO wp_options (option_name, option_value, autoload) VALUES ('a', 'b', 'no')` →
    `INSERT INTO "wp_options" ("option_name", "option_value", "autoload") VALUES ('a', 'b', 'no') ON CONFLICT DO NOTHING RETURNING "option_id" AS "__speedyspec_id"`.
  - `insert_returns_id`: `INSERT INTO wp_posts (post_title) VALUES (?)` → `INSERT INTO "wp_posts" ("post_title") VALUES ($1) RETURNING "ID" AS "__speedyspec_id"`, `RowCount::Returning`.
  - `explicit_id_moves_sequence`: `INSERT INTO wp_posts (ID, post_title) VALUES (500, 'x')` → the insert with
    `RETURNING "ID" AS "__speedyspec_id"` and then the `setval` statement for `"wp_posts"`/`ID` (`RowCount::Hidden`).
  - `replace_splits`: `REPLACE INTO wp_options (option_name, option_value, autoload) VALUES (?, ?, ?)` with params
    `[a, b, yes]` → `DELETE FROM "wp_options" WHERE "option_name" = $1` (params `[a]`) and
    `INSERT INTO "wp_options" ("option_name", "option_value", "autoload") VALUES ($1, $2, $3) RETURNING "option_id" AS "__speedyspec_id"`.
  - `update_guard`: `UPDATE wp_options SET option_value = ? WHERE option_name = ?` with `[v, n]` →
    `UPDATE "wp_options" SET "option_value" = $1 WHERE ("option_name" = $2) AND "option_value" COLLATE "C" IS DISTINCT FROM $3 COLLATE "C"`,
    params `[v, n, v]`; `UPDATE wp_posts SET post_status = 'draft', menu_order = menu_order + 1 WHERE ID = 3` →
    `UPDATE "wp_posts" SET "post_status" = 'draft', "menu_order" = "menu_order" + 1 WHERE ("ID" = 3) AND ("post_status" COLLATE "C", "menu_order") IS DISTINCT FROM ('draft' COLLATE "C", "menu_order" + 1)`.
  - `delete_limit`: `DELETE FROM wp_options WHERE autoload = 'no' LIMIT 100` →
    `DELETE FROM "wp_options" WHERE "ctid" IN (SELECT "ctid" FROM "wp_options" WHERE "autoload" = 'no' LIMIT 100)`.
  - `expired_transients`: `DELETE a, b FROM wp_options a, wp_options b WHERE a.option_name LIKE '\\_transient\\_%' AND a.option_name NOT LIKE '\\_transient\\_timeout\\_%' AND b.option_name = CONCAT('_transient_timeout_', SUBSTRING(a.option_name, 12)) AND b.option_value < 1700000000`
    → `WITH "__speedyspec_doomed" AS (SELECT "a"."ctid" AS "c0", "b"."ctid" AS "c1" FROM "wp_options" AS "a", "wp_options" AS "b" WHERE "a"."option_name" COLLATE "und-x-icu" ILIKE '\_transient\_%' AND "a"."option_name" COLLATE "und-x-icu" NOT ILIKE '\_transient\_timeout\_%' AND "b"."option_name" = ('_transient_timeout_' || SUBSTRING("a"."option_name", 12)::text) AND speedyspec_double("b"."option_value") < 1700000000), "__speedyspec_d0" AS (DELETE FROM "wp_options" WHERE "ctid" IN (SELECT "c0" FROM "__speedyspec_doomed" UNION SELECT "c1" FROM "__speedyspec_doomed") RETURNING 1) SELECT count(*) AS "__speedyspec_count" FROM "__speedyspec_d0"`,
    planned as `RowCount::CountColumn`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test writes` — Expected: FAIL.
- [ ] **Step 3: Implement** the six passes per the rules above.
- [ ] **Step 4: Run** all tests of the crate — Expected: PASS (earlier tasks' expectations unchanged).
- [ ] **Step 5: Commit** — `git commit -am "feat(data-postgres): upserts, replace, insert ids and changed-rows guards"`

### Task 5: Loose `GROUP BY` (`PgLooseGroupBy`)

**Files:**
- Create: `src/passes/group_by.rs`; modify `src/dialect.rs`
- Test: `tests/group_by.rs`

**Interfaces:**
- Consumes: Tasks 1–4; `SchemaCache` primary keys.
- Produces: `PgLooseGroupBy` per the Decisions. A column is "covered" if it equals a `GROUP BY` expression, sits
  inside an aggregate (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP_CONCAT`'s translation, `any_value`), or belongs to
  a table whose whole primary key is grouped. A `SELECT` with aggregates and no `GROUP BY` treats every bare column as
  uncovered. Wrapped projection items keep the MySQL name (`AS "<column as written>"`).

- [ ] **Step 1: Write the failing tests:**
  - `wraps_ungrouped`: `SELECT post_type, post_status, COUNT(*) AS n FROM wp_posts GROUP BY post_type` →
    `SELECT "post_type", any_value("post_status") AS "post_status", COUNT(*) AS "n" FROM "wp_posts" GROUP BY "post_type"`.
  - `primary_key_dependency_untouched`: `SELECT wp_posts.* FROM wp_posts INNER JOIN wp_postmeta ON (wp_posts.ID = wp_postmeta.post_id) GROUP BY wp_posts.ID`
    → `SELECT "wp_posts".* FROM "wp_posts" INNER JOIN "wp_postmeta" ON ("wp_posts"."ID" = "wp_postmeta"."post_id") GROUP BY "wp_posts"."ID"`.
  - `aggregate_without_group_by`: `SELECT p.post_title, COUNT(*) FROM wp_posts p` →
    `SELECT any_value("p"."post_title") AS "post_title", COUNT(*) AS "COUNT(*)" FROM "wp_posts" AS "p"`.
  - `star_expanded`: `SELECT * FROM wp_postmeta GROUP BY meta_key` →
    `SELECT any_value("meta_id") AS "meta_id", any_value("post_id") AS "post_id", "meta_key", any_value("meta_value") AS "meta_value" FROM "wp_postmeta" GROUP BY "meta_key"`.
  - `having_alias_and_order_by`: `SELECT meta_key, COUNT(*) AS c FROM wp_postmeta GROUP BY meta_key HAVING c > 1 ORDER BY post_id`
    → `SELECT "meta_key", COUNT(*) AS "c" FROM "wp_postmeta" GROUP BY "meta_key" HAVING COUNT(*) > 1 ORDER BY any_value("post_id")`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test group_by` — Expected: FAIL.
- [ ] **Step 3: Implement** `PgLooseGroupBy`; it runs per `SELECT` body, subqueries included.
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-postgres): MySQL's loose GROUP BY through any_value"`

### Task 6: Values in and out as mysqli (`PgParam`, `PgCell`)

**Files:**
- Create: `src/value.rs`; modify `src/lib.rs`
- Test: `tests/values.rs`

**Interfaces:**
- Consumes: plan 1 `DbValue`, `DbError`.
- Produces: `pub(crate) struct PgParam<'a>(pub &'a DbValue)` implementing `postgres::types::ToSql` (`accepts` →
  true; `encode_format` → `Format::Text`; `Null` → `IsNull::Yes`; others write `to_mysql_text()`, except `Bytes` for
  a `BYTEA` parameter, written as `\x` and lowercase hex; non-UTF-8 `Bytes` for a non-`BYTEA` parameter is an
  error naming the parameter's type). If the pinned `postgres` lacks `encode_format`, stop and report it.
  `pub(crate) struct PgCell(pub DbValue)` implementing `FromSql` (`accepts` → true; `from_sql_null` → `Null`)
  decoding binary by type: `bool` → `Int(1|0)`; `int2|int4|int8|oid` → `Int`; `float4|float8` → `Float`;
  `numeric` → `Text` of the decimal with exactly `dscale` fraction digits (`NaN` → `Text("NaN")`);
  `text|varchar|bpchar|name|unknown|json` → `Text`; `bytea` → `Bytes`; `date` (days from 2000-01-01) →
  `Text("YYYY-MM-DD")`; `time` → `Text("HH:MM:SS")`; `timestamp`/`timestamptz` (µs from 2000-01-01 UTC) →
  `Text("YYYY-MM-DD HH:MM:SS")`, fraction appended only when non-zero; `-infinity` → the zero strings,
  `infinity` → `9999-12-31 23:59:59` / `9999-12-31`; `uuid` → `Text` hyphenated; any other type →
  `Unsupported("result column of PostgreSQL type <name>")`. Also `pub(crate) fn mysql_column_info(col:
  &postgres::Column, table: Option<&str>, values: &[DbValue], index: usize) -> ColumnInfo` (name; `sql_type` in the
  spelling `MySqlStore` uses for the matching MySQL type: `int8`→`bigint`, `int4`→`int`, `int2`→`smallint`,
  `numeric`→`decimal`, `float8`→`double`, `float4`→`float`, `text`→`blob`, `varchar`→`var_string`,
  `bpchar`→`string`, `timestamp`→`datetime`, `date`, `time`, `bytea`→`blob`, `bool`→`tinyint`; `charset` 63 for
  binary types, else 246; `max_length` the longest value's byte length).

- [ ] **Step 1: Write the failing tests** (raw bytes, no server): `decodes_integers_and_bools` (`int8` bytes of `-5` →
  `Int(-5)`; `bool` `[1]` → `Int(1)`); `decodes_numeric_with_scale` (ndigits 2, weight 0, sign positive, dscale 2,
  digits `[1, 5000]` → `Text("1.50")`; digits `[1, 2345, 6780]`, weight 1, negative, dscale 3 → `Text("-12345.678")`;
  zero digits, dscale 0 → `Text("0")`); `decodes_timestamps` (0 µs → `"2000-01-01 00:00:00"`; `i64::MIN` →
  `"0000-00-00 00:00:00"`; `i64::MAX` → `"9999-12-31 23:59:59"`; 1 500 000 µs → `"2000-01-01 00:00:01.500000"`;
  date `i32::MIN` → `"0000-00-00"`); `float_round_trips` (`float8` 0.1 → `Float(0.1)`); `unknown_type_unsupported`
  (`Type::POINT`); `params_in_text_format` (`Int(5)` writes `b"5"`; `Null` is `IsNull::Yes`; `Bytes([0x0a, 0xff])`
  for `BYTEA` writes `b"\\x0aff"`; `encode_format` is `Text` for every type).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test values` — Expected: FAIL. (The test reaches the
  crate-private types through a `#[doc(hidden)] pub mod __test` re-export in `lib.rs`.)
- [ ] **Step 3: Implement** `value.rs`.
- [ ] **Step 4: Run** the tests — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-postgres): mysqli-shaped values from PostgreSQL's binary results"`

### Task 7: Connection, support objects and `PostgresIntrospector`

**Files:**
- Create: `src/connection.rs`, `src/error.rs`, `src/support.rs`, `src/support.sql`, `src/introspect.rs`; modify `src/lib.rs`
- Test: `tests/introspect.rs` (live)

**Interfaces:**
- Consumes: Tasks 1–2, 6.
- Produces:
  - `pub(crate) struct Connection` (holds `Option<postgres::Client>`, the config, the schema) with `client(&mut
    self) -> Result<&mut postgres::Client, DbError>` (connects on first use or after a loss: version check, then
    `SET TimeZone = 'UTC'; SET DateStyle = 'ISO, YMD'; SET IntervalStyle = 'postgres'; SET extra_float_digits = 1;
    SET standard_conforming_strings = on; SET client_encoding = 'UTF8'; SET search_path = "<schema>"; LISTEN
    speedyspec_schema`, then `support::ensure`), and `mark_lost(&mut self)`.
  - `pub(crate) fn check_server_version(num: u32, text: &str) -> Result<(), DbError>`.
  - `pub(crate) fn map_error(engine: &str, e: postgres::Error) -> DbError`: SQLSTATE class `23` → `Constraint`;
    `40P01`, `40001` → `Deadlock`; class `08`, closed client or I/O → `Connection`; others → `Engine { code:
    Some(<sqlstate>), .. }`; message is PostgreSQL's.
  - `pub const SUPPORT_SQL: &str = include_str!("support.sql")` (public so the CLI can pre-install it) and
    `pub(crate) fn ensure(client: &mut Client, schema: &str) -> Result<(), DbError>`: if
    `obj_description('speedyspec_signed(text)'::regprocedure, 'pg_proc')` is not `speedyspec-support 1`, run
    `SUPPORT_SQL` in a transaction under `pg_advisory_xact_lock(7316)`. `support.sql` creates the `speedyspec_ci`
    collation and the five functions (`LANGUAGE plpgsql IMMUTABLE`, `RETURNS NULL ON NULL INPUT`).
  - `pub struct PostgresIntrospector` with `new(conn: Rc<RefCell<Connection>>) -> PostgresIntrospector` and
    `impl Introspector`: `server_version` → `"8.4.0-speedyspec-postgres-<server_version>"`
    (prefix per plan 5); `tables(like)` from `pg_class` in the schema, kinds `r`/`p`, sorted; `columns(t)` from
    `pg_attribute` in `attnum` order, reporting the comment's `mysql_type` through `mysql_display` (reverse map for
    columns without one: `int8`→`bigint`, `int4`→`int`, `int2`→`smallint`, `numeric(p,s)`→`decimal(p,s)`,
    `float8`→`double`, `float4`→`float`, `text`→`text`, `varchar(n)`, `bpchar(n)`→`char(n)`,
    `timestamp`→`datetime`, `date`, `time`, `bytea`→`longblob`, `bool`→`tinyint(1)`), `key` `PRI`/`UNI`/`MUL` by
    MySQL's rules, `default` with casts stripped and `-infinity` as the zero date, `extra` `auto_increment` for
    identity columns, `collation` from the comment (else `utf8mb4_unicode_520_ci` for `speedyspec_ci`, `utf8mb4_bin`
    for `"C"`, none for non-text); `indexes(t)` with `PRIMARY` for the primary key and names and prefix lengths from
    `parse_index_name`; `table_collation(t)` from the table comment, default `utf8mb4_unicode_520_ci`. It caches
    nothing (the dialect's `SchemaCache` does). Its catalog queries are fixed adapter SQL with `$n`.

- [ ] **Step 1: Write the failing tests** (live; each sets up its schema, runs `PostgresDialect::new(None)`'s output
  for the DDL on a `Connection`): `support_functions_are_lenient` (`speedyspec_signed('12abc') = 12`, `('') = 0`,
  `(NULL) IS NULL`; `speedyspec_double('1.5e3x') = 1500`; `speedyspec_date('2024-02-30') IS NULL`;
  `speedyspec_decimal(' -3.25 ') = -3.25`); `ci_collation_behaves_like_520_ci` (`'Hello' = 'hello' COLLATE
  speedyspec_ci` and `'élan' = 'ELAN' COLLATE speedyspec_ci` are true); `wp_options_round_trip` (after the
  `wp_options` DDL from Task 2: `columns` gives `option_id` `bigint unsigned` `PRI` `auto_increment`, `option_name`
  `varchar(191)` `UNI` collation `utf8mb4_unicode_520_ci`, `option_value` `longtext`, `autoload` `varchar(20)` `MUL`
  default `yes`; `indexes` gives `PRIMARY`, `option_name` (unique), `autoload`; `table_collation` is
  `utf8mb4_unicode_520_ci`); `same_index_name_in_two_tables` (`CREATE TABLE a_one (id int, name varchar(20), KEY name
  (name))` and the same for `a_two` both succeed; each reports index `name`); `prefix_length_reported`
  (`wp_postmeta`'s `meta_key` index columns are `[("meta_key", Some(191))]`); `zero_date_default_reported`
  (`default == Some("0000-00-00 00:00:00")`); `server_version_shape` (starts with `8.4.0-speedyspec-postgres-`);
  and, not live, `refuses_old_server` (`check_server_version(150004, "15.4")` is `Err(Connection { .. })`,
  `160000` is `Ok`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test introspect -- --include-ignored` with PostgreSQL
  up — Expected: FAIL.
- [ ] **Step 3: Implement** the files above.
- [ ] **Step 4: Run** the same command — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-postgres): connection setup, support objects and the introspector"`

### Task 8: `PostgresStore`, the factory, and the SQL Store suite

**Files:**
- Create: `src/store.rs`, `src/factory.rs`; modify `src/lib.rs`
- Test: `tests/conformance.rs`, `tests/store.rs` (live)

**Interfaces:**
- Consumes: Tasks 1–7; plan 1 `SqlStore`, `Engine`, `EngineFactory`, `EngineHandle`, `EngineConfig`,
  `Capabilities`, `EngineRegistry`, `sql_store_suite`; plan 5's emulated-op executor and found-rows protocol.
- Produces:
  - `pub struct PostgresStore` (`impl SqlStore`; engine name from its config). `capabilities()` = transactions,
    savepoints, `returning: true`, `upsert: UpsertForm::OnConflict`, the rest false. `query`/`execute` plan the
    statement; `Emulated` goes to plan 5's executor with the introspector; each `PgEmitted` runs with `PgParam`s
    through a per-connection prepared-statement cache (`IndexMap<String, postgres::Statement>`, 256 entries, oldest
    evicted; cleared on DDL, `reset` and reconnect): `Rows` → `Rows` of `PgCell`s with `mysql_column_info`;
    `Affected` → command-tag count; `Returning` → rows returned; `Upsert` → 1 per row with `__speedyspec_inserted`
    true, 2 otherwise; `CountColumn` → the value; `Hidden` → not counted. `insert_id` = smallest `__speedyspec_id`
    returned, `None` when none. `SELECT LAST_INSERT_ID()` (sole projection, no `FROM`) answers from the store's last
    insert id (0 when none). A plan of several statements runs inside `BEGIN`/`COMMIT` when no transaction is open.
    Inside a transaction each caller statement runs under `SAVEPOINT speedyspec_stmt` (released on success, rolled
    back to on error). After any `Ddl` statement: `NOTIFY speedyspec_schema` and `dialect.invalidate_schema()`.
    SQLSTATE `42703`/`42P01` also invalidates. `begin`/`commit`/`rollback`/savepoints and `Transaction(..)`
    statements keep `in_transaction`. A `Connection` error marks the connection lost; the next call reconnects.
    `reset()`: rollback if open, `DISCARD ALL`, re-run the session setup, clear the statement cache, found-rows state
    and last insert id, and drop the schema cache if a `speedyspec_schema` notification arrived.
  - `pub struct PostgresEngine` (`impl Engine`, `kind() == Postgres`, opens one store per worker) and
    `pub struct PostgresEngineFactory` (`scheme() == "postgres"`; URL `postgres://user:password@host[:port]/database`
    parsed with `postgres::Config::from_str`; options `schema` (default `public`), `application_name` (default
    `speedyspec`), `sslmode` (only `disable`)).

- [ ] **Step 1: Write the failing tests:** `tests/conformance.rs`: `postgres_conformance` (build through
  `PostgresEngineFactory` and an `EngineRegistry` from `pg_url()` with `schema=t_conformance`, then
  `sql_store_suite(&*store, "speedyspec_conformance")`); `reconnects_after_terminate` (`SELECT
  pg_terminate_backend(<pid>)` from a second client, then one `Connection` error and the next query succeeds).
  `tests/store.rs`, each on the WordPress tables created through the store:
  - `changed_rows_match_mysql`: `add_option`'s upsert of `('a','b','auto')` → `rows_affected 1`, `insert_id Some(1)`;
    the same again → `0`; with value `'c'` → `2`; `UPDATE wp_options SET option_value = 'c' WHERE option_name = 'a'`
    → `0`; `'C'` → `1` and a following `SELECT` returns `C`; `INSERT IGNORE` of `'a'` → `0`, `insert_id None`.
  - `result_names_follow_mysql`: `SELECT * FROM wp_posts` names `ID`; `SELECT id FROM wp_posts` names `id`;
    `SELECT COUNT(*) FROM wp_posts` names `COUNT(*)` and returns `Int` whose text is `"0"`.
  - `zero_dates_round_trip`: insert a post with `post_date_gmt '0000-00-00 00:00:00'` and one with `'2024-01-01
    00:00:00'`; `SELECT post_date_gmt .. ORDER BY post_date_gmt` gives `"0000-00-00 00:00:00"` first; `WHERE
    post_date_gmt = '0000-00-00 00:00:00'` finds one row.
  - `case_insensitive_text`: `option_name 'Hello'` then `'hello'` → `Err(Constraint)`; `WHERE option_name =
    'HÉLLO'` finds it; `LIKE 'HEL%'` finds it.
  - `explicit_id_bumps_identity`: insert post `ID 500`, then one without `ID` gets `insert_id Some(501)`.
  - `delete_expired_transients_runs`: insert `_transient_x` / `_transient_timeout_x = '100'` and `_transient_y` /
    `_transient_timeout_y = '9999999999'`; the multi-table `DELETE` with `< 1700000000` gives `rows_affected 2` and
    leaves only the `y` pair.
  - `error_inside_transaction_does_not_abort_it`: `execute(parse_one("START TRANSACTION"))`, insert `p`, insert `p`
    again (`Err(Constraint)`), insert `q`, `COMMIT`: `p` and `q` present.
  - `replace_is_atomic`: `REPLACE` whose insert violates `NOT NULL` leaves the old row in place.
  - `mysqli_values`: `SELECT 1 = 1, CAST('1.5' AS DECIMAL(10,2)), NULL` gives `Int(1)`, `Text("1.50")`, `Null`.
  - `found_rows_and_show`: `SELECT SQL_CALC_FOUND_ROWS ID FROM wp_posts LIMIT 1` then `SELECT FOUND_ROWS()` gives the
    full count; `SHOW FULL COLUMNS FROM wp_options` gives `Type` `bigint unsigned`, `Key` `PRI`, `Extra`
    `auto_increment` for `option_id`; `SELECT LAST_INSERT_ID()` after an insert gives its id.
  - `reset_rolls_back_and_clears`: raw `START TRANSACTION`, insert, `reset()`: `in_transaction()` false, row gone.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres -- --include-ignored` with PostgreSQL up — Expected:
  FAIL.
- [ ] **Step 3: Implement** the store, engine and factory per the rules above.
- [ ] **Step 4: Run** the same command — Expected: PASS, the conformance suite printing no failed check.
- [ ] **Step 5: Run** `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --check` — Expected: clean.
- [ ] **Step 6: Commit** — `git commit -am "feat(data-postgres): the PostgreSQL store and engine factory"`

### Task 9: The SQL corpus on PostgreSQL

**Files:**
- Create: `crates/speedyspec-data-postgres/tests/corpus.rs`; the PostgreSQL expected-unsupported file at plan 5's
  per-dialect location (for example `corpus/unsupported-postgres.json`)

**Interfaces:**
- Consumes: plan 5's corpus and corpus runner; Tasks 1–8.

- [ ] **Step 1: Write the tests:** `corpus_translates_for_postgres` (not live: the runner with
  `PostgresDialect::new(Some(<FakeIntrospector loaded from the corpus's WordPress core schema>))`; every statement
  translates or is listed with its construct); `corpus_prepares_on_postgres` (live: create the core schema through
  `PostgresStore`, then `client.prepare` every translated non-DDL statement without running it; a failure fails the
  test with the statement and error unless its SQLSTATE is `42P01` for a plugin table not in the schema, which is
  counted and printed as unverified).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-postgres --test corpus -- --include-ignored` — Expected: FAIL,
  listing statements.
- [ ] **Step 3: Fix each failure** in the owning pass (with a new `assert_pg` case in that pass's test file), or list
  it in the unsupported file with its construct when PostgreSQL has no faithful form. Never list a statement only to
  pass.
- [ ] **Step 4: Run** the same command — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "test(data-postgres): the SQL corpus translates and prepares on PostgreSQL"`

### Task 10: Cross-engine replay of `compat/wp-db` on PostgreSQL

**Files:**
- Modify: `compat/wp-db/breaks.json`; the replay test's engine list where plan 5 keeps it
- Test: plan 5's replay test, with PostgreSQL added

**Interfaces:**
- Consumes: plan 5's cross-engine replayer; plan 2's `WpDb`; Tasks 1–8.
- Allowed break causes (each `breaks.json` entry names exactly one, engine `postgres`): `pg-like-accent-sensitive`,
  `pg-pad-space` (trailing spaces count in `=`), `pg-collation-order` (ICU root versus UCA 5.2.0 order),
  `pg-regex-dialect`, `pg-unsigned` (no unsigned ranges; `CAST AS UNSIGNED` keeps the sign), `pg-text-number-compare`
  (only literals and params are converted), `pg-assign-order` (`SET a = b, b = a` uses old values),
  `pg-result-names` (expression spacing), `pg-upsert-several-unique`, `pg-partial-zero-dates` (`'2024-00-00'`),
  `pg-timezone`, `pg-match-against`, `pg-week-modes`, `pg-distinct-order-by`, `pg-on-update-timestamp`,
  `pg-index-name-length`, `pg-nul-bytes`, `pg-bool-arithmetic`, `pg-insert-ignore-scope` (only conflicts are
  ignored), `pg-dbh`, `pg-external-ddl` (schema changed outside SpeedySpec is seen after restart).

- [ ] **Step 1: Add PostgreSQL** (from `SPEEDYSPEC_TEST_POSTGRES_URL`) to the replay's engines, `#[ignore]` as above.
- [ ] **Step 2: Run** the replay with PostgreSQL and MySQL up — Expected: FAIL, listing differing scenarios.
- [ ] **Step 3: For each difference:** fix it in the owning pass or the store (with a unit or live test there), or,
  when its cause is one of the allowed causes, add a `breaks.json` entry naming the scenario and the cause. A
  difference with no allowed cause is a bug: fix it.
- [ ] **Step 4: If plan 5 runs WordPress's `tests/phpunit/tests/db.php`, `dbdelta.php` and `db/` against SQLite,
  run them against PostgreSQL the same way** — Expected: pass apart from tests whose failure is one of the allowed
  causes, listed the way plan 5 lists SQLite's.
- [ ] **Step 5: Run** the replay again — Expected: PASS.
- [ ] **Step 6: Commit** — `git commit -am "test(wp-db): cross-engine replay on PostgreSQL with its breaks"`

### Task 11: The options suite on PostgreSQL (only if plan 4 is merged)

**Files:**
- Modify: plan 4's options repository test, adding PostgreSQL

**Interfaces:**
- Consumes: plan 4's `SqlOptionRepository` and its repository and recorded-compatibility suites; Task 8.

- [ ] **Step 1: Check** that plan 4 is on the base branch. If not, mark this task blocked in the plan and stop; do
  not stub it.
- [ ] **Step 2: Add** a `#[ignore]`d PostgreSQL case to the options suites, building `SqlOptionRepository` over
  `PostgresStore` (schema `t_options`), and run them — Expected: PASS, or failures fixed in this crate or listed with
  an allowed cause from Task 10 in `compat/options/breaks.json`.
- [ ] **Step 3: Commit** — `git commit -am "test(wp-options): the options suites on PostgreSQL"`

### Task 12: Hand-off note for the runtime's owner

**Files:**
- Create: a note in the shared notes folder named `postgres-engine-runtime-handoff.md`

- [ ] **Step 1: Write the note:** the worker's `EngineRegistry` must `register_factory(Rc::new(PostgresEngineFactory))`;
  a `postgres://` URL configures `default` or any other engine; the server must be PostgreSQL 16+ with ICU; the
  database user needs `CREATE` on the schema the first time (or an operator runs
  `speedyspec_data_postgres::SUPPORT_SQL` beforehand); `reset_all()` at request end is required, because it also
  refreshes the schema cache. Nothing in `worker.rs` is changed by this plan.
- [ ] **Step 2: Commit** nothing in the repository for this task; report the note's path.
