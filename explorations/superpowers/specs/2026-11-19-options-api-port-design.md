# Options API port design

Date: 2026-11-19
Status: approved 2026-11-19

Step 1 of the port order in `2026-11-19-native-plugin-features-design.md`, second half. Builds on
`2026-11-19-database-layer-design.md`.

## Intent

Port WordPress's options API (`option.php`) to Rust, with the object cache it relies on, so that Rust code in the
runtime (first the firewall's settings, then the response cache and the native responders) reads and writes options
without entering PHP, and PHP plugins keep calling `get_option()` and the rest unchanged.

## Goals

- `get_option`, `get_options`, `add_option`, `update_option`, `delete_option`, `wp_load_alloptions`,
  `wp_prime_option_caches`, `wp_prime_option_caches_by_group`, `wp_set_option_autoload_values`,
  `wp_set_options_autoload`, `wp_set_option_autoload`, `wp_determine_option_autoload_value`,
  `wp_filter_default_autoload_value_via_option_size`, `wp_autoload_values_to_autoload`, `wp_protect_special_option`,
  `form_option`: ported with WordPress's exact behaviour, as commands and queries.
- **Every hook unchanged,** in name, arguments and order: `pre_option_{$option}`, `pre_option`,
  `default_option_{$option}`, `option_{$option}`, `pre_wp_load_alloptions`, `pre_cache_alloptions`, `alloptions`,
  `pre_update_option_{$option}`, `pre_update_option`, `update_option`, `update_option_{$option}`, `updated_option`,
  `add_option`, `add_option_{$option}`, `added_option`, `delete_option`, `delete_option_{$option}`, `deleted_option`,
  `wp_autoload_values_to_autoload`, `wp_default_autoload_value`, `wp_max_autoloaded_option_size`.
- **Autoload as WordPress 6.6+ has it:** `on`, `off`, `auto`, `auto-on`, `auto-off`, with legacy `yes` and `no` read
  as `on` and `off`; the 150000-byte size rule.
- **`alloptions` and `notoptions`** cached in the object cache's `options` group, as WordPress does, so a persistent
  Caching Store (Valkey, ReDB, SQL, files) shares them across workers and pools.
- **The object cache ported** (`wp_cache_*` and `WP_Object_Cache`) over the Caching Store, because options depend on
  it and nothing else would port it first.
- Plain objects: `Option` is a plain struct (and a plain PHP object for PHP callers that want one), loaded and saved
  by an `OptionMapper` and an `OptionRepository`.

## Non-goals

- Changing option names, the `wp_options` schema or the serialized format.
- Options on ReDB, Valkey or a Direct Store. `wp_options` is a WordPress table, so it lives on the SQL Store (project
  owner's decision in the database design). Only the *cache* of options may live elsewhere.
- Porting `sanitize_option()` (it belongs to `formatting.php`, which is not ported); see "Sanitizing".
- In this design's first phases: transients, site and network options, user settings, `register_setting()`. See
  "Phases".
- Compatibility with the two existing PHP option package sets' APIs. Their names and entities are reference only.

## Anti-goals

- A Rust caller that enters PHP to read an option when no PHP callback is on any hook the read fires.
- A second cache of options beside the object cache. `alloptions`, `notoptions` and single options live where
  WordPress keeps them, so cache-purging code that deletes them (`wp_cache_delete( 'alloptions', 'options' )`)
  still works.
- An option entity that saves or loads itself.

## Constraints

- Everything in the database design's Constraints.
- Option values are PHP-serialized (`maybe_serialize`). Rust must read and write that format itself (see "Values").
- `option.php` calls functions that are not ported: `wp_installing()`, `is_multisite()`, `sanitize_option()`,
  `_deprecated_argument()`, `_doing_it_wrong()`, `wp_protect_special_option()`'s `wp_die()`. Each is an outbound port
  of the options area with a PHP adapter calling WordPress's function, and a Rust adapter where Rust can answer alone
  (see "Ports to unported code").

## Approaches considered

This port follows the Hook API's pattern, and the project owner fixed the database choices in the database design, so
there was one sensible way. Two smaller choices were the designer's:

- **Port the object cache here, or wait.** Options cannot be ported faithfully without it (every read path goes through
  `wp_cache_get`). **Chosen: port it here**, as its own area.
- **Build on which PHP package set.** `packages/wp-option/*` has working use cases and a facade over WordPress;
  `packages/speedyspec-wp-option-*` is mostly contracts with stubbed repositories. **Chosen: neither as code.** Their
  entity and use-case names inform the Rust names; both are deleted with `option.php` once the port's compatibility
  tests pass.

## Design

### Crates

| Crate | Holds |
|---|---|
| `speedyspec-wp-object-cache` | The object cache API (`wp_cache_*`, `WP_Object_Cache`'s behaviour: groups, global groups, non-persistent groups, blog-id key prefixes, `wp_cache_supports`) as commands and queries over the Caching Store. |
| `speedyspec-wp-object-cache-php` | Native `wp_cache_*` functions and a native `WP_Object_Cache` class; the `$wp_object_cache` global. |
| `speedyspec-wp-options` | The options API: commands, queries, domain services, outbound ports (`OptionRepository`, the ports to unported code). |
| `speedyspec-wp-options-sql` | `SqlOptionRepository` and `OptionMapper` over a `SqlStore` (`wp_options` through the builder). |
| `speedyspec-wp-options-memory` | `MemoryOptionRepository`, for tests. |
| `speedyspec-wp-options-php` | Native `get_option()` and the rest; PHP adapters for the ports to unported code. |
| `speedyspec-php-serialize` | PHP's `serialize`/`unserialize` format as a domain service, shared by options, the object cache and later meta. |

### Patterns, and where they are used

| Pattern | Where |
|---|---|
| **Repository** | `OptionRepository`: `find(name)`, `find_many(names)`, `autoloaded(values)`, `by_group`… , `add`, `update`, `remove`, `set_autoload(names, value)`. SQL and memory implementations. |
| **Data Mapper** | `OptionMapper`: `wp_options` rows to `Option` and back. |
| **Serialized LOB** | `option_value` holds PHP-serialized data; the mapper keeps it as bytes and `speedyspec-php-serialize` decodes it on demand. |
| **Unit of Work** | Rust callers that change several options (the firewall's settings save, later the admin API's settings) register them in the request's unit of work and commit once. The PHP functions keep WordPress's write-at-once behaviour. |
| **Identity Map** | Within a unit of work, one `Option` per name. |
| **Decorator** (two-level cache) | The object cache: a request-lifetime `MemoryCache` in front of an optional persistent Caching Store, as `WP_Object_Cache` plus a persistent drop-in behave. Non-persistent groups stop at the first level. |
| **Strategy** | `AutoloadPolicy` (`wp_determine_option_autoload_value` and its filters); the Caching Store backend. |
| **Command / CQRS** | Every function a command or query, handlers bound in the container as in the Hook API port. |
| **Adapter** | PHP adapters for the ports to unported code; the native functions over the commands. |
| **Facade** | `OptionFacade`-like Rust convenience for runtime features (`options.get::<T>(name)`), over the queries. |
| **Observer** | Every option hook, through the Hook API. |

### The entity and values

- `Option { name: OptionName, value: SerializedValue, autoload: Autoload }`. `OptionName` trims and rejects empty
  names as WordPress does. `Autoload` is the 6.6 set plus legacy `yes`/`no`. `option_id` is the identity field the
  mapper keeps and the entity ignores.
- `SerializedValue` is the stored bytes. `speedyspec-php-serialize` turns it into a PHP-shaped value (the Hook API's
  `Value`) for Rust callers. Serialized objects cannot become PHP objects in Rust; they stay opaque and are
  materialized only when the value crosses into PHP. Rust code reading its own options uses scalars and arrays, so it
  never meets one.
- `maybe_serialize` and `maybe_unserialize` are ported with WordPress's rules (including `is_serialized` and the
  double-serialization of already-serialized strings).

### Read path (`get_option`), as WordPress has it

1. Deprecated option names (`blacklist_keys`, `comment_whitelist`) are mapped, with `_deprecated_argument` unless
   installing.
2. Empty name: false.
3. `pre_option_{$option}`, then `pre_option`: a non-false result is returned as is.
4. Unless installing: `wp_load_alloptions()`; a hit is the value.
5. Otherwise `notoptions` in the cache: a hit returns the filtered default (`default_option_{$option}`).
6. Otherwise `wp_cache_get( $option, 'options' )`; otherwise the repository; found values are cached; missing names
   are added to `notoptions` and return the filtered default.
7. `home`, `siteurl` and `upload_path`-like special cases as WordPress has them.
8. `option_{$option}` on the unserialized value.

`wp_load_alloptions` runs `pre_wp_load_alloptions`, then the cache's `alloptions`, then the repository's autoloaded
options (the values from `wp_autoload_values_to_autoload`), then `pre_cache_alloptions` before caching, then
`alloptions`.

### The object cache's two levels

- **The first level holds values, not bytes** (designer's call). It is a request-lifetime store of PHP-shaped values,
  with `WP_Object_Cache`'s semantics: objects are cloned on set and on get, and any value PHP can hold is accepted,
  closures included. A PHP value that Rust has not converted stays a PHP value (the Hook API's opaque value), so a
  PHP `wp_cache_set` followed by `wp_cache_get` never serializes.
- **The second level is the persistent Caching Store, in bytes.** A value is written there in PHP's `serialize`
  format, through `speedyspec-php-serialize`, so PHP and Rust read the same entries (`alloptions` and `notoptions`
  included). A value `serialize()` refuses stays in the first level only, and the miss is logged, as the persistent
  object-cache drop-ins behave.
- Non-persistent groups never reach the second level.

### Write paths

`add_option`, `update_option` and `delete_option` follow WordPress's order of checks, hooks, writes and cache updates
exactly; the recorded scenarios are the specification. Notes:

- **Through a unit of work** (Rust callers only; designer's call): the filters (`pre_update_option_{$option}`,
  `pre_update_option`) run when the change is registered, because they decide the value written. The actions
  (`add_option`, `update_option`, `delete_option` and their named and past-tense forms) and the cache invalidation run
  after a successful commit, in registration order, so no callback sees an uncommitted value. If the commit fails,
  no action fires and the cache is untouched. Options all live on one engine, so a partial commit cannot split them.

- `add_option` writes with `INSERT ... ON DUPLICATE KEY UPDATE`, as core does. The builder produces it and the
  database layer translates it for PostgreSQL and SQLite; this is the first real user of that pass.
- Each write invalidates exactly the cache entries WordPress touches (`alloptions`, `notoptions`, the single option),
  so a persistent Caching Store stays right across workers and pools.

### Ports to unported code

| Port | Rust adapter | PHP adapter |
|---|---|---|
| `Installing` (`wp_installing()`) | false outside the installer; the runtime knows when it runs the installer | calls `wp_installing()` |
| `Multisite` (`is_multisite()`, current blog and network ids) | from site configuration | calls WordPress |
| `Sanitizing` (`sanitize_option()`) | handles names `sanitize_option()` has no case for: it only fires `sanitize_option_{$option}` for those | calls `sanitize_option()` for core option names until `formatting.php` is ported |
| `Deprecations` (`_deprecated_argument`, `_doing_it_wrong`) | logs | calls WordPress |
| `ProtectedOptions` (`wp_protect_special_option`) | the fixed list (`alloptions`, `notoptions`) and an error | calls `wp_die()` |

A Rust native path that writes a core option name needing sanitizing must therefore go to PHP; the project's own
option names (prefixed) never do. The firewall and cache only write their own.

### Native paths and PHP callbacks

A native path (for example the firewall reading its settings in stage 1) asks the worker's hook registry (native
features design, "PHP callbacks on native paths") before reading. The options area exposes, per call, the hooks it may
fire (`pre_option_{$name}`, `pre_option`, `default_option_{$name}`, `option_{$name}`, and the `alloptions` hooks when
`alloptions` is not yet loaded). A read that would reach the database also fires `query`, because the repository
goes through the `wpdb` port (database design, "Two entry points, one path"), so `query` is on the list for such a
read. With a persistent object cache that read rarely happens. If none of the hooks has a PHP callback, the read stays
in Rust. If any does, the request goes to PHP.

**The registry itself is not built here.** It belongs to the native features work. Until it exists, the options area
offers its list of hooks, and the native paths that will use it (the firewall first) do not exist yet either.

### PHP surface

- Native functions with WordPress's signatures and defaults for every function in Goals, dispatching through handlers
  resolved once, as the Hook API's do.
- `option.php` (`packages/wp-option/wp-option-legacy/functions/option.php`) keeps only the functions of later phases
  until those are ported; the phase-1 functions are removed from it, then the file is deleted.
- The object cache's native functions replace `cache.php` and `class-wp-object-cache.php` in
  `packages/wp-object-cache/wp-object-cache-legacy`. An `object-cache.php` drop-in is not loaded; a site that has one
  gets an error naming it (persistence comes from configuring the Caching Store instead).

## Phases

1. Object cache over `MemoryCache` (request lifetime only, as WordPress's default), native functions, recorded
   compatibility.
2. Single-site options (everything in Goals) over `SqlOptionRepository`; recorded compatibility; WordPress's
   `tests/phpunit/tests/option/` suite.
3. A persistent Caching Store behind the object cache (configurable: SQL, Valkey, ReDB, files).
4. Transients (on the object cache when it is persistent, otherwise on options, as WordPress decides), site and
   network options, `register_setting()` and registered settings. User settings (cookie-based) wait for the auth port.

## Testing

- Recorded compatibility: an `options` catalogue (every function, every hook, autoload values old and new, the size
  rule, `notoptions` behaviour, deprecated names, protected names, serialized values of every shape) recorded against
  WordPress and replayed against the port; an `object-cache` catalogue the same way.
- WordPress's own `tests/phpunit/tests/option/` and `tests/phpunit/tests/cache.php` against the native functions.
  **A phase is done when it passes on MySQL** (designer's call), which the database design's step 1 provides. The
  same suites are added to the SQLite and PostgreSQL runs when the database design's steps 3 and 4 land; `add_option`'s
  upsert is then the first case the upsert translation must pass.
- Cross-process: two workers sharing a persistent Caching Store see each other's `update_option` at once.

## Left to the builder

- Exact command and query names, following the Hook API port's naming.
- The `Facade` convenience's shape for runtime features.
- How `notoptions` and `alloptions` are keyed in non-SQL caches.
