# SQLite engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** WordPress's tables on SQLite: the neutral MySQL-to-portable rewrite passes (shared with PostgreSQL), the
SQLite dialect and store with MySQL's results semantics, the SQL corpus that proves every statement WordPress and
popular plugins send translates, and the `wp-db` compatibility catalogue replayed on SQLite.

**Architecture:** A new crate `speedyspec-data-translate` holds everything both non-MySQL engines need: the neutral
passes, placeholder numbering, MySQL type parsing, a MySQL schema catalogue kept in three `_speedyspec_*` tables
(Metadata Mapping), the session variables, and the answers to `EmulatedOp`. `speedyspec-data-query` and
`speedyspec-data-mysql` are not touched: MySQL keeps its empty chain. `speedyspec-data-sqlite` (rusqlite) adds the
SQLite passes, an emitter that plans each statement into steps with counting rules, the store, the introspector over
the catalogue, and MySQL's functions as connection functions. `speedyspec-data-corpus` ingests and checks the SQL
corpus.

**Tech Stack:** Rust 1.99, edition 2024; `rusqlite` (latest release, features `bundled`, `functions`, `collation`,
`column_metadata`), `regex`, `unicode-normalization`, `chrono`, `md-5`, `uuid` (`v4`), `serde` + `serde_json`
(corpus), `tempfile` (dev). Hand-written `Display`, as plan 1 does.

