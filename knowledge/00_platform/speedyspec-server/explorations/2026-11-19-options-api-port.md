# Exploration: The options API port

**Status:** built in part
**Created:** 2026-11-19
**Author:** Claude, for the project owner
**Context:** Port WordPress's options API, and the object cache it reads through, to Rust, with every hook unchanged, so Rust features read options without entering PHP.
**Approved by:** the project owner on 2026-11-19
**Socialized with:** not socialized

---

## Problem Statement

The firewall, the response cache and later native responders need their settings. In WordPress those are options,
read through `get_option()`, which goes through the object cache and `$wpdb`. Reading one from Rust meant entering
PHP. Options also carry many hooks that plugins depend on, and their values are in PHP's serialize format, so a port
had to match WordPress exactly or break plugins. Every read path calls `wp_cache_get`, so the object cache could not be
left behind.

## Current State

- `option.php`, `cache.php` and `class-wp-object-cache.php` ran as WordPress's PHP in the wordpress-develop fork.
- Two PHP package sets existed in the fork: `packages/wp-option/*` (working use cases and a facade over WordPress) and
  `packages/speedyspec-wp-option-*` (mostly contracts with stubbed repositories).
- The Hook API was ported (speedyspec-server #4, #5). The database layer and `wpdb` port were being built in parallel
  (see the database layer exploration).

## Proposed Solution

### Architecture

```
  PHP callers: get_option(), wp_cache_get() ...       Rust features: Options facade
        |                                                  |
  -wp-options-php, -wp-object-cache-php (native functions) |
        |                                                  v
        +-------> speedyspec-wp-options (OptionsApi, NetworkOptionsApi, TransientsApi, SettingsRegistry)
                     |                    |                         |
            OptionRepository     ObjectCache (2 levels)       ports to unported WordPress
            SitemetaRepository     | first: PHP-shaped values    (installing, multisite, sanitizing,
              (SQL via WpDb,       | second: CachingStore bytes   deprecations, protected names,
               memory)             v                              external object cache, settings store)
                              SqlCache / KvCache / FileCache,
                              chosen from the "object-cache" engine route
```

### Data Model

- `Option { name: OptionName, value: SerializedValue, autoload: Autoload }`. The name is trimmed and rejects empty
  and `"0"`; `option_id` is kept by the mapper, not the entity.
- `Autoload`: WordPress 6.6's `on`, `off`, `auto`, `auto-on`, `auto-off`, plus legacy `yes` and `no`.
- `SerializedValue`: the stored bytes (Serialized LOB), decoded on demand by `speedyspec-php-serialize`.
- `NetworkMeta { network_id, key, value }` for `wp_sitemeta`.
- Cache layout exactly as WordPress: group `options` with `alloptions`, `notoptions` and single names;
  `site-options`, `transient` and `site-transient` groups.

### Interfaces

- `OptionRepository`: one method per `$wpdb` call `option.php` makes, so query counts match WordPress's.
- `OptionsApi::hooks_fired_by_get(&OptionName) -> Vec<HookName>`: the hooks a read could fire, listed without
  firing any, for the future PHP-callback registry. It includes `query` when the read would reach the database.
- `Options::get::<T>(name)`: the Rust facade for runtime features.
- `stage_add`, `stage_update`, `stage_delete`: option writes through a unit of work.

## Design Decisions

1. **Port the object cache here, as its own area.** Every read path goes through it (designer's call).
   *Trade-off:* a second area, with its own native PHP surface and compatibility catalogue, inside this port.
2. **Build on neither PHP package set as code.** Their names informed the Rust names; both are deleted with
   `option.php` (designer's call; [ADR 0014](../../adr/0014-port-wordpress-to-rust-and-delete-its-php.md)).
   *Trade-off:* working PHP use cases are thrown away.
3. **The first cache level holds values, not bytes** (designer's call). A PHP `wp_cache_set` then `wp_cache_get` never
   serializes, and closures can be cached. *Trade-off:* PHP values must be released at request end, before PHP shuts
   down.
4. **The second level stores PHP's serialize format**, so PHP and Rust read the same entries. A value `serialize()`
   refuses stays in the first level and the miss is logged, as persistent drop-ins behave.
   *Trade-off:* a full PHP-compatible serializer in Rust, tested against PHP's own output.
5. **Hooks fire through the Hook API's empty-hook fast path.** With no PHP callback, a Rust read stays in Rust.
   *Trade-off:* each hook site checks for callbacks before building arguments.
6. **Option SQL goes through `wpdb`'s statement entry**, so `query` fires and `num_queries` counts as in WordPress
   (WordPress's tests assert query counts). *Trade-off:* Rust option reads pass through the `wpdb` port.
7. **Unit-of-work writes for Rust callers** (designer's call): filters run when a change is registered, actions and
   cache invalidation only after a successful commit. PHP functions keep write-at-once.
   *Trade-off:* two write paths to keep equal.
8. **Unported WordPress sits behind ports** with PHP and Rust adapters
   ([ADR 0015](../../adr/0015-ports-and-adapters-with-cqrs-use-cases.md),
   [ADR 0017](../../adr/0017-php-twins-and-the-override-flag.md)). A Rust write of a core option name that needs
   `sanitize_option()` must go to PHP. *Trade-off:* the project's own options must use prefixed names to stay native.
   Building revealed a sixth port the spec lacked, for the setting groups `register_setting()` fills.
9. **Persistent means routed** (phase 3 decision). The object cache is persistent exactly when the site routes the
   `object-cache` use to an engine; a selector builds `SqlCache`, `KvCache` or `FileCache` from it.
   *Trade-off:* no persistence without engine configuration; `object-cache.php` drop-ins are refused.
10. **`wp_using_ext_object_cache()` and the settings globals stay in PHP.** `load.php` is not ported, and
    `$wp_registered_settings` stays the real store so PHP code reading it keeps working. REST settings parts stay PHP.
    *Trade-off:* more ports to unported code.
11. **A failing second level degrades.** Reads miss, writes are dropped, the database answers. As built, the first
    failure turns the second level off for the rest of that request, so an unreachable host costs one timeout.
    *Trade-off:* a request may run without persistence after one failure.

## Implementation Phases

1. **Object cache, request-lifetime.** `speedyspec-php-serialize`, the two-level `ObjectCache`, native `wp_cache_*`
   and `WP_Object_Cache`, the `object-cache` catalogue, WordPress's `cache.php` tests. *Delivers:* a native object
   cache. Shipped in speedyspec-server #14.
2. **Single-site options.** The entity, ports, autoload policy and size rule, read and write paths, CQRS messages,
   unit-of-work writes, SQL and memory repositories, native functions, the `options` catalogue, WordPress's option
   tests on MySQL. *Delivers:* `get_option()` and fifteen more functions in Rust. Shipped in #15.
3. **Persistent second level.** Engine-routed selection of the Caching Store; cross-process tests. Shipped in #19.
4. **Transients, site and network options, registered settings.** Shipped in #19, with 22 more native functions.
5. **Follow-ups.** Purge on SQLite and PostgreSQL (#22); priority deferred findings, including atomic `KvCache`
   counters, `FileCache` locking, settings writes that now throw, and a cross-process transients test (#25).
6. **Wiring and deletion.** The worker creates the areas, picks the second level at start and sets
   `$_wp_using_ext_object_cache`; the ported functions are removed from `option.php` and the object-cache files.
   *Delivers:* sites served by the port. Not shipped: `worker.rs` on main does not reference it, and `option.php`
   still defines `get_option` in the wordpress-develop fork.

## Fleet Impact

- **speedyspec-server:** new `speedyspec-wp-object-cache*`, `speedyspec-wp-options*`, `speedyspec-php-serialize` and
  `speedyspec-cache-routing` crates (confirmed by the merged PRs).
- **PHP runtime:** must add the object cache and options areas, choose the second level per worker, and tell the
  installing port when the installer runs. Not done yet (checked `worker.rs` on main).
- **wordpress-develop fork:** loses the ported functions in `option.php`, `cache.php` and `class-wp-object-cache.php`,
  then the PHP option packages. Not done yet (checked the fork).
- **Blog, front-end API and admin API pools:** share `alloptions` and `notoptions` through a persistent second level
  once wired. Unverified.
- **CLI, admin site:** no change found. Unverified.

## Open Questions

1. **When does the runtime wire the areas in?** Settled by the worker changes landing.
2. **When are `option.php` and the object-cache PHP files deleted?** After wiring, per
   [ADR 0021](../../adr/0021-recorded-compatibility.md); `option.php` then keeps only user settings and
   `register_initial_settings()`.
3. **User settings** wait for the auth port. Settled when that port exists.
4. **`sanitize_option()`** stays in PHP until `formatting.php` is ported. Settled by that port.
5. **The PHP-callback registry** that consumes `hooks_fired_by_get` belongs to the native features work and is not
   built. Settled when the firewall's native path is built.
6. The Minor review findings are listed in the [deferred findings](../../../../deferred/speedyspec-server/README.md).

## Spike Findings

**Settles:** whether a second process sees option and transient writes at once through a shared second level.
**Built:** cross-process tests that spawn a child test process to write while the parent reads after a new request,
counting SQL through the `query` filter (#19, transients added in #25).
**Established:** `update_option`, `add_option`, `delete_option` and transients written in one process are read in the
next request of another with no SQL, on the SQL, file and Valkey second levels.
**Does not establish:** behaviour on the ReDB second level, which these runs did not include.
**Consequence:** the design's cross-process test requirement is met for three backends.

## Success Criteria

- [x] The `object-cache` catalogue replays against WordPress's recording; WordPress's `cache.php` tests pass on single
  site and multisite.
- [x] The `options` catalogue replays; WordPress's option tests pass on MySQL.
- [x] Phases 3–4: 113 recorded scenarios, single site and multisite; option test runs pass with 0 failures (258
  options, 71 network options, 105 multisite, 97 with a persistent cache) (#19).
- [x] Two processes sharing a persistent Caching Store see each other's `update_option` at once.
- [x] A Rust read of a cached option with no PHP callbacks does not cross into PHP (boundary-crossing test).
- [x] Expired-transient purge works on MySQL, SQLite and PostgreSQL (#22).
- [ ] Option suites also run on SQLite and PostgreSQL: the engine plans made this conditional; not confirmed.
- [ ] Sites are served by the native functions (needs wiring and deletion).

---

## Non-Goals

- **Changing option names, the `wp_options` schema or the serialized format** — plugins depend on them.
- **Options stored on ReDB, Valkey or files** — `wp_options` is a WordPress table, so it stays on SQL; only the cache
  may live elsewhere.
- **Porting `sanitize_option()`** — it belongs to `formatting.php`.
- **A second cache of options** — cache-purging code must keep working.
- **API compatibility with the existing PHP option packages** — reference only.

## Alternatives Considered

### Option: Wait for someone else to port the object cache
- **Pros:** a smaller options port.
- **Cons:** options cannot be faithful without it; nothing else would port it first.
- **Verdict:** rejected; ported here.

### Option: Build on `packages/wp-option/*`
- **Pros:** working use cases already exist.
- **Cons:** PHP code over WordPress, not a Rust port.
- **Verdict:** names reused, code not.

### Option: Build on `packages/speedyspec-wp-option-*`
- **Pros:** contracts already drafted.
- **Cons:** mostly stubs.
- **Verdict:** names reused, code not.

## Dependencies

- The database layer and `wpdb` port (speedyspec-server #12, #13), for `wp_options` access and the `query` hook.
- The Caching Store and the NoSQL and file stores (#14, #18), for the persistent second level.
- The Hook API, CQRS buses, container and recorded compatibility (#4, #5).
- [ADR 0003](../../adr/0003-native-features-wait-for-api-ports.md): options is the second half of step 1.

## References

- [ADR 0003](../../adr/0003-native-features-wait-for-api-ports.md), [ADR 0009](../../adr/0009-database-port-requirements.md), [ADR 0014](../../adr/0014-port-wordpress-to-rust-and-delete-its-php.md), [ADR 0015](../../adr/0015-ports-and-adapters-with-cqrs-use-cases.md), [ADR 0017](../../adr/0017-php-twins-and-the-override-flag.md), [ADR 0021](../../adr/0021-recorded-compatibility.md)
- [Database layer exploration](2026-11-19-database-layer.md)
- [Deferred findings](../../../../deferred/speedyspec-server/README.md)
- [SpeedySpec/speedyspec-server](https://github.com/SpeedySpec/speedyspec-server): [#14](https://github.com/SpeedySpec/speedyspec-server/pull/14), [#15](https://github.com/SpeedySpec/speedyspec-server/pull/15), [#19](https://github.com/SpeedySpec/speedyspec-server/pull/19), [#22](https://github.com/SpeedySpec/speedyspec-server/pull/22), [#25](https://github.com/SpeedySpec/speedyspec-server/pull/25)
- Source spec: `2026-11-19-options-api-port-design.md`
- Source plans: `2026-11-19-options-api-port.md`, `2026-11-19-persistent-cache-transients-network-options.md`, `2026-11-19-caching-store-and-object-cache.md`
