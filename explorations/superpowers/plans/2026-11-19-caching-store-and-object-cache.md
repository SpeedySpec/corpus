# Caching Store and object cache Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The Caching Store contract with its memory and SQL implementations and a shared conformance suite, PHP's
`serialize` format in Rust, and WordPress's object cache (`wp_cache_*`, `WP_Object_Cache`) ported as commands and
queries with a value-holding first level and an optional persistent second level, native in PHP and proven against
WordPress by a recorded `object-cache` catalogue and WordPress's own `tests/phpunit/tests/cache.php`.

**Architecture:** `speedyspec-cache` holds only the contract (bytes, groups, expiry). `speedyspec-cache-memory` and
`speedyspec-cache-sql` implement it (Strategy); `speedyspec-cache-conformance` is the suite both run.
`speedyspec-php-serialize` is a domain service with its own `PhpValue`. `speedyspec-wp-object-cache` is the area:
`ObjectCache` (two levels, Decorator over an optional `CachingStore`), its outbound ports, and its use cases on the
CQRS buses, handled by `CurrentObjectCache`. `speedyspec-wp-object-cache-php` is the native PHP surface (Facade) and
the PHP adapters of the ports. All blocking and single-threaded (`Rc`, `RefCell`, `Cell`).

**Tech Stack:** Rust 1.99, edition 2024; `indexmap`, `sha2` (SqlCache long keys); the workspace's
`speedyspec-hooks`, `speedyspec-cqrs`, `speedyspec-container`, `speedyspec-php`, `speedyspec-php-ext`,
`speedyspec-compat`; plan 1's `speedyspec-data`, `speedyspec-data-query`, `speedyspec-data-mysql`. No `thiserror`
(hand-written `Display`, as the Hook API crates do).

**Spec:** `docs/superpowers/specs/2026-11-19-database-layer-design.md` ("The Caching Store", Testing) and
`docs/superpowers/specs/2026-11-19-options-api-port-design.md` ("The object cache's two levels", "PHP surface",
Phases (phase 1), Testing). Plan index (repository, worktree, shared names):
`docs/superpowers/plans/2026-11-19-data-and-options-ports-index.md`. Reference pattern: the Hook API port
(`crates/speedyspec-wp-hooks`, `crates/speedyspec-wp-hooks-php`, `crates/speedyspec-compat`, `compat/hooks/`).

## Global Constraints