**Spec:** `docs/superpowers/specs/2026-11-19-database-layer-design.md` ("The statement pipeline", "Translating MySQL
for PostgreSQL and SQLite", "Testing", "Order of work" step 3). Plan index (repository, worktree, shared names):
`docs/superpowers/plans/2026-11-19-data-and-options-ports-index.md`. Consumes plan 1
(`2026-11-19-data-layer-foundation.md`) exactly as written.

## Global Constraints

- New worktree of `SpeedySpec/speedyspec-server`, branch `feat/sqlite-engine`, from `origin/main` once plans 1 and 2
  are merged. New crates under `crates/`; no workspace `Cargo.toml` edit. Never touch
  `packages/speedyspec-server-runtime-gate/` or `packages/speedyspec-server/`. Do not edit `crates/speedyspec-php-runtime`
  (Task 12 hands the wiring off).
- Each crate's `Cargo.toml`: `edition.workspace = true`, `rust-version.workspace = true`, `license.workspace = true`,
  version `0.1.0`, a one-line `description`.
- Do not edit `speedyspec-data`, `speedyspec-data-query`, `speedyspec-data-mysql` or `speedyspec-compat`. A gap found
  in them is reported in the PR, not patched here.
- No WordPress or PHP knowledge in `speedyspec-data-translate`, `speedyspec-data-sqlite` or `speedyspec-data-corpus`
  (MySQL knowledge is their subject). WordPress-specific collection scripts live in `tools/sql-corpus/`.
- Every statement reaches SQLite only as a parsed `Statement` run through the chain. Text the dialect builds itself
  (the `ON UPDATE` trigger, the statement transaction) is emitted output, never input SQL.
- A construct with no faithful SQLite form fails with `DbError::Unsupported { construct, dialect: DialectKind::Sqlite }`
  (`dialect: Postgres`/`Sqlite` from the caller's `PassContext.target` in the neutral crate).
- The bundled SQLite must be 3.45 or later (target-less `ON CONFLICT DO UPDATE` 3.35, `RETURNING` 3.35, `DROP COLUMN`
  3.35, `UPDATE ... FROM` 3.33, `IS [NOT] DISTINCT FROM` 3.39, aggregate `ORDER BY` 3.44).
- The MySQL this port imitates is 8.4 (the recorded fixture's server): its `SHOW` output, type display (no integer
  display width except `tinyint(1)`) and default `sql_mode`.
- Patterns: Visitor and Chain of Responsibility (every pass, `PassChain`), Strategy (`SqliteDialect`), Abstract
  Factory (`SqliteEngine`), Adapter (`SqliteStore` over rusqlite), Metadata Mapping (the MySQL schema catalogue),
  Interpreter (the corpus normaliser reuses the parser). No others.
- After any Linux container test run, delete that run's `target-linux*/` folder or Docker target volume.

## Review Focus

- **An `UPDATE` that changes only letter case** (`'Hello'` to `'hello'`) on a case-insensitive column: it writes and
  counts 1, as MySQL does; the changed-rows predicate compares `COLLATE BINARY`. Test in Tasks 7 and 8.
- **`LIKE 'wp\_%'` from `esc_like`**: `_` is literal because MySQL's default `ESCAPE '\'` is added. Tests in Tasks 1
  and 8.
- **Two worker processes writing one SQLite file:** the second waits up to `busy_timeout` instead of failing; writes
  inside transactions take the lock up front (`BEGIN IMMEDIATE`). Test in Task 8.
- **`dbDelta()` run again on an existing table** finds nothing to change: `DESCRIBE` and `SHOW INDEX` give back MySQL
  8.4's types, defaults and prefix lengths exactly. Test in Task 8.
- **Params after a rewrite moves or repeats expressions** (multi-table `UPDATE`, the changed-rows predicate, a
  multi-row upsert split): each value binds where MySQL would have used it, through numbered placeholders. Tests in
  Tasks 2 and 7.

## Assumed from plan 2

- `compat/wp-db/` (`scenarios.json`, `source.json`, `wordpress-<version>.json`, `breaks.json`) exists, recorded on
  MySQL, in `speedyspec-compat`'s format; scenarios create their own tables through `$wpdb`, so they run on an empty
  database.
- `speedyspec-wp-db-php` has a test-support `ScenarioRunner` for `wp-db`. This plan calls it as
  `WpDbScenarioRunner::for_engine(config: EngineConfig) -> Result<WpDbScenarioRunner, String>`; if plan 2's runner is
  built only from the MySQL URL, Task 10 Step 1 adds this constructor beside it, registering `SqliteEngineFactory`.
- `wpdb` turns `DbError::Unsupported` and `DbError::Parse` into `last_error`, and reads `db_version()` /
  `db_server_info()` from `Introspector::server_version`.
- A harness runs WordPress PHPUnit files against the native class with a chosen engine; this plan calls it as
  `SPEEDYSPEC_DB_URL=<url> scripts/wp-phpunit.sh <test path>... --log-junit <file>`.

From plan 1 beyond the index: `Rewrite::Split` statements carry their own params; `sqlparser` parses the MySQL
constructs below (a construct it rejects is a parser gap, handled by plan 1 Task 3's rule and listed in the PR).

---

### Task 1: `speedyspec-data-translate`: placeholder numbering and the lexical, hint and expression passes

**Files:**
- Create: `crates/speedyspec-data-translate/Cargo.toml`, `src/lib.rs`, `src/params.rs`, `src/lexical.rs`,
  `src/hints.rs`, `src/expressions.rs`
- Test: `crates/speedyspec-data-translate/tests/support/mod.rs`, `tests/lexical.rs`

**Interfaces:**
- Consumes: plan 1's `Statement`, `DbError`, `DialectKind`, `RewritePass`, `Rewrite`, `PassContext`, `PassChain`,
  `Rewritten`, `parse_one`.
- Produces:
  - `pub fn number_placeholders(stmt: &mut Statement)`: every `Value::Placeholder("?")` becomes `"?1"`, `"?2"`, … in
    `VisitorMut` order (which is MySQL text order for every statement kind in this plan's tests); already numbered
    placeholders are left alone. After this, passes may move and repeat expressions freely; params never change.
  - `pub struct LexicalPass` (`name() == "lexical"`): backtick identifiers become `"`-quoted; `"`-strings become
    single-quoted; `LIMIT o, c` becomes `LIMIT c OFFSET o`; `<=>` becomes `IS NOT DISTINCT FROM`; `||` parsed as
    string concatenation from MySQL text becomes `OR`; `LIKE`/`NOT LIKE` without `ESCAPE` gets `ESCAPE '\'`;
    `DISTINCTROW` becomes `DISTINCT`.
  - `pub struct HintsPass` (`"hints"`): drops index hints (`USE|FORCE|IGNORE INDEX`), `HIGH_PRIORITY`,
    `LOW_PRIORITY`, `DELAYED`, `QUICK`, `SQL_NO_CACHE`, `SQL_CACHE`, `SQL_SMALL_RESULT`, `SQL_BIG_RESULT`,
    `SQL_BUFFER_RESULT`, and `STRAIGHT_JOIN` (as `JOIN`). Keeps `SQL_CALC_FOUND_ROWS` (Task 2 owns it).
  - `pub struct ExpressionPass` (`"expressions"`): `CAST(x AS SIGNED|UNSIGNED)` to `CAST(x AS BIGINT)`, `AS CHAR[(n)]`
    to `AS TEXT`; `DATE_ADD|ADDDATE(d, INTERVAL n U)` and `d + INTERVAL n U` to `speedyspec_date_add(d, n, 'U')`,
    `DATE_SUB|SUBDATE` and `d - INTERVAL n U` to `speedyspec_date_sub(d, n, 'U')` (compound units such as
    `HOUR_MINUTE` keep their string amount); `@@[SESSION.|GLOBAL.|LOCAL.]name` to `speedyspec_sysvar('name')`, aliased
    with the original text (`@@SESSION.sql_mode`) when it is an unaliased projection.
  - `pub fn neutral_chain() -> PassChain` (filled in by Tasks 2 and 3; order `admin`, `ddl`, `lexical`, `hints`,
    `expressions`, `insert`).
- Test support (`tests/support/mod.rs`, shared by Tasks 1–4): `rewrite_one(pass: impl RewritePass + 'static, sql:
  &str, cx: &PassContext) -> Result<Rewritten, DbError>` (parses with `parse_one`, runs a one-pass chain) and
  `assert_sql(stmt: &Statement, expected: &str)` (compares `stmt.tree().to_string()` with `expected` parsed by
  `sqlparser`'s `GenericDialect` and printed again, so spacing and keyword case are `sqlparser`'s).

Translation pairs (input is MySQL text; output after the named pass alone):

```text
lexical   IN  SELECT `option_value` FROM `wp_options` WHERE `option_name` = "siteurl" LIMIT 1
          OUT SELECT "option_value" FROM "wp_options" WHERE "option_name" = 'siteurl' LIMIT 1
lexical   IN  SELECT ID FROM wp_posts ORDER BY post_date DESC LIMIT 20, 10
          OUT SELECT ID FROM wp_posts ORDER BY post_date DESC LIMIT 10 OFFSET 20
lexical   IN  SELECT * FROM t WHERE a <=> NULL OR b = 1 || c = 2
          OUT SELECT * FROM t WHERE a IS NOT DISTINCT FROM NULL OR b = 1 OR c = 2
lexical   IN  SELECT option_name FROM wp_options WHERE option_name LIKE '\\_transient\\_%'      (Rust: r"...")
          OUT SELECT option_name FROM wp_options WHERE option_name LIKE '\_transient\_%' ESCAPE '\'
lexical   IN  SELECT a FROM t WHERE a LIKE 'x|_%' ESCAPE '|'          OUT unchanged
lexical   IN  SELECT 'it\'s', "a\"b"                                    OUT SELECT 'it''s', 'a"b'
hints     IN  SELECT HIGH_PRIORITY SQL_NO_CACHE ID FROM wp_posts FORCE INDEX (type_status_date) WHERE post_type = 'post'
          OUT SELECT ID FROM wp_posts WHERE post_type = 'post'
exprs     IN  SELECT CAST(meta_value AS SIGNED), CAST(meta_value AS UNSIGNED), CAST(ID AS CHAR) FROM wp_postmeta
          OUT SELECT CAST(meta_value AS BIGINT), CAST(meta_value AS BIGINT), CAST(ID AS TEXT) FROM wp_postmeta
exprs     IN  SELECT * FROM wp_posts WHERE post_date > DATE_SUB(NOW(), INTERVAL 7 DAY) AND post_modified < post_date + INTERVAL ? MONTH
          OUT SELECT * FROM wp_posts WHERE post_date > speedyspec_date_sub(NOW(), 7, 'DAY') AND post_modified < speedyspec_date_add(post_date, ?, 'MONTH')
exprs     IN  SELECT DATE_ADD(d, INTERVAL '1:30' HOUR_MINUTE) FROM t    OUT SELECT speedyspec_date_add(d, '1:30', 'HOUR_MINUTE') FROM t
exprs     IN  SELECT @@SESSION.sql_mode                                 OUT SELECT speedyspec_sysvar('sql_mode') AS "@@SESSION.sql_mode"
```

- [ ] **Step 1: Write the failing tests** in `tests/lexical.rs`: one test per pair above (`lexical_backticks_and_strings`,
  `lexical_limit_comma`, `lexical_null_safe_and_pipes`, `lexical_like_gets_mysql_escape`, `lexical_explicit_escape_kept`,
  `lexical_string_escapes`, `hints_dropped`, `expressions_casts`, `expressions_intervals`, `expressions_compound_interval`,
  `expressions_sysvar_alias`), each `assert_sql(&statement_of(rewrite_one(..)?), OUT)`; plus
  `number_placeholders_in_text_order` (for `UPDATE wp_posts p JOIN wp_postmeta m ON m.meta_key = ? SET p.post_status = ? WHERE m.meta_value = ? LIMIT ?`,
  `INSERT INTO t (a, b) VALUES (?, ?), (?, ?) ON DUPLICATE KEY UPDATE b = ?` and
  `SELECT * FROM t WHERE a IN (SELECT b FROM u WHERE c = ?) AND d = ? LIMIT ?`, the numbered text reads `?1`, `?2`,
  … left to right) and `number_placeholders_is_idempotent`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-translate` — Expected: FAIL (crate missing).
- [ ] **Step 3: Implement** the three passes and `number_placeholders` with `sqlparser`'s `VisitorMut`
  (`visit_expressions_mut`, `visit_relations_mut`), and `neutral_chain` with these three for now. If numbering order
  differs from text order for a tested statement, number by walking that statement's fields in text order instead.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-translate` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data-translate && git commit -m "feat(data-translate): placeholder numbering and the lexical, hint and expression passes"`

### Task 2: Neutral statement passes: inserts, administrative statements, found rows, row limits

**Files:**
- Create: `crates/speedyspec-data-translate/src/insert.rs`, `src/admin.rs`, `src/found_rows.rs`, `src/row_limit.rs`;
  modify `src/lib.rs`
- Test: `crates/speedyspec-data-translate/tests/statements.rs`

**Interfaces:**
- Consumes: Task 1.
- Produces:
  - `pub struct InsertPass` (`"insert"`): `INSERT ... SET a = 1, b = 2` to `(a, b) VALUES (1, 2)`; `ON DUPLICATE KEY
    UPDATE` to a target-less `ON CONFLICT DO UPDATE SET`, with `VALUES(col)` and the MySQL 8 row alias
    (`VALUES (..) AS new ... new.col`) both becoming `excluded.col`; `INSERT IGNORE` to `ON CONFLICT DO NOTHING`.
  - `pub struct AdminPass` (`"admin"`), returning `Rewrite::Emulate`: `SHOW TABLES [LIKE p]` →
    `ShowTables { like }`; `SHOW [FULL] COLUMNS|FIELDS FROM t [LIKE p]` and `DESCRIBE|DESC|EXPLAIN t` →
    `ShowColumns { table, full, like }` (`full: false` for `DESCRIBE`); `SHOW INDEX|INDEXES|KEYS FROM t` →
    `ShowIndex`; `SHOW CREATE TABLE t` → `ShowCreateTable`; `SHOW [SESSION|GLOBAL] VARIABLES [LIKE p]` →
    `ShowVariables`; `SHOW TABLE STATUS [LIKE p]` → `ShowTableStatus`; `SET NAMES c [COLLATE k]` →
    `SetSession { name: "names", value: "c" | "c k" }`; `SET [SESSION|@@SESSION.|@@]name = literal` →
    `SetSession { name: lowercased name, value: literal text }`; `SELECT FOUND_ROWS()` (alone, aliased or not) →
    `FoundRows`; `LOCK TABLES`, `UNLOCK TABLES`, `OPTIMIZE|ANALYZE|CHECK|REPAIR TABLE`, `SET autocommit = 1` → `Noop`.
    Errors: `SET @v = ..` → `Unsupported { construct: "user variable" }`; `SET autocommit = 0` →
    `Unsupported { construct: "SET autocommit = 0" }`; `FOUND_ROWS()` anywhere else →
    `Unsupported { construct: "FOUND_ROWS() in an expression" }`. LIKE patterns are passed as the unescaped literal.
  - `pub fn take_found_rows(stmt: &mut Statement) -> Option<Statement>`: when the top-level `SELECT` carries
    `SQL_CALC_FOUND_ROWS`, removes the modifier and returns `SELECT COUNT(*) FROM (<the query without its top-level
    ORDER BY, LIMIT and OFFSET>) AS speedyspec_found_rows` with the same params (numbered placeholders make unused
    ones harmless); otherwise `None`.
  - `pub struct RowLimitPass` with `RowLimitPass::new(row_id: &'static str)` (`"row_limit"`): a single-table
    `UPDATE` or `DELETE` with `ORDER BY` and/or `LIMIT` becomes `... WHERE <row_id> IN (SELECT <row_id> FROM t WHERE
    <where> ORDER BY .. LIMIT n)`. Not in `neutral_chain` (SQLite pushes it with `"rowid"`, PostgreSQL with `"ctid"`).
  - `neutral_chain()` now `admin`, `lexical`, `hints`, `expressions`, `insert` (Task 3 inserts `ddl` second).

```text
insert IN  INSERT INTO wp_options (option_name, option_value, autoload) VALUES ('a', 'b', 'auto') ON DUPLICATE KEY UPDATE option_name = VALUES(option_name), option_value = VALUES(option_value), autoload = VALUES(autoload)
       OUT INSERT INTO wp_options (option_name, option_value, autoload) VALUES ('a', 'b', 'auto') ON CONFLICT DO UPDATE SET option_name = excluded.option_name, option_value = excluded.option_value, autoload = excluded.autoload
insert IN  INSERT INTO t (a, b) VALUES (1, 2) AS new ON DUPLICATE KEY UPDATE b = new.b + t.b
       OUT INSERT INTO t (a, b) VALUES (1, 2) ON CONFLICT DO UPDATE SET b = excluded.b + t.b
insert IN  INSERT IGNORE INTO wp_term_relationships (object_id, term_taxonomy_id) VALUES (1, 2)
       OUT INSERT INTO wp_term_relationships (object_id, term_taxonomy_id) VALUES (1, 2) ON CONFLICT DO NOTHING
insert IN  INSERT INTO wp_postmeta SET post_id = 1, meta_key = 'k', meta_value = 'v'
       OUT INSERT INTO wp_postmeta (post_id, meta_key, meta_value) VALUES (1, 'k', 'v')
rowlim IN  DELETE FROM wp_options WHERE option_name LIKE 'x%' ORDER BY option_id LIMIT 100
       OUT DELETE FROM wp_options WHERE rowid IN (SELECT rowid FROM wp_options WHERE option_name LIKE 'x%' ORDER BY option_id LIMIT 100)
rowlim IN  UPDATE wp_posts SET post_status = 'draft' WHERE post_author = 1 LIMIT 5
       OUT UPDATE wp_posts SET post_status = 'draft' WHERE rowid IN (SELECT rowid FROM wp_posts WHERE post_author = 1 LIMIT 5)
found  IN  SELECT SQL_CALC_FOUND_ROWS wp_posts.ID FROM wp_posts WHERE 1=1 AND post_type = ?1 ORDER BY wp_posts.post_date DESC LIMIT ?2, 10
       OUT stmt  SELECT wp_posts.ID FROM wp_posts WHERE 1=1 AND post_type = ?1 ORDER BY wp_posts.post_date DESC LIMIT ?2, 10
           count SELECT COUNT(*) FROM (SELECT wp_posts.ID FROM wp_posts WHERE 1=1 AND post_type = ?1) AS speedyspec_found_rows
```

- [ ] **Step 1: Write the failing tests** in `tests/statements.rs`: one per pair above; `found_rows_absent_is_none`;
  `admin_emulations` (each statement in the `AdminPass` list gives the exact `EmulatedOp`, e.g.
  ``SHOW TABLES LIKE 'wp\\_%'`` → `ShowTables { like: Some(r"wp\_%".into()) }`,
  ``SHOW FULL COLUMNS FROM `wp_posts` `` → `ShowColumns { table: "wp_posts", full: true, like: None }`,
  `DESCRIBE wp_posts` → `full: false`, `SET NAMES 'utf8mb4' COLLATE 'utf8mb4_unicode_520_ci'` →
  `SetSession { name: "names", value: "utf8mb4 utf8mb4_unicode_520_ci" }`,
  `SET @@SESSION.sql_mode = 'NO_ENGINE_SUBSTITUTION'` → `SetSession { name: "sql_mode", value:
  "NO_ENGINE_SUBSTITUTION" }`, `UNLOCK TABLES` → `Noop`, `SELECT FOUND_ROWS() AS n` → `FoundRows`);
  `admin_refusals` (`SET @a = 1`, `SET autocommit = 0`, `SELECT FOUND_ROWS() + 1` give the three `Unsupported`
  constructs with `dialect == cx.target`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-translate --test statements` — Expected: FAIL.
- [ ] **Step 3: Implement** the passes and `take_found_rows`. `SHOW TABLE STATUS` and `DESCRIBE`'s forms map from
  whatever `ast` variants the pinned `sqlparser` produces for them.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-translate` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-translate): neutral insert, administrative, found-rows and row-limit passes"` (add new files first).

### Task 3: MySQL types, the schema catalogue and the `dbDelta` DDL pass

**Files:**
- Create: `crates/speedyspec-data-translate/src/mysql_type.rs`, `src/catalogue.rs`, `src/ddl.rs`,
  `src/testing.rs` (behind feature `testing`); modify `src/lib.rs`, `Cargo.toml` (`[features] testing = []`)
- Test: `crates/speedyspec-data-translate/tests/ddl.rs`

**Interfaces:**
- Consumes: Tasks 1–2; plan 1's `Introspector`, `ColumnDef`, `IndexDef`, `Query`.
- Produces:
  - `pub struct MySqlType { pub base: String, pub args: Vec<String>, pub unsigned: bool, pub zerofill: bool }` with
    `MySqlType::from_ast(dt: &ast::DataType) -> Result<MySqlType, DbError>`, `MySqlType::parse(text: &str) ->
    Result<MySqlType, DbError>` (through `sqlparser`'s data-type parser), `display(&self) -> String` (MySQL 8.4's
    `SHOW COLUMNS` form: lowercase, `integer` → `int`, integer display widths dropped except `tinyint(1)`, ` unsigned`
    / ` zerofill` suffixes), `family(&self) -> TypeFamily`, `implicit_default(&self) -> &'static str` (MySQL's
    non-strict implicit default: `""` text, `"0"` numbers, `"0000-00-00 00:00:00"` datetime/timestamp,
    `"0000-00-00"` date, `"00:00:00"` time, `"0000"` year; enums use their first value through `args[0]`).
    `pub enum TypeFamily { Integer, Decimal { scale: u32 }, Float, Text, Enum, Binary, Temporal(Temporal) }`,
    `pub enum Temporal { Date, DateTime, Timestamp, Time, Year }`.
  - The catalogue: tables `_speedyspec_tables (table_name, collation)`, `_speedyspec_columns (table_name, ordinal,
    column_name, column_type, nullable, column_default, extra, collation)`, `_speedyspec_indexes (table_name,
    index_name, kind, seq, column_name, sub_part)`, keys `(table_name)`, `(table_name, column_name)`,
    `(table_name, index_name, seq)`. `pub fn catalogue_ddl() -> Vec<Statement>` (MySQL-form `CREATE TABLE IF NOT
    EXISTS`, translated like any DDL); `pub fn is_catalogue_table(name: &str) -> bool` (prefix `_speedyspec_`);
    `pub fn index_object_name(table: &str, index: &str) -> String` (`"<table>__<index>"`: SQLite and PostgreSQL index
    names are schema-wide, MySQL's are per table).
  - `pub struct CatalogueColumn { pub name: String, pub column_type: MySqlType, pub nullable: bool, pub default:
    Option<String>, pub extra: String, pub collation: Option<String> }`; `pub enum IndexKind { Primary, Unique, Index,
    Fulltext }`; `pub struct CatalogueIndex { pub name: String, pub kind: IndexKind, pub parts: Vec<(String,
    Option<u32>)> }`; `pub struct CatalogueTable { pub name: String, pub collation: Option<String>, pub columns:
    Vec<CatalogueColumn>, pub indexes: Vec<CatalogueIndex> }` with `column_defs(&self) -> Vec<ColumnDef>` (`sql_type`
    = `display()`, `key` from `column_key`), `index_defs(&self) -> Vec<IndexDef>` (primary named `PRIMARY`),
    `to_create_table(&self) -> Statement` (MySQL form, used by SQLite's table rebuild).
  - `pub fn catalogue_table(create: &ast::Statement) -> Result<CatalogueTable, DbError>` (from a MySQL
    `CREATE TABLE`: text columns inherit the table's `COLLATE`, or its `CHARSET`'s default `utf8mb4_0900_ai_ci`;
    unnamed keys take MySQL's name, the first column, then `_2`, `_3` on a clash; column-level `UNIQUE` /
    `PRIMARY KEY` become indexes; `extra` is `auto_increment`, `DEFAULT_GENERATED`, `DEFAULT_GENERATED on update
    CURRENT_TIMESTAMP` as MySQL 8.4 shows them).
  - `pub fn column_key(column: &str, indexes: &[CatalogueIndex]) -> &'static str` (MySQL's rule: `PRI` if in the
    primary key; `UNI` if the first column of a single-column unique index; `MUL` if the first column of any other
    index; else `""`; highest of `PRI`, `UNI`, `MUL` wins).
  - `pub enum SchemaChange { CreateTable(CatalogueTable), DropTable(String), RenameTable { from: String, to: String },
    AddColumn { table: String, column: CatalogueColumn, position: ColumnPosition }, ChangeColumn { table: String,
    old_name: String, column: CatalogueColumn, position: ColumnPosition }, DropColumn { table: String, column: String },
    AddIndex { table: String, index: CatalogueIndex }, DropIndex { table: String, name: String }, SetCollation { table:
    String, collation: String } }`, `pub enum ColumnPosition { Last, First, After(String) }`, and
    `pub fn catalogue_writes(change: &SchemaChange) -> Vec<Statement>` (literals, no params; `CreateTable` deletes the
    table's old rows first; ordinals renumbered for `First`/`After`).
  - `pub fn read_catalogue(run: &dyn Fn(&Statement) -> Result<Rows, DbError>, table: &str) ->
    Result<Option<CatalogueTable>, DbError>` and `pub fn catalogue_tables(run: ..) -> Result<Vec<String>, DbError>`.
  - `pub struct DdlPass` (`"ddl"`, second in `neutral_chain`), returning `Split` of the statements below plus
    `catalogue_writes`; catalogue tables themselves pass through untouched:
    - `CREATE TABLE`: table options (`ENGINE`, `[DEFAULT] CHARSET|CHARACTER SET`, `COLLATE`, `ROW_FORMAT`, `COMMENT`)
      dropped; `KEY`/`INDEX`/`UNIQUE KEY` lifted out as `CREATE [UNIQUE] INDEX <index_object_name> ON t (cols)`
      without prefix lengths; `FULLTEXT KEY` kept only in the catalogue; `PRIMARY KEY` stays in the table.
      `CREATE TABLE IF NOT EXISTS t` when `cx.introspector` lists `t` → `Emulate(Noop)`. `CREATE TABLE .. LIKE`,
      `CREATE TABLE .. SELECT`, and the `AUTO_INCREMENT=n` table option → `Unsupported` naming each.
    - `DROP TABLE [IF EXISTS] a, b` → one `DROP TABLE` per table.
    - `ALTER TABLE` with several actions → one statement per action: `ADD [UNIQUE] KEY|INDEX` → `CREATE INDEX`;
      `DROP INDEX|KEY k` → `DROP INDEX <index_object_name>`; `ADD FULLTEXT` → catalogue only (a no-op when the
      catalogue has it); `ENGINE=`, `[DEFAULT] CHARSET`, `CONVERT TO CHARACTER SET` → catalogue only; column actions
      and `ADD|DROP PRIMARY KEY`, `RENAME [TO]` stay as single-action `ALTER TABLE` for the dialect (`AFTER`/`FIRST`
      removed from the SQL, kept in the catalogue).
    - `CREATE [UNIQUE] INDEX k ON t (c(n))` → `CREATE [UNIQUE] INDEX t__k ON t (c)`; `DROP INDEX k ON t` →
      `DROP INDEX t__k`; `RENAME TABLE a TO b` → `ALTER TABLE a RENAME TO b`.
  - `testing::StaticIntrospector` (`from_mysql_ddl(ddl: &[&str]) -> StaticIntrospector`, `impl Introspector` over
    `catalogue_table` of each; `server_version` `"8.4.0"`), used by Tasks 4, 6, 7.

```text
ddl IN  CREATE TABLE wp_options ( option_id bigint(20) unsigned NOT NULL auto_increment, option_name varchar(191) NOT NULL default '', option_value longtext NOT NULL, autoload varchar(20) NOT NULL default 'yes', PRIMARY KEY  (option_id), UNIQUE KEY option_name (option_name), KEY autoload (autoload) ) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_520_ci
    OUT [0] CREATE TABLE wp_options (option_id bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT, option_name varchar(191) NOT NULL DEFAULT '', option_value longtext NOT NULL, autoload varchar(20) NOT NULL DEFAULT 'yes', PRIMARY KEY (option_id))
        [1] CREATE UNIQUE INDEX wp_options__option_name ON wp_options (option_name)
        [2] CREATE INDEX wp_options__autoload ON wp_options (autoload)
        [3..] catalogue writes
ddl IN  CREATE TABLE wp_postmeta ( meta_id bigint(20) unsigned NOT NULL auto_increment, post_id bigint(20) unsigned NOT NULL default '0', meta_key varchar(255) default NULL, meta_value longtext, PRIMARY KEY  (meta_id), KEY post_id (post_id), KEY meta_key (meta_key(191)) )
    OUT [2] CREATE INDEX wp_postmeta__meta_key ON wp_postmeta (meta_key)        (catalogue sub_part 191)
ddl IN  ALTER TABLE wp_t ADD COLUMN c varchar(10) NOT NULL DEFAULT '' AFTER b, ADD KEY c (c), DROP INDEX old
    OUT [0] ALTER TABLE wp_t ADD COLUMN c varchar(10) NOT NULL DEFAULT ''   [1] CREATE INDEX wp_t__c ON wp_t (c)   [2] DROP INDEX wp_t__old   [3..] catalogue
ddl IN  CREATE INDEX k ON t (c(10))          OUT [0] CREATE INDEX t__k ON t (c)
ddl IN  DROP INDEX k ON t                    OUT [0] DROP INDEX t__k
ddl IN  DROP TABLE IF EXISTS a, b            OUT [0] DROP TABLE IF EXISTS a   [1] DROP TABLE IF EXISTS b   [2..] catalogue
```

- [ ] **Step 1: Write the failing tests** in `tests/ddl.rs`: the pairs above (`ddl_wp_options`, `ddl_prefix_index`,
  `ddl_alter_split`, `ddl_create_index`, `ddl_drop_index`, `ddl_drop_tables`); `mysql_type_display`
  (`bigint(20) unsigned` → `bigint unsigned`, `int(11)` → `int`, `INTEGER` → `int`, `tinyint(1)` → `tinyint(1)`,
  `varchar(191)`, `decimal(10,2)`, `enum('a','b')`, `longtext`, `double` unchanged; `implicit_default` of `bigint`,
  `varchar`, `datetime`, `enum('x','y')` is `"0"`, `""`, `"0000-00-00 00:00:00"`, `"x"`);
  `catalogue_rows_for_wp_options` (the writes insert column rows `('wp_options', 1, 'option_id', 'bigint unsigned', 0,
  NULL, 'auto_increment', NULL)` and `('wp_options', 2, 'option_name', 'varchar(191)', 0, '', '',
  'utf8mb4_unicode_520_ci')`, and index rows `PRIMARY/PRIMARY/option_id`, `option_name/UNIQUE/option_name`,
  `autoload/INDEX/autoload`); `column_key_rules` (single-column unique → `UNI`, first column of a two-column unique →
  `MUL`, in primary → `PRI`, second column of an index → `""`); `unnamed_keys_named_like_mysql` (`KEY (a), KEY (a, b)`
  → `a`, `a_2`); `fulltext_only_in_catalogue`; `ddl_refusals` (`CREATE TABLE a LIKE b`, `CREATE TABLE t (id int)
  AUTO_INCREMENT=100`); `create_if_not_exists_existing_is_noop` (with `StaticIntrospector` holding `t`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-translate --features testing --test ddl` — Expected: FAIL.
- [ ] **Step 3: Implement** the modules above and put `DdlPass` second in `neutral_chain`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-translate --features testing` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-translate): MySQL types, the schema catalogue and the dbDelta DDL pass"` (add new files first).

### Task 4: Session variables and MySQL-shaped answers to `EmulatedOp`

**Files:**
- Create: `crates/speedyspec-data-translate/src/session.rs`, `src/emulate.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-data-translate/tests/emulate.rs`

**Interfaces:**
- Consumes: Tasks 2–3; plan 1's `EmulatedOp`, `Rows`, `Row`, `ColumnInfo`, `Introspector`.
- Produces:
  - `pub struct SessionVars` (`Send`, `Clone`) with `SessionVars::mysql_defaults(database: &str, version: &str) ->
    SessionVars`, `database(&self) -> &str`, `get(&self, name: &str) -> Option<&str>` (case-insensitive), `set(&mut
    self, name: &str, value: &str)`, `set_names(&mut self, charset: &str, collation: Option<&str>)` (sets
    `character_set_client`, `character_set_connection`, `character_set_results`, `collation_connection`; the
    charset's default collation when `None`), `like(&self, pattern: Option<&str>) -> Vec<(String, String)>` (sorted,
    MySQL `LIKE` with `\` escape, case-insensitive), `reset(&mut self)`. Defaults: `version` as given,
    `version_comment` `"SpeedySpec"`, `sql_mode` `"ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"`,
    the three `character_set_*` `"utf8mb4"`, `collation_connection` `"utf8mb4_0900_ai_ci"`, `time_zone` `"+00:00"`,
    `max_allowed_packet` `"67108864"`, `lower_case_table_names` `"0"`, `autocommit` `"ON"`.
  - `pub fn answer_emulated(op: &EmulatedOp, introspector: &dyn Introspector, session: &mut SessionVars, found_rows:
    u64) -> Result<Rows, DbError>`, with MySQL 8.4's column names and order:
    - `ShowTables`: one column `Tables_in_<db>` (with ` (<pattern>)` appended when `like` is set), catalogue tables
      excluded, sorted, filtered by case-sensitive MySQL `LIKE`.
    - `ShowColumns`: `full` → `Field, Type, Collation, Null, Key, Default, Extra, Privileges, Comment` (Privileges
      `select,insert,update,references`, Comment `""`); else `Field, Type, Null, Key, Default, Extra`. `Null` is
      `YES`/`NO`; `Default` NULL when none.
    - `ShowIndex`: `Table, Non_unique, Key_name, Seq_in_index, Column_name, Collation, Cardinality, Sub_part, Packed,
      Null, Index_type, Comment, Index_comment, Visible, Expression` (Collation `A`, Cardinality `0`, Index_type
      `BTREE`, Visible `YES`).
    - `ShowCreateTable`: `Table, Create Table`, text from `render_create_table`.
    - `ShowVariables`: `Variable_name, Value` from `session.like`.
    - `ShowTableStatus`: MySQL's 18 columns; Engine `InnoDB`, Row_format `Dynamic`, Collation from
      `table_collation`, numbers `0`, times NULL.
    - `SetSession`: applies to `session` (`names` through `set_names`), empty `Rows`. `Noop`: empty `Rows`.
    - `FoundRows`: one column `FOUND_ROWS()`, one row `Int(found_rows)`.
    Text values are `DbValue::Text`, counts `DbValue::Int`.
  - `pub fn render_create_table(table: &str, columns: &[ColumnDef], indexes: &[IndexDef], collation: Option<&str>) ->
    String`: MySQL 8.4's layout (two-space indent, backticks, `PRIMARY KEY`, `UNIQUE KEY`, `KEY` with `(col(n))` prefix,
    closing `) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=<collation>`).

- [ ] **Step 1: Write the failing tests** in `tests/emulate.rs` over `StaticIntrospector::from_mysql_ddl` of the
  `wp_options` and `wp_postmeta` DDL from Task 3, database `wordpress`: `show_tables_like` (`like: Some(r"wp\_%")`
  → column `Tables_in_wordpress (wp\_%)`, rows `wp_options`, `wp_postmeta`); `show_full_columns_row`
  (`option_id` row is `["option_id", "bigint unsigned", NULL, "NO", "PRI", NULL, "auto_increment",
  "select,insert,update,references", ""]`); `describe_postmeta` (rows `meta_id|bigint unsigned|NO|PRI|NULL|auto_increment`,
  `post_id|bigint unsigned|NO|MUL|0|`, `meta_key|varchar(255)|YES|MUL|NULL|`, `meta_value|longtext|YES||NULL|`);
  `show_index_sub_part` (`meta_key` row: `Non_unique` 1, `Sub_part` 191, `Null` `YES`); `show_create_table_text`
  (exact text for `wp_options` with collation `utf8mb4_unicode_520_ci`); `set_names_and_sql_mode` (after
  `SetSession { names, "utf8mb4 utf8mb4_unicode_520_ci" }`, `ShowVariables { like: Some("collation_connection") }`
  gives that collation; `reset()` restores `utf8mb4_0900_ai_ci`); `found_rows_value` (`FoundRows` with 15 → one row
  `Int(15)`, column `FOUND_ROWS()`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-translate --features testing --test emulate` — Expected: FAIL.
- [ ] **Step 3: Implement** `session.rs` and `emulate.rs`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-translate --features testing` — Expected: PASS.
- [ ] **Step 5: Run** `cargo clippy -p speedyspec-data-translate --all-targets --features testing -- -D warnings` and `cargo fmt --check` — Expected: clean.
- [ ] **Step 6: Commit** — `git commit -am "feat(data-translate): session variables and MySQL-shaped emulated answers"` (add new files first).

### Task 5: `speedyspec-data-sqlite`: connection, collation and MySQL's functions

**Files:**
- Create: `crates/speedyspec-data-sqlite/Cargo.toml`, `src/lib.rs`, `src/connection.rs`, `src/collation.rs`,
  `src/functions/mod.rs`, `src/functions/dates.rs`, `src/functions/strings.rs`, `src/functions/misc.rs`
- Test: `crates/speedyspec-data-sqlite/tests/functions.rs`

**Interfaces:**
- Consumes: Task 4's `SessionVars`.
- Produces:
  - `pub fn open_connection(target: &SqliteTarget, busy_timeout: Duration, session: Arc<Mutex<SessionVars>>) ->
    Result<rusqlite::Connection, DbError>` with `pub enum SqliteTarget { Memory, File(PathBuf) }`: for a file,
    `PRAGMA journal_mode=WAL` and `synchronous=NORMAL`; always `busy_timeout`, `foreign_keys=OFF`,
    `ATTACH DATABASE ':memory:' AS information_schema` (Task 8 fills it), then `register(..)`.
  - `pub fn register(conn: &rusqlite::Connection, session: Arc<Mutex<SessionVars>>) -> rusqlite::Result<()>`
    (rusqlite functions must be `Send`, hence `Arc<Mutex<..>>`). Collation `speedyspec_ci`: compare after NFD, removing
    combining marks, Unicode lowercasing and trimming trailing spaces (MySQL's PAD SPACE). Functions, MySQL semantics,
    deterministic where MySQL's are:
    - Dates (UTC; zero dates `0000-00-00[ 00:00:00]` give `0` from parts and `NULL` from `date_format`):
      `now`, `sysdate`, `utc_timestamp`, `curdate`, `utc_date`, `curtime`, `unix_timestamp(/1)`,
      `from_unixtime(1/2)`, `date_format`, `speedyspec_date_add`, `speedyspec_date_sub` (month-end clamping; date in →
      date out for day-or-larger units, else datetime), `date` (override: keeps zero dates), `year`, `month`,
      `dayofmonth`, `day`, `hour`, `minute`, `second`, `dayofweek` (Sunday 1), `weekday` (Monday 0), `dayofyear`,
      `week(1/2)` (modes 0–7), `quarter`, `last_day`, `datediff`.
    - Strings: `concat` (override: NULL if any argument is NULL), `length` (override: bytes), `char_length`,
      `character_length`, `lower`, `upper`, `lcase`, `ucase` (overrides: Unicode), `left`, `right`, `substring`,
      `substr`, `mid` (MySQL positions: 0 gives `''`, negative counts from the end), `substring_index`, `locate(2/3)`,
      `field`, `md5`, `like(2/3)` (override: `speedyspec_ci` folding; case-sensitive when either operand is a BLOB),
      `regexp` (the `REGEXP` operator: `regex` crate, case-insensitive unless an operand is a BLOB).
    - Other: `if`, `isnull`, `greatest`, `least` (NULL if any NULL; numeric when all numeric), `rand(0/1)` (float in
      `[0, 1)`), `uuid`, `inet_aton`, `inet_ntoa`, `get_lock`/`release_lock`/`is_free_lock` (always `1`: one writer at
      a time already), `connection_id` (process id), `version` (`SessionVars` `version`), `database`,
      `speedyspec_sysvar(name)` (`SessionVars::get`, NULL when unknown).
    - SQLite's own, already MySQL's semantics, left as they are: `ifnull`, `coalesce`, `nullif`, `concat_ws`, `instr`,
      `replace`, `trim`, `abs`, `round`, `max`/`min` (aggregates), `count`, `sum`, `avg`.

- [ ] **Step 1: Write the failing tests** in `tests/functions.rs` on `Connection::open_in_memory()` with `register`,
  each asserting one `SELECT`'s single value:
  `sqlite_is_new_enough` (`sqlite_version()` ≥ `3.45.0`);
  strings: `concat('a', NULL)` → NULL, `concat('a','b')` → `'ab'`, `length('é')` → 2, `char_length('é')` → 1,
  `lower('ÉCOLE')` → `'école'`, `substring('hello', 0)` → `''`, `substring('hello', -3)` → `'llo'`,
  `substring('hello', 2, 3)` → `'ell'`, `substring_index('a.b.c', '.', 2)` → `'a.b'`, `substring_index('a.b.c', '.', -1)`
  → `'c'`, `field('b','a','b','c')` → 2, `field('z','a')` → 0, `md5('a')` → `'0cc175b9c0f1b6a831c399e269772661'`;
  like and collation: `'ÉCOLE' LIKE 'ecole'` → 1, `'a_c' LIKE 'a\_c' ESCAPE '\'` → 1, `'abc' LIKE 'a\_c' ESCAPE '\'`
  → 0, `CAST('ABC' AS BLOB) LIKE 'abc'` → 0, `'Hello' REGEXP '^h'` → 1, `'Hello' = 'hello' COLLATE speedyspec_ci` →
  1, `'a ' = 'a' COLLATE speedyspec_ci` → 1, and `SELECT column1 FROM (VALUES ('banana'), ('Apple'), ('cherry')) ORDER
  BY column1 COLLATE speedyspec_ci` → `Apple, banana, cherry`;
  dates: `speedyspec_date_add('2024-01-31', 1, 'MONTH')` → `'2024-02-29'`,
  `speedyspec_date_sub('2024-03-01 00:00:00', 1, 'SECOND')` → `'2024-02-29 23:59:59'`,
  `speedyspec_date_add('2024-01-01', '1:30', 'HOUR_MINUTE')` → `'2024-01-01 01:30:00'`,
  `date_format('2024-03-05 14:07:09', '%Y-%m-%d %H:%i:%s %W %M %e %c %j')` → `'2024-03-05 14:07:09 Tuesday March 5 3 065'`,
  `unix_timestamp('1970-01-02 00:00:00')` → 86400, `from_unixtime(86400)` → `'1970-01-02 00:00:00'`,
  `unix_timestamp('0000-00-00 00:00:00')` → 0, `year('0000-00-00 00:00:00')` → 0, `dayofweek('2024-03-05')` → 3,
  `weekday('2024-03-05')` → 1, `week('2024-03-05', 1)` → 10, `week('2024-01-01', 0)` → 0;
  other: `greatest(1, 5, 3)` → 5, `greatest(1, NULL)` → NULL, `least('b','a')` → `'a'`, `if(1 > 0, 'y', 'n')` → `'y'`,
  `isnull(NULL)` → 1, `inet_aton('10.0.0.1')` → 167772161, `rand()` in `[0, 1)`;
  session: `speedyspec_sysvar('sql_mode')` → the default, and after `session.lock().set("sql_mode", "X")` → `'X'`;
  `version()` and `database()` → the session's values.
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-sqlite --test functions` — Expected: FAIL.
- [ ] **Step 3: Implement** `connection.rs`, `collation.rs` and the function modules (`chrono` for dates,
  `unicode-normalization` for folding, `md-5`, `uuid`, `regex`, with a per-connection compiled-pattern cache).
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-sqlite --test functions` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data-sqlite && git commit -m "feat(data-sqlite): connection, MySQL collation and functions"`

### Task 6: SQLite DDL passes: column types, `AUTO_INCREMENT`, implicit defaults, `ALTER` and table rebuilds

**Files:**
- Create: `crates/speedyspec-data-sqlite/src/passes/mod.rs`, `src/passes/column_types.rs`, `src/passes/alter.rs`,
  `src/passes/quote.rs`, `src/plan.rs`, `src/dialect.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-data-sqlite/tests/ddl.rs`, `tests/support/mod.rs`

**Interfaces:**
- Consumes: Tasks 1–4 (`neutral_chain`, `number_placeholders`, `take_found_rows`, `RowLimitPass`, `MySqlType`,
  `catalogue_*`, `StaticIntrospector`).
- Produces:
  - `pub struct SqliteStep { pub sql: EmittedSql, pub count: CountRule, pub insert_id: InsertIdRule, pub returns_rows:
    bool }`; `pub enum CountRule { Changes, Upsert, None }`; `pub enum InsertIdRule { None, LastRowid, Returning }`;
    `pub enum SqlitePlan { Run { steps: Vec<SqliteStep>, found_rows: Option<EmittedSql>, information_schema: bool },
    Emulated(EmulatedOp) }`.
  - `pub struct SqliteDialect` with `new() -> SqliteDialect`, `plan(&self, stmt: &Statement, introspector:
    Option<&dyn Introspector>) -> Result<SqlitePlan, DbError>`, and `impl Dialect` (`kind() == Sqlite`; `prepare` is
    `plan(stmt, None)` flattened to `Prepared::Sql` (steps, then the found-rows count) or `Prepared::Emulated`).
    `plan`: clone, `number_placeholders`, `take_found_rows`, then each statement through the chain: `neutral_chain()`,
    then `RowLimitPass::new("rowid")`, `sqlite_alter`, `sqlite_types`, `sqlite_dml` (Task 7),
    `sqlite_numeric_compare` (Task 7), `sqlite_functions` (Task 7), `sqlite_quote`; then emission.
  - Pass `sqlite_types` (CREATE TABLE and ADD COLUMN): integer types, `bit`, `bool`, `year` → `INTEGER`;
    `decimal|numeric|float|double|real` → `REAL`; char/text/enum/set/json → `TEXT COLLATE speedyspec_ci` (`COLLATE
    BINARY` when the MySQL collation ends `_bin` or the charset is `binary`); binary/blob types → `BLOB`; temporal →
    `TEXT`. The single-column primary key with `AUTO_INCREMENT` becomes `"<col>" INTEGER PRIMARY KEY AUTOINCREMENT
    NOT NULL` and leaves the table constraints; `AUTO_INCREMENT` elsewhere → `Unsupported { construct: "AUTO_INCREMENT
    outside a single-column primary key" }`. A `NOT NULL` column without `DEFAULT` gets `DEFAULT
    <implicit_default>` (WordPress runs MySQL without strict mode, where an omitted column takes it). `CHARACTER SET`,
    `COMMENT` dropped; `DEFAULT CURRENT_TIMESTAMP` kept; `ON UPDATE CURRENT_TIMESTAMP` left for emission.
  - Pass `sqlite_alter` (uses `cx.introspector`; without one, `Unsupported { construct: "ALTER TABLE without schema
    information" }`): `ADD COLUMN` native; `CHANGE`/`MODIFY` whose SQLite column (affinity, `NOT NULL`, default,
    collation) is unchanged → catalogue only; otherwise, and for `ADD|DROP PRIMARY KEY` and `DROP COLUMN` of an
    indexed column, a rebuild: `CREATE TABLE "<t>__speedyspec_rebuild"` (from the altered `CatalogueTable::
    to_create_table`), `INSERT INTO .. (<kept cols>) SELECT <kept cols> FROM "<t>"`, `DROP TABLE "<t>"`, `ALTER TABLE
    .. RENAME TO "<t>"`, every non-fulltext index re-created; `DROP COLUMN` of an unindexed column and `RENAME COLUMN`
    native; `RENAME TO` native plus each index dropped and re-created under the new `index_object_name`;
    `TRUNCATE TABLE t` → `DELETE FROM "t"` and `DELETE FROM sqlite_sequence WHERE name = 't'` (MySQL resets the
    counter).
  - Pass `sqlite_quote` (last): every table, column, index and alias identifier `"`-quoted; not function names,
    collation names, types, `rowid`, `excluded`, `temp`, `sqlite_sequence`.
  - Emission (`plan.rs`): `tree().to_string()`; a column with `ON UPDATE CURRENT_TIMESTAMP` is emitted without it,
    followed by the step `CREATE TRIGGER "<t>__<c>__on_update" AFTER UPDATE ON "<t>" FOR EACH ROW WHEN NEW."<c>" IS
    OLD."<c>" BEGIN UPDATE "<t>" SET "<c>" = CURRENT_TIMESTAMP WHERE rowid = NEW.rowid; END` (text built here:
    `sqlparser` has no SQLite trigger body). Rules: when the original statement is `Ddl`, every step counts `None`;
    otherwise Task 7's rules.
- Test support (`tests/support/mod.rs`): `plan_steps(sql: &str, ddl: &[&str]) -> Vec<String>` (plans with a
  `StaticIntrospector` from `ddl`, returns each step's SQL) and `assert_sqlite_sql(actual: &str, expected: &str)`
  (both parsed with `sqlparser`'s `SQLiteDialect` and printed again).

```text
IN  wp_options CREATE TABLE from Task 3
OUT [0] CREATE TABLE "wp_options" ("option_id" INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, "option_name" TEXT NOT NULL DEFAULT '' COLLATE speedyspec_ci, "option_value" TEXT NOT NULL DEFAULT '' COLLATE speedyspec_ci, "autoload" TEXT NOT NULL DEFAULT 'yes' COLLATE speedyspec_ci)
    [1] CREATE UNIQUE INDEX "wp_options__option_name" ON "wp_options" ("option_name")
    [2] CREATE INDEX "wp_options__autoload" ON "wp_options" ("autoload")
IN  CREATE TABLE t (id int NOT NULL, a tinyint(1) NOT NULL DEFAULT 0, b decimal(10,2), c double, d datetime NOT NULL DEFAULT '0000-00-00 00:00:00', e varbinary(16), f enum('x','y') NOT NULL, g varchar(10) COLLATE utf8mb4_bin, h timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP, PRIMARY KEY (id))
OUT [0] CREATE TABLE "t" ("id" INTEGER NOT NULL, "a" INTEGER NOT NULL DEFAULT 0, "b" REAL, "c" REAL, "d" TEXT NOT NULL DEFAULT '0000-00-00 00:00:00', "e" BLOB, "f" TEXT NOT NULL DEFAULT 'x' COLLATE speedyspec_ci, "g" TEXT COLLATE BINARY, "h" TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY ("id"))
    [1] CREATE TRIGGER "t__h__on_update" AFTER UPDATE ON "t" FOR EACH ROW WHEN NEW."h" IS OLD."h" BEGIN UPDATE "t" SET "h" = CURRENT_TIMESTAMP WHERE rowid = NEW.rowid; END   (exact text)
IN  ALTER TABLE t ADD COLUMN c varchar(10) NOT NULL          (t has id, b)
OUT [0] ALTER TABLE "t" ADD COLUMN "c" TEXT NOT NULL DEFAULT '' COLLATE speedyspec_ci
IN  ALTER TABLE t CHANGE COLUMN c c varchar(255) NOT NULL DEFAULT ''      (c was varchar(10) NOT NULL DEFAULT '')
OUT catalogue steps only
IN  ALTER TABLE t MODIFY c bigint(20) NOT NULL DEFAULT 0     (t: id int PK, c varchar(10), KEY c (c))
OUT [0] CREATE TABLE "t__speedyspec_rebuild" ("id" INTEGER NOT NULL, "c" INTEGER NOT NULL DEFAULT 0, PRIMARY KEY ("id"))
    [1] INSERT INTO "t__speedyspec_rebuild" ("id", "c") SELECT "id", "c" FROM "t"
    [2] DROP TABLE "t"   [3] ALTER TABLE "t__speedyspec_rebuild" RENAME TO "t"   [4] CREATE INDEX "t__c" ON "t" ("c")   [5..] catalogue
IN  ALTER TABLE a RENAME TO b                                 (a has KEY k (x))
OUT [0] ALTER TABLE "a" RENAME TO "b"   [1] DROP INDEX "a__k"   [2] CREATE INDEX "b__k" ON "b" ("x")   [3..] catalogue
IN  TRUNCATE TABLE wp_t
OUT [0] DELETE FROM "wp_t"   [1] DELETE FROM sqlite_sequence WHERE name = 'wp_t'
```

- [ ] **Step 1: Write the failing tests** in `tests/ddl.rs`: one per pair (`create_wp_options`, `create_type_map_and_on_update`,
  `add_column_implicit_default`, `change_column_catalogue_only`, `modify_column_rebuilds`, `rename_table_renames_indexes`,
  `truncate_resets_sequence`), each also asserting every step's `count == CountRule::None`;
  `autoincrement_outside_pk_unsupported` (`CREATE TABLE t (a int AUTO_INCREMENT, b int, PRIMARY KEY (a, b))`);
  `prepare_without_introspector` (`SqliteDialect::new().prepare(&parse_one("ALTER TABLE t MODIFY c int")?)` is
  `Unsupported { construct: "ALTER TABLE without schema information", dialect: Sqlite }`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-sqlite --test ddl` — Expected: FAIL.
- [ ] **Step 3: Implement** the passes, `plan.rs` and `dialect.rs` (Task 7's passes as empty placeholders that
  return `Continue`, so the chain order is fixed now).
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-sqlite` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-sqlite): DDL translation, AUTO_INCREMENT, implicit defaults and table rebuilds"` (add new files first).

### Task 7: SQLite DML passes: changed rows, upserts, `REPLACE`, multi-table statements, functions and comparisons

**Files:**
- Create: `crates/speedyspec-data-sqlite/src/passes/dml.rs`, `src/passes/numeric_compare.rs`,
  `src/passes/functions.rs`; modify `src/passes/mod.rs`, `src/plan.rs`
- Test: `crates/speedyspec-data-sqlite/tests/dml.rs`

**Interfaces:**
- Consumes: Task 6.
- Produces:
  - Pass `sqlite_dml`:
    - **Changed rows.** SQLite's `changes()` counts every row an `UPDATE` matched; MySQL's `rows_affected` counts the
      rows whose values changed. So the pass makes SQLite match only rows that change: it ANDs `("c1" IS NOT v1
      COLLATE BINARY OR "c2" IS NOT v2 COLLATE BINARY ...)` onto the `WHERE`, one term per `SET` column, `vN` the
      assigned expression (qualified with the target's alias in a multi-table `UPDATE`). SQLite evaluates every `vN`
      against the old row, the one `SET` sees; `IS NOT` is null-safe; `COLLATE BINARY` makes a case-only change count
      as a change, as MySQL's row-image comparison does; column affinity makes `'5'` equal `5` on an integer column,
      as MySQL does. `changes()` is then MySQL's changed-rows count. The predicate is added after `RowLimitPass`, so it
      sits outside the `rowid IN (...)` subquery and `LIMIT` still counts matched rows, as MySQL's does. No predicate
      when an assigned expression calls `rand`, `uuid`, `now`, `sysdate`, `curdate`, `curtime`, `utc_timestamp` (each
      row then counts as changed).
    - **Upserts.** `ON CONFLICT DO UPDATE SET ..` gets the same predicate as `WHERE` against `excluded` (unchanged →
      no update, `changes()` 0); a multi-row upsert becomes one single-row upsert per row (`Split`, the shared
      `SET` placeholders repeated by number); with an `auto_increment` target (from `cx.introspector`), `RETURNING
      rowid` is appended.
    - **`REPLACE INTO t (cols) VALUES ..`**: per row, `DELETE FROM t WHERE (<u1 = v>) OR (<u2a = va AND u2b = vb>)`
      over the unique and primary indexes (from `cx.introspector`) whose columns are all in `cols`, then the
      `INSERT`; no such index → just the `INSERT`. `REPLACE .. SELECT` → `Unsupported { construct: "REPLACE ...
      SELECT" }`.
    - **Multi-row `INSERT`** into an `auto_increment` table gets `RETURNING rowid` (MySQL reports the first id).
    - **Multi-table `DELETE a, b FROM ..`** (and `DELETE .. USING`): `DROP TABLE IF EXISTS temp."speedyspec_delete"`,
      `CREATE TEMP TABLE "speedyspec_delete" AS SELECT "a".rowid AS "r0", "b".rowid AS "r1" FROM <joins> WHERE <where>`,
      one `DELETE FROM "<table of target i>" WHERE rowid IN (SELECT "r<i>" FROM temp."speedyspec_delete")` per
      target, `DROP TABLE temp."speedyspec_delete"`; the row sets are fixed before any delete, since targets may be
      the same table. A single target needs no temp table: `DELETE FROM "t" WHERE rowid IN (SELECT "a".rowid FROM
      <joins> WHERE <where>)`.
    - **Multi-table `UPDATE a JOIN b ON x SET a.c = .. WHERE w`** → `UPDATE "t1" AS "a" SET "c" = .. FROM "t2" AS "b"
      WHERE (x) AND (w)`; `SET` on two tables, or the target on the outer side of a `LEFT JOIN` → `Unsupported`
      naming it.
  - Pass `sqlite_numeric_compare`: a comparison (`= <> < <= > >=`, `BETWEEN`, `IN`) between a column whose catalogue
    family is `Text` or `Enum` and a numeric literal, or a numbered placeholder whose param is `Int`/`UInt`/`Float`,
    wraps the column in `CAST(.. AS REAL)` (MySQL compares a string with a number as numbers; SQLite would compare
    text). Aliases resolve through the statement's `FROM`; unknown columns are left alone.
  - Pass `sqlite_functions`: `a DIV b` → `CAST(a AS INTEGER) / CAST(b AS INTEGER)`; `a / b` → `CAST(a AS REAL) / b`;
    `GROUP_CONCAT([DISTINCT] x [ORDER BY ..] [SEPARATOR s])` → `group_concat(x, s ORDER BY ..)` (`s` default `','`;
    `DISTINCT` with a separator other than `','` → `Unsupported { construct: "GROUP_CONCAT(DISTINCT ... SEPARATOR)" }`);
    `SUBSTRING(x FROM a FOR b)` → `SUBSTRING(x, a, b)`; `x [NOT] RLIKE p` → `x [NOT] REGEXP p`; `CAST(x AS DATE)` →
    `date(x)`, `AS DATETIME|TIME` → `AS TEXT`, `AS DECIMAL[(p,s)]` → `AS REAL`, `AS BINARY` → `AS BLOB` (SQLite's own
    reading of `DATE` would give the date numeric affinity: `'2024-01-01'` becomes `2024`); `LAST_INSERT_ID()` → `last_insert_rowid()` aliased
    `"LAST_INSERT_ID()"` when unaliased; `BINARY x` → `x COLLATE BINARY`, but `CAST(x AS BLOB)` as a `LIKE`/`REGEXP`
    operand; `COLLATE <x>_bin` → `COLLATE BINARY`, other `COLLATE utf8mb4_*` → `COLLATE speedyspec_ci`;
    `FOR UPDATE`, `FOR SHARE`, `LOCK IN SHARE MODE` dropped (`BEGIN IMMEDIATE` holds the write lock);
    `MATCH (..) AGAINST (..)` → `Unsupported { construct: "MATCH ... AGAINST" }`; `information_schema.tables|columns|
    statistics` kept (sets the plan's `information_schema`), unaliased bare column projections from them aliased to
    the uppercase name (MySQL 8 names them so); any other `information_schema` table → `Unsupported` naming it;
    `START TRANSACTION`/`BEGIN` → `BEGIN IMMEDIATE` (`READ ONLY` → `BEGIN DEFERRED`).
  - Plan rules (`plan.rs`): a step's `count` is `Upsert` for an `INSERT .. ON CONFLICT DO UPDATE`, `Changes` for any
    other `INSERT`/`UPDATE`/`DELETE` on a table that is not `temp.*`, `sqlite_sequence` or a catalogue table, else
    `None`; `insert_id` is `Returning` when the step ends `RETURNING rowid`, `LastRowid` for any other single-row
    `INSERT`, else `None`; `returns_rows` for `SELECT`s.

```text
IN  UPDATE wp_options SET option_value = 'b', autoload = 'auto' WHERE option_name = 'a'
OUT UPDATE "wp_options" SET "option_value" = 'b', "autoload" = 'auto' WHERE ("option_name" = 'a') AND ("option_value" IS NOT 'b' COLLATE BINARY OR "autoload" IS NOT 'auto' COLLATE BINARY)
IN  UPDATE wp_posts SET post_status = 'draft' WHERE post_author = 1 LIMIT 5
OUT UPDATE "wp_posts" SET "post_status" = 'draft' WHERE (rowid IN (SELECT rowid FROM "wp_posts" WHERE "post_author" = 1 LIMIT 5)) AND ("post_status" IS NOT 'draft' COLLATE BINARY)
IN  UPDATE t SET a = RAND()                OUT UPDATE "t" SET "a" = RAND()
IN  UPDATE wp_posts p JOIN wp_postmeta m ON m.post_id = p.ID AND m.meta_key = ? SET p.post_status = ? WHERE m.meta_value = ?      params [k, s, v]
OUT UPDATE "wp_posts" AS "p" SET "post_status" = ?2 FROM "wp_postmeta" AS "m" WHERE ("m"."post_id" = "p"."ID" AND "m"."meta_key" = ?1) AND ("m"."meta_value" = ?3) AND ("p"."post_status" IS NOT ?2 COLLATE BINARY)      params [k, s, v]
IN  the wp_options ON DUPLICATE KEY UPDATE insert from Task 2
OUT INSERT INTO "wp_options" ("option_name", "option_value", "autoload") VALUES ('a', 'b', 'auto') ON CONFLICT DO UPDATE SET "option_name" = excluded."option_name", "option_value" = excluded."option_value", "autoload" = excluded."autoload" WHERE "option_name" IS NOT excluded."option_name" COLLATE BINARY OR "option_value" IS NOT excluded."option_value" COLLATE BINARY OR "autoload" IS NOT excluded."autoload" COLLATE BINARY RETURNING rowid      count Upsert, insert_id Returning
IN  INSERT INTO t (a, b) VALUES (?, ?), (?, ?) ON DUPLICATE KEY UPDATE b = ?          (t without auto_increment)
OUT [0] INSERT INTO "t" ("a", "b") VALUES (?1, ?2) ON CONFLICT DO UPDATE SET "b" = ?5 WHERE "b" IS NOT ?5 COLLATE BINARY
    [1] INSERT INTO "t" ("a", "b") VALUES (?3, ?4) ON CONFLICT DO UPDATE SET "b" = ?5 WHERE "b" IS NOT ?5 COLLATE BINARY
IN  REPLACE INTO wp_options (option_name, option_value, autoload) VALUES ('a', 'b', 'no')
OUT [0] DELETE FROM "wp_options" WHERE ("option_name" = 'a')   [1] INSERT INTO "wp_options" ("option_name", "option_value", "autoload") VALUES ('a', 'b', 'no')
IN  DELETE a, b FROM wp_options a, wp_options b WHERE a.option_name LIKE '\\_transient\\_%' AND a.option_name NOT LIKE '\\_transient\\_timeout\\_%' AND b.option_name = CONCAT('_transient_timeout_', SUBSTRING(a.option_name, 12)) AND b.option_value < 1700000000
OUT [0] DROP TABLE IF EXISTS temp."speedyspec_delete"
    [1] CREATE TEMP TABLE "speedyspec_delete" AS SELECT "a".rowid AS "r0", "b".rowid AS "r1" FROM "wp_options" AS "a", "wp_options" AS "b" WHERE "a"."option_name" LIKE '\_transient\_%' ESCAPE '\' AND "a"."option_name" NOT LIKE '\_transient\_timeout\_%' ESCAPE '\' AND "b"."option_name" = CONCAT('_transient_timeout_', SUBSTRING("a"."option_name", 12)) AND CAST("b"."option_value" AS REAL) < 1700000000
    [2] DELETE FROM "wp_options" WHERE rowid IN (SELECT "r0" FROM temp."speedyspec_delete")
    [3] DELETE FROM "wp_options" WHERE rowid IN (SELECT "r1" FROM temp."speedyspec_delete")
    [4] DROP TABLE temp."speedyspec_delete"                 counts None, None, Changes, Changes, None
IN  SELECT GROUP_CONCAT(term_id ORDER BY term_id SEPARATOR ';') FROM t      OUT SELECT group_concat("term_id", ';' ORDER BY "term_id") FROM "t"
IN  SELECT a DIV b, a / b FROM t                                           OUT SELECT CAST("a" AS INTEGER) / CAST("b" AS INTEGER), CAST("a" AS REAL) / "b" FROM "t"
IN  SELECT SUBSTRING(post_title FROM 2 FOR 3) FROM wp_posts               OUT SELECT SUBSTRING("post_title", 2, 3) FROM "wp_posts"
IN  SELECT LAST_INSERT_ID()                                                OUT SELECT last_insert_rowid() AS "LAST_INSERT_ID()"
IN  SELECT CAST(post_date AS DATE) FROM wp_posts WHERE post_name RLIKE '^a'  OUT SELECT date("post_date") FROM "wp_posts" WHERE "post_name" REGEXP '^a'
IN  SELECT * FROM wp_posts WHERE BINARY post_name = 'Hello' AND post_title LIKE BINARY 'He%'
OUT SELECT * FROM "wp_posts" WHERE "post_name" COLLATE BINARY = 'Hello' AND "post_title" LIKE CAST('He%' AS BLOB) ESCAPE '\'
IN  SELECT * FROM wp_options WHERE option_name = 'a' FOR UPDATE           OUT SELECT * FROM "wp_options" WHERE "option_name" = 'a'
IN  SELECT table_name, data_length FROM information_schema.tables WHERE table_schema = DATABASE()
OUT SELECT "table_name" AS "TABLE_NAME", "data_length" AS "DATA_LENGTH" FROM "information_schema"."tables" WHERE "table_schema" = DATABASE()      information_schema: true
IN  SELECT meta_id FROM wp_postmeta WHERE meta_value > ?      params [Int(5)]      OUT ... WHERE CAST("meta_value" AS REAL) > ?1
IN  SELECT SQL_CALC_FOUND_ROWS wp_posts.ID FROM wp_posts WHERE 1=1 AND post_type = ? ORDER BY wp_posts.post_date DESC LIMIT 0, 10
OUT SELECT "wp_posts"."ID" FROM "wp_posts" WHERE 1 = 1 AND "post_type" = ?1 ORDER BY "wp_posts"."post_date" DESC LIMIT 10 OFFSET 0
    found_rows SELECT COUNT(*) FROM (SELECT "wp_posts"."ID" FROM "wp_posts" WHERE 1 = 1 AND "post_type" = ?1) AS "speedyspec_found_rows"
IN  START TRANSACTION                                                      OUT BEGIN IMMEDIATE
```

- [ ] **Step 1: Write the failing tests** in `tests/dml.rs` with `plan_steps` over `StaticIntrospector` of
  `wp_options`, `wp_posts` (`ID`, `post_author`, `post_name`, `post_title`, `post_status`, `post_type`,
  `post_date`), `wp_postmeta` and `t`: one test per pair above, asserting SQL, params and the counting and insert-id
  rules shown; plus `refusals` (`REPLACE INTO t SELECT * FROM u`, `UPDATE a JOIN b ON a.x = b.x SET a.c = 1, b.d = 2`,
  `SELECT GROUP_CONCAT(DISTINCT a SEPARATOR ';') FROM t`, `SELECT ID FROM wp_posts WHERE MATCH (post_title) AGAINST
  ('x')`, `SELECT * FROM information_schema.processlist`, each the named `Unsupported` with `dialect: Sqlite`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-sqlite --test dml` — Expected: FAIL.
- [ ] **Step 3: Implement** the three passes and the plan rules.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-sqlite` — Expected: PASS (Tasks 5–7 tests).
- [ ] **Step 5: Commit** — `git commit -am "feat(data-sqlite): changed rows, upserts, REPLACE, multi-table statements and MySQL functions"` (add new files first).

### Task 8: `SqliteStore`, its introspector and engine, passing `sql_store_suite`

**Files:**
- Create: `crates/speedyspec-data-sqlite/src/store.rs`, `src/introspect.rs`, `src/info_schema.rs`, `src/results.rs`,
  `src/errors.rs`, `src/factory.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-data-sqlite/tests/conformance.rs`, `tests/store.rs`

**Interfaces:**
- Consumes: Tasks 4–7; plan 1's `SqlStore`, `Introspector`, `Capabilities`, `Engine`, `EngineFactory`,
  `EngineRegistry`, `sql_store_suite`.
- Produces:
  - `pub struct SqliteEngineFactory` (`scheme() == "sqlite"`; URL `sqlite://:memory:` or `sqlite:///<absolute path>`;
    options `busy_timeout_ms` (default `5000`), `database` (the name `DATABASE()` and `SHOW TABLES` report; default
    the file stem, `main` in memory)); `pub struct SqliteEngine` (`kind() == EngineKind::Sqlite`, `open` →
    `EngineHandle::Sql(Rc<SqliteStore>)`).
  - `pub struct SqliteStore` with `SqliteStore::open(config: &EngineConfig) -> Result<SqliteStore, DbError>` (opens,
    runs `catalogue_ddl()` through its own plan), `plan(&self, stmt: &Statement) -> Result<SqlitePlan, DbError>` (with
    its introspector), `busy_timeout(&self) -> Duration`, and `impl SqlStore`:
    - `capabilities()`: `transactions: true, savepoints: true, ordered_ranges: false, expiry: false,
      atomic_increment: true, returning: true, upsert: UpsertForm::OnConflict`.
    - Running a `Run` plan: `information_schema` → refresh first; more than one step → wrapped in `BEGIN IMMEDIATE`
      .. `COMMIT` when in autocommit, else `SAVEPOINT speedyspec_stmt` .. `RELEASE`, rolled back on any error so the
      statement is atomic as in MySQL. Each step binds `params[i - 1]` for `i` in `1..=parameter_count()`.
      `rows_affected` sums the steps: `Changes` adds `changes()`; `Upsert` adds `0` when `changes()` is 0, `1` when
      `last_insert_rowid()` moved (inserted), else `2` (updated), MySQL's `ON DUPLICATE KEY` counts. `insert_id` is
      the first step's: `LastRowid` → `Some(last_insert_rowid())` when the step changed a row and the rowid moved;
      `Returning` → the first returned rowid; else `None`. A table with no `auto_increment` column never reports one.
      `query()` returns the `returns_rows` step's rows (empty `Rows` for none); `execute()` of a row-returning plan
      reports the row count as `rows_affected`, as `mysqli_affected_rows` does.
    - Found rows: with `found_rows`, the count query runs after the select and its value is kept on the store; any
      other `SELECT` keeps its row count; `Emulated(op)` answers through `answer_emulated(op, introspector,
      session, found_rows)`, `execute()` returning `rows_affected: 0`.
    - `begin()` is `BEGIN IMMEDIATE`; `in_transaction()` is `!conn.is_autocommit()`; `reset()` rolls back an open
      transaction, resets `SessionVars`, zeroes found rows and drops `temp.speedyspec_*`.
  - Results (`results.rs`): `Integer` → `DbValue::Int`, `Real` → `Float`, `Text` → `Text`, `Blob` → `Bytes`, `Null`;
    a value whose origin column (rusqlite column metadata) is `decimal(p,s)` in the catalogue → `Text` with exactly `s`
    decimals, as MySQL returns it. `ColumnInfo`: `name` the result name, `org_name`/`org_table` the origin, `table`
    the origin table (SQLite reports no alias), `sql_type` the same strings `MySqlStore` produces for the origin's
    MySQL type (expressions: `bigint` for integers, `double` for reals, `longtext` for text), `max_length` the longest
    value's byte length, `nullable`, `decimals`, `charset` 63 for numbers and binaries, else 255.
  - `SqliteIntrospector` (`introspect()`): `server_version()` = `"8.4.0-speedyspec-sqlite-<sqlite_version>"`
    (`db_version()` reads `8.4.0`); `tables`, `columns`, `indexes`, `table_collation` from `read_catalogue`, falling
    back to `PRAGMA table_info`/`index_list` for tables with no catalogue rows (types by affinity); catalogue tables
    never listed.
  - `information_schema` (attached in memory): `tables (TABLE_CATALOG, TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE, ENGINE,
    TABLE_ROWS, DATA_LENGTH, INDEX_LENGTH, AUTO_INCREMENT, TABLE_COLLATION)`, `columns (TABLE_SCHEMA, TABLE_NAME,
    COLUMN_NAME, ORDINAL_POSITION, COLUMN_DEFAULT, IS_NULLABLE, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, COLUMN_TYPE,
    COLUMN_KEY, EXTRA, COLLATION_NAME, CHARACTER_SET_NAME)`, `statistics (TABLE_SCHEMA, TABLE_NAME, NON_UNIQUE,
    INDEX_NAME, SEQ_IN_INDEX, COLUMN_NAME, SUB_PART)`, rewritten from the introspector on refresh (`TABLE_ROWS` by
    `COUNT(*)`, lengths `0`).
  - Errors (`errors.rs`): `SQLITE_CONSTRAINT_*` → `Constraint`; `SQLITE_BUSY`/`SQLITE_LOCKED` → `Engine { code:
    Some("SQLITE_BUSY"|"SQLITE_LOCKED") }`; `SQLITE_CANTOPEN`/`SQLITE_NOTADB` → `Connection`; others → `Engine` with
    the extended code's name. `engine` is the configured name.

- [ ] **Step 1: Write the failing tests.** `tests/conformance.rs`: `sqlite_conformance_in_memory` and
  `sqlite_conformance_file` (a `tempfile` dir, `sqlite:///<dir>/site.db`), each through `SqliteEngineFactory` and an
  `EngineRegistry`, then `sql_store_suite(&*store, "speedyspec_conformance")`; the file case also checks a second
  plain rusqlite connection reads `PRAGMA journal_mode` = `wal` and `store.busy_timeout()` is 5 s.
  `tests/store.rs`, on the Task 3 `wp_options` and `wp_postmeta` DDL run through `execute`:
  `changed_rows_like_mysql` (insert `a`/`x`; same `UPDATE` again → 0; `'Hello'` then `'hello'` → 1 and the value reads
  `hello`; the Task 2 upsert on a new name → 1, repeated → 0, with a new value → 2; `REPLACE` of an existing name → 2;
  `INSERT IGNORE` of a duplicate → 0);
  `insert_ids` (first insert `Some(1)`; a two-row insert after it → `Some(2)`; insert into `CREATE TABLE n (a int)` →
  `None`; ignored duplicate → `None`);
  `found_rows` (15 posts; `SQL_CALC_FOUND_ROWS .. LIMIT 0, 10` gives 10 rows and `SELECT FOUND_ROWS()` 15; `SELECT *
  FROM wp_posts LIMIT 3` then 3; `reset()` then 0);
  `results_stringified_like_mysql` (`decimal(10,2)` holding 5 → `Text("5.00")`; `double` 1.5 → `to_mysql_text` `1.5`;
  NULL → `None`; `SELECT option_name AS n FROM wp_options o` → `name == "n"`, `org_name == "option_name"`,
  `org_table == "wp_options"`);
  `implicit_default_applies` (insert omitting `option_value` succeeds and reads `''`);
  `like_escape` (rows `wp_a`, `wpxa`; `LIKE 'wp\\_%'` returns only `wp_a`);
  `unique_is_case_insensitive` (`Key` then `key` into `option_name` → `Constraint`);
  `dbdelta_round_trip` (`DESCRIBE wp_postmeta` and `SHOW INDEX FROM wp_postmeta` give exactly Task 4's rows, and
  `SHOW CREATE TABLE wp_options` its text: what MySQL 8.4 shows);
  `emulated_admin` (`SHOW TABLES` lists no `_speedyspec_` table; `SELECT VERSION()` starts `8.4.0`; `SELECT DATABASE()`
  is the configured `database`; `SET SESSION sql_mode = 'NO_ENGINE_SUBSTITUTION'` then `SELECT @@SESSION.sql_mode`
  gives it; after `reset()` the default);
  `information_schema_tables` (`SELECT table_name FROM information_schema.tables WHERE table_schema = DATABASE() ORDER
  BY table_name` → column `TABLE_NAME`, rows `wp_options`, `wp_postmeta`);
  `transients_cleanup` (expired pair `_transient_a`/`_transient_timeout_a` = `999`, live pair with `99999999999`; the
  Task 7 multi-table `DELETE` with `1700000000` → `rows_affected == 2`, the live pair remains);
  `statement_is_atomic` (a `REPLACE` whose insert fails on `NOT NULL` with an explicit NULL leaves the old row);
  `concurrent_writer_waits` (a thread opens its own store on the same file, `begin()`s, inserts, sleeps 200 ms,
  commits; the main store's insert started meanwhile succeeds after ≥ 150 ms; with `busy_timeout_ms=50` it fails with
  `Engine { code: Some("SQLITE_BUSY"), .. }`);
  `reset_rolls_back_raw_transaction` (`START TRANSACTION`, insert, `reset()` → row gone, `in_transaction()` false).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-sqlite` — Expected: FAIL (store missing).
- [ ] **Step 3: Implement** the modules above. The catalogue lookups for result conversion are cached per store and
  cleared after any `Ddl` plan.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-sqlite` — Expected: PASS, the conformance suite printing no failed check.
- [ ] **Step 5: Run** `cargo clippy --workspace --all-targets --all-features -- -D warnings` and `cargo fmt --check` — Expected: clean.
- [ ] **Step 6: Commit** — `git commit -am "feat(data-sqlite): the SQLite store, introspector and engine"` (add new files first).

### Task 9: The SQL corpus

**Files:**
- Create: `crates/speedyspec-data-corpus/Cargo.toml`, `src/lib.rs`, `src/shape.rs`, `src/check.rs`, `src/main.rs`
  (bin `speedyspec-sql-corpus`); `tools/sql-corpus/phpunit-bootstrap.php`, `tools/sql-corpus/corpus-logger.php`,
  `tools/sql-corpus/collect-phpunit.sh`, `tools/sql-corpus/collect-schema.sh`, `tools/sql-corpus/collect-plugins.sh`;
  `compat/sql-corpus/plugins.json`, `compat/sql-corpus/corpus.json`, `compat/sql-corpus/unsupported.json`,
  `compat/sql-corpus/parse-gaps.json`
- Test: `crates/speedyspec-data-corpus/tests/shape.rs`, `tests/corpus.rs`; `crates/speedyspec-data-sqlite/tests/corpus.rs`

**Interfaces:**
- Consumes: plan 1's `parse_one`, `parse_many`, `DialectKind`; `speedyspec_compat::compat_dir()`; Task 8's
  `SqliteStore::plan`.
- Produces:
  - `pub struct CorpusEntry { pub shape: Option<String>, pub example: String, pub sources: BTreeSet<String>, pub
    count: u64 }` (serde; `shape` `None` when the statement does not parse); `pub fn shape_of(sql: &str) ->
    Result<String, DbError>` (parse, every literal and placeholder → `?`, an `IN` list of only those → `IN (?)`,
    printed); `pub fn ingest(lines: impl BufRead, into: &mut BTreeMap<String, CorpusEntry>)` (JSON lines `{"sql",
    "source"}`; keyed by shape, or by the text when it does not parse; first text kept as `example`).
  - `pub struct UnsupportedEntry { pub construct: String, pub dialect: String, pub why: String }`;
    `pub struct CheckReport { pub planned: usize, pub listed_unsupported: usize, pub unlisted_unsupported: Vec<(String,
    String)>, pub parse_failures: Vec<String>, pub errors: Vec<(String, String)> }` with `ok(&self) -> bool` (all three
    lists empty); `pub fn check(entries: &[CorpusEntry], dialect: DialectKind, plan: &dyn Fn(&Statement) -> Result<(),
    DbError>, unsupported: &[UnsupportedEntry], parse_gaps: &[String]) -> CheckReport` (an `Unsupported` error is
    accepted only if its construct is listed for `dialect`; a parse failure only if its text is in `parse_gaps`).
  - bin: `speedyspec-sql-corpus ingest <file.jsonl>...` merges into `compat/sql-corpus/corpus.json` (sorted, pretty).
  - Collection (WordPress side, in a `wordpress-develop` checkout with its Docker environment, MySQL as usual):
    - `phpunit-bootstrap.php`: `require_once __DIR__ . '/includes/functions.php';` then `tests_add_filter( 'query', .. ,
      PHP_INT_MAX )` appending `{"sql": $query, "source": "wordpress-phpunit"}` lines to `dirname( __DIR__, 2 ) .
      '/speedyspec-corpus.jsonl'`, then `require __DIR__ . '/includes/bootstrap.php';`.
    - `collect-phpunit.sh <wordpress-develop dir> <out.jsonl>`: copies the bootstrap to
      `tests/phpunit/speedyspec-corpus-bootstrap.php`, runs `npm run test:php -- --bootstrap
      tests/phpunit/speedyspec-corpus-bootstrap.php` and again with `-c tests/phpunit/multisite.xml`, moves the log to
      `<out.jsonl>`, removes the copy.
    - `collect-schema.sh <dir> <out.jsonl>`: `npm run env:cli -- eval` printing `wp_get_db_schema( 'all' )` and
      `wp_get_db_schema( 'ms_global' )`, split into statements, source `wordpress-schema`.
    - `corpus-logger.php` (a mu-plugin) and `collect-plugins.sh <dir> <out.jsonl>`: for each plugin in
      `plugins.json`, after `npm run env:reset && npm run env:install`, installs the logger (source
      `plugin:<slug>@<version>`), `env:cli -- plugin install <slug> --version=<version> --activate`, `env:cli -- cron
      event run --all`, `curl`s `/`, `/?s=corpus`, `/?p=1`, `/feed/`, `/wp-json/`, `/wp-login.php` on
      `http://localhost:8889`, then `plugin deactivate <slug> --uninstall`, and removes the logger.
    - `plugins.json`: `woocommerce`, `wordpress-seo`, `contact-form-7`, `elementor`, `akismet`, `jetpack`,
      `wpforms-lite`, `wordfence`, `redirection`, `advanced-custom-fields`, each with the version current at collection.
  - `unsupported.json` starts with `MATCH ... AGAINST` for `sqlite` ("SQLite full-text search is not wired up yet").
    `parse-gaps.json` holds statement texts `sqlparser` rejects, each with the upstream issue; it must contain no
    statement whose only source is `wordpress-phpunit` or `wordpress-schema`.

- [ ] **Step 1: Write the failing tests:** `tests/shape.rs`: `shape_replaces_literals` (`SELECT * FROM wp_posts WHERE ID
  = 5 AND post_type IN ('a', 'b')` → `SELECT * FROM wp_posts WHERE ID = ? AND post_type IN (?)`, and the same shape for
  `ID = 7 AND post_type IN ('c')`); `ingest_dedupes_and_counts` (three lines of two shapes → two entries, counts 2 and
  1, sources merged); `unparseable_kept_verbatim`; `check_classifies` (a planner closure returning `Unsupported {
  construct: "X" }` is listed or unlisted by `unsupported.json` content; a parse failure is accepted only through
  `parse_gaps`). `crates/speedyspec-data-corpus/tests/corpus.rs`: `corpus_parses` (every entry of the committed
  `corpus.json` parses or is in `parse-gaps.json`; none of the gaps is core-only). `crates/speedyspec-data-sqlite/tests/corpus.rs`:
  `corpus_translates_for_sqlite` (an in-memory `SqliteStore` runs every `wordpress-schema` entry, then
  `check(entries, Sqlite, &|s| store.plan(s).map(|_| ()), ..)` is `ok()`, printing the report).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-corpus --test shape` — Expected: FAIL.
- [ ] **Step 3: Implement** the crate and the bin.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-corpus --test shape` — Expected: PASS.
- [ ] **Step 5: Collect.** In a `wordpress-develop` checkout with `npm run env:start` and `npm run env:install` done:
  `tools/sql-corpus/collect-schema.sh <dir> /tmp/schema.jsonl`, `tools/sql-corpus/collect-phpunit.sh <dir>
  /tmp/phpunit.jsonl`, `tools/sql-corpus/collect-plugins.sh <dir> /tmp/plugins.jsonl`, then `cargo run -p
  speedyspec-data-corpus -- ingest /tmp/schema.jsonl /tmp/phpunit.jsonl /tmp/plugins.jsonl` — Expected:
  `compat/sql-corpus/corpus.json` written, the plugin versions recorded in `plugins.json`.
- [ ] **Step 6: Run** `cargo test -p speedyspec-data-corpus -p speedyspec-data-sqlite --test corpus` — Expected: at
  first, a report of failures. For each: a translation gap is fixed in the pass that owns the construct, with a new
  pair in that task's test file; a construct with no faithful SQLite form becomes an `Unsupported` in its pass and an
  `unsupported.json` entry with its why; a parse failure goes to `parse-gaps.json` with an upstream issue (none from
  core). Repeat until both tests PASS.
- [ ] **Step 7: Commit** — `git add crates/speedyspec-data-corpus tools/sql-corpus compat/sql-corpus crates/speedyspec-data-sqlite crates/speedyspec-data-translate && git commit -m "feat(data-corpus): the SQL corpus, and every statement in it translating for SQLite"`

### Task 10: Cross-engine replay of `compat/wp-db` and WordPress's database tests on SQLite

**Files:**
- Create: `crates/speedyspec-wp-db-php/tests/common/mod.rs`, `crates/speedyspec-wp-db-php/tests/replay_sqlite.rs`,
  `compat/phpunit-sqlite.json`
- Modify: `crates/speedyspec-wp-db-php/Cargo.toml` (dev-dependency `speedyspec-data-sqlite`), `compat/wp-db/breaks.json`
  (adds `engine_breaks`), plan 2's runner only if it lacks `for_engine`

**Interfaces:**
- Consumes: plan 2's `compat/wp-db` and runner (see "Assumed from plan 2"); `speedyspec_compat::{load, replay_with,
  Breaks, Break}`; Task 8's `SqliteEngineFactory`.
- Produces:
  - `compat/wp-db/breaks.json` gains `"engine_breaks": { "sqlite": [ { "scenario", "step"?, "why" } ] }`.
    `speedyspec_compat::Breaks` ignores the extra key (as it ignores `hooks`' `limits`), so MySQL replays keep
    exactly plan 2's breaks.
  - `common::breaks_for_engine(area: &str, engine: &str) -> Breaks`: `breaks.json`'s `breaks` plus
    `engine_breaks[engine]`.
  - `compat/phpunit-sqlite.json`: `{ "expected_failures": [ { "test": "<Class>::<method>", "why" } ] }`, shared with
    Task 11 and plan 6's PostgreSQL equivalent.

- [ ] **Step 1: Write the failing tests** in `tests/replay_sqlite.rs`: `wp_db_replays_on_sqlite` (load `wp-db`;
  runner `WpDbScenarioRunner::for_engine(EngineConfig { name: "default", url: "sqlite://:memory:", options: {} })`,
  a fresh database per scenario; `replay_with(&catalogue, &fixture, &breaks_for_engine("wp-db", "sqlite"), &mut
  runner)` is empty, each mismatch printed with `Display`); `sqlite_breaks_are_still_needed` (a replay with only the
  shared `breaks` gives a mismatch for every scenario named in `engine_breaks.sqlite`, so no entry is stale). If
  plan 2's runner lacks `for_engine`, add it first: same as its MySQL constructor, with `SqliteEngineFactory`
  registered and `config` as `default`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-db-php --test replay_sqlite` — Expected: FAIL with the list of
  mismatches.
- [ ] **Step 3: Triage each mismatch.** A translation or result-mapping fault is fixed in the module that owns it
  (Tasks 1–8), with a test pair or store test pinning it. A genuine engine difference gets an `engine_breaks.sqlite`
  entry whose `why` names the difference. Differences this plan expects may appear (add only those the replay shows):
  accent and `ß` ordering under `speedyspec_ci` versus `utf8mb4_unicode_520_ci`; division results without MySQL's
  `div_precision_increment` zeros (`0.5` for `0.5000`); `SET a = b, b = a` (MySQL assigns left to right, SQLite from
  the old row); `information_schema` sizes reported as 0; `NOW()` in UTC; `Duplicate entry` wording; unsigned
  `bigint` beyond `i64`; `table` in column info being the table, not its alias; DDL inside a transaction not
  committing it.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-db-php --test replay_sqlite` — Expected: PASS.
- [ ] **Step 5: Run WordPress's database tests on SQLite:** `SPEEDYSPEC_DB_URL=sqlite:///tmp/speedyspec-phpunit.db
  scripts/wp-phpunit.sh tests/phpunit/tests/db.php tests/phpunit/tests/dbdelta.php tests/phpunit/tests/db --log-junit
  /tmp/sqlite-db.xml`, and the same with plan 2's MySQL URL into `/tmp/mysql-db.xml` — Expected: every test passing
  on MySQL passes on SQLite, except entries in `compat/phpunit-sqlite.json`. Fix faults as in Step 3; list genuine
  differences there with their why. Delete `/tmp/speedyspec-phpunit.db*` after.
- [ ] **Step 6: Commit** — `git add compat crates && git commit -m "test(wp-db): replay the wp-db catalogue and WordPress's database tests on SQLite"`

### Task 11: WordPress's options tests on SQLite (only once plan 4 has landed)

**Files:**
- Modify: `compat/phpunit-sqlite.json` (only if a genuine difference appears); fixes go to the owning module

- [ ] **Step 1: Check** that `crates/speedyspec-wp-options` is on `origin/main` (`git ls-tree origin/main crates/ | grep
  speedyspec-wp-options`). If it is not, skip this task and say so in the PR description.
- [ ] **Step 2: Run** `SPEEDYSPEC_DB_URL=sqlite:///tmp/speedyspec-options.db scripts/wp-phpunit.sh
  tests/phpunit/tests/option --log-junit /tmp/sqlite-option.xml`, and with plan 2's MySQL URL into
  `/tmp/mysql-option.xml` — Expected: the same passing set on both, except entries in `compat/phpunit-sqlite.json`.
- [ ] **Step 3: Fix** each difference in its owning module with a test, or list a genuine engine difference in
  `compat/phpunit-sqlite.json`; re-run Step 2 until it holds. Delete `/tmp/speedyspec-options.db*`.
- [ ] **Step 4: Commit** (only if anything changed) — `git commit -am "test(options): WordPress's option tests pass on SQLite"`

### Task 12: Hand the SQLite engine's wiring to the runtime's owner

**Files:**
- Create: `/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/sqlite-engine.md`
  (outside the repository: not committed)

- [ ] **Step 1: Write the note**, titled "SQLite engine: wiring request for the runtime", stating:
  - What exists: `speedyspec_data_sqlite::SqliteEngineFactory` (scheme `sqlite`, URLs `sqlite:///<path>` and
    `sqlite://:memory:`, options `busy_timeout_ms`, `database`) on branch `feat/sqlite-engine` of
    `SpeedySpec/speedyspec-server`; `crates/speedyspec-php-runtime` was not touched.
  - The request: register `SqliteEngineFactory` in the worker's `EngineRegistry` beside `MySqlEngineFactory`; when the
    site configuration gives `default` a `sqlite://` URL, use it instead of the `DB_*` constants from `wp-config.php`
    (the key name is the runtime owner's call).
  - What the runtime must keep: every worker process of a site opens the same file, so the file and its `-wal` and
    `-shm` siblings must be writable by the workers' user and not on storage shared between replicas (with
    `[cluster]`, SQLite is per replica, as files are); `reset()` at request end stays required; `:memory:` is for
    tests only, since each worker would hold its own database.
- [ ] **Step 2: Check** — `ls "/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/"` — Expected: `sqlite-engine.md` listed.
