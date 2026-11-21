# Persistent object cache, transients and network options Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Options design phases 3 and 4: a persistent Caching Store as the object cache's second level, chosen by
engine routing; transients and site transients; site and network options over `wp_sitemeta`; `register_setting()` and
registered settings; all with WordPress's exact behaviour and hooks, proven by recorded compatibility and WordPress's
own option tests on MySQL.

**Architecture:** A small Abstract Factory (`speedyspec-cache-routing`) turns the engine routed as `object-cache` into
a `SqlCache`, `KvCache` or `FileCache`, which plan 3's `ObjectCache` uses as its second level (Decorator).
Transients, network options and the settings registry are new domain services in `speedyspec-wp-options` beside
`OptionsApi`, each a set of commands and queries handled as the Hook API's are. `wp_sitemeta` gets its own aggregate
repository (`SitemetaRepository`, SQL and memory). Unported WordPress (`wp_using_ext_object_cache()`, `is_main_site()`,
the REST settings controller, `__()`) stays behind ports or in PHP.

**Tech Stack:** Rust 1.99, edition 2024; the crates of plans 1–4 and 7; `indexmap`; the embedded PHP of
`speedyspec-php` for the PHP surface and WordPress's PHPUnit tests.

**Spec:** `docs/superpowers/specs/2026-11-19-options-api-port-design.md` (Phases 3 and 4, "The object cache's two
levels", Testing) and `docs/superpowers/specs/2026-11-19-database-layer-design.md` (Engines and the registry, The
Caching Store). Plan index (repository, worktree, shared names):
`docs/superpowers/plans/2026-11-19-data-and-options-ports-index.md`. WordPress's original:
`packages/wp-option/wp-option-legacy/functions/option.php` (wordpress-develop), lines 677–846 and 1381–3256.

## Global Constraints

- Work in a new worktree of `SpeedySpec/speedyspec-server` branched from `origin/main` after plans 4 and 7 merged,
  branch `feat/persistent-cache-transients-network-options`. Never in `packages/speedyspec-server-runtime-gate/` or
  `packages/speedyspec-server/`. New crates under `crates/`; no workspace `Cargo.toml` edit.
- No edit to `crates/speedyspec-php-runtime`. Worker changes go into the hand-over note (Task 10).
- New crates: `edition.workspace`, `rust-version.workspace`, `license.workspace`, version `0.1.0`, one-line
  `description`. Contracts never share a crate with implementations; no `-domain`, `-app`, `-infra`, `-core` crates.
- Every hook keeps WordPress's name, arguments and order; recorded scenarios are the specification where this plan
  and `option.php` seem to differ.
- Every SQL statement is parsed (`parse_one` or the builder) and sent through `WpDb` (plan 2), so `query` fires as in
  WordPress. Entities never load, save or query themselves.
- Object cache groups and keys exactly as WordPress: `transient`, `site-transient`, `site-options` with keys
  `"{network_id}:{option}"` and `"{network_id}:notoptions"`; `site-options` and `site-transient` are global groups.
- Patterns: Abstract Factory (`PersistentCacheSelector`), Decorator (two-level object cache), Strategy (Caching Store
  backend), Repository and Data Mapper (`SitemetaRepository`, `SitemetaMapper`), Serialized LOB (`meta_value`),
  Command/Query (every function), Adapter (PHP adapters of ports, native functions), Observer (hooks), Registry
  (`SettingsRegistry` over a `SettingsStore` port).
- User settings (`wp_user_settings` and the rest) are out of scope: they wait for the auth port.
- Tests needing MySQL are `#[ignore = "needs MySQL: set SPEEDYSPEC_TEST_MYSQL_URL"]`; needing Valkey too,
  `#[ignore = "needs MySQL and Valkey: set SPEEDYSPEC_TEST_MYSQL_URL and SPEEDYSPEC_TEST_VALKEY_URL"]`. Valkey for
  tests: `docker run -d --name speedyspec-valkey -p 36379:6379 valkey/valkey:8`,
  `SPEEDYSPEC_TEST_VALKEY_URL=valkey://127.0.0.1:36379`. After Linux container runs, delete that run's `target-linux*/`.

## Decisions

- **Persistent means routed.** The object cache is persistent exactly when the site configuration routes the use
  `object-cache` to an engine (even `default`). Unrouted, it stays request-lifetime, as WordPress without a drop-in.
