# Options API port Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** WordPress's single-site options API (`get_option` and the fifteen other functions of the spec's Goals) in
Rust, with every hook unchanged, over `wp_options` through the `wpdb` port and over the ported object cache, with
native PHP functions in front of it, proven by a recorded `options` catalogue and WordPress's own
`tests/phpunit/tests/option/` suite on MySQL.

**Architecture:** `speedyspec-wp-options` holds the entity and values, the ports (`OptionRepository`, the ports to
unported code, a value codec), `AutoloadPolicy`, `OptionsApi` (the handler of every command and query) and the
unit-of-work write path. Adapters: `speedyspec-wp-options-sql` (through `WpDb::query_statement`, so `query` fires and
is counted), `speedyspec-wp-options-memory`, `speedyspec-wp-options-php` (native functions and PHP adapters). Every
function follows `option.php`'s order of checks, hooks, queries and cache operations exactly; the recorded scenarios
are the specification.

**Tech Stack:** Rust 1.99, edition 2024; `speedyspec-wp-hooks`, `speedyspec-cqrs`, `speedyspec-container`,
`speedyspec-php`, `speedyspec-compat` (existing); `speedyspec-data*` (plan 1), `speedyspec-wp-db*` (plan 2),
`speedyspec-wp-object-cache*` and `speedyspec-php-serialize` (plan 3); `tracing` for logs.

**Spec:** `docs/superpowers/specs/2026-11-19-options-api-port-design.md` (phase 2), with
`docs/superpowers/specs/2026-11-19-database-layer-design.md`. Plan index (repository, worktree, shared names):
`docs/superpowers/plans/2026-11-19-data-and-options-ports-index.md`. WordPress's original:
`packages/wp-option/wp-option-legacy/functions/option.php` in wordpress-develop (read it beside each task).

## Assumed from plans 2 and 3

**Reconciled after plans 2 and 3 were written (these win over the list below):**

- `WpDb::query_statement` returns `QueryOutcome::{Rows(Rc<Vec<WpRow>>), Affected { rows: u64, insert_id: u64 },
  Done}`; a `WpRow` is `fields: Vec<(String, Option<Vec<u8>>)>` (MySQL text semantics, `None` for NULL).
- Table names: `WpDb::table_name(&self, logical: &str) -> String` (plan 2 adds it), not `table_names()`.
- `WpDb::process_fields(&self, table: &str, data: &[(String, WpValue)], format: Option<&[String]>) ->
  Result<Vec<Field>, WpDbError>`; where `wpdb` would return false the result is `Err(WpDbError::ProcessFields(..))`.
- `WpDbArea::api(&self) -> Rc<WpDb>` exists.
- The object cache: `ObjectCacheArea::api(&self) -> Rc<CurrentObjectCache>` (plan 3 adds it); the cache in use is
  `CurrentObjectCache::current(&self) -> Rc<ObjectCache>`. `ObjectCache`'s methods take PHP-shaped keys and groups:
  `get(&self, key: &Value, group: &Value, force: bool) -> Result<Option<Value>, ObjectCacheError>`, `set`/`add(&self,
  key: &Value, data: Value, group: &Value, expire: i64) -> Result<bool, ObjectCacheError>`, `delete(&self, key:
  &Value, group: &Value) -> Result<bool, ObjectCacheError>`, `get_multiple(&self, keys: &[Value], group: &Value,
  force: bool) -> Result<Vec<(Key, Value)>, ObjectCacheError>` (a miss is `Value::Bool(false)`). Tests build one
  with `ObjectCache::new(ObjectCachePorts::rust(), None)`. `exists` is still added by this plan's Task 7.
- `speedyspec_php_serialize`: objects cross into `Value` as `Value::Opaque(Rc<OpaquePhpValue>)`. Plan 3 also has a
  `ValueCodec` port in `speedyspec-wp-object-cache`; this plan's `ValueCodec` lives in `speedyspec-wp-options` and is
  a separate trait (keep both names; they never meet in one module).


Names the index fixes are used as written. These further shapes are assumed; if the merged plan differs, adapt the
call sites and keep the behaviour:

- `WpDb::query_statement(&self, stmt: Statement) -> Result<QueryOutcome, WpDbError>`; `QueryOutcome` gives the
  selected `Rows` for reads and `rows_affected` for writes. A statement a `query` callback empties or rewrites is
  handled as `wpdb::query` handles it (an empty query fails; changed text is parsed again).
- `WpDb::suppress_errors(&self, suppress: bool) -> bool` (returns the previous setting), `WpDb::table_names(&self)
  -> &TableNames`, and `WpDb::process_fields(&self, table: &str, fields: Vec<(String, DbValue)>) ->
  Result<Option<Vec<(String, DbValue)>>, WpDbError>` (`None` where `wpdb::update`/`delete` would return false over a
  charset check). If `process_fields` is not public, Task 9 adds a public wrapper in `speedyspec-wp-db`.
- `WpDbArea` exposes `fn api(&self) -> Rc<WpDb>`; the native `wpdb` counts statements sent through
  `query_statement` in `$wpdb->num_queries`.
- `ObjectCache` (concrete, `Rc`-shared): `get(&self, key: &str, group: &str, force: bool) -> Result<Option<CacheValue>,
  CacheError>`, `get_multiple(&self, keys: &[String], group: &str, force: bool) -> Result<Vec<(String,
  Option<CacheValue>)>, CacheError>`, `set`/`add(&self, key: &str, value: CacheValue, group: &str, expire: i64) ->
  Result<bool, CacheError>`, `delete(&self, key: &str, group: &str) -> Result<bool, CacheError>`,
  `ObjectCache::in_memory() -> ObjectCache` (phase 1: `MemoryCache` only). If it has no side-effect-free
  `exists(&self, key: &str, group: &str) -> bool` (no hit or miss counted), Task 7 adds one to
  `speedyspec-wp-object-cache`. `ObjectCacheArea::api(&self) -> Rc<ObjectCache>`.
- `speedyspec_php_serialize::{maybe_serialize, maybe_unserialize, is_serialized}` work on `PhpValue`; converting a
  `PhpValue` holding an `Object` into `speedyspec_hooks::Value` gives `Value::Opaque(Rc<PhpValue>)`.
- `speedyspec-compat` has an `area` hook for per-step state (from plan 2's `wp-db` area). If it has none, Task 11 adds
  `StepResult.state: Option<serde_json::Value>` (`#[serde(default, skip_serializing_if = "Option::is_none")]`, so the
  `hooks` fixture stays byte-identical) and fills it from a probe the area's setup defines.

## Global Constraints

- Work in a new worktree of `SpeedySpec/speedyspec-server`, branched from `origin/main` after plans 2 and 3 merged,
  branch `feat/options-api-port`. New crates go under `crates/`; `members = ["crates/*"]` needs no edit.
- Each crate's `Cargo.toml`: `edition.workspace = true`, `rust-version.workspace = true`, `license.workspace = true`,
  version `0.1.0`, a one-line `description`. Contracts and implementations never share a crate; no crate named
  `-domain`, `-app`, `-infra` or `-core`.
- The entity is named `Option` (index). Never glob-import it; files that use it write `use crate::Option as
  OptionEntity;` (or `speedyspec_wp_options::Option as OptionEntity`), and `Option<T>` elsewhere is always std's.
- Names are trimmed with PHP's `trim()` set (`" \t\n\r\0\x0B"`); a name that is then empty or `"0"` (PHP's `empty()`)
  makes every function return false before any hook fires.
- Every hook's name, arguments and order exactly as `option.php` has them; every query WordPress sends is sent (one
  `query_statement` per `$wpdb` call), and no other, because `get_num_queries()` is asserted by WordPress's tests.
- Cache keys and shapes as WordPress's: group `options`; `alloptions` an array of name → stored value, `notoptions` an
  array of name → `true`, single options under their name. No second cache of options anywhere.
- Hooks fire through the Hook API; an action or filter with no callbacks goes through `HookApi::skip_if_empty`, so a
  Rust caller with no PHP callbacks never enters PHP.