- Work in a new worktree of `SpeedySpec/speedyspec-server`, branch `feat/caching-store-and-object-cache`, from
  `origin/main` once plan 1 (`feat/data-layer-foundation`) is merged (Task 3 needs `speedyspec-data*`; Tasks 1, 2 and
  4–11 do not, and may start from plan 1's branch). New crates go under `crates/`; `members = ["crates/*"]` needs no
  edit.
- Each crate's `Cargo.toml` uses `edition.workspace = true`, `rust-version.workspace = true`,
  `license.workspace = true`, version `0.1.0`, and a one-line `description`.
- Contracts never share a crate with implementations. No crate named `-domain`, `-app`, `-infra` or `-core`.
  `speedyspec-cache` depends on nothing.
- No PHP or WordPress knowledge in `speedyspec-cache*`. No PHP runtime types in `speedyspec-php-serialize` or
  `speedyspec-wp-object-cache` (they depend on `speedyspec-hooks` for `Value`, not on `speedyspec-php`).
- Never edit `crates/speedyspec-php-runtime`. Its changes go in a hand-over note (Task 12).
- Phase 1 scope: the worker runs the object cache with **no persistent level**, as WordPress's default.
  `MemoryCache` stands in as the second level in tests. `SqlCache` is built and passes the conformance suite here;
  configuring a persistent level for a site is plan 8.
- Patterns used here: Strategy (`CachingStore` backends; `ValueCodec`), Decorator (the object cache's two levels),
  Adapter (`SqlCache` over `SqlStore`; the PHP adapters of the area's ports), Serialized LOB (second-level values in
  PHP's format), Command / CQRS (every `wp_cache_*` a command or query), Facade (the native `WP_Object_Cache` and
  functions), Registry (native `WP_Object_Cache` objects to their Rust caches). Nothing else.
- Command and query keys are `SpeedySpec\WpObjectCache\<Name>Handler`.
- Tests that need MySQL are `#[ignore = "needs MySQL: set SPEEDYSPEC_TEST_MYSQL_URL"]`; start it with
  `docker run -d --name speedyspec-mysql -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=speedyspec_test -p 33306:3306 mysql:8.4`
  and run with `SPEEDYSPEC_TEST_MYSQL_URL=mysql://root:password@127.0.0.1:33306/speedyspec_test`.
- PHP tests are `harness = false` binaries with their own `main()` and `check(name, ok, detail)`, as
  `speedyspec-wp-hooks-php/tests/php.rs` does (PHP boots once per process).

## Review Focus

- **A request ends with PHP objects and arrays in the first level** (pass-through zvals): every PHP value is dropped
  in `request_end`, before PHP's request shutdown, and the next request starts empty without a crash. Test
  `values_released_at_request_end` in Task 9.
- **`replace` with the value already stored** on MySQL, where `UPDATE` counts changed rows: `replace` returns `true`.
  Check `replace_same_value_true` in the suite, Task 2 (run by MySQL in Task 3).
- **Keys `"5"` and `5`** are one entry, as PHP's array keys are; `"05"`, `" 5"` and `"-0"` are not integers. Test
  `numeric_string_key_is_int_key` in Task 6.
- **Keys longer than the SQL column, or with `:`, multibyte or NUL characters**, stay distinct and readable. Check
  `long_and_odd_keys` in the suite, Task 2.
- **Corrupt or truncated bytes in the second level** (another writer, a cut-off value): a miss with a notice, never a
  panic or an error to the caller. Test `corrupt_second_level_is_a_miss` in Task 7.

---

### Task 1: The Caching Store contract (`speedyspec-cache`)

**Files:**
- Create: `crates/speedyspec-cache/Cargo.toml`, `src/lib.rs`, `src/error.rs`
- Test: `crates/speedyspec-cache/tests/contract.rs`

**Interfaces:**
- Produces:
  - `pub enum CacheError { Backend { cache: String, message: String }, NotANumber { group: String, key: String } }`
    with `Display` and `std::error::Error`.
  - `pub trait CachingStore` (object-safe, all `&self`): `fn name(&self) -> &str;`
    `fn get(&self, group: &str, key: &str) -> Result<Option<Vec<u8>>, CacheError>;`
    `fn get_many(&self, group: &str, keys: &[&str]) -> Result<Vec<Option<Vec<u8>>>, CacheError>;` (one entry per
    key, in order, duplicates included)
    `fn set(&self, group: &str, key: &str, value: &[u8], expiry: Option<Duration>) -> Result<(), CacheError>;`
    `fn add(&self, group: &str, key: &str, value: &[u8], expiry: Option<Duration>) -> Result<bool, CacheError>;`
    (false when a live entry exists) `fn replace(..same..) -> Result<bool, CacheError>;` (false when none does)
    `fn delete(&self, group: &str, key: &str) -> Result<bool, CacheError>;` (true when a live entry was removed)
    `fn increment(&self, group: &str, key: &str, by: i64) -> Result<Option<i64>, CacheError>;`
    `fn decrement(&self, group: &str, key: &str, by: i64) -> Result<Option<i64>, CacheError>;`
    `fn flush(&self) -> Result<(), CacheError>;` `fn flush_group(&self, group: &str) -> Result<(), CacheError>;`
- Rules, each in the trait's doc comments: values are opaque bytes; `None` expiry never expires, `Some(ZERO)` is
  expired at once; an expired entry behaves as absent everywhere; `increment`/`decrement` read the value as an ASCII
  signed decimal integer (`NotANumber` otherwise), store the result the same way, keep the entry's expiry, return
  `None` when absent; `decrement` floors at 0 (memcached's rule); `flush_group` touches only that group.

- [ ] **Step 1: Write the failing test** `caching_store_is_object_safe` (a function taking `&dyn CachingStore`
  compiles) and `cache_error_display` (`Backend { cache: "sql:default", message: "gone" }` displays text containing
  both; `NotANumber { group: "g", key: "k" }` contains `g` and `k`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-cache` — Expected: FAIL (crate missing).
- [ ] **Step 3: Implement** the trait and error above.
- [ ] **Step 4: Run** `cargo test -p speedyspec-cache` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-cache && git commit -m "feat(cache): the Caching Store contract"`

### Task 2: The conformance suite and `MemoryCache`

**Files:**
- Create: `crates/speedyspec-cache-conformance/Cargo.toml`, `src/lib.rs`;
  `crates/speedyspec-cache-memory/Cargo.toml`, `src/lib.rs`
- Test: `crates/speedyspec-cache-memory/tests/conformance.rs`

**Interfaces:**
- Consumes: Task 1.
- Produces: `pub fn caching_store_suite(cache: &dyn CachingStore)` in `speedyspec-cache-conformance` (its own crate,
  not `speedyspec-data-conformance`, so the cache contract's suite does not pull in the data layer; plan 7's
  `KvCache` and `FileCache` run it too), panicking with the failing check's name. It starts with `flush()`.
  `pub struct MemoryCache` with `MemoryCache::new() -> MemoryCache`, `Default`, `impl CachingStore` (`name()` is
  `"memory"`), entries in `RefCell<HashMap<String, HashMap<String, (Vec<u8>, Option<Instant>)>>>`.
- Checks, in order (group `"g"` unless stated):
  1. `miss_is_none`: `get("g", "absent") == None`.
  2. `set_get_bytes`: `set("k", b"a\0\xff")` then `get` is `Some(b"a\0\xff")`.
  3. `set_overwrites`: `set("k", b"2")` then `get` is `Some(b"2")`.
  4. `add_only_if_absent`: `add("k", b"3")` is `false`, value still `b"2"`; `add("k2", b"x")` is `true`.
  5. `replace_only_if_present`: `replace("absent", ..)` is `false` and creates nothing; `replace("k", b"4")` is `true`.
  6. `replace_same_value_true`: `replace("k", b"4")` again is `true`.
  7. `delete_reports_presence`: `delete("k2")` is `true`, again `false`.
  8. `get_many_in_order`: keys `["k", "absent", "k"]` give `[Some(b"4"), None, Some(b"4")]`.
  9. `groups_are_separate`: `set("g2", "k", b"other")`; `get("g", "k")` still `b"4"`.
  10. `flush_group_only_that_group`: `flush_group("g2")` removes `g2/k`, keeps `g/k`.
  11. `increment_decimal`: `set("n", b"5")`, `increment("n", 3) == Some(8)`, `get("n") == Some(b"8")`;
      `increment("absent", 1) == None`; on `b"x"` it is `Err(NotANumber { .. })`.
  12. `decrement_floors_at_zero`: `set("n", b"2")`, `decrement("n", 5) == Some(0)`.
  13. `expiry_zero_is_expired`: `set("z", b"v", Some(Duration::ZERO))` then `get("z") == None`, and `add("z", ..)`
      is `true`.
  14. `expiry_elapses`: `set("e", b"v", Some(1s))` is readable at once; after sleeping 2.1 s it is absent,
      `delete("e")` is `false`, `replace("e", ..)` is `false` and `add("e", ..)` is `true`.
  15. `long_and_odd_keys`: a 300-character key and its 299-character prefix are distinct entries; keys `"é:\0x"` and
      `"1:5"` round-trip; group `""` works.
  16. `empty_value`: `set("empty", b"")` gives `Some(b"")`, not `None`.
  17. `flush_all`: `flush()` empties every group.

- [ ] **Step 1: Write the failing test** `memory_cache_conformance` in `speedyspec-cache-memory/tests/conformance.rs`:
  `caching_store_suite(&MemoryCache::new())`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-cache-memory` — Expected: FAIL (crates missing).
- [ ] **Step 3: Implement** the suite (a dev-dependency of each implementation) and `MemoryCache`; expired entries are
  dropped when met.
- [ ] **Step 4: Run** `cargo test -p speedyspec-cache-memory` — Expected: PASS, every check printed `ok`.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-cache-conformance crates/speedyspec-cache-memory && git commit -m "feat(cache): the Caching Store conformance suite and MemoryCache"`

### Task 3: `SqlCache` over any `SqlStore` (`speedyspec-cache-sql`)

**Files:**
- Create: `crates/speedyspec-cache-sql/Cargo.toml`, `src/lib.rs`, `src/schema.rs`, `src/statements.rs`
- Test: `crates/speedyspec-cache-sql/tests/statements.rs`, `tests/conformance.rs`

**Interfaces:**
- Consumes: Task 1, Task 2's suite; plan 1's `SqlStore`, `Statement`, `DbValue`, `DbError`, `Capabilities`,
  `UpsertForm`, `parse_one`, `Query`, `EngineRegistry`, `MySqlEngineFactory`.
- Produces: `pub struct SqlCache` with `SqlCache::new(store: Rc<dyn SqlStore>, table: &str) -> Result<SqlCache,
  CacheError>` (parses its statement templates once), `install(&self) -> Result<(), CacheError>`,
  `purge_expired(&self) -> Result<u64, CacheError>` (for plan 8's scheduling), and `impl CachingStore` (`name()` is
  `sql:<engine>`). `pub fn stored_key(text: &str) -> String`.
- The table (pinned; `table` is the caller's full name, e.g. `wp_speedyspec_cache`):
  ```sql
  CREATE TABLE IF NOT EXISTS <table> (
    cache_group varchar(191) NOT NULL,
    cache_key varchar(191) NOT NULL,
    cache_value longblob NOT NULL,
    expires_at bigint NULL,
    PRIMARY KEY (cache_group, cache_key),
    KEY expires_at (expires_at)
  ) DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
  ```
- Rules:
  - `stored_key`: a group or key over 191 characters is stored as `sha256:` plus the lowercase hex SHA-256 of its
    UTF-8 bytes; others as is. `utf8mb4_bin` keeps `a` and `A` distinct, as WordPress's array keys are.
  - `expires_at` is Unix seconds: `now + ceil(expiry)`; `NULL` for none. "Live" is `expires_at IS NULL OR
    expires_at > now`, with `now` from `SystemTime` passed as a parameter (never the engine's clock function).
  - Every statement is a parsed template with `?` placeholders bound through `Statement::new(tree.clone(), params)`,
    or built with `Query` (`get_many`'s `where_in`). Nothing else reaches the store.
  - `get`/`get_many` select `cache_key, cache_value, expires_at` and drop expired rows in Rust; a `DbValue::Text`
    value is its UTF-8 bytes.
  - `set`: `Query::insert_into(..).values(..).on_duplicate_update(&["cache_value", "expires_at"])` when
    `capabilities().upsert != UpsertForm::None`; otherwise `DELETE` then `INSERT` in one transaction.
  - `add`: delete that key's expired row, then `INSERT`; `DbError::Constraint` means `false`.
  - `replace`: a live-row `SELECT`, then `set`; true when the row was live (changed-row counts are never read).
  - `increment`/`decrement`: `SELECT .. FOR UPDATE` and `UPDATE` inside `begin`/`commit` when the store has
    transactions and none is open; without, the same two statements.
  - `flush`: `DELETE FROM <table>` (not `TRUNCATE`, which commits implicitly). `flush_group`: `DELETE .. WHERE
    cache_group = ?`.
  - `DbError` maps to `CacheError::Backend { cache: name(), message: error.to_string() }`.

- [ ] **Step 1: Write the failing tests.** `tests/statements.rs` (a recording fake `SqlStore` reporting
  `UpsertForm::OnDuplicateKey`): `set_emits_upsert` (the executed statement's text contains `ON DUPLICATE KEY UPDATE
  cache_value = VALUES(cache_value)` and its params are `[Text("g"), Text("k"), Bytes(b"v"), Null]`);
  `set_without_upsert_deletes_then_inserts` (fake reports `UpsertForm::None`: log is `begin, DELETE.., INSERT..,
  commit`); `stored_key_hashes_long_keys` (`stored_key(&"a".repeat(191))` is unchanged; a 192-character key becomes
  `sha256:` + 64 hex characters). `tests/conformance.rs` (ignored without MySQL): `sql_cache_conformance` (store from
  `SPEEDYSPEC_TEST_MYSQL_URL` through `MySqlEngineFactory` and an `EngineRegistry`; `SqlCache::new(store,
  "speedyspec_cache_conformance")`, `install()` twice, then `caching_store_suite`); `purge_removes_only_expired`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-cache-sql` and, with MySQL up, `cargo test -p speedyspec-cache-sql
  -- --ignored` — Expected: FAIL.
- [ ] **Step 3: Implement** `SqlCache` per the rules.
- [ ] **Step 4: Run** both commands — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-cache-sql && git commit -m "feat(cache-sql): a Caching Store over any SQL Store"`

### Task 4: PHP's serialize format (`speedyspec-php-serialize`)

**Files:**
- Create: `crates/speedyspec-php-serialize/Cargo.toml`, `src/lib.rs`, `src/value.rs`, `src/write.rs`,
  `src/read.rs`, `src/float.rs`
- Test: `crates/speedyspec-php-serialize/tests/format.rs`

**Interfaces:**
- Produces:
  - `pub enum PhpKey { Int(i64), Str(Vec<u8>) }`.
  - `pub enum PhpValue { Null, Bool(bool), Int(i64), Float(f64), Str(Vec<u8>), Array(Vec<(PhpKey, PhpValue)>),
    Object { class: String, properties: Vec<(PhpKey, PhpValue)> }, Custom { class: String, data: Vec<u8> },
    Enum { class: String, case: String }, Ref(u32), ObjectRef(u32) }` (`Custom` is `C:`, a `Serializable`'s payload;
    `Ref` is `R:n`, a PHP reference; `ObjectRef` is `r:n`, the same object again), with `PartialEq` (floats
    bitwise) and `contains_object(&self) -> bool` (`Object`, `Custom`, `Enum` or `ObjectRef` anywhere).
  - `pub fn serialize(value: &PhpValue) -> Vec<u8>`.
  - `pub struct UnserializeError { pub offset: usize, pub length: usize }`, `Display` as PHP's notice: `Error at
    offset {offset} of {length} bytes`.
  - `pub fn unserialize(bytes: &[u8]) -> Result<PhpValue, UnserializeError>`: bytes after the first complete value
    are ignored (PHP returns the value); nesting deeper than 4096 is an error (`unserialize_max_depth`); a declared
    length or count beyond the input is an error before any allocation; in arrays a string key that is a canonical
    decimal `i64` becomes `PhpKey::Int` (PHP's symbol-table rule), in objects an integer key becomes `PhpKey::Str`;
    `b:` accepts only `0` and `1`.
- Floats (`float.rs`) are written in PHP's `serialize_precision = -1` form: the shortest round-trip digits, `INF`,
  `-INF`, `NAN`, `-0`; exponent form `<m>E<sign><exp>` with at least one fraction digit (`1.0E+25`) when the decimal
  exponent is below -4 or at least 15. Task 9's differential test against PHP is the authority; fix this file, not
  the test, if they disagree.

- [ ] **Step 1: Write the failing tests:** `scalars` (`Null` → `N;`, `Bool(true)` → `b:1;`, `Int(-5)` → `i:-5;`,
  `Str(b"h\"\xffi")` → `s:4:"h"\xffi";`); `floats` (`0.1` → `d:0.1;`, `1.0` → `d:1;`, `-0.0` → `d:-0;`, `1e14` →
  `d:100000000000000;`, `1e15` → `d:1.0E+15;`, `0.0001` → `d:0.0001;`, `0.00001` → `d:1.0E-5;`, `1.5e300` →
  `d:1.5E+300;`, `f64::INFINITY` → `d:INF;`, `f64::NAN` → `d:NAN;`); `nested_array`
  (`a:2:{i:0;s:1:"a";s:1:"k";a:1:{i:5;b:0;}}` round-trips exactly both ways); `objects`
  (`O:8:"stdClass":1:{s:3:"foo";s:3:"bar";}`, a property named `"\0Foo\0bar"` (private), `C:3:"Foo":5:{hello}`,
  `E:11:"Suit:Hearts";` each round-trip and give `contains_object() == true`); `references`
  (`a:2:{i:0;i:1;i:1;R:2;}` gives `Array[(Int 0, Int 1), (Int 1, Ref(2))]` and round-trips;
  `a:2:{i:0;O:8:"stdClass":0:{}i:1;r:2;}` round-trips); `key_normalisation` (`a:1:{s:1:"5";i:1;}` → key `Int(5)`;
  `"05"` and `"-0"` stay strings; in an object, `i:1` → `Str(b"1")`); `errors` (`s:5:"ab";` → `Err { offset, length:
  8 }` with `offset > 0`; `s:999999999:"x";` errors without allocating; `b:2;`, `i:;`, `""` and 4097 nested
  `a:1:{i:0;` all `Err`); `trailing_bytes_ignored` (`i:1;junk` → `Int(1)`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-php-serialize` — Expected: FAIL.
- [ ] **Step 3: Implement** a hand-written byte reader and writer (no regex, no serde).
- [ ] **Step 4: Run** `cargo test -p speedyspec-php-serialize` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-php-serialize && git commit -m "feat(php-serialize): PHP's serialize and unserialize"`

### Task 5: `maybe_serialize`, `is_serialized` and conversions to the Hook API's `Value`

**Files:**
- Create: `crates/speedyspec-php-serialize/src/maybe.rs`, `src/convert.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-php-serialize/tests/maybe.rs`, `tests/convert.rs`

**Interfaces:**
- Consumes: Task 4; `speedyspec_hooks::{Value, Key}`.
- Produces:
  - `pub fn is_serialized(data: &[u8], strict: bool) -> bool`, `pub fn maybe_serialize(value: &PhpValue) ->
    PhpValue`, `pub fn maybe_unserialize(value: &PhpValue) -> PhpValue`: WordPress's `functions.php` rules exactly
    (trim of `" \t\n\r\0\x0B"`; `N;`; the length-4 and `data[1] == ':'` checks; strict and non-strict `;`/`}`
    checks; tokens `s a O E b i d`; non-strict `is_serialized` strings are serialized again, the double
    serialization). `maybe_unserialize` of a string that fails to unserialize gives `Bool(false)` (`@unserialize`).
  - `pub struct OpaquePhpValue(pub PhpValue)`: what Rust cannot represent, carried in `Value::Opaque`.
  - `pub fn to_hook_value(value: &PhpValue) -> Value` (infallible): `Str` → `Value::Str` when UTF-8, else `Bytes`;
    `Array` → `Value::Array`; `Object`, `Custom`, `Enum` → `Value::Opaque(Rc<OpaquePhpValue>)` of that value, the
    same `Rc` for every `r:` to it; `Ref(n)` → a copy of slot `n`. Slots are numbered as PHP's `var_hash` numbers
    them: from 1 for the top value, in order, each array, object and element value one slot, keys none (Task 9's
    differential test pins it). The whole value becomes one `OpaquePhpValue` when a reference points at an ancestor
    (a cycle) or an array key is not UTF-8.
  - `pub enum ConvertError { Foreign }` (`Display`, `Error`); `pub fn from_hook_value(value: &Value) ->
    Result<PhpValue, ConvertError>`: reads the value first (`Value::read`, converting a bridge's readable opaques);
    `Bytes` → `Str`; an `Opaque` holding an `OpaquePhpValue` → its value; any other `Opaque` → `Foreign`.

- [ ] **Step 1: Write the failing tests.** `tests/maybe.rs`, cases from WordPress's own `is_serialized` tests:
  strict true for `N;`, `b:0;`, `i:5;`, `d:0.1;`, `s:0:"";`, `a:0:{}`, `O:8:"stdClass":0:{}`, `E:7:"Foo:Bar";`,
  `"  i:5;  "`; strict false for `"N"`, `"i:5"`, `"s:4:test;"`, `"a:0:"`, `"x:1;"`; `"s:3:\"abc"` false strict,
  true non-strict. `maybe_serialize`: `Array[]` → `Str(b"a:0:{}")`; `Str(b"i:5;")` → `Str(b"s:4:\"i:5;\";")`
  (double serialization); `Str(b"plain")` and `Int(5)` unchanged. `maybe_unserialize`: `Str(b"a:1:{i:0;i:1;}")` →
  the array; `Str(b"s:5:\"ab\";")` → `Bool(false)`; `Str(b"plain")` unchanged. `tests/convert.rs`:
  `scalars_and_arrays_round_trip` (`Value::Array([(Str "a", Int 1), (Int 0, Str "x")])` → `PhpValue` → same
  `Value`); `objects_become_opaque_and_return` (`O:8:"stdClass":0:{}` → `Opaque`, back to the same `PhpValue`);
  `object_ref_shares_rc` (`a:2:{i:0;O:8:"stdClass":0:{}i:1;r:2;}` → both elements `Rc::ptr_eq`);
  `value_ref_is_copied` (`a:2:{i:0;i:1;i:1;R:2;}` → `[1, 1]`); `foreign_opaque_refused`
  (`Value::Opaque(Rc::new(5u8))` → `Err(Foreign)`); `invalid_utf8_string_is_bytes`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-php-serialize` — Expected: FAIL.
- [ ] **Step 3: Implement** `maybe.rs` and `convert.rs`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-php-serialize` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(php-serialize): maybe_serialize, is_serialized and Value conversions"` (add new files first)

### Task 6: `ObjectCache`'s first level (`speedyspec-wp-object-cache`)

**Files:**
- Create: `crates/speedyspec-wp-object-cache/Cargo.toml`, `src/lib.rs`, `src/cache.rs`, `src/keys.rs`,
  `src/numeric.rs`, `src/ports.rs`, `src/error.rs`
- Test: `crates/speedyspec-wp-object-cache/tests/first_level.rs`

**Interfaces:**
- Consumes: `speedyspec_hooks::{Value, Key}`; Task 1's `CachingStore` (held, unused until Task 7).
- Produces:
  - Ports (`ports.rs`), each with its Rust adapter here: `pub trait SiteContext { fn is_multisite(&self) -> bool; fn
    current_blog_id(&self) -> i64; }` (`SingleSite`: false, 1); `pub trait CacheNotices { fn doing_it_wrong(&self,
    function: &str, message: &str, version: &str); fn deprecated_function(&self, function: &str, version: &str,
    replacement: &str); fn persistence_miss(&self, group: &str, key: &str, why: &str); }` (`LoggedNotices`:
    `eprintln!("speedyspec_wordpress: ..")`); `pub trait AdditionSuspension { fn suspended(&self) -> bool; }`
    (`NeverSuspended`); `pub enum CodecError { Refused(String), Corrupt(String) }`; `pub trait ValueCodec { fn
    encode(&self, value: &Value) -> Result<Vec<u8>, CodecError>; fn decode(&self, bytes: &[u8]) -> Result<Value,
    CodecError>; }` (`SerializeCodec`, Task 7); `pub struct ObjectCachePorts { pub site: Rc<dyn SiteContext>, pub
    notices: Rc<dyn CacheNotices>, pub suspension: Rc<dyn AdditionSuspension>, pub codec: Rc<dyn ValueCodec> }` with
    `ObjectCachePorts::rust() -> ObjectCachePorts`.
  - `pub enum ObjectCacheError { IllegalOffset(String), Unsupported(String) }` (`Display`, `Error`).
  - `pub fn array_key(value: &Value) -> Result<Key, ObjectCacheError>` (PHP's array-key cast: canonical decimal
    strings to `Int`, `Bool` to 0/1, `Null` to `""`, finite `Float` truncated; an array or object is
    `IllegalOffset(<type>)`; non-UTF-8 `Bytes` is `Unsupported`).
  - `pub enum CacheValue { Php(Value), Bytes(Vec<u8>) }` (a first-level entry: a PHP-shaped value, or second-level
    bytes not yet decoded, Task 7).
  - `pub struct ObjectCache` with `ObjectCache::new(ports: ObjectCachePorts, persistent: Option<Rc<dyn
    CachingStore>>) -> ObjectCache` and, mirroring `WP_Object_Cache`'s methods (`key` and `group` are PHP-shaped,
    so validation and casts are WordPress's):
    `add(&self, key: &Value, data: Value, group: &Value, expire: i64) -> Result<bool, ObjectCacheError>`,
    `replace(..same..)`, `set(..same..)`,
    `add_multiple(&self, data: Vec<(Key, Value)>, group: &Value, expire: i64) -> Result<Vec<(Key, bool)>,
    ObjectCacheError>`, `set_multiple(..same..)`,
    `get(&self, key: &Value, group: &Value, force: bool) -> Result<Option<Value>, ObjectCacheError>` (`None` is a
    miss: PHP's `false` with `$found = false`),
    `get_multiple(&self, keys: &[Value], group: &Value, force: bool) -> Result<Vec<(Key, Value)>, ObjectCacheError>`
    (a miss is `Value::Bool(false)`),
    `delete(&self, key: &Value, group: &Value) -> Result<bool, ObjectCacheError>`,
    `delete_multiple(&self, keys: &[Value], group: &Value) -> Result<Vec<(Key, bool)>, ObjectCacheError>`,
    `incr(&self, key: &Value, offset: i64, group: &Value) -> Result<Option<Value>, ObjectCacheError>`, `decr(..)`,
    `flush(&self) -> bool`, `flush_runtime(&self) -> bool`, `flush_group(&self, group: &Value) -> Result<bool,
    ObjectCacheError>`, `add_global_groups(&self, groups: &[Value])`, `add_non_persistent_groups(&self, groups:
    &[Value])`, `switch_to_blog(&self, blog_id: i64)`, `reset(&self)`, `pub fn supports(feature: &str) -> bool`;
    the properties `hits(&self) -> i64`, `misses(&self) -> i64`, `set_hits(i64)`, `set_misses(i64)`,
    `global_groups(&self) -> Vec<Key>`, `blog_prefix(&self) -> String`, `set_blog_prefix(&self, prefix: String)`,
    `is_multisite(&self) -> bool`, `set_multisite(&self, on: bool)`, `runtime_entries(&self) -> Vec<(Key, Vec<(Key,
    Value)>)>` (the `$cache` property, decoding any `Bytes` entry), `set_runtime_entries(&self, entries: Vec<(Key,
    Vec<(Key, Value)>)>)`.
- Rules (`class-wp-object-cache.php` and `cache.php`, literally):
  - The constructor reads `site` once: `blog_prefix` is `"<id>:"` on multisite, else `""`.
  - `add` checks `suspension.suspended()` first (false), then the key, then existence.
  - A valid key is an `Int`, or a `Str` whose trim is non-empty. Otherwise `doing_it_wrong("WP_Object_Cache::<m>",
    msg, "6.1.0")` with `<m>` the `WP_Object_Cache` method that checked it (`add_multiple` reports `add`,
    `get_multiple` reports `get`), `msg` `Cache key must not be an empty string.` for strings, else `Cache key must
    be an integer or a non-empty string, <type> given.` with PHP's `gettype` names (`NULL`, `boolean`, `double`,
    `array`, `object`), and the call returns false (`get_multiple` and the other `_multiple`s put false under that
    key).
  - An empty group (`""`, `"0"`, `Null`, `false`, `0`, `0.0`, empty array) is `default`; `flush_group` does not apply
    this rule. On multisite a group not in the global groups prefixes the id with `blog_prefix`.
  - `incr`/`decr`: absent is `None`; a value that is not numeric (PHP's `is_numeric`, in `numeric.rs`: leading
    whitespace, optional sign, digits, decimal point, exponent, trailing whitespace) becomes `Int(0)`; `Int`
    arithmetic that overflows gives `Float`, as PHP; a numeric string with `.` or an exponent gives `Float`; a result
    below 0 becomes `Int(0)`; the stored value is the result.
  - `supports` is true for exactly `add_multiple`, `set_multiple`, `get_multiple`, `delete_multiple`,
    `flush_runtime`, `flush_group`.
  - `switch_to_blog` sets `blog_prefix` to `"<id>:"` on multisite, else `""`. `reset` calls
    `deprecated_function("reset", "3.5.0", "WP_Object_Cache::switch_to_blog()")` and empties every non-global group.
  - `get` counts a hit or a miss. Objects are cloned at the PHP boundary (Task 9), not here: Rust values are
    immutable.

- [ ] **Step 1: Write the failing tests** (a recording `CacheNotices` and a `SiteContext` double in the test file):
  `add_get_set_replace_delete` (WordPress's `test_add`, `test_replace`, `test_set`, `test_delete` sequences);
  `stores_false_null_zero` (`set("k", Bool(false))` then `get == Some(Bool(false))`); `numeric_string_key_is_int_key`
  (`set(Str "5", ..)` then `get(Int 5)` finds it; `"05"`, `" 5"`, `"-0"` do not); `invalid_keys` (keys `""`, `" "`,
  `Null`, `Float(1.5)`, `Bool(true)`, `Array[]`: false, and the notices recorded with the exact messages above and
  function `WP_Object_Cache::add`); `empty_group_is_default`; `incr_decr` (`5` +1 → `Int(6)`; `"x"` +1 → `Int(1)`;
  `2` decr 5 → `Int(0)`; `"1.5"` +1 → `Float(2.5)`; `i64::MAX` +1 → `Float(9.223372036854775808e18)`; absent →
  `None`); `multiple_ops` (`add_multiple`, `set_multiple`, `get_multiple` with a missing key, `delete_multiple`, keys
  and order as WordPress returns them); `flush_and_flush_group` (`flush_group("")` leaves `default`);
  `multisite_prefix` (site multisite, blog 1: `set("k")` in `posts`; `switch_to_blog(2)` misses it; global group
  `users` is seen from both); `single_site_ignores_switch_to_blog`; `reset_keeps_global_groups_and_notes_deprecation`;
  `suspended_addition_refuses_add`; `hit_and_miss_counts`; `supports_exact_list`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-object-cache` — Expected: FAIL.
- [ ] **Step 3: Implement** the first level: `RefCell<IndexMap<Key, IndexMap<Key, CacheValue>>>` (insertion order,
  as PHP's arrays), `Cell` counters, the global and non-persistent group sets.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-object-cache` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-wp-object-cache && git commit -m "feat(wp-object-cache): WP_Object_Cache's first level"`

### Task 7: The second level

**Files:**
- Create: `crates/speedyspec-wp-object-cache/src/persistent.rs`, `src/codec.rs`; modify `src/cache.rs`,
  `src/lib.rs`, `Cargo.toml` (`speedyspec-php-serialize`; dev: `speedyspec-cache-memory`)
- Test: `crates/speedyspec-wp-object-cache/tests/second_level.rs`

**Interfaces:**
- Consumes: Tasks 1, 5, 6.
- Produces: `pub struct SerializeCodec` (`impl ValueCodec`: `from_hook_value` then `serialize`; `Foreign` is
  `Refused("an object Rust cannot serialize")`; `decode` is `unserialize` then `to_hook_value`, an error is
  `Corrupt(<UnserializeError>)`), used by `ObjectCachePorts::rust()`.
- Rules, when `persistent` is `Some` and the group is not non-persistent (otherwise Task 6's behaviour is unchanged):
  - The second level's group is the group's key text and its key is the id's text (`Int` in decimal), so the blog
    prefix is part of it (`"2:k"`); expiry is `Some(Duration::from_secs(expire))` when `expire > 0`, else `None`.
  - `set` writes both levels; a `CodecError::Refused` keeps the value in the first level only and calls
    `persistence_miss(group, key, why)`; `set` still returns true.
  - `get`: a first-level hit returns it unless `force`; otherwise the second level is read, a hit is stored in the
    first level as `CacheValue::Bytes` and decoded when returned (replacing it with `Php`), and counts as a hit.
    `get_multiple` reads every first-level miss with one `get_many`.
  - `add`: false if the first level has it, else the second level's `add` decides; `replace`: true if either level has
    it, then `set`. `delete` removes from both, true if either had it. `incr`/`decr` read through `get`, apply Task 6's
    rule, then `set`: read-modify-write, not atomic across workers, since the value is PHP-serialized.
  - `flush` empties both; `flush_runtime` only the first; `flush_group` both (the second with `flush_group`).
  - A `CacheError` from the second level, or a `CodecError::Corrupt` on decode, is a `persistence_miss` and is
    treated as a miss (or, for a write, as first-level only); it never reaches the caller.

- [ ] **Step 1: Write the failing tests** (`MemoryCache` as the second level, plus a failing `CachingStore` double):
  `writes_php_serialize_format` (`set("k", Array[(Int 0, Str "a")], "g", 0)` → second level `get("g", "k") ==
  Some(b"a:1:{i:0;s:1:\"a\";}")`); `reads_second_level_after_flush_runtime` (value returned, `hits() == 1`);
  `non_persistent_groups_stay_first_level`; `refused_value_stays_first_level` (`Value::Opaque(Rc::new(5u8))`: first
  level returns the same `Rc`, second level empty, one `persistence_miss` naming `g` and `k`);
  `blog_prefix_in_second_level_key`; `add_sees_another_worker` (entry written straight into the shared
  `MemoryCache` → `add` false); `delete_flush_group_flush_reach_second_level`; `expire_becomes_duration` (a recording
  double sees `Some(30s)` for 30 and `None` for 0 and -1); `incr_writes_serialized` (second level holds `b"i:6;"`);
  `force_rereads_second_level`; `corrupt_second_level_is_a_miss` (`b"a:1:{i:0;"` in the store: `get` is `None`, one
  notice); `failing_store_is_a_miss`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-object-cache --test second_level` — Expected: FAIL.
- [ ] **Step 3: Implement** `persistent.rs` (the Decorator's second-level calls) and `SerializeCodec`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-object-cache` — Expected: PASS (both test files).
- [ ] **Step 5: Commit** — `git commit -am "feat(wp-object-cache): the persistent second level in PHP's format"` (add new files first)

### Task 8: Use cases on the buses

**Files:**
- Create: `crates/speedyspec-wp-object-cache/src/current.rs`, `src/messages.rs`; modify `src/lib.rs`, `Cargo.toml`
  (`speedyspec-cqrs`, `speedyspec-container`)
- Test: `crates/speedyspec-wp-object-cache/tests/messages.rs`

**Interfaces:**
- Consumes: Tasks 6 and 7; `speedyspec_cqrs::{Command, Query, CommandHandler, QueryHandler, bind_command_handler,
  bind_query_handler, CommandBus, QueryBus}`, `speedyspec_container::Container`.
- Produces:
  - `pub struct CurrentObjectCache` with `new(ports: ObjectCachePorts, persistent: Option<Rc<dyn CachingStore>>) ->
    CurrentObjectCache` (holds a fresh current cache), `create(&self) -> Rc<ObjectCache>` (a new cache sharing the
    ports and second level), `current(&self) -> Rc<ObjectCache>`, `select(&self, cache: Rc<ObjectCache>)`,
    `end_request(&self)` (drops every cache it holds and starts a fresh current one). It is the handler of every
    message, delegating to `current()`.
  - Commands (fields as the `ObjectCache` method's arguments; `Result` as its return): `CacheAdd { key, data, group,
    expire }`, `CacheAddMultiple { data, group, expire }`, `CacheReplace`, `CacheSet`, `CacheSetMultiple`,
    `CacheDelete { key, group }`, `CacheDeleteMultiple { keys, group }`, `CacheIncr { key, offset, group }`,
    `CacheDecr`, `CacheFlush`, `CacheFlushRuntime`, `CacheFlushGroup { group }`, `CacheAddGlobalGroups { groups }`,
    `CacheAddNonPersistentGroups { groups }`, `CacheSwitchToBlog { blog_id }`, `CacheReset`. Queries: `CacheGet {
    key, group, force }`, `CacheGetMultiple { keys, group, force }`, `CacheSupports { feature: String }` (`Result =
    bool`). Keys `SpeedySpec\WpObjectCache\<Name>Handler`, e.g. `SpeedySpec\WpObjectCache\CacheGetHandler`.
  - `pub fn register_object_cache_handlers(container: &Container, current: Rc<CurrentObjectCache>)`.

- [ ] **Step 1: Write the failing tests:** `bus_round_trip` (register, then `CommandBus::dispatch(CacheSet { .. })`
  and `QueryBus::ask(CacheGet { .. })` return the value); `select_switches_target` (a second cache from `create()`,
  selected: the bus reads it, the first keeps its own entries); `end_request_drops_values` (after `end_request` the
  value held by an `Rc` in the cache has a strong count of 1); `override_by_key` (binding another
  `CommandHandler<CacheSet>` under the same key replaces the default, as in the Hook API).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-object-cache --test messages` — Expected: FAIL.
- [ ] **Step 3: Implement** the messages and handlers in the shape of `speedyspec-wp-hooks/src/messages.rs`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-object-cache` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(wp-object-cache): the object cache's commands and queries"` (add new files first)

### Task 9: The native PHP surface (`speedyspec-wp-object-cache-php`)

**Files:**
- Create: `crates/speedyspec-wp-object-cache-php/Cargo.toml`, `src/lib.rs`, `src/wp-object-cache.php` (the facade),
  `src/ports.rs`, `src/registry.rs`
- Test: `crates/speedyspec-wp-object-cache-php/tests/php.rs` (`harness = false`)

**Interfaces:**
- Consumes: Tasks 5–8; `speedyspec_php::{NativeArea, native, global, object_id, call, eval}`,
  `speedyspec_cqrs::{HandlerRef, QueryRef}`.
- Produces: `pub struct ObjectCacheArea` with `ObjectCacheArea::new(container: Rc<Container>, persistent:
  Option<Rc<dyn CachingStore>>) -> ObjectCacheArea` and `api(&self) -> Rc<CurrentObjectCache>` (for plan 4) (builds `CurrentObjectCache` with the PHP ports and registers
  the handlers), `api(&self) -> Rc<CurrentObjectCache>`, `impl NativeArea`; `pub fn
  object_cache_scenario_runner(php: Php) -> speedyspec_compat::PhpRunner` (Task 10); PHP adapters `PhpSiteContext`
  (`is_multisite()` and `get_current_blog_id()` when defined, else false and 1), `PhpCacheNotices` (`__()` then
  `_doing_it_wrong()` / `_deprecated_function()` when defined; `persistence_miss` logs), `PhpAdditionSuspension`
  (`wp_suspend_cache_addition()` when defined), `PhpSerializeCodec` (`SerializeCodec` first; on `Refused`, PHP's
  `serialize()` on the pass-through value, a thrown exception being `Refused(<message>)`; on decode, bytes whose
  `PhpValue::contains_object()` go to PHP's `unserialize()`).
- Rules:
  - `request_start` evaluates the facade (`speedyspec://wp-object-cache`), which declares, with WordPress's exact
    signatures and defaults, every function in `cache.php` (`wp_cache_init`, `wp_cache_add` … `wp_cache_reset`,
    `wp_cache_close`) and the class `WP_Object_Cache` (`#[AllowDynamicProperties]`, not final, no declared
    properties). `cache-compat.php`'s salted functions are not declared; WordPress's `cache-compat.php` keeps them.
  - Each function: if `get_class($wp_object_cache) !== 'WP_Object_Cache'`, call the method on it, as WordPress does;
    otherwise call `Native\cache_<name>($wp_object_cache, ..)`, which selects that object's cache as current when it
    is not already, and dispatches the use case through its `HandlerRef`/`QueryRef` (resolved once).
  - Each `WP_Object_Cache` method calls `Native\object_<name>($this, ..)`, straight on that object's cache (no
    selection). `__construct` calls `Native\object_construct($this)` (the registry maps `object_id` to
    `CurrentObjectCache::create()`); `__destruct` releases it. `__get`/`__set`/`__isset`/`__unset` serve
    `cache_hits`, `cache_misses`, `cache`, `global_groups`, `blog_prefix`, `multisite` from Rust and anything else as a
    real dynamic property, as WordPress's magic methods do. `stats()` is WordPress's, in the facade, over those
    properties.
  - `wp_cache_init()` throws `Error` with `The object-cache.php drop-in (<WP_CONTENT_DIR>/object-cache.php) is not
    supported: SpeedySpec's object cache is native. Remove the file; persistence comes from configuring a Caching
    Store.` when `WP_CONTENT_DIR` is defined and that file exists; otherwise `$GLOBALS['wp_object_cache'] = new
    WP_Object_Cache();` and selects it.
  - `get` returns `$found` through `Native\found()` after the call. Set and add clone a top-level object before
    storing; get clones a top-level object before returning (WordPress's shallow clones). Data arguments pass through
    unconverted (`Args::pass`); keys and groups are converted. Values returned to PHP that hold an `OpaquePhpValue`
    are materialised with PHP's `unserialize()`. `ObjectCacheError::IllegalOffset` throws `TypeError` and
    `Unsupported` throws `Error`.
  - `request_end` clears the registry and calls `end_request()`, before PHP's request shutdown.

- [ ] **Step 1: Write the failing tests** in `tests/php.rs` (areas `twins_area()`, `ContainerArea`,
  `ObjectCacheArea::new(container, None)`): `functions_declared` (every `cache.php` function exists;
  `wp_cache_init()` makes `$wp_object_cache` a `WP_Object_Cache`); `object_refs` (WordPress's `test_object_refs` in
  PHP); `found_by_ref`; `instances_are_separate` (`new WP_Object_Cache` does not see the global's entries;
  `assertEquals`-style `==` between two fresh instances is true); `subclass_method_called` (a PHP subclass overriding
  `get` assigned to `$wp_object_cache`: `wp_cache_get` calls the override); `rust_caller_sees_php_set` (PHP
  `wp_cache_set('k', 'v', 'g')`, then `QueryBus::ask(CacheGet)` from Rust gives `Str("v")`); `drop_in_refused` (with
  `WP_CONTENT_DIR` at a temporary directory holding `object-cache.php`, `wp_cache_init()` throws an `Error` whose
  message contains the file's path); `no_boundary_crossing_for_get` (`boundary_crossings()` unchanged across 100
  `wp_cache_get` calls); `values_released_at_request_end` (set an object and an array, restart the request, `get` is
  false, no crash); `closure_refused_by_second_level` (a second `ObjectCacheArea` with `MemoryCache`: a closure set
  stays readable in the request, the second level is empty, a `persistence_miss` is logged);
  `serialize_matches_php` (the differential test: for a corpus of integers, the floats of Task 4 plus 1000
  fixed-seed `f64::from_bits` values, strings with NUL, quotes, multibyte and invalid UTF-8, nested arrays with
  numeric-string keys, a reference pair, `stdClass`, a class with private and protected properties, `ArrayObject`, a
  `Serializable` class and an enum: Rust's `serialize` of the converted value equals PHP's `serialize()`, and Rust's
  `unserialize` of PHP's bytes re-serializes to the same bytes).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-object-cache-php --test php` — Expected: FAIL.
- [ ] **Step 3: Implement** the area, facade, registry and PHP adapters per the rules, in the shape of
  `speedyspec-wp-hooks-php` (`STATE` thread-local, `Handlers` resolved once, `native::function`/`raw_function`).
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-object-cache-php --test php` — Expected: PASS, every case `ok`.
  A `serialize_matches_php` mismatch is fixed in `speedyspec-php-serialize`.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-wp-object-cache-php && git commit -m "feat(wp-object-cache-php): native wp_cache_* functions and WP_Object_Cache"`

### Task 10: Recorded compatibility (`compat/object-cache`)

**Files:**
- Create: `crates/speedyspec-compat/src/object_cache_stubs.php`, `compat/object-cache/scenarios.json`,
  `compat/object-cache/source.json` (a copy of `compat/hooks/source.json`), `compat/object-cache/wordpress-7.1.5.json`
  (recorded), `compat/object-cache/breaks.json` (empty `breaks`)
- Modify: `crates/speedyspec-compat/src/record.rs` (`setup_for`), `src/lib.rs`
- Test: `crates/speedyspec-wp-object-cache-php/tests/php.rs` (one more case)

**Interfaces:**
- Consumes: Task 9's `object_cache_scenario_runner`; `speedyspec_compat::{record, replay, PhpRunner}`.
- Produces: `pub const OBJECT_CACHE_STUBS: &str` in `speedyspec-compat`: PHP declaring `is_multisite()` and
  `get_current_blog_id()` from the globals `speedyspec_compat_multisite` (default false) and
  `speedyspec_compat_blog_id` (default 1); `__()` returning its text; `_doing_it_wrong()` and `_deprecated_function()`
  appending an invocation (`fixture` is the function name, `args` normalised) as the driver's `_deprecated_hook`
  does; WordPress's `wp_suspend_cache_addition()`; `wp_using_ext_object_cache()` returning false;
  `speedyspec_compat_cache_get_found($key, $group = '', $force = false)` returning `['value' => .., 'found' => ..]`;
  `speedyspec_compat_object(array $props)` (`(object) $props`), `speedyspec_compat_props(object $o)`
  (`get_object_vars`), `speedyspec_compat_prop_set(object $o, string $name, mixed $value)`,
  `speedyspec_compat_new_cache()` (`new WP_Object_Cache()`). The driver itself is unchanged, so the hooks fixture
  is unchanged.
- `setup_for("object-cache")` evaluates the stubs, requires `wp-includes/class-wp-object-cache.php` and
  `wp-includes/cache.php`, and calls `wp_cache_init()`. The port's runner evaluates the stubs and calls
  `wp_cache_init()`.
- Scenarios (one per name; each in a fresh request): `add_get`, `add_existing_is_false`, `stores_null_false_zero`,
  `replace_missing_is_false`, `set_overwrites`, `delete_existing_and_missing`, `get_miss_counts`, `found_flag`,
  `object_cloned_on_set_and_get`, `array_holding_object_shares_it`, `empty_group_is_default`,
  `zero_string_group_is_default`, `numeric_string_key_is_int_key`, `invalid_keys_doing_it_wrong` (`""`, `" "`, null,
  1.5, true, `[]`), `incr_numeric`, `incr_non_numeric_resets`, `decr_clamps_at_zero`,
  `incr_float_and_numeric_string`, `incr_int_overflow` (`{"int": "PHP_INT_MAX"}`), `incr_offset_cast`, `multiple_ops`,
  `get_multiple_with_invalid_key`, `flush`, `flush_runtime`, `flush_group_raw_group`, `supports_each_feature`,
  `non_persistent_groups_noop`, `global_groups_on_multisite`, `switch_to_blog_single_site_ignored`,
  `switch_to_blog_multisite`, `reset_is_deprecated`, `suspend_addition`, `hit_miss_properties` (`$c->__get` of
  `cache_hits` and `cache_misses` on a `speedyspec_compat_new_cache()` instance kept `as: "c"`), `dynamic_property`, `cache_property_shape`, `close_is_true`, `expire_ignored`, `instances_separate`.
  Multisite scenarios set `speedyspec_compat_multisite` and `speedyspec_compat_blog_id` with `global:replace`, then
  call `wp_cache_init`.

- [ ] **Step 1: Write** the stubs, `setup_for`, the catalogue and `source.json`.
- [ ] **Step 2: Record:** `cargo run -p speedyspec-compat -- record object-cache --wordpress 7.1.5` — Expected:
  `wrote compat/object-cache/wordpress-7.1.5.json`. Read the fixture: every `invalid_keys_doing_it_wrong` step has a
  `_doing_it_wrong` invocation, and `object_cloned_on_set_and_get` shows `alpha`.
- [ ] **Step 3: Write the failing test** `replayed_scenarios_match_wordpress` in `tests/php.rs`
  (`speedyspec_compat::replay("object-cache", &mut object_cache_scenario_runner(..))` is empty; print each mismatch).
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-object-cache-php --test php` — Expected: PASS. A mismatch is a bug
  in the port; only a difference the port cannot remove goes in `breaks.json`, with its `why`.
- [ ] **Step 5: Run** `cargo test -p speedyspec-wp-hooks-php` — Expected: PASS (the hooks catalogue is unaffected).
- [ ] **Step 6: Commit** — `git add compat/object-cache crates/speedyspec-compat crates/speedyspec-wp-object-cache-php && git commit -m "test(compat): the object-cache catalogue, recorded and replayed"`

### Task 11: WordPress's `tests/phpunit/tests/cache.php`

**Files:**
- Create: `tests/php/wordpress/cache.php` (copied unchanged from wordpress-develop's
  `tests/phpunit/tests/cache.php`), `tests/php/bootstrap-object-cache.php`, `tests/php/phpunit-object-cache.xml`,
  `crates/speedyspec-wp-object-cache-php/tests/common/mod.rs`, `tests/wordpress.rs`, `tests/wordpress_multisite.rs`
  (both `harness = false`)
- Modify: `tests/php/WP_UnitTestCase.php` (add `setExpectedIncorrectUsage()`, `flush_cache()`, and the incorrect-usage
  check in `assert_post_conditions`)

**Interfaces:**
- Consumes: Task 9's `ObjectCacheArea`; the pinned PHPUnit (`tests/php/phpunit.json`), fetched and checked as
  `speedyspec-wp-hooks-php/tests/wordpress.rs` does.
- Rules: `bootstrap-object-cache.php` requires `bootstrap.php`, then declares `is_multisite()` (true when
  `SPEEDYSPEC_TEST_MULTISITE` is `1`), `get_current_blog_id()` (1), `__()`, `_doing_it_wrong()` (recorded for
  `setExpectedIncorrectUsage`, as WordPress's test case does through `doing_it_wrong_run`), `_deprecated_function()`,
  `wp_suspend_cache_addition()`, `wp_using_ext_object_cache()` (false), and calls `wp_cache_init()`. `flush_cache()`
  is WordPress's (`wp_cache_flush_runtime()`, `wp_cache_flush()`, the global and non-persistent groups).
  `phpunit-object-cache.xml` has one suite with `wordpress/cache.php`, excludes `documented-break`, and is separate
  from `phpunit.xml` so the hooks run is unchanged. `wordpress_multisite.rs` sets `SPEEDYSPEC_TEST_MULTISITE=1`
  before PHP boots.

- [ ] **Step 1: Write the failing tests** `wordpress_cache_tests_pass` in both binaries (the junit totals: status 0,
  `tests >= 30`, no failures or errors) and `documented_breaks_are_listed` (as the hooks test does, against
  `compat/object-cache/breaks.json`'s `wordpress_tests`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-object-cache-php --test wordpress --test wordpress_multisite` —
  Expected: FAIL until the copied suite and shim are in place, then any failure is a port bug.
- [ ] **Step 3: Fix** the port (not the copied test) until both pass; a test that cannot pass is marked `@group
  documented-break` and listed with its `why`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-object-cache-php` and `cargo test -p speedyspec-wp-hooks-php` —
  Expected: PASS.
- [ ] **Step 5: Run** `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --check` — Expected:
  clean. After any Linux container run, delete that run's `target-linux*/` folder.
- [ ] **Step 6: Commit** — `git add tests/php crates/speedyspec-wp-object-cache-php && git commit -m "test(wp-object-cache-php): WordPress's cache tests against the native object cache"`

### Task 12: Hand-over note for the runtime's owner

**Files:**
- Create: `/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/object-cache-worker-request.md`
  (outside the repository; no commit)

- [ ] **Step 1: Write the note**, short, naming the branch and commit range, asking the owner of
  `crates/speedyspec-php-runtime/src/worker.rs` to:
  1. add `ObjectCacheArea::new(container.clone(), None)` to the worker's areas after `WpHooksArea` (phase 1: no
     persistent level; plan 8 passes a configured `CachingStore`);
  2. keep one `ObjectCacheArea` per worker for its life (its `request_end` releases every PHP value before PHP's
     shutdown);
  3. know that the `object-cache.php` drop-in is refused by the native `wp_cache_init()`, so no worker check is
     needed; a startup check that names the file earlier is welcome but optional;
  4. make sure WordPress's own `wp-includes/cache.php` and `class-wp-object-cache.php` are not loaded beside the
     native functions (they would redeclare them), as was done for `plugin.php`.
- [ ] **Step 2: Check** the note names `ObjectCacheArea`, the branch, and items 1–4. Expected: yes.

---

## Self-review notes

- Spec coverage: Caching Store contract and groups/expiry (Task 1); `MemoryCache` and `SqlCache` (Tasks 2, 3); store
  contract tests (Tasks 2, 3); `speedyspec-php-serialize` incl. references, floats, `O:`/`C:`/`E:`, WordPress's maybe
  rules and `Value` conversions (Tasks 4, 5); object cache semantics, two levels, refused values, non-persistent
  groups (Tasks 6, 7); CQRS (Task 8); native functions, class, global, drop-in refusal (Task 9); recorded catalogue
  (Task 10); WordPress's `cache.php` (Task 11); worker changes handed over (Task 12). Persistent-level configuration,
  the cross-process test and transients are plan 8.
- Not in scope, said here so no task is expected: retiring `packages/wp-object-cache/wp-object-cache-legacy` in
  wordpress-develop (done after this merges, in that repository, as the Hook API's legacy code was).