- **`wp_using_ext_object_cache()` stays in PHP** (`load.php` is not ported; WordPress's tests toggle it). Rust reads it
  through a new port `ExternalObjectCache`: the PHP adapter calls the function, the Rust adapter answers
  `ObjectCache::is_persistent()`. The worker sets `$_wp_using_ext_object_cache = true` at request start when
  persistent (hand-over note), so `wp_start_object_cache()` skips `cache.php` as it does for a drop-in.
- **Settings state lives behind a `SettingsStore` port**, as hooks live behind `HookStore`: in PHP the store *is*
  `$wp_registered_settings` and `$new_allowed_options` (with `$new_whitelist_options` a reference), so PHP code that
  reads those globals keeps working.
- **REST parts stay PHP.** `show_in_rest` is carried as given; `WP_REST_Settings_Controller` reads
  `get_registered_settings()` unchanged. `register_initial_settings()` stays in `option.php` (it is data plus `__()`,
  and translation is not ported) and calls the native `register_setting()`.
- Messages passed to `_deprecated_argument`/`_doing_it_wrong` are WordPress's English text; recordings use `en_US`.
- Removing ported functions from `option.php` follows plan 4's mechanism; after this plan `option.php` holds only the
  user-settings functions and `register_initial_settings()`.

## Assumed from plans 3, 4 and 7

**Reconciled after those plans were written (these win over the list below):**

- Plan 3: `SqlCache::new(store: Rc<dyn SqlStore>, table: &str) -> Result<SqlCache, CacheError>`, `install`,
  `purge_expired(&self) -> Result<u64, CacheError>`. The second level is given at construction:
  `ObjectCache::new(ports, Some(store))`, `CurrentObjectCache::new(ports, persistent)`, `ObjectCacheArea::new(container,
  persistent: Option<Rc<dyn CachingStore>>)`, so no `with_persistent` is needed. `ObjectCache`'s methods take
  PHP-shaped keys and groups (`&Value`) and return `Result<_, ObjectCacheError>`; see plan 3, Task 6.
- Plan 4: see its own "Reconciled" block for `WpDb` and object-cache access; its other names are as this plan lists.
- Plan 7: `KvCache::add` is atomic through `NoSqlStore::put_if_absent` (added to plan 1).
- `EngineRegistry::routed`, which this plan adds to plan 1's registry, is the only change this plan makes to plan 1's
  crates.


- Plan 3: `CachingStore`, `CacheError`, `ObjectCache` with `get`, `get_multiple`, `set`, `add`, `delete`,
  `add_global_groups`, `switch_to_blog`, `flush_runtime` taking `(key: &str, group: &str, ..)` and
  `speedyspec_hooks::Value`; crate `speedyspec-cache-sql` with `SqlCache::new(store: Rc<dyn SqlStore>)` (its default
  table) and `SqlCache::install(&self) -> Result<(), CacheError>`; `ObjectCacheArea`. If `ObjectCache` has no way to
  take a second level after construction, Task 2 adds `with_persistent`.
- Plan 4: `OptionsApi` methods `get_option`, `add_option`, `update_option`, `delete_option`, `wp_load_alloptions`,
  `wp_prime_option_caches` (WordPress's arguments; `get_option` knows whether a default was passed), error type
  `OptionsError`, ports `Installing`, `Multisite` (`is_multisite`, `current_blog_id`, `current_network_id`),
  `Sanitizing` (`sanitize(name, value)`), `Deprecations` (`deprecated_argument(function, version, message)`,
  `doing_it_wrong(function, message, version)`), `ProtectedOptions` (`protect(name)`), with Rust adapters in
  `speedyspec-wp-options` and PHP adapters in `-php`; message keys `SpeedySpec\WpOptions\<Name>Handler`; the compat
  recorder and replay test for `compat/options`; the WordPress PHPUnit runner `crates/speedyspec-wp-options-php/tests/wordpress.rs`
  (shaped like `speedyspec-wp-hooks-php/tests/wordpress.rs`) with a list of test files.
- Plan 7: `KvCache::new(store: Rc<dyn NoSqlStore>)` in `speedyspec-cache-kv`, `FileCache::new(store: Rc<dyn
  DirectStore>)` in `speedyspec-cache-file` (file Direct Stores report `kind() == "file"`), the Valkey engine factory
  (scheme `valkey`) and file engine factory (scheme `file`).

## Review Focus

- **The second-level store fails mid-request** (Valkey stopped): `get_option` still answers from the database, the
  failure is logged, nothing panics. Test in Task 2.
- **A non-numeric network id** (`'abc'`, an array) to any `*_network_option`: `false` at once, no hook, no query.
  Test in Task 4.
- **Odd expirations** (`'60'`, `'abc'`, `-10`, `60.9`): PHP's `(int)` cast; a negative one expires on the next read.
  Test in Task 5.
- **Multisite with a persistent cache:** a site transient set on blog 1 is read on blog 2; a plain transient is not.
  Test in Task 5.
- **A transient name too long for `option_name`** (over 172 bytes) on options storage: `set_transient` returns
  false and fires no `set_transient*` action, as WordPress on MySQL. Recorded scenario in Task 9.

---

### Task 1: Choosing the persistent Caching Store (`speedyspec-cache-routing`)

**Files:**
- Create: `crates/speedyspec-cache-routing/Cargo.toml`, `src/lib.rs`
- Modify: `crates/speedyspec-data-query/src/registry.rs` (add `routed`)
- Test: `crates/speedyspec-cache-routing/tests/select.rs`, `crates/speedyspec-data-query/tests/registry.rs`

**Interfaces:**
- Consumes: `EngineRegistry`, `EngineHandle`, `DbError` (plan 1); `SqlCache`, `KvCache`, `FileCache`, `CachingStore`.
- Produces: `EngineRegistry::routed(&self, use_name: &str) -> Option<String>` (the explicit route only, no `default`
  fallback); `pub const OBJECT_CACHE_USE: &str = "object-cache";`; `pub struct PersistentCacheSelector` with
  `new() -> PersistentCacheSelector` and `select(&self, registry: &EngineRegistry, use_name: &str) ->
  Result<Option<Rc<dyn CachingStore>>, DbError>`: unrouted → `None`; `Sql` → `SqlCache`; `NoSql` → `KvCache`;
  `Direct` with `kind() == "file"` → `FileCache`; any other `Direct` → `WrongKind { engine, expected: "sql, nosql or
  file" }`. One instance per call site; the worker keeps the result for its life.

- [ ] **Step 1: Write the failing tests:** `routed_has_no_default_fallback` (route `object-cache` → `cache`:
  `routed("object-cache") == Some("cache")`, `routed("x") == None`); in `select.rs` with fake factories (schemes
  `fsql`, `fkv`, `ffile`, `fdirect`) whose stores record calls: `unrouted_is_none`, `sql_engine_gives_sql_cache`
  (a `set` through the result reaches the fake `SqlStore`), `nosql_engine_gives_kv_cache`, `file_engine_gives_file_cache`,
  `other_direct_is_wrong_kind` (`Err(WrongKind { engine: "d", .. })`), `routing_to_default_is_persistent` (route
  `object-cache` → `default` gives `Some`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-query -p speedyspec-cache-routing` — Expected: FAIL.
- [ ] **Step 3: Implement** `routed` and `PersistentCacheSelector` (a `match` on the handle; `SqlCache::install()` is
  not called here: the cache table is created by the installer path plan 3 defines).
- [ ] **Step 4: Run** the same command — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-cache-routing crates/speedyspec-data-query && git commit -m "feat(cache-routing): choose the persistent Caching Store from the object-cache route"`

### Task 2: The object cache's second level, `ExternalObjectCache`, and cross-process proof

**Files:**
- Modify: `crates/speedyspec-wp-object-cache/src/lib.rs` (`is_persistent`; `with_persistent` if absent)
- Create: `crates/speedyspec-wp-options/src/external_cache.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-wp-object-cache/tests/persistent.rs`, `crates/speedyspec-wp-options-sql/tests/cross_process.rs`

**Interfaces:**
- Consumes: Task 1; `ObjectCache`, `OptionsApi`, `SqlOptionRepository`, `WpDb`.
- Produces: `ObjectCache::is_persistent(&self) -> bool`; `ObjectCache::with_persistent(self, store: Rc<dyn
  CachingStore>) -> ObjectCache`; `pub trait ExternalObjectCache { fn using(&self) -> bool; }` and its Rust adapter
  `pub struct ObjectCachePersistence(pub Rc<ObjectCache>)`.
- Rule: a second-level error on read is a miss, on write is ignored; both are logged once per request per operation
  kind, and the first level keeps working (as persistent drop-ins degrade).

- [ ] **Step 1: Write the failing tests.** `persistent.rs`: `persistent_iff_second_level` (`is_persistent()` false
  for a bare cache, true after `with_persistent(MemoryCache)`); `second_level_shared_between_instances` (two
  `ObjectCache`s over one `MemoryCache`: `set("k", "v", "g")` on A, `get("k", "g")` on B is `"v"`);
  `non_persistent_group_stays_local`; `get_option_survives_failing_second_level` (a `CachingStore` fake returning
  `CacheError` on everything: `get_option("blogname")` over the memory repository still returns the stored value).
  `cross_process.rs` (MySQL; runs the same four checks for backends `sql` (`SqlCache` on the test MySQL), `file`
  (`FileCache` in a temp dir) and, ignored separately, `valkey`): the parent spawns `std::env::current_exe()` with
  `--exact child_writer --ignored --nocapture` and `SPEEDYSPEC_CROSS_ROLE=<step>`, waits for exit 0, then calls
  `flush_runtime()` (a new request) before reading; a counter on the `query` filter measures SQL:
  `update_option_seen_at_once` (parent `add_option("ss_cross", "one", "on")`, reads it; child
  `update_option("ss_cross", "two")`; before `flush_runtime` the parent still reads `"one"` (first level, as WordPress);
  after it `"two"` with 0 queries), `notoptions_invalidated` (parent `get_option("ss_missing")` is false; child
  `add_option("ss_missing", "here", "off")`; parent reads `"here"`), `alloptions_invalidated` (child adds autoloaded
  `ss_auto`; parent `wp_load_alloptions()` contains `ss_auto => "x"`), `delete_seen` (child deletes `ss_cross`; parent
  gets false). `child_writer` returns at once when `SPEEDYSPEC_CROSS_ROLE` is unset.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-object-cache --test persistent` and, with MySQL (and Valkey),
  `cargo test -p speedyspec-wp-options-sql --test cross_process -- --ignored` — Expected: FAIL.
- [ ] **Step 3: Implement** `is_persistent`, `with_persistent` (if plan 3 lacks it), the error rule above,
  `ExternalObjectCache` and `ObjectCachePersistence`.
- [ ] **Step 4: Run** both commands — Expected: PASS for every backend.
- [ ] **Step 5: Commit** — `git commit -m "feat(object-cache): a persistent second level shared across processes"` (add new files first).

### Task 3: `SitemetaRepository` (port, SQL, memory)

**Files:**
- Create: `crates/speedyspec-wp-options/src/sitemeta.rs`; `crates/speedyspec-wp-options-sql/src/sitemeta.rs`;
  `crates/speedyspec-wp-options-memory/src/sitemeta.rs`, `src/suites.rs`; modify each crate's `lib.rs`
- Test: `crates/speedyspec-wp-options-memory/tests/sitemeta.rs`, `crates/speedyspec-wp-options-sql/tests/sitemeta.rs`

**Interfaces:**
- Consumes: `SerializedValue` (plan 4), `WpDb`, `TableNames`, `Query`, `DbError`, `Identified`.
- Produces: `pub struct NetworkMeta { pub network_id: i64, pub key: String, pub value: SerializedValue }`
  (`Identified`, id `(i64, String)`; `meta_id` is kept by the mapper only); `pub trait SitemetaRepository { fn
  find(&self, network_id: i64, key: &str) -> Result<Option<NetworkMeta>, DbError>; fn find_many(&self, network_id: i64,
  keys: &[String]) -> Result<Vec<NetworkMeta>, DbError>; fn exists(&self, network_id: i64, key: &str) -> Result<bool,
  DbError>; fn add(&self, meta: NetworkMeta) -> Result<bool, DbError>; fn update(&self, meta: NetworkMeta) ->
  Result<u64, DbError>; fn remove(&self, network_id: i64, key: &str) -> Result<u64, DbError>; }`;
  `SqlSitemetaRepository::new(db: Rc<WpDb>)`, `SitemetaMapper`; `MemorySitemetaRepository::new()`;
  `pub fn sitemeta_repository_suite(repo: &dyn SitemetaRepository)` in `speedyspec_wp_options_memory::suites`.
- SQL, WordPress's own: `find` → `SELECT meta_value FROM {sitemeta} WHERE meta_key = ? AND site_id = ?` (first row);
  `find_many` → `SELECT meta_key, meta_value FROM {sitemeta} WHERE meta_key IN (?, ..) AND site_id = ?`; `exists` →
  `SELECT meta_id FROM {sitemeta} WHERE meta_key = ? AND site_id = ?` and a non-zero `meta_id`; writes through
  `WpDb`'s `Insert`, `Update`, `Delete` handlers with columns `site_id` (`%d`), `meta_key`, `meta_value` (`%s`), so
  charset stripping matches `$wpdb->insert`. `{sitemeta}` is `TableNames::name("sitemeta")`. `update` and `remove`
  return changed rows.

- [ ] **Step 1: Write the failing tests.** The suite checks: `add_then_find` (value bytes equal); `find_many_skips_missing`;
  `networks_are_isolated` (key on network 1 absent on 2); `update_counts_changed_rows` (new value 1, same value 0);
  `remove_counts_rows` (1, then 0); `exists_follows_rows`. `memory/tests/sitemeta.rs` runs it on
  `MemorySitemetaRepository`; `sql/tests/sitemeta.rs` (MySQL) creates `wp_sitemeta` with WordPress's schema
  (`meta_id bigint(20) unsigned NOT NULL auto_increment, site_id bigint(20) unsigned NOT NULL default '0', meta_key
  varchar(255) default NULL, meta_value longtext, PRIMARY KEY (meta_id), KEY meta_key (meta_key(191)), KEY site_id
  (site_id)`) and runs it on `SqlSitemetaRepository`, plus `query_filter_fires` (a `query` callback sees each statement).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-memory --test sitemeta` and, with MySQL,
  `cargo test -p speedyspec-wp-options-sql --test sitemeta -- --ignored` — Expected: FAIL.
- [ ] **Step 3: Implement** the port, mapper and both repositories.
- [ ] **Step 4: Run** both — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): the sitemeta repository, SQL and memory"`

### Task 4: Site and network options (`NetworkOptionsApi`)

**Files:**
- Create: `crates/speedyspec-wp-options/src/network.rs`, `src/network_messages.rs`; modify `src/lib.rs` and the
  `Multisite` port (`is_main_site`, `is_main_network`) with its Rust adapter
- Test: `crates/speedyspec-wp-options-memory/tests/network_options.rs`

**Interfaces:**
- Consumes: Task 3; `OptionsApi`, `ObjectCache`, `HookApi`, ports `Multisite`, `Installing`, `Sanitizing`,
  `ProtectedOptions`; `maybe_serialize`, `maybe_unserialize`.
- Produces:
  - `pub fn resolve_network_id(raw: &HookValue, multisite: &dyn Multisite) -> Option<i64>`: truthy and not
    `is_numeric` → `None`; otherwise PHP `(int)`, and 0 → `current_network_id()`.
  - `pub struct NetworkPorts { pub multisite: Rc<dyn Multisite>, pub installing: Rc<dyn Installing>, pub sanitizing:
    Rc<dyn Sanitizing>, pub protected: Rc<dyn ProtectedOptions> }`.
  - `pub struct NetworkOptionsApi` with `new(options: Rc<OptionsApi>, sitemeta: Rc<dyn SitemetaRepository>, cache:
    Rc<ObjectCache>, hooks: Rc<HookApi>, ports: NetworkPorts) -> NetworkOptionsApi` (calls
    `cache.add_global_groups(&["site-options", "site-transient"])`), and `get_network_option(&self, network_id:
    &HookValue, option: &str, default: HookValue) -> Result<HookValue, OptionsError>`, `add_network_option(&self,
    network_id: &HookValue, option: &str, value: HookValue) -> Result<bool, OptionsError>`, `update_network_option(..same)
    -> Result<bool, OptionsError>`, `delete_network_option(&self, network_id: &HookValue, option: &str) -> Result<bool,
    OptionsError>`, `prime_network_option_caches(&self, network_id: &HookValue, options: &[String]) -> Result<(),
    OptionsError>`, `load_core_site_options(&self, network_id: &HookValue) -> Result<(), OptionsError>`.
  - `pub trait Multisite` gains `fn is_main_site(&self) -> bool; fn is_main_network(&self) -> bool;`.
  - Messages (`SpeedySpec\WpOptions\<Name>Handler`, handled by `NetworkOptionsApi`, bound by
    `register_network_option_handlers(container: &Container, api: Rc<NetworkOptionsApi>)`): `GetNetworkOption`,
    `AddNetworkOption`, `UpdateNetworkOption`, `DeleteNetworkOption`, `PrimeNetworkOptionCaches`,
    `LoadCoreSiteOptions`. The `*_site_option` functions are the same messages with `network_id: HookValue::Null`.
- Behaviour: `option.php` lines 677–846 and 2002–2500, step for step: the `pre_site_option_{$o}` / `pre_site_option`
  short circuit; the network `notoptions` check before the multisite branch; non-multisite delegating to
  `get_option($o, filtered default)`, `add_option($o, $v, '', false)`, `update_option($o, $v, false)`,
  `delete_option($o)`; multisite reading sitemeta, caching in `site-options`, `sanitize` then `maybe_serialize` on
  writes; the empty `notoptions` array written when none was cached; `update` returning false when `===` or the
  serialized forms match, and delegating to `add_network_option` when the old value is false. `load_core_site_options`
  primes WordPress's ten names (`site_name`, `siteurl`, `active_sitewide_plugins`,
  `_site_transient_timeout_theme_roots`, `_site_transient_theme_roots`, `site_admins`, `can_compress_scripts`,
  `global_terms_enabled`, `ms_files_rewriting`, `WPLANG`) only when multisite and not installing.

- [ ] **Step 1: Write the failing tests** (memory repositories, a memory object cache, recording hooks, a settable
  `Multisite` fake): `single_site_falls_back_to_options` (`add_network_option(Null, "x", "v")` stores option `x` with
  `Autoload::Off`; `get_site_option` reads `"v"`; update then delete go through options); `multisite_uses_sitemeta`
  (`get_network_option(2, "x")` caches `"2:x"` in `site-options`; the options repository is untouched);
  `notoptions_per_network` (missing `x` on 1 sets `1:notoptions[x]`; network 2's is unset); `found_option_sets_empty_notoptions`;
  `add_and_update_clear_notoptions`; `delete_sets_notoptions`; `update_same_value_is_false` (an array holding an
  equal object, ticket 44956); `hooks_in_order` (multisite add fires `pre_add_site_option_x(v, x, 1)`,
  `add_site_option_x(x, v, 1)`, `add_site_option(x, v, 1)`; delete fires `pre_delete_site_option_x(x, 1)`,
  `delete_site_option_x(x, 1)`, `delete_site_option(x, 1)`; update fires the get's filters, then
  `pre_update_site_option_x(v, old, x, 1)`, `update_site_option_x(x, v, old, 1)`, `update_site_option(x, v, old, 1)`);
  `pre_site_option_short_circuits` (no repository read); `network_id_parameter` (`1`, `'1'`, `2`, `null`, `false`, `0`,
  `'0'` accepted; `'abc'` and an array give false with no hook fired and no repository call); `prime_skips_cached_and_notoptions`
  (one `find_many` with only the uncached names; missing ones join `notoptions`); `prime_single_site_uses_option_prime`;
  `load_core_site_options_only_on_multisite`; `protected_names_refused` (`add_network_option(1, "alloptions", ..)`
  reaches `ProtectedOptions`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-memory --test network_options` — Expected: FAIL.
- [ ] **Step 3: Implement** the service, the messages and the `Multisite` additions (Rust adapter: from site
  configuration, a single network answers true to both).
- [ ] **Step 4: Run** the test — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): site and network options"`

### Task 5: Transients and site transients (`TransientsApi`)

**Files:**
- Create: `crates/speedyspec-wp-options/src/transients.rs`, `src/transient_messages.rs`, `src/clock.rs`,
  `src/php_cast.rs` (unless plan 4 already has PHP's `(int)` cast); `crates/speedyspec-wp-options-memory/src/clock.rs`
- Test: `crates/speedyspec-wp-options-memory/tests/transients.rs`

**Interfaces:**
- Consumes: Tasks 2 and 4; `OptionsApi`, `ObjectCache`, `HookApi`, `Installing`, `Multisite`.
- Produces:
  - `pub trait Clock { fn now(&self) -> i64; }`, `SystemClock`; `FixedClock` (`new(now: i64)`, `set(&self, now: i64)`)
    in `-memory`. `pub fn php_int(value: &HookValue) -> i64` (null 0, bools 0/1, floats truncated, strings by leading
    numeric prefix, else 0).
  - `pub enum TransientPrefix { Transient, SiteTransient }` with `value(&self) -> &'static str` (`_transient_`,
    `_site_transient_`) and `timeout(&self) -> &'static str` (`_transient_timeout_`, `_site_transient_timeout_`);
    `pub trait ExpiredTransients { fn purge_options(&self, prefix: TransientPrefix, now: i64) -> Result<u64, DbError>;
    fn purge_sitemeta(&self, now: i64) -> Result<u64, DbError>; }`.
  - `pub struct TransientPorts { pub external: Rc<dyn ExternalObjectCache>, pub installing: Rc<dyn Installing>, pub
    multisite: Rc<dyn Multisite>, pub clock: Rc<dyn Clock>, pub expired: Rc<dyn ExpiredTransients> }`;
    `pub struct TransientsApi` with `new(options: Rc<OptionsApi>, network: Rc<NetworkOptionsApi>, cache:
    Rc<ObjectCache>, hooks: Rc<HookApi>, ports: TransientPorts)`, `get_transient(&self, name: &str) ->
    Result<HookValue, OptionsError>`, `set_transient(&self, name: &str, value: HookValue, expiration: &HookValue) ->
    Result<bool, OptionsError>`, `delete_transient(&self, name: &str) -> Result<bool, OptionsError>`, the same three as
    `get_site_transient`, `set_site_transient`, `delete_site_transient`, and `delete_expired_transients(&self,
    force_db: bool) -> Result<(), OptionsError>`.
  - Messages bound by `register_transient_handlers(container: &Container, api: Rc<TransientsApi>)`: `GetTransient`,
    `SetTransient`, `DeleteTransient`, `GetSiteTransient`, `SetSiteTransient`, `DeleteSiteTransient`,
    `DeleteExpiredTransients`.
- Behaviour: `option.php` 1381–1690 and 2512–2735 step for step. Branch on `external.using() || installing`: object
  cache groups `transient` / `site-transient` with the expiration as the cache expiry; otherwise options (local) or
  site options (site). Local: `(int)` cast *before* `pre_set_transient_{$t}(value, expiration, t)`; site:
  `pre_set_site_transient_{$t}(value, t)` *before* the cast. Local reads skip the timeout check when `_transient_{$t}`
  is in `wp_load_alloptions()`; site reads skip it for `update_core`, `update_plugins`, `update_themes`. Timeouts are
  `clock.now() + expiration`; an expired read deletes both rows. The success actions are `set_transient_{$t}(value,
  expiration, t)`, `set_transient(t, value, expiration)`, then `do_action_deprecated('setted_transient', [t, value,
  expiration], '6.8.0', 'set_transient')` (site forms likewise); deletes fire `delete_transient_{$t}(t)` first and
  `deleted_transient(t)` on success. `delete_expired_transients`: nothing when `!force_db && external.using()`; else
  `purge_options(Transient)`, then `purge_options(SiteTransient)` on single site, or `purge_sitemeta` when multisite
  and `is_main_site() && is_main_network()`.

- [ ] **Step 1: Write the failing tests** (memory everything, `FixedClock(1000)`, a recording `ExpiredTransients`):
  `options_storage_basics` (no expiry: `_transient_t` autoloaded, no timeout row; get `"v"`; delete true, rows gone,
  `deleted_transient` fired); `timeout_makes_both_non_autoload` (60 → `_transient_timeout_t == 1060`, both
  `Autoload::Off`); `expired_read_deletes` (clock 1061 → false, both rows gone); `adding_timeout_to_existing_recreates`
  (ticket 22807: value row becomes non-autoload); `existing_with_timeout_updates_timeout`; `object_cache_branch`
  (external true: value in `transient` group, no option written; delete removes it); `installing_uses_object_cache`;
  `set_hooks_in_order` (names and arguments above, `setted_transient` through the deprecated path);
  `pre_transient_short_circuits` (no option read); `odd_expirations` (`'60'` → 1060, `'abc'` → no timeout row,
  `60.9` → 1060, `-10` → 990 and the next read is false); `site_transient_single_site_in_options` (`_site_transient_t`
  option, `Autoload::Off`); `site_transient_multisite_in_sitemeta`; `site_core_names_skip_timeout`
  (`get_site_transient("update_core")` reads no `_site_transient_timeout_update_core`); `site_pre_set_args`
  (`pre_set_site_transient_t` gets `(value, t)`); `site_transient_is_global_across_blogs` (persistent cache, multisite:
  set on blog 1, `switch_to_blog(2)`, `get_site_transient` is the value, `get_transient` of the same name set on blog 1
  is false); `delete_expired_branches` (external without force → no purge; single site → `[Transient, SiteTransient]`;
  multisite main → `[Transient]` and sitemeta; multisite not main → `[Transient]` only).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-memory --test transients` — Expected: FAIL.
- [ ] **Step 3: Implement** the clock, cast, port, service and messages.
- [ ] **Step 4: Run** the test — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): transients and site transients"`

### Task 6: Purging expired transients in SQL (`SqlExpiredTransients`)

**Files:**
- Create: `crates/speedyspec-wp-options-sql/src/expired.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-wp-options-sql/tests/expired.rs`

**Interfaces:**
- Consumes: Task 5's `ExpiredTransients`, `TransientPrefix`; `WpDb::query_statement`, `TableNames`, `parse_one`.
- Produces: `SqlExpiredTransients::new(db: Rc<WpDb>)`, `impl ExpiredTransients`.
- Statements, WordPress's text with `?` for its three `%s`/`%d` values (`\_transient\_%`, `\_transient\_timeout\_%`,
  `now`, or the `\_site\_transient\_` forms): `DELETE a, b FROM {options} a, {options} b WHERE a.option_name LIKE ? AND
  a.option_name NOT LIKE ? AND b.option_name = CONCAT( '_transient_timeout_', SUBSTRING( a.option_name, 12 ) ) AND
  b.option_value < ?` (site form: `'_site_transient_timeout_'`, `17`); sitemeta: the same over `meta_key`/`meta_value`
  of `{sitemeta}`. Returns rows affected. If the multi-table `DELETE` does not parse, stop and report (the data
  layer's no-bypass rule).

- [ ] **Step 1: Write the failing test** `purges_only_expired_pairs` (MySQL; options rows: `_transient_a` +
  timeout 999, `_transient_b` + timeout 2000, `_transient_c` with no timeout, `_site_transient_d` + timeout 999;
  `purge_options(Transient, 1000)` leaves `b`, its timeout, `c` and `d`; `purge_options(SiteTransient, 1000)` removes
  `d`'s pair) and `purges_sitemeta` (the same on `wp_sitemeta`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-sql --test expired -- --ignored` — Expected: FAIL.
- [ ] **Step 3: Implement** `SqlExpiredTransients`.
- [ ] **Step 4: Run** it — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options-sql): purge expired transients"`

### Task 7: Registered settings (`SettingsRegistry`)

**Files:**
- Create: `crates/speedyspec-wp-options/src/settings.rs`, `src/settings_messages.rs`;
  `crates/speedyspec-wp-options-memory/src/settings.rs`
- Test: `crates/speedyspec-wp-options-memory/tests/settings.rs`

**Interfaces:**
- Consumes: `HookApi`, `Callback`, `HookValue`, `Deprecations`, `OptionsApi` (test only).
- Produces:
  - `pub trait SettingsStore { fn registered(&self) -> IndexMap<String, HookValue>; fn get(&self, name: &str) ->
    Option<HookValue>; fn put(&self, name: &str, args: HookValue); fn remove(&self, name: &str); fn allow(&self, group:
    &str, name: &str); fn disallow(&self, group: &str, name: &str); }` (`allow` appends as `$a[$group][] = $name`;
    `disallow` is `array_search` strict then `unset`, leaving PHP's index holes); `MemorySettingsStore`.
  - `pub enum SettingArgsInput { Array(HookValue), Callable(Callback) }`; `pub const FILTER_DEFAULT_OPTION: &str =
    "filter_default_option";`.
  - `pub struct SettingsRegistry` with `new(store: Rc<dyn SettingsStore>, hooks: Rc<HookApi>, deprecations: Rc<dyn
    Deprecations>)`, `register_setting(&self, group: &str, name: &str, args: SettingArgsInput) -> Result<(),
    OptionsError>`, `unregister_setting(&self, group: &str, name: &str, deprecated: Option<Callback>) -> Result<(),
    OptionsError>`, `registered_settings(&self) -> IndexMap<String, HookValue>`, `filter_default_option(&self,
    default: HookValue, option: &str, passed_default: bool) -> HookValue`.
  - Messages bound by `register_settings_handlers(container: &Container, api: Rc<SettingsRegistry>)`:
    `RegisterSetting`, `UnregisterSetting`, `GetRegisteredSettings`, `FilterDefaultOption`.
- Behaviour: `option.php` 2997–3256. Defaults in order `type 'string'`, `group`, `label ''`, `description ''`,
  `sanitize_callback null`, `show_in_rest false`; a callable becomes `['sanitize_callback' => cb]`;
  `register_setting_args(args, defaults, group, name)` runs on the raw args, then `wp_parse_args` (defaults' keys
  first, then the args' other keys in their order); an `array` type with `show_in_rest` lacking `schema.items` calls
  `doing_it_wrong("register_setting", <WordPress's message>, "5.4.0")`; `misc` → `general` (deprecated `3.0.0`),
  `privacy` → `reading` (`3.5.0`); `allow`; a non-empty `sanitize_callback` is added to `sanitize_option_{$name}`
  (10, 1); a `default` key adds `Callback::rust(FILTER_DEFAULT_OPTION, ..)` to `default_option_{$name}` (10, 3);
  `register_setting(group, name, args)` fires; `put`. Unregister mirrors lines 3109–3190 (`4.7.0` deprecation for a
  passed callback; `unregister_setting(group, name)` only when registered).

- [ ] **Step 1: Write the failing tests:** `defaults_merged_in_order` (args `['default' => 'x']` gives keys `type,
  group, label, description, sanitize_callback, show_in_rest, default`); `callable_becomes_sanitize_callback`
  (`apply_filters('sanitize_option_n', 'smart')` is the callback's result); `default_filter_registered`
  (`has_filter('default_option_n', 'filter_default_option') == 10`); `filter_default_option_rules` (passed default
  kept; unregistered kept; registered gives its default); `get_option_uses_registered_default` (over `OptionsApi`:
  `get_option("n")` is `"x"`, `get_option("n", "y")` is `"y"`); `args_filter_sees_raw_args`;
  `deprecated_groups` (`misc` stored under `general`, `privacy` under `reading`, each one `deprecated_argument`);
  `array_type_without_items_is_doing_it_wrong`; `unregister_removes_filters_and_fires`; `unregister_unknown_is_silent`
  (no action, no error); `allowed_keeps_php_indices` (register a, b, c in one group, unregister b → indices `0, 2`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-memory --test settings` — Expected: FAIL.
- [ ] **Step 3: Implement** the port, the memory store, the registry and the messages.
- [ ] **Step 4: Run** the test — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options): register_setting and registered settings"`

### Task 8: The PHP surface (`speedyspec-wp-options-php`)

**Files:**
- Modify: `crates/speedyspec-wp-options-php/src/lib.rs` (area wiring), create `src/transients.rs`, `src/network.rs`,
  `src/settings.rs`, `src/ports.rs` (new PHP adapters), `src/settings_store.rs`; the area's PHP bootstrap file
- Modify (wordpress-develop, plan 4's mechanism): `packages/wp-option/wp-option-legacy/functions/option.php`
- Test: `crates/speedyspec-wp-options-php/tests/php.rs`

**Interfaces:**
- Consumes: Tasks 2–7 messages; `OptionsArea`.
- Produces: native functions with WordPress's signatures and defaults: `get_transient($transient)`,
  `set_transient($transient, $value, $expiration = 0)`, `delete_transient($transient)`,
  `delete_expired_transients($force_db = false)`, `get_site_transient`, `set_site_transient`, `delete_site_transient`
  (same shapes), `get_site_option($option, $default_value = false, $deprecated = true)`, `add_site_option($option,
  $value)`, `update_site_option($option, $value)`, `delete_site_option($option)`, `get_network_option($network_id,
  $option, $default_value = false)`, `add_network_option($network_id, $option, $value)`,
  `update_network_option($network_id, $option, $value)`, `delete_network_option($network_id, $option)`,
  `wp_prime_site_option_caches(array $options)`, `wp_prime_network_option_caches($network_id, array $options)`,
  `wp_load_core_site_options($network_id = null)`, `register_setting($option_group, $option_name, $args = array())`,
  `unregister_setting($option_group, $option_name, $deprecated = '')`, `get_registered_settings()`,
  `filter_default_option($default_value, $option, $passed_default)`. PHP adapters: `PhpExternalObjectCache`
  (`wp_using_ext_object_cache()`), `is_main_site()`/`is_main_network()` on the PHP `Multisite` adapter,
  `PhpSettingsStore` (reads and writes `$wp_registered_settings` and `$new_allowed_options`; the bootstrap binds
  `$GLOBALS['new_whitelist_options'] = &$GLOBALS['new_allowed_options']`). `register_setting`'s `$args`: callable →
  `Callable`; array → `Array`; anything else through `wp_parse_args` first.

- [ ] **Step 1: Write the failing tests** (embedded PHP, as `speedyspec-wp-hooks-php/tests/php.rs`):
  `signatures_match_wordpress` (`ReflectionFunction` parameter names and defaults for every function above);
  `transient_roundtrip` (`set_transient('t', ['a' => 1], 60)`, `get_transient('t') === ['a' => 1]`, `_transient_t` in
  `wp_options`); `ext_toggle_switches_storage` (`wp_using_ext_object_cache(true)`; `set_transient('u', 1)` writes no
  option row; `wp_cache_get('u', 'transient') === 1`); `site_option_single_site` (`add_site_option('s', 'v')`, row in
  `wp_options` with `autoload = 'off'`); `settings_globals` (after `register_setting('g', 'n', ['default' => 'x'])`:
  `$wp_registered_settings['n']['default'] === 'x'`, `$new_whitelist_options === $new_allowed_options`,
  `in_array('n', $new_allowed_options['g'], true)`); `register_initial_settings_runs` (still PHP; registers
  `blogname` with `show_in_rest['name'] === 'title'`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-options-php --test php` — Expected: FAIL.
- [ ] **Step 3: Implement** the native functions (handlers resolved once, as the Hook API's are), the adapters and the
  store; remove the now-native functions from `option.php`, leaving the user-settings functions and
  `register_initial_settings()`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options-php` — Expected: PASS, including plan 4's tests.
- [ ] **Step 5: Commit** — `git commit -m "feat(wp-options-php): native transients, network options and settings"` (and,
  in wordpress-develop, `git commit -m "refactor: remove option.php functions the Rust port replaces"`).

### Task 9: Recorded compatibility and WordPress's tests on MySQL

**Files:**
- Modify: `compat/options/scenarios.json`, `compat/options/breaks.json`; the compat recorder's options driver
  (multisite flag); `crates/speedyspec-wp-options-php/tests/wordpress.rs` (file lists and runs)
- Create: `compat/options/wordpress-<version>-multisite.json` (recorded)

**Interfaces:**
- Consumes: Tasks 1–8; plan 4's recorder and replay test.
- Produces: scenarios (single site unless marked *ms*): `transient_basics`, `transient_serialized`,
  `transient_with_timeout`, `transient_expired_read_deletes`, `transient_add_timeout_to_existing`,
  `transient_ext_object_cache`, `transient_installing`, `transient_hooks_order`, `transient_long_name_fails` (a
  180-byte name), `transient_odd_expirations`, `site_transient_single_site`, `site_transient_core_no_timeout`,
  `site_option_fallback`, `network_option_bad_network_id`, `network_option_update_same_object`,
  `delete_expired_transients_single`, `register_setting_defaults`, `register_setting_callable`,
  `register_setting_deprecated_groups`, `unregister_setting_removes_default`, `filter_default_option_passed`;
  *ms*: `network_option_per_network`, `network_option_notoptions`, `network_option_hooks_order`,
  `site_transient_multisite`, `delete_expired_transients_multisite`, `prime_network_option_caches`. A scenario with
  `"multisite": true` is recorded on a multisite install into the `-multisite` fixture.

- [ ] **Step 1: Add the scenarios and record** both fixtures against WordPress on the test MySQL (the recorder's
  command from plan 4, once single site and once with `--multisite`).
- [ ] **Step 2: Run** the replay (`cargo test -p speedyspec-wp-options-php --test compat -- --ignored`) — Expected:
  FAIL before any fix, or PASS; fix every difference in the owning task's code, or list it in `breaks.json` with the
  reason.
- [ ] **Step 3: Add to `wordpress.rs`** three runs on MySQL: single site with `transient.php`, `siteTransient.php`,
  `siteOption.php`, `networkOption.php`, `registration.php`, `wpPrimeNetworkOptionCaches.php`; multisite
  (`WP_TESTS_MULTISITE=1`, `ms-required` included) with the same files plus `multisite.php`; and single site with
  `object-cache` routed to a `file://` engine in a temp dir (`transient.php`, `siteTransient.php`, `option.php`).
  `userSettings.php` and `wpUserSettings.php` stay out (user settings wait for the auth port).
- [ ] **Step 4: Run** `cargo test -p speedyspec-wp-options-php --test wordpress -- --ignored` — Expected: PASS; any
  `@group documented-break` test is listed in `breaks.json`.
- [ ] **Step 5: Run** `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --check` — Expected: clean.
- [ ] **Step 6: Commit** — `git commit -m "test(options): recorded compatibility and WordPress tests for phases 3 and 4"`

### Task 10: Hand-over note for the runtime's owner

**Files:**
- Create: `/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/2026-11-19-persistent-object-cache-worker-request.md`

- [ ] **Step 1: Write the note**, asking `speedyspec-php-runtime`'s worker to: (1) at worker start, call
  `PersistentCacheSelector::new().select(&registry, OBJECT_CACHE_USE)` and give the result to the worker's
  `ObjectCache` through `with_persistent`, keeping it for the worker's life; (2) at each request start, before
  `wp-settings.php`, set `$GLOBALS['_wp_using_ext_object_cache'] = true` when `is_persistent()`; (3) at request end,
  `flush_runtime()` the first level; (4) build the `Multisite` Rust adapter's network and site ids from site
  configuration; (5) note that a `redb://` route needs plan 7's owner process running, and that the `SqlCache` table
  must exist before the first request. Name this branch and the commits it depends on.
- [ ] **Step 2: Check** — the note exists and no file under `crates/speedyspec-php-runtime` changed
  (`git diff --stat origin/main -- crates/speedyspec-php-runtime` is empty).