- Entities never load, save or query themselves.
- Patterns used here: Repository (`OptionRepository`), Data Mapper (`OptionMapper`), Serialized LOB
  (`SerializedValue`), Unit of Work and Identity Map (the staged write path), Strategy (`AutoloadPolicy`,
  `ValueCodec`), Command/CQRS (the messages), Adapter (ports to unported code, native functions), Facade (`Options`),
  Observer (every hook).
- MySQL tests are `#[ignore = "needs MySQL: set SPEEDYSPEC_TEST_MYSQL_URL"]`, run with `-- --ignored` against the
  index's container. After Linux container test runs, delete that run's `target-linux*/` folder.

## Review Focus

- **A PHP `alloptions` callback that returns a non-array** (`false`, `null`, a string): `get_option` behaves as
  WordPress (the name is not found there; notoptions, the single cache, then the database), with no panic. Test in
  Task 5.
- **An option name of `"0"`, `"  "`, `" foo "`, an integer or `true` from PHP:** `"0"` and blank names return false
  with no hook; `" foo "` fires `pre_option_foo`; `get_option(123)` reads `"123"`. Tests in Tasks 1 and 10.
- **Stored values that are not strings:** `update_option('n', 5)` caches the integer `5` in `alloptions` (as
  WordPress caches `maybe_serialize(5)`), and an already-serialized string is serialized again. Test in Task 6.
- **A `query` callback that empties the `UPDATE`** (WordPress's own failure test): `wp_set_option_autoload_values`
  reports false for every name and leaves the cache alone. Test in Task 9.
- **An option value that is not valid UTF-8 given to `update_option`:** false and nothing written, as `wpdb::update`'s
  charset check decides; `add_option` (raw `query`) writes it. Test in Task 9.

---

### Task 1: The entity and values (`speedyspec-wp-options`)

**Files:**
- Create: `crates/speedyspec-wp-options/Cargo.toml`, `src/lib.rs`, `src/name.rs`, `src/autoload.rs`, `src/value.rs`,
  `src/entity.rs`, `src/error.rs`
- Test: `crates/speedyspec-wp-options/tests/values.rs`

**Interfaces:**
- Consumes: `speedyspec_wp_hooks::{HookValue, HookError}`, `speedyspec_data::{Identified, CommitError}`,
  `speedyspec_wp_object_cache::CacheError`.
- Produces:
  - `pub struct OptionName(String)` (`Clone, Eq, Hash, Debug, Display`): `OptionName::new(raw: &str) ->
    Option<OptionName>` (trim, then `None` for empty or `"0"`), `as_str(&self) -> &str`,
    `deprecated_replacement(&self) -> Option<&'static str>` (`blacklist_keys` → `disallowed_keys`,
    `comment_whitelist` → `comment_previously_approved`).
  - `pub enum Autoload { On, Off, Auto, AutoOn, AutoOff, Yes, No, Other(String) }` with `parse(&str) -> Autoload`,
    `as_str(&self) -> &str`; `pub const DEFAULT_AUTOLOAD_VALUES: [&str; 4] = ["yes", "on", "auto-on", "auto"];`.
  - `pub enum AutoloadRequest { Default, Bool(bool), Text(String) }` with `from_value(&HookValue) -> AutoloadRequest`
    (`Null` and every non-bool, non-string value are `Default`, since PHP 8's `switch` matches none of them).
  - `pub struct SerializedValue(Vec<u8>)`: `new(Vec<u8>)`, `as_bytes(&self) -> &[u8]`, `to_value(&self) ->
    HookValue` (`Str` when UTF-8, else `Bytes`), `for_insert(value: &HookValue) -> Result<SerializedValue,
    OptionsError>` (as `prepare('%s')` formats: `Null`/`false` → `""`, `true` → `"1"`), `for_update(value:
    &HookValue) -> Result<Option<SerializedValue>, OptionsError>` (as `wpdb::update` formats: `Null` → `None`, SQL
    `NULL`). Arrays and opaque values are `Err(Serialize)`: callers serialize first.
  - `pub fn php_string(value: &HookValue) -> Option<Vec<u8>>` (PHP's string conversion; floats with `precision=14`,
    `%.14G` rules) and `pub fn php_empty(value: &HookValue) -> bool` (PHP's `empty()`).
  - `pub struct Option { pub name: OptionName, pub value: SerializedValue, pub autoload: Autoload }` with
    `impl Identified` (`Id = OptionName`). `option_id` is not a field.
  - `pub enum OptionsError { Hook(HookError), Cache(CacheError), Db(String), Serialize(String), Protected(String),
    NeedsPhp { function: &'static str, option: String }, Php(HookValue), Commit(CommitError) }` with `Display`,
    `Error` and `From` for the wrapped errors. `Php` carries a PHP exception (an opaque value) to rethrow.

- [ ] **Step 1: Write the failing tests:** `name_trims_and_rejects_empty` (`" foo\n"` → `"foo"`; `""`, `"   "`,
  `"0"`, `" 0 "` → `None`; `"00"` → `Some`); `deprecated_names_map`; `autoload_round_trips` (each of `on off auto
  auto-on auto-off yes no weird` parses and prints back); `autoload_request_from_value` (`Null` → `Default`,
  `Bool(false)` → `Bool(false)`, `Int(1)` → `Default`, `Str("no")` → `Text("no")`); `php_string_matches_php`
  (`Float(0.1)` → `"0.1"`, `Float(1.0)` → `"1"`, `Float(0.1 + 0.2)` → `"0.3"`, `Float(1e20)` → `"1.0E+20"`,
  `Float(-0.0)` → `"-0"`, `Float(f64::INFINITY)` → `"INF"`, `Float(f64::NAN)` → `"NAN"`, `Bool(true)` → `"1"`,
  `Bool(false)` → `""`, `Int(-7)` → `"-7"`, `Array(..)` → `None`); `php_empty_matches_php` (`""`, `"0"`, `0`, `0.0`,
  `false`, `Null`, empty array are empty; `"0.0"`, `" "`, `"a"` are not); `serialized_value_formats`
  (`for_insert(Null)` is `""`, `for_update(Null)` is `None`, `for_update(Int(5))` is `"5"`, `to_value` of bytes
  `[0xff]` is `Bytes`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options --test values` — Expected: FAIL (crate missing).
- [ ] **Step 3: Implement** the items above.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options --test values` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-wp-options && git commit -m "feat(wp-options): the option entity and its values"`

### Task 2: Ports and their Rust adapters

**Files:**
- Create: `crates/speedyspec-wp-options/src/ports.rs`, `src/adapters.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-wp-options/tests/adapters.rs`

**Interfaces:**
- Consumes: Task 1; `speedyspec_wp_hooks::HookApi`; `speedyspec_php_serialize`.
- Produces (object-safe, `&self`):
  - `pub trait OptionRepository` — one method per `$wpdb` call `option.php` makes, so query counts match. A failure
    `wpdb` would hide (it returns false or null and records `last_error`) is `Ok` with the empty answer; `Err` only for
    failures `wpdb` would not hide. Methods: `engine(&self) -> String`; `value(&self, name: &OptionName, quiet: bool)
    -> Result<Option<SerializedValue>, OptionsError>` (`quiet` suppresses `wpdb` errors, as the installing path does);
    `autoload_of(&self, name: &OptionName) -> Result<Option<Autoload>, OptionsError>`; `values_of(&self, names:
    &[String]) -> Result<Vec<(String, SerializedValue)>, OptionsError>`; `autoloaded(&self, values: &[String]) ->
    Result<Vec<(String, SerializedValue)>, OptionsError>` and `all(&self) -> Result<Vec<(String, SerializedValue)>,
    OptionsError>` (both quiet, in table order); `needing_autoload_change(&self, groups: &[(Autoload, Vec<String>)])
    -> Result<Vec<String>, OptionsError>` (names whose autoload differs from their group's value);
    `set_autoload(&self, names: &[String], autoload: &Autoload) -> Result<u64, OptionsError>`; `add(&self, option:
    &OptionEntity) -> Result<u64, OptionsError>` (upsert; MySQL's affected-row count: 1 new, 2 changed, 0 same);
    `update(&self, name: &OptionName, value: Option<&SerializedValue>, autoload: Option<&Autoload>) -> Result<u64,
    OptionsError>`; `remove(&self, name: &OptionName) -> Result<u64, OptionsError>`.
  - `pub trait ValueCodec { fn maybe_serialize(&self, value: &HookValue) -> Result<HookValue, OptionsError>; fn
    maybe_unserialize(&self, value: HookValue) -> Result<HookValue, OptionsError>; fn clone_value(&self, value:
    HookValue) -> HookValue; }` (Strategy: Rust values here, PHP objects in Task 10). `maybe_serialize` returns
    non-array scalars unchanged, as WordPress does.
  - `pub trait Installing { fn installing(&self) -> bool; fn setup_config(&self) -> bool; }` (`setup_config`:
    `defined('WP_SETUP_CONFIG')`); `pub trait Multisite { fn is_multisite(&self) -> bool; fn blog_id(&self) -> u64;
    fn network_id(&self) -> u64; }`; `pub trait Sanitizing { fn sanitize_option(&self, name: &OptionName, value:
    HookValue) -> Result<HookValue, OptionsError>; }`; `pub trait Deprecations { fn deprecated_argument(&self,
    function: &str, version: &str, message: &str) -> Result<(), OptionsError>; }` (no phase-2 function calls
    `_doing_it_wrong`, so it is left out until one does); `pub trait ProtectedOptions { fn protect(&self, name: &str)
    -> Result<(), OptionsError>; }`; `pub trait OptionGroups { fn options_in(&self, group: &str) ->
    Option<Vec<String>>; }` (the `$new_allowed_options` global `wp_prime_option_caches_by_group` reads; a sixth port
    the spec's list lacks, since that global comes from `register_setting()`, phase 4).
  - Rust adapters: `RustCodec` (through `speedyspec-php-serialize`); `InstallingFlag` (`new() -> InstallingFlag`,
    `set(&self, installing: bool)`, a `Cell`; `setup_config` false); `SiteConfiguration { pub multisite: bool, pub
    blog_id: u64, pub network_id: u64 }`; `HookSanitizing::new(hooks: Rc<HookApi>)`; `LogDeprecations`
    (`tracing::warn!`); `FixedProtectedOptions`; `NoOptionGroups`.
  - `pub const CORE_SANITIZED_OPTIONS: &[&str]` — the names `sanitize_option()`'s `switch` handles, copied from
    `formatting.php` (wordpress-develop `packages/wp-text/wp-text-legacy/functions/formatting.php`): `admin_email
    new_admin_email thumbnail_size_w thumbnail_size_h medium_size_w medium_size_h medium_large_size_w
    medium_large_size_h large_size_w large_size_h mailserver_port comment_max_links page_on_front page_for_posts
    rss_excerpt_length default_category default_email_category default_link_category close_comments_days_old
    comments_per_page thread_comments_depth users_can_register start_of_week site_icon fileupload_maxk posts_per_page
    posts_per_rss default_ping_status default_comment_status blogdescription blogname blog_charset blog_public
    date_format time_format mailserver_url mailserver_login mailserver_pass upload_path ping_sites gmt_offset siteurl
    home illegal_names limited_email_domains banned_email_domains timezone_string permalink_structure category_base
    tag_base default_role moderation_keys disallowed_keys WPLANG`.

- [ ] **Step 1: Write the failing tests:** `hook_sanitizing_fires_named_filter` (a Rust callback on
  `sanitize_option_my_opt` sees args `("v", "my_opt", "v")` and its return is the result);
  `hook_sanitizing_refuses_core_names` (`blogname` → `Err(NeedsPhp { function: "sanitize_option", option:
  "blogname" })`, no hook fired); `protected_names` (`alloptions`, `notoptions` → `Err(Protected(..))`; `Alloptions`
  → `Ok`); `installing_flag_toggles`; `rust_codec_round_trips` (array `["a" => 1]` serializes to
  `a:1:{s:1:"a";i:1;}` and back; `Str("a:0:{}")` serializes to `s:6:"a:0:{}";`; `Int(5)` stays `Int(5)`;
  `maybe_unserialize(Str("b:0;"))` is `Bool(false)`; an object payload `O:8:"stdClass":0:{}` unserializes to an
  `Opaque`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options --test adapters` — Expected: FAIL.
- [ ] **Step 3: Implement** the traits and adapters.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options --test adapters` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-wp-options && git commit -m "feat(wp-options): ports to the repository and to unported code, with Rust adapters"`

### Task 3: The repository suite and `MemoryOptionRepository` (`speedyspec-wp-options-memory`)

**Files:**
- Create: `crates/speedyspec-wp-options/src/conformance.rs` (behind feature `conformance`);
  `crates/speedyspec-wp-options-memory/Cargo.toml`, `src/lib.rs`
- Test: `crates/speedyspec-wp-options-memory/tests/conformance.rs`

**Interfaces:**
- Consumes: Task 2.
- Produces: `pub fn option_repository_suite(repo: &dyn OptionRepository)` (panics naming the failing check; expects an
  empty table); `pub struct MemoryOptionRepository` with `new() -> MemoryOptionRepository`, `impl OptionRepository`
  (`engine()` is `"memory"`), and `statements(&self) -> u64` (calls made, one per method call, for query-count tests).
- Suite checks, in order: `add_new` (1); `add_same` (0); `add_changed` (2, value replaced); `value_found_and_missing`;
  `autoload_of`; `autoloaded_filters_by_values` (rows `a:on`, `b:off`, `c:auto`, `d:yes` with values `["yes", "on",
  "auto-on", "auto"]` → `a, c, d` in insertion order); `all`; `values_of` (missing names absent, no error);
  `needing_autoload_change` (`[(On, [a, b])]` → `[b]`); `set_autoload` (`[b, c]` to `Off` is 1, since `b` is already
  `off`; the same call again is 0); `update_value_only` (autoload kept); `update_same_is_zero`; `update_missing_is_zero`;
  `remove` (1, then 0).

- [ ] **Step 1: Write the failing test** `memory_conformance` (`option_repository_suite(&MemoryOptionRepository::new())`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-memory` — Expected: FAIL.
- [ ] **Step 3: Implement** the suite and the repository (`RefCell<IndexMap<String, (SerializedValue, Autoload)>>`).
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options-memory` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options-memory): option repository suite and memory repository"` (add new files first).

### Task 4: `AutoloadPolicy`

**Files:**
- Create: `crates/speedyspec-wp-options/src/autoload_policy.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-wp-options/tests/autoload_policy.rs`

**Interfaces:**
- Consumes: Tasks 1–2, `HookApi`.
- Produces: `pub trait AutoloadPolicy { fn determine(&self, name: &str, value: &HookValue, serialized: &HookValue,
  requested: &AutoloadRequest) -> Result<Autoload, OptionsError>; fn values_to_autoload(&self) -> Result<HookValue,
  OptionsError>; }`; `pub struct WordPressAutoloadPolicy` (`new(hooks: Rc<HookApi>)`); `pub fn
  filter_default_autoload_value_via_option_size(hooks: &HookApi, autoload: HookValue, name: &str, serialized:
  &HookValue) -> Result<HookValue, OptionsError>`; `pub fn register_default_filters(hooks: &HookApi) -> Result<(),
  OptionsError>`; `pub fn autoload_values(values: &HookValue) -> Vec<String>`.
- Rules, from `option.php`: `determine` — `Bool(b)` → `On`/`Off`; `Text("on"|"yes")` → `On`; `Text("off"|"no")` →
  `Off`; otherwise `wp_default_autoload_value` with `(null, name, value, serialized)`, a bool result → `AutoOn`/
  `AutoOff`, anything else `Auto`. `values_to_autoload` — `wp_autoload_values_to_autoload` on `DEFAULT_AUTOLOAD_VALUES`
  as a list, then PHP's `array_intersect(filtered, defaults)` (filtered keys kept). The size filter —
  `(int) wp_max_autoloaded_option_size(150000, name)`; size is `strlen(serialized)` unless `empty(serialized)`;
  larger than the maximum → `Bool(false)`, else `autoload` unchanged. `register_default_filters` adds the size filter
  as a Rust callback with id `wp_filter_default_autoload_value_via_option_size`, priority 5, 4 args, as
  `default-filters.php` does; for hosts with no PHP. A PHP host gets the same id from `default-filters.php`, so
  `add_filter` replaces rather than duplicates.

- [ ] **Step 1: Write the failing tests:** `explicit_values` (`Bool(true)` → `On`, `Text("no")` → `Off`, no hook
  fired); `default_without_filters_is_auto`; `filter_bool_gives_auto_on_off`; `size_rule` (with
  `register_default_filters`: a 150000-byte serialized value → `Auto`, 150001 bytes → `AutoOff`; a
  `wp_max_autoloaded_option_size` callback returning `"10"` makes 11 bytes `AutoOff`); `values_to_autoload_default`
  (`[0 => yes, 1 => on, 2 => auto-on, 3 => auto]`); `values_to_autoload_only_removes` (a filter returning `["on",
  "extra"]` → `[0 => "on"]`; one unsetting key 0 → keys `1, 2, 3`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options --test autoload_policy` — Expected: FAIL.
- [ ] **Step 3: Implement** per the rules.
- [ ] **Step 4: Run** the test — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(wp-options): autoload policy and the size rule"` (add new files first).

### Task 5: `OptionsApi` and the read path

**Files:**
- Create: `crates/speedyspec-wp-options/src/api.rs`, `src/read.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-wp-options/tests/support/mod.rs`, `tests/read.rs`

**Interfaces:**
- Consumes: Tasks 1–4; `ObjectCache`, `CacheValue` (plan 3).
- Produces:
  - `pub struct OptionsDeps { pub hooks: Rc<HookApi>, pub cache: Rc<ObjectCache>, pub repository: Rc<dyn
    OptionRepository>, pub autoload: Rc<dyn AutoloadPolicy>, pub codec: Rc<dyn ValueCodec>, pub installing: Rc<dyn
    Installing>, pub multisite: Rc<dyn Multisite>, pub sanitizing: Rc<dyn Sanitizing>, pub deprecations: Rc<dyn
    Deprecations>, pub protected: Rc<dyn ProtectedOptions>, pub groups: Rc<dyn OptionGroups> }` with
    `OptionsDeps::rust(hooks, cache, repository) -> OptionsDeps` (every Rust adapter of Task 2, single site, blog 1).
  - `pub struct OptionsApi` with `new(deps: OptionsDeps) -> OptionsApi`, and: `get_option(&self, name: &str, default:
    Option<HookValue>) -> Result<HookValue, OptionsError>` (`None` = no default passed: `false`, `$passed_default`
    false); `load_alloptions(&self, force_cache: bool) -> Result<HookValue, OptionsError>`;
    `prime_option_caches(&self, names: &[String]) -> Result<(), OptionsError>`; `prime_option_caches_by_group(&self,
    group: &str) -> Result<(), OptionsError>`; `get_options(&self, names: &[String]) -> Result<Vec<(String,
    HookValue)>, OptionsError>`; `protect_special_option(&self, name: &str) -> Result<(), OptionsError>`.
  - Test support: `support::fixture() -> Fixture { api, hooks, cache, repo: Rc<MemoryOptionRepository>, installing:
    Rc<InstallingFlag>, log: Rc<RefCell<Vec<(String, Vec<HookValue>)>>> }` and `support::record(&fixture, hook:
    &str)` (adds a Rust callback that logs its hook and arguments and returns its first argument).
- Order, from `option.php` (`get_option`): trim/empty → false; deprecated name (unless installing):
  `deprecated_argument("get_option", "5.5.0", "The \"<old>\" option key has been renamed to \"<new>\".")` and recurse
  with the new name and the same default; `pre_option_{n}` `(false, n, default)`, then `pre_option` `(pre, n,
  default)`, non-`false` returned; `setup_config` → false; not installing: `load_alloptions(false)`, a set key wins;
  else `notoptions` (created as `[]` and set when not an array), a hit → `default_option_{n}` `(default, n, passed)`;
  else the single cache; else `repository.value(n, false)` → `cache.add`, or `notoptions[n] = true`, set, and the
  default filter. Installing: `repository.value(n, true)`, no cache. Then `home` with `""` → `get_option("siteurl")`;
  `siteurl`, `home`, `category_base`, `tag_base` are right-trimmed of `/` and `\`; `option_{n}` `(maybe_unserialize(v),
  n)`. `load_alloptions` and `prime_option_caches` follow `wp_load_alloptions` and `wp_prime_option_caches` line for
  line (`pre_wp_load_alloptions` array → returned without `alloptions`; the cache read and the `pre_cache_alloptions`
  plus `cache.add` both only when `!installing || !multisite`; `autoloaded`, then `all` when empty; prime sets found
  values one `cache.set` each, and `notoptions` only when it gained a name). `prime_option_caches_by_group` primes
  `groups.options_in(group)` when `Some`. `get_options` primes, then `get_option(name, None)` each.

- [ ] **Step 1: Write the failing tests** in `tests/read.rs` (each with `support::fixture()`, the memory repository's
  `statements()` standing in for `get_num_queries()`):
  `get_option_hook_order` (record every read hook; for a stored, non-autoloaded `foo` the log is `pre_option_foo,
  pre_option, pre_wp_load_alloptions, wp_autoload_values_to_autoload, pre_cache_alloptions, alloptions, option_foo`
  with WordPress's arguments; with one autoloaded row `a` seeded, `statements()` is 2: `autoloaded`, then `value`);
  `pre_option_short_circuits` (a `pre_option` callback returning `"x"` → `"x"`, no
  statement); `missing_goes_to_notoptions_once` (two `get_option("nope", Some("d"))` → `"d"` twice, one `value` call,
  `notoptions == ["nope" => true]`, `default_option_nope` args `("d", "nope", true)`); `blank_and_zero_names` (`""`,
  `"0"`, `" 0 "` → `false`, no hook); `trimmed_name` (`" foo "` fires `pre_option_foo`); `deprecated_name_maps`
  (`blacklist_keys` reads `disallowed_keys`, one deprecation; none while installing); `installing_skips_cache`
  (`installing.set(true)`: two reads → two `value(_, true)` calls, cache untouched); `home_falls_back_to_siteurl`
  (`home = ""`, `siteurl = "http://x/"` → `"http://x"`); `alloptions_filter_non_array` (an `alloptions` callback
  returning `Bool(false)` → `get_option("foo")` still reads `foo` from the single cache or the repository);
  `prime_skips_known` (with `a` autoloaded, `b` in notoptions, `c` cached: priming `[a, b, c, d, e]` with `d` stored
  sends one `values_of([d, e])`, caches `d`, adds `e` to notoptions); `get_options_returns_in_order`;
  `protect_special_option_errors`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options --test read` — Expected: FAIL.
- [ ] **Step 3: Implement** `OptionsApi` and the read path per the order above; each filter and action goes through
  `skip_if_empty` before building its arguments.
- [ ] **Step 4: Run** the test — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): OptionsApi and the read path"` (add new files first).

### Task 6: The write path and autoload setters

**Files:**
- Create: `crates/speedyspec-wp-options/src/write.rs`, `src/autoload_set.rs`; modify `src/api.rs`
- Test: `crates/speedyspec-wp-options/tests/write.rs`

**Interfaces:**
- Consumes: Task 5.
- Produces, on `OptionsApi`: `add_option(&self, name: &str, value: HookValue, deprecated: HookValue, autoload:
  HookValue) -> Result<bool, OptionsError>`; `update_option(&self, name: &str, value: HookValue, autoload: HookValue)
  -> Result<bool, OptionsError>`; `delete_option(&self, name: &str) -> Result<bool, OptionsError>`;
  `set_option_autoload_values(&self, options: Vec<(String, HookValue)>) -> Result<Vec<(String, bool)>,
  OptionsError>`; `set_options_autoload(&self, names: &[String], autoload: HookValue) -> Result<Vec<(String, bool)>,
  OptionsError>`; `set_option_autoload(&self, name: &str, autoload: HookValue) -> Result<bool, OptionsError>`.
  Internally `fn plan_add(..) -> Result<Option<PlannedWrite>, OptionsError>`, `fn plan_update(..)`, `fn
  plan_delete(..)` (checks and filters up to the write) and `fn finish(&self, planned: &PlannedWrite, rows: u64) ->
  Result<bool, OptionsError>` (cache updates and actions after it), so Task 8 can split them around a commit.
  `pub(crate) enum PlannedWrite { Add { option: OptionEntity, value: HookValue, stored: HookValue }, Update { name:
  OptionName, old: HookValue, value: HookValue, stored: HookValue, autoload: Option<Autoload> }, Delete { name:
  OptionName, autoload: Autoload } }` (`stored` is `maybe_serialize(value)`, what the caches hold).
- Order: each function exactly as `option.php` (`add_option` checks `deprecated` with `php_empty` before trimming and
  calls `deprecated_argument("add_option", "2.3.0", "")`; protect; `codec.clone_value`; `sanitize_option`; the
  notoptions / `default_option_{n}` `(false, n, false)` existence check; `update_option`'s equality test is `value ==
  old` or equal `maybe_serialize` results, and an old value equal to the `default_option_{n}` result becomes a full
  `add_option` (so `sanitize_option` runs twice, as in WordPress); a `None` autoload reads `autoload_of` and
  re-determines only `auto`, `auto-on`, `auto-off`). The `update_option` action fires before the write;
  `update_option_{n}`, `updated_option` after the cache updates; `add_option` before, `add_option_{n}`, `added_option`
  after; `delete_option` before, `delete_option_{n}`, `deleted_option` after and only when a row was deleted.
  `wp_set_option_autoload_values` groups by `'off' === $a || 'no' === $a || false === $a`, protects each name, sends
  one `needing_autoload_change`, at most two `set_autoload`, and then either deletes each `on` name and `alloptions`
  from the cache, or removes the `off` names from a forced `load_alloptions` and sets it.

- [ ] **Step 1: Write the failing tests:** `add_hook_order_and_caches` (autoloaded add: log `add_option`,
  `add_option_x`, `added_option`; `alloptions["x"]` set; `notoptions` loses `x`); `add_existing_is_false`;
  `add_deprecated_argument_warns`; `update_hook_order` (`pre_update_option_x (new, old, x)`, `pre_update_option (new,
  x, old)`, `update_option (x, old, new)`, `update_option_x (old, new, x)`, `updated_option (x, old, new)`);
  `update_unchanged_is_false` (same value and same serialized form, no write); `update_missing_adds`
  (`statements()` and hooks match an `add_option`); `update_reevaluates_auto_only` (stored `auto` + 150001-byte value
  with the size filter → `auto-off`, moved from `alloptions` to the single cache; stored `on` is kept);
  `update_caches_raw_stored_value` (`update_option("n", Int(5))` with `n` autoloaded → `alloptions["n"] == Int(5)`);
  `update_double_serializes` (`update_option("s", Str("a:0:{}"))` stores `s:6:"a:0:{}";`, `get_option("s")` is
  `"a:0:{}"`); `delete_hook_order_and_notoptions`; `delete_missing_is_false_without_hooks`;
  `set_autoload_values_groups` (four names, mixed `true`/`"no"`/`false`/`"yes"`, two already right → one select, two
  updates, results per name, `alloptions` deleted); `set_autoload_values_empty_is_empty` (no statement);
  `set_option_autoload_missing_is_false`; `protected_name_errors_before_any_write`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options --test write` — Expected: FAIL.
- [ ] **Step 3: Implement** per the order above.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): add, update and delete options, and the autoload setters"` (add new files first).

### Task 7: Commands, queries, the hooks a read may fire, and the Facade

**Files:**
- Create: `crates/speedyspec-wp-options/src/messages.rs`, `src/fired.rs`, `src/facade.rs`; modify `src/lib.rs`;
  modify `crates/speedyspec-wp-object-cache/src/*` only if `exists` is missing (see the assumptions)
- Test: `crates/speedyspec-wp-options/tests/messages.rs`, `tests/fired.rs`

**Interfaces:**
- Consumes: Tasks 5–6; `speedyspec_cqrs`, `speedyspec_container::Container`.
- Produces:
  - Queries (keys `SpeedySpec\WpOptions\<Name>Handler`, handled by `OptionsApi`): `GetOption { name: String,
    default: Option<HookValue> }` → `Result<HookValue, OptionsError>`; `GetOptions { names: Vec<String> }`;
    `LoadAlloptions { force_cache: bool }`; `DetermineOptionAutoloadValue { name: String, value: HookValue,
    serialized: HookValue, autoload: HookValue }` → `Result<HookValue, OptionsError>` (the string);
    `FilterDefaultAutoloadValueViaOptionSize { autoload: HookValue, name: String, value: HookValue, serialized:
    HookValue }`; `AutoloadValuesToAutoload` → `Result<HookValue, OptionsError>`. Commands: `AddOption { name,
    value, deprecated, autoload }` → `Result<bool, OptionsError>`; `UpdateOption { name, value, autoload }`;
    `DeleteOption { name }`; `PrimeOptionCaches { names: Vec<String> }` → `Result<(), OptionsError>`;
    `PrimeOptionCachesByGroup { group: String }`; `SetOptionAutoloadValues { options: Vec<(String, HookValue)> }` →
    `Result<Vec<(String, bool)>, OptionsError>`; `SetOptionsAutoload { names: Vec<String>, autoload: HookValue }`;
    `SetOptionAutoload { name: String, autoload: HookValue }` → `Result<bool, OptionsError>`; `ProtectSpecialOption
    { name: String }` → `Result<(), OptionsError>`. `form_option` has no message: it is `esc_attr(get_option())`,
    and escaping belongs to `formatting.php` (Task 10's facade).
  - `pub fn register_options_handlers(container: &Container, api: Rc<OptionsApi>)`, binding each as the Hook API's
    `register_hook_handlers` does.
  - `OptionsApi::hooks_fired_by_get(&self, name: &OptionName) -> Vec<HookName>` — the hooks a `get_option(name)` now
    could fire, without firing any or counting cache stats (`cache.exists`): always `pre_option_{n}`, `pre_option`,
    `default_option_{n}`, `option_{n}`; unless installing, `pre_wp_load_alloptions` and `alloptions`; when
    `alloptions` is not cached (or installing on multisite), `wp_autoload_values_to_autoload`, `pre_cache_alloptions`
    and `query`; `query` also when the name is in none of `alloptions`, `notoptions` and the single cache, or when
    installing; for `home`, `siteurl`'s four hooks too; for a deprecated name, the new name's hooks. No duplicates;
    in that order.
  - `pub struct Options` (Facade): `new(api: Rc<OptionsApi>)`, `get<T: FromOptionValue>(&self, name: &str) ->
    Result<Option<T>, OptionsError>` (`None` when `get_option(name)` returns `false`, which a stored option never
    does, since `false` is stored as `""`, or when the value does not convert); `pub trait FromOptionValue: Sized { fn from_option_value(value:
    HookValue) -> Option<Self>; }` for `String`, `i64` (PHP numeric strings), `bool` (PHP truthiness) and `HookValue`.

- [ ] **Step 1: Write the failing tests:** `messages.rs` — `handlers_are_bound` (resolve every key above from a
  `Container` after `register_options_handlers`; `UpdateOption` then `GetOption` through the bus round-trips a value);
  `fired.rs` — `cold_read_lists_query` (fresh cache: list contains `query` and `pre_cache_alloptions`);
  `warm_autoloaded_read_has_no_query` (after one `get_option` of an autoloaded name: no `query`, no
  `pre_cache_alloptions`); `known_missing_has_no_query` (name in `notoptions`); `home_includes_siteurl`;
  `listing_fires_nothing` (`did_filter("pre_option")` unchanged, cache hit and miss counts unchanged); `facade_get`
  (`"5"` as `i64` → `Some(5)`, missing → `None`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options` — Expected: FAIL.
- [ ] **Step 3: Implement** the messages, the binding, `hooks_fired_by_get` and the Facade.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): commands and queries, the hooks a read fires, and the Options facade"` (add new files first).

### Task 8: The unit-of-work write path

**Files:**
- Create: `crates/speedyspec-wp-options/src/staged.rs`; modify `src/api.rs`, `src/lib.rs`
- Test: `crates/speedyspec-wp-options/tests/staged.rs`

**Interfaces:**
- Consumes: Task 6's `plan_*`/`finish`; `speedyspec_data::{UnitOfWork, PendingChange, ChangeKind}`,
  `speedyspec_data_uow::DefaultUnitOfWork` (plan 1).
- Produces: on `OptionsApi`, `stage_add(&self, uow: &DefaultUnitOfWork, name: &str, value: HookValue, autoload:
  HookValue) -> Result<Staged, OptionsError>`, `stage_update(&self, uow: &DefaultUnitOfWork, name: &str, value:
  HookValue, autoload: HookValue) -> Result<Staged, OptionsError>`, `stage_delete(&self, uow: &DefaultUnitOfWork,
  name: &str) -> Result<Staged, OptionsError>`; `pub enum Staged { Registered, Unchanged, Exists, Missing, Rejected }`
  (the cases where the direct function would return false before writing: `Unchanged` for update, `Exists` for add,
  `Missing` for delete, `Rejected` for a blank name). A Rust caller commits with `uow.commit()`.
- Rules (spec, "Write paths"): `plan_*` runs at registration, so the filters (`pre_update_option_{n}`,
  `pre_update_option`, `sanitize_option`, `wp_default_autoload_value`) fire then. The pending change (`engine()` is
  `repository.engine()`, kind `New`/`Dirty`/`Removed`) calls the repository write in `apply` and keeps its row count in
  a `Cell`. One `after_commit` callback per staged write calls `finish` with that count: actions and cache updates run
  only after a successful commit, in registration order; a count of 0 skips them, as WordPress's `if ( ! $result )`
  does. A failed commit runs no callback, so no action fires and the cache is untouched. Within one unit of work the
  identity map holds one `OptionEntity` per name: a second `stage_update` of a name reads its old value from the
  staged entity, not the cache, and replaces the pending change's value.

- [ ] **Step 1: Write the failing tests** (memory repository; a test `EngineLookup` handing a dummy `Direct` handle
  for `"memory"`, and a failing `PendingChange` registered last to force a commit error):
  `filters_at_registration_actions_after_commit` (after `stage_update`: `pre_update_option_x` logged,
  `update_option` not; after `commit`: `update_option`, `update_option_x`, `updated_option`, in that order);
  `failed_commit_fires_nothing` (no action, `alloptions` and single cache unchanged, `get_option` returns the old
  value); `registration_order_across_options` (`a` then `b` → `updated_option` for `a` before `b`);
  `second_stage_sees_first` (`stage_update(x, 1)`, `stage_update(x, 2)` → `pre_update_option_x` old value `1`, one
  repository write of `2`); `unchanged_is_reported` (`Staged::Unchanged`, nothing registered).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options --test staged` — Expected: FAIL.
- [ ] **Step 3: Implement** per the rules.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): option writes through a unit of work"` (add new files first).

### Task 9: `SqlOptionRepository` and `OptionMapper` (`speedyspec-wp-options-sql`)

**Files:**
- Create: `crates/speedyspec-wp-options-sql/Cargo.toml`, `src/lib.rs`, `src/mapper.rs`, `src/repository.rs`;
  modify `crates/speedyspec-wp-db/src/*` only for the `process_fields` wrapper, if needed
- Test: `crates/speedyspec-wp-options-sql/tests/mapper.rs`, `tests/mysql.rs`

**Interfaces:**
- Consumes: Tasks 1–3; `WpDb`, `TableNames`, `WpDbError` (plan 2); `Query`, `Order` (plan 1).
- Produces: `pub struct OptionMapper` with `to_entity(row: &Row) -> Result<OptionEntity, OptionsError>` (columns
  `option_name`, `option_value`, `autoload`; `option_id` ignored) and `to_values(option: &OptionEntity) ->
  Vec<DbValue>` (name, value bytes as `Text` when UTF-8 else `Bytes`, autoload); `pub struct SqlOptionRepository` with
  `new(wpdb: Rc<WpDb>, engine: String) -> SqlOptionRepository` and `impl OptionRepository`.
- Statements (built with `Query`, never text; the table is `wpdb.table_names().name("options")` on every call, so
  `set_blog_id` is followed): `value` — `SELECT option_value FROM <t> WHERE option_name = ? LIMIT 1`; `autoload_of` —
  `SELECT autoload FROM <t> WHERE option_name = ? LIMIT 1`; `values_of` — `SELECT option_name, option_value FROM <t>
  WHERE option_name IN (..)`; `autoloaded` — `SELECT option_name, option_value FROM <t> WHERE autoload IN (..)`;
  `all` — the same without `WHERE`; `needing_autoload_change` — `SELECT option_name FROM <t> WHERE (autoload != ? AND
  option_name IN (..)) OR (..)` through `Partial::expr`; `set_autoload` — `UPDATE <t> SET autoload = ? WHERE
  option_name IN (..)`; `add` — `Query::insert_into(<t>, &["option_name", "option_value", "autoload"])` with
  `on_duplicate_update` on all three; `update` and `remove` — `UPDATE <t> SET option_value = ?[, autoload = ?] WHERE
  option_name = ?` and `DELETE FROM <t> WHERE option_name = ?`, each after `wpdb.process_fields` (a `None` is 0 rows,
  sent nothing). Every statement goes through `wpdb.query_statement`; `quiet` reads wrap it in `suppress_errors(true)`
  and restore the previous setting. A `WpDbError` is the empty answer (`None`, `[]`, 0) per the port's rule.

- [ ] **Step 1: Write the failing tests:** `mapper.rs` — `maps_rows_both_ways` (a `Row` with `option_id`,
  `option_name`, `option_value`, `autoload` → entity; non-UTF-8 value → `Bytes`). `mysql.rs` (ignored without MySQL;
  creates `wp_options` with WordPress's `schema.php` definition — `option_id bigint(20) unsigned NOT NULL
  auto_increment, option_name varchar(191) NOT NULL default '', option_value longtext NOT NULL, autoload varchar(20)
  NOT NULL default 'yes', PRIMARY KEY (option_id), UNIQUE KEY option_name (option_name), KEY autoload (autoload)` —
  in `utf8mb4`, empties it between checks): `mysql_conformance` (`option_repository_suite`);
  `query_fires_and_counts` (a Rust callback on `query` sees `SELECT option_value FROM wp_options ...` text; the
  `wpdb` query count rises by exactly one per repository call); `emptied_update_fails` (a `query` callback returning
  `""` for text starting `UPDATE ` → `set_autoload` is 0 and `OptionsApi::set_option_autoload_values` over this
  repository reports false for each name, `alloptions` cached before is unchanged); `invalid_utf8_update_is_zero`
  (`update` with value bytes `[0x61, 0xff]` → 0, row unchanged; `add` of the same → 1); `per_site_table` (after
  `set_blog_id(2)` with a multisite prefix the statements name `wp_2_options`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-sql` and, with MySQL up, `cargo test -p
  speedyspec-wp-options-sql -- --ignored` — Expected: FAIL.
- [ ] **Step 3: Implement** the mapper and repository per the statements above.
- [ ] **Step 4: Run** both commands — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-wp-options-sql && git commit -m "feat(wp-options-sql): options over wp_options through wpdb"`

### Task 10: Native functions and PHP adapters (`speedyspec-wp-options-php`)

**Files:**
- Create: `crates/speedyspec-wp-options-php/Cargo.toml`, `src/lib.rs`, `src/options.php`, `src/native.rs`,
  `src/adapters.rs`, `src/codec.rs`
- Test: `crates/speedyspec-wp-options-php/tests/php.rs` (`harness = false`)

**Interfaces:**
- Consumes: Tasks 1–9; `WpHooksArea`, `WpDbArea`, `ObjectCacheArea`; `speedyspec_php::{NativeArea, call_catching}`,
  `speedyspec_php::native::{Args, NativeError}`.
- Produces: `pub struct OptionsArea` with `new(container: Rc<Container>, hooks: Rc<HookApi>, cache: Rc<ObjectCache>,
  wpdb: Rc<WpDb>) -> OptionsArea` (builds `SqlOptionRepository` for engine `default`, the PHP adapters,
  `WordPressAutoloadPolicy`, `OptionsApi`, and calls `register_options_handlers`), `api(&self) -> Rc<OptionsApi>`,
  `impl NativeArea`.
- PHP surface: `options.php`, evaluated at request start like `hooks.php`, declares WordPress's names with
  WordPress's signatures and defaults, each a one-line call to `SpeedySpec\WpOptions\Native\<name>(...func_get_args())`:
  `get_option($option, $default_value = false)`, `get_options($options)`, `add_option($option, $value = '',
  $deprecated = '', $autoload = null)`, `update_option($option, $value, $autoload = null)`,
  `delete_option($option)`, `wp_load_alloptions($force_cache = false)`, `wp_prime_option_caches($options)`,
  `wp_prime_option_caches_by_group($option_group)`, `wp_set_option_autoload_values(array $options)`,
  `wp_set_options_autoload(array $options, $autoload)`, `wp_set_option_autoload($option, $autoload)`,
  `wp_determine_option_autoload_value($option, $value, $serialized_value, $autoload)`,
  `wp_filter_default_autoload_value_via_option_size($autoload, $option, $value, $serialized_value)`,
  `wp_autoload_values_to_autoload()`, `wp_protect_special_option($option)`, and `form_option($option)` as `echo
  esc_attr(get_option($option));` (WordPress's body). `func_get_args()` keeps `get_option`'s `$passed_default`.
  Each native function dispatches through a `HandlerRef`/`QueryRef` resolved once.
- Precedence: the area declares these names at request start, before any WordPress file loads, so they are the
  functions every caller reaches. A file that declares them again (today's `option.php`) fails with "Cannot redeclare";
  removing them from `option.php` is the hand-over in Task 13, not done here.
- Name conversion in the native functions: strings as is; `int`, `float`, `bool` by PHP's string conversion
  (`true` → `"1"`, `false` → `""`); `null` → `""`; arrays and objects → `false` returned, with no hook (documented
  break `non_scalar_option_name`, Task 11).
- PHP adapters: `PhpInstalling` (`wp_installing()`, `defined('WP_SETUP_CONFIG')`), `PhpMultisite` (`is_multisite()`,
  `get_current_blog_id()`, `get_current_network_id()`), `PhpSanitizing` (core names call `sanitize_option()`; others
  use `HookSanitizing`, no crossing), `PhpDeprecations` (`_deprecated_argument()`), `PhpProtectedOptions` (WordPress's
  message `"%s is a protected WP option and may not be modified"` with `esc_html($option)`, through `wp_die()`),
  `PhpOptionGroups` (`$GLOBALS['new_allowed_options'][$group]`), `PhpCodec` (as `RustCodec`, except values holding
  a PHP object: `serialize()` and `unserialize()` run in PHP, and `clone_value` clones objects). Each calls with
  `call_catching`; a PHP exception becomes `OptionsError::Php(exception)`, which the native function turns into
  `NativeError::Rethrow`, so `WPDieException` reaches PHPUnit.

- [ ] **Step 1: Write the failing tests** in `tests/php.rs` (boots PHP with `twins_area()`, `ContainerArea`,
  `WpHooksArea`, `WpDbArea` on MySQL, `ObjectCacheArea`, `OptionsArea`; ignored-equivalent: exits 0 with a skip line
  when `SPEEDYSPEC_TEST_MYSQL_URL` is unset): `functions_declared_with_defaults` (`ReflectionFunction` of each name
  reports WordPress's parameter names and defaults); `round_trip_from_php` (`add_option('k', ['a' => new
  stdClass])`, `get_option('k')['a'] instanceof stdClass`); `passed_default_seen` (a `default_option_nope` callback
  receives `$passed_default` `false` for `get_option('nope')` and `true` for `get_option('nope', false)`);
  `scalar_names` (`get_option(123)` reads `"123"`; `get_option(['x'])` is `false` with no hook); `protected_dies`
  (a `wp_die_handler` that throws → `update_option('alloptions', 1)` throws that exception); `form_option_escapes`
  (value `<a>` echoes `&lt;a&gt;`); `no_crossing_without_php_callbacks` (`boundary_crossings()` unchanged across a
  Rust-side `OptionsApi::get_option` of a cached option).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-php --test php` — Expected: FAIL.
- [ ] **Step 3: Implement** the area, facade, native functions and adapters.
- [ ] **Step 4: Run** the test with MySQL up — Expected: every check prints `ok:`.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-wp-options-php && git commit -m "feat(wp-options-php): native option functions and PHP adapters"`

### Task 11: Recorded compatibility (`compat/options`)

**Files:**
- Create: `compat/options/scenarios.json`, `compat/options/source.json` (copy `compat/hooks/source.json`'s pin),
  `compat/options/setup.php`, `compat/options/breaks.json`, `compat/options/wordpress-7.1.5.json` (recorded);
  modify `crates/speedyspec-compat/src/record.rs` (`setup_for("options")`, and `pub fn wordpress_release(version:
  &str) -> Result<PathBuf, String>` factored out of `record`: fetch, check SHA-256, unpack once under
  `CARGO_TARGET_TMPDIR`-style cache), `src/lib.rs`, `src/driver.php` (the state probe, if plan 2 added none)
- Test: `crates/speedyspec-wp-options-php/tests/compat.rs` (`harness = false`)

**Interfaces:**
- Consumes: Task 10; `speedyspec_compat::{replay, PhpRunner, Scenario}`.
- Produces: `pub fn options_scenario_runner(php: Php, release: &Path) -> PhpRunner` in `speedyspec-wp-options-php`.
- `setup.php` (shared by both sides; takes the release root and whether the port is loaded): defines `ABSPATH`,
  `WPINC`, `WP_DEBUG = false`; loads the release's `load.php`, `formatting.php`, `functions.php`, `kses.php`; stubs
  `__()` and `_x()` (return the text); adds a `wp_die_handler` filter whose handler throws `WPDieException`; creates
  or empties `wp_options` (Task 9's definition) and seeds `siteurl`, `home`, `blogname` with autoload `on`. The
  recorder's side also loads `plugin.php`, `cache.php`, `class-wp-object-cache.php`, `class-wpdb.php` (`$wpdb` from
  `SPEEDYSPEC_TEST_MYSQL_URL`, prefix `wp_`) and `option.php`, plus `add_filter('wp_default_autoload_value',
  'wp_filter_default_autoload_value_via_option_size', 5, 4)`; the port's side adds that filter only. A function a
  scenario reaches that neither side defines is stubbed here with WordPress's default-path behaviour and a comment.
- State recorded after each step: `num_queries` since the step began, the `options` group's `alloptions` and
  `notoptions` entries and the single entries of the names the scenario touches, and the `wp_options` rows except the
  seeds.
- Catalogue (every function and hook; recording callbacks on every hook the step can fire), at least: each Goals
  function's happy path; `get_option` from alloptions, the single cache, the database, notoptions, a missing name with
  and without a passed default, `pre_option_{n}` and `pre_option` short-circuits, installing on and off,
  `WP_SETUP_CONFIG`, `home` → `siteurl`, the four untrailingslashed names; names `""`, `"0"`, `" x "`, `123`, `true`,
  `['x']`; deprecated `blacklist_keys` and `comment_whitelist` through `get_option`, `update_option` and
  `add_option`; protected `alloptions` and `notoptions` through each writer and `wp_set_option_autoload_values`;
  `add_option` with `$deprecated` set, an existing name, a `default_option_{n}` filter; `update_option` unchanged,
  missing (→ add), with each autoload value; autoload values `on off auto auto-on auto-off yes no` stored and
  requested, `true`/`false`/`null`; the size rule at 150000 and 150001 bytes and with `wp_max_autoloaded_option_size`;
  `wp_autoload_values_to_autoload` filtered to remove, add and reorder; `wp_load_alloptions` with
  `pre_wp_load_alloptions` and `pre_cache_alloptions` and `force_cache`; `wp_prime_option_caches` with mixed known and
  unknown names; `wp_prime_option_caches_by_group` with and without the group; values `null`, `false`, `true`, `0`,
  `1.5`, `""`, `"0"`, a serialized string, a numeric string, a nested array, `stdClass`, an object of a declared
  class, a non-UTF-8 string, an array holding an object; `form_option` with HTML.
- `breaks.json`: `non_scalar_option_name` ("A non-scalar option name returns false with no hook; WordPress converts
  it to a string with a warning.") and any other difference the replay shows, each with its reason. A difference is a
  bug to fix unless it is listed with the project owner's agreement.

- [ ] **Step 1: Write** `scenarios.json`, `setup.php`, `source.json` and the recorder's `options` setup.
- [ ] **Step 2: Record** with MySQL up — `cargo run -p speedyspec-compat -- record options --wordpress 7.1.5` —
  Expected: `wrote .../compat/options/wordpress-7.1.5.json`, no recorder error.
- [ ] **Step 3: Write the replay test** `options_replay_matches` in `tests/compat.rs` (`replay("options",
  &mut options_scenario_runner(..))` is empty, else print each `Mismatch` and exit 1), and check `cargo test -p
  speedyspec-compat` still passes with the `hooks` fixture unchanged (`git diff --exit-code compat/hooks`).
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options-php --test compat` — Expected: `ok: options_replay_matches`
  (fix the port, not the fixture, for each mismatch).
- [ ] **Step 5: Commit** — `git add compat/options crates/speedyspec-compat crates/speedyspec-wp-options-php && git commit -m "test(wp-options): recorded options compatibility"`

### Task 12: WordPress's option tests on MySQL

**Files:**
- Create: `tests/php/wordpress/option/` (copied from wordpress-develop `tests/phpunit/tests/option/`: `option.php`,
  `getOptions.php`, `updateOption.php`, `slashes.php`, `wpAutoloadValuesToAutoload.php`,
  `wpDetermineOptionAutoloadValue.php`, `wpLoadAlloptions.php`, `wpPrimeOptionCaches.php`,
  `wpPrimeOptionCachesByGroup.php`, `wpSetOptionAutoload.php`, `wpSetOptionAutoloadValues.php`,
  `wpSetOptionsAutoload.php`, with their GPL headers), `tests/php/options-bootstrap.php`, `tests/php/phpunit-options.xml`
- Modify: `tests/php/WP_UnitTestCase.php` (additions only, so the hook tests are unaffected)
- Test: `crates/speedyspec-wp-options-php/tests/wordpress.rs` (`harness = false`)

**Interfaces:**
- Consumes: Tasks 10–11 (`setup.php` with the port loaded, `wordpress_release`).
- Left out, with reasons in the bootstrap's comment: `multisite.php`, `networkOption.php`, `siteOption.php`,
  `siteTransient.php`, `transient.php`, `registration.php`, `themeMods.php`, `userSettings.php`, `wpUserSettings.php`,
  `wpPrimeNetworkOptionCaches.php` (phases 3–4 or other areas) and `sanitizeOption.php` (`sanitize_option()` is not
  ported).
- `WP_UnitTestCase` additions: when `$GLOBALS['wpdb']` is set, `set_up` runs `START TRANSACTION` and
  `tear_down` `ROLLBACK`, then `wp_cache_flush()`; `flush_cache()` (`wp_cache_flush()`); `WPDieException` declared in
  the bootstrap. `options-bootstrap.php` requires `bootstrap.php`'s pieces and `compat/options/setup.php`.

- [ ] **Step 1: Copy** the twelve files; write the bootstrap, `phpunit-options.xml` (suite `wordpress-options`,
  directory `wordpress/option`, excluding `documented-break`) and the test case additions.
- [ ] **Step 2: Write** `tests/wordpress.rs` as the Hook API's `wordpress.rs` (pinned PHPUnit, JUnit totals), booting
  the six areas of Task 10, with checks `documented_breaks_are_listed` (against `compat/options/breaks.json`'s
  `wordpress_tests`) and `wordpress_option_tests_pass` (exit 0, `tests >= 89`, the copied files' test methods before data sets, no failures or errors).
- [ ] **Step 3: Run** with MySQL up — `cargo test -p speedyspec-wp-options-php --test wordpress` — Expected: both
  checks `ok:`. A failing test is fixed in the port; only a difference the project owner accepts is marked `@group
  documented-break` and listed.
- [ ] **Step 4: Run** `cargo test --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo fmt
  --check` — Expected: clean (the hook tests still pass with the extended test case).
- [ ] **Step 5: Commit** — `git add tests/php crates/speedyspec-wp-options-php && git commit -m "test(wp-options): WordPress's option tests pass on MySQL"`

### Task 13: Hand-over note for work outside this repository

**Files:**
- Create: `/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/options-api-port.md`
  (outside the repository: not committed)

- [ ] **Step 1: Write the note**, titled "Options API port (phase 2): hand-over", stating:
  - What exists: branch `feat/options-api-port` of `SpeedySpec/speedyspec-server` with the four crates, `compat/options`
    and WordPress's option tests passing on MySQL; `crates/speedyspec-php-runtime` was not touched.
  - For wordpress-develop: remove from `packages/wp-option/wp-option-legacy/functions/option.php` the sixteen phase-2
    functions (`get_option`, `wp_prime_option_caches`, `wp_prime_option_caches_by_group`, `get_options`,
    `wp_set_option_autoload_values`, `wp_set_options_autoload`, `wp_set_option_autoload`,
    `wp_protect_special_option`, `form_option`, `wp_load_alloptions`, `update_option`, `add_option`, `delete_option`,
    `wp_determine_option_autoload_value`, `wp_filter_default_autoload_value_via_option_size`,
    `wp_autoload_values_to_autoload`), keeping transients, site and network options, user settings and
    `register_setting()` for phases 3–4; the native functions are declared at request start, so the file fails with
    "Cannot redeclare" until they are removed. `default-filters.php`'s `wp_default_autoload_value` line stays (it names
    the native function). `packages/wp-option/*` and `packages/speedyspec-wp-option-*` are deleted with `option.php`,
    after phase 4, per the spec.
  - For the runtime's owner (`worker.rs`): create `OptionsArea::new(container, hooks, cache, wpdb)` after
    `WpDbArea` and `ObjectCacheArea`, from the same container; tell the Rust `Installing` adapter when the runtime
    runs the installer.
  - For the native features work: `OptionsApi::hooks_fired_by_get(&OptionName) -> Vec<HookName>` is ready for the
    PHP-callback registry; it lists `query` for reads that would reach the database.
  - Breaks recorded in `compat/options/breaks.json`, and the assumptions this plan made about plans 2 and 3 that the
    merged code changed, if any.
- [ ] **Step 2: Check** — `ls "/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/"` — Expected: `options-api-port.md` listed.
