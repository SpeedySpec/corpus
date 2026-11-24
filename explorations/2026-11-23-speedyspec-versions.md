# Exploration: SpeedySpec versions, from WordPress-compatible to SpeedySpec's own APIs

**Status:** decided; nothing built
**Created:** 2026-11-23
**Author:** Claude, for the project owner
**Context:** What each SpeedySpec major promises plugins, how SpeedySpec 1 stays WordPress, and how later majors
replace WordPress's old systems.
**Approved by:** the project owner, in discussion on 2026-11-23
**Socialized with:** not socialized

---

## Problem Statement

SpeedySpec is a Rust port of WordPress, with plugins still written in PHP. It serves two audiences whose needs
conflict:

- **Site owners** want to put SpeedySpec on an existing WordPress install and have every plugin they run keep working,
  through every upgrade.
- **The project** wants plugins moved onto SpeedySpec's own APIs (blocks in place of widgets, rebindable ports, the
  detached admin), so that WordPress's old systems and deprecated APIs can eventually go, and tables can be changed.

Nothing yet says which releases may break a plugin. Each component has its own SemVer, and none of those numbers says
what a plugin can rely on.

## Current State

- The components version separately: `speedyspec-cli` is on 1.x, `speedyspec-php-runtime` on 1.4.0, so SpeedySpec 1
  is already released, and `speedyspec-server` on 0.x. The admin micro-frontends exploration puts the admin on its own SemVer as well.
- [ADR-014](../knowledge/00_platform/adr/014-port-wordpress-to-rust-and-delete-its-php.md) ports WordPress to Rust and
  keeps observable behaviour, apart from documented breaks listed in each area's design and `breaks.json`.
- [ADR-021](../knowledge/00_platform/adr/021-recorded-compatibility.md) holds each port to WordPress 7.1.5's recorded
  behaviour.
- Several departures from WordPress already exist (see the register below). They are spread across area designs, and
  none says, from a plugin author's side, how to work with it.
- The widgets design keeps a Rust widget API that builds blocks, with an adapter for `WP_Widget` subclasses.

## Proposed Solution

Three majors, each with a different relation to WordPress. The rules are in
[ADR-028](../knowledge/00_platform/adr/028-speedyspec-versions.md); this exploration holds the reasoning, the plan for
each major, and the departures register.

### Architecture

```
 SpeedySpec 1                       SpeedySpec 2                      SpeedySpec 3
 ───────────────────────────        ───────────────────────────       ───────────────────────────
 WordPress API (all of it)          WordPress API (minus what 1       WordPress API (minus more
   = adapters ─────────┐              deprecated) = adapters ──┐        old-style APIs) = adapters ─┐
                       v                                        v                                    v
 SpeedySpec API (new, additive)     SpeedySpec API                    SpeedySpec API (some changed,
                       v                                        v       after 2 deprecated them)     v
 Rust ports, held to WordPress      Rust ports; tables migrated       Rust ports
 follows every WordPress release    follows WordPress where it fits   follows WordPress where it fits
```

At every major the new SpeedySpec API is the only implementation. The WordPress API a major still carries is a layer of
Adapters over it, held to WordPress's recorded behaviour. Each major removes the adapters the one before deprecated.

**What carries the promise.** Only the PHP runtime's version is the SpeedySpec version, because the runtime is what
plugins run against. The admin REST API (`wp-admin/v1`) and the blog API are SpeedySpec's own and already break
WordPress's shapes in places, so their endpoints are outside the promise. They are consumers of it instead: an admin
or blog API release declares the majors it supports, and must honour those majors' PHP APIs, so a menu page or setting
a plugin registers the WordPress way still appears and works.

### Data Model

- **Plugin header:** `Requires SpeedySpec: <SemVer range>`, for example `1 || 2`. Absent means `1`, because a plugin
  without it was written for WordPress.
- **Component manifests:** the admin REST API and the blog API state the SpeedySpec majors they support, in the same
  SemVer range form the release manifests already use for `requires`.
- **Deprecations:** each one names the API, the minor that deprecates it, the replacement, and the major that removes
  it. The SpeedySpec Guide shows them per symbol.
- **`breaks.json`:** stays per area ([ADR-021](../knowledge/00_platform/adr/021-recorded-compatibility.md)); every entry
  that a plugin can observe also appears in the register below.

### Interfaces

- A plugin whose `Requires SpeedySpec:` range excludes the running major is not activated, and the plugins screen says
  why, as WordPress does for `Requires at least`.
- A deprecated API fires WordPress's deprecation hooks (`deprecated_function_run`, `deprecated_class_run`,
  `deprecated_hook_run` and the rest), so the tools plugin authors already use report it.

## Design Decisions

1. **The runtime's major is the SpeedySpec version.** The project owner placed the WordPress promise in the runtime,
   and only there. A separate platform number that only the runtime implemented would be a second name for the same
   thing. *Trade-off:* a plugin-observable fix in a port crate in the server workspace is versioned as a runtime
   change.
2. **The admin REST API and the blog API are outside the promise, but must honour it.** The project owner: the admin
   API already breaks plugin compatibility in places, while the WordPress APIs it serves must keep working.
   *Trade-off:* each admin or blog API release has to be checked against the PHP APIs of every major it claims.
3. **Old APIs are adapters over the new ones.** The project owner: old-style APIs must still work as expected, but hook
   into the new APIs. *Trade-off:* the adapter has to reproduce WordPress's behaviour over an API shaped differently.
4. **SpeedySpec 1 follows WordPress for good.** The project owner chose open-ended tracking over a fixed window after
   SpeedySpec 2, expecting LLM-assisted porting to keep the work small. *Trade-off:* every WordPress release is ported
   into 1.x and checked against later majors, with no end date. WordPress's own changes, including its schema upgrades
   and deprecations, pass through in 1.x minors, because a WordPress site gets them too.
5. **Removal only after deprecation in the previous major** (designer). A plugin author sees every removal, and its
   replacement, a full major ahead. *Trade-off:* an API SpeedySpec wants gone lives at least one major longer.
6. **An undeclared plugin is a SpeedySpec 1 plugin** (designer). *Trade-off:* unconverted plugins do not run on
   SpeedySpec 2 until someone declares them, which is the push toward conversion.
7. **WordPress's own `wp/v2` REST API is outside the promise.** The project owner. Plugin routes registered with
   `register_rest_route()` keep working, because that is a PHP API; the shapes of core's `wp/v2` routes may change in
   a minor. *Trade-off:* a client written against stock `wp/v2` responses is not protected by SpeedySpec 1. The
   widgets design still keeps `wp/v2` widget shapes, as its own choice.
8. **Later majors' fix windows are decided per major.** The project owner: how long SpeedySpec 2 or 3 gets fixes once
   the next major ships depends on how hard the fixes are and how many installs still run it. *Trade-off:* no date a
   site owner can plan against in advance.
9. **The project converts plugins itself, with Claude.** The project owner plans to convert as many free, open-source
   WordPress plugins as possible, all of them if it can (about 30,000), onto the later majors' APIs, using Claude.
   *Trade-off:* conversions at that scale need an automated way to prove each one, per major it claims.

## Implementation Phases

### Phase 1: The register and the header

Publish the departures register below and keep it current with each area's design. Teach the runtime to read
`Requires SpeedySpec:`. Show deprecations and departures in the SpeedySpec Guide.

### Phase 2: Following WordPress

When WordPress releases, record it ([ADR-021](../knowledge/00_platform/adr/021-recorded-compatibility.md)), replay
every area, port the differences into a 1.x minor, and update the register.

### Phase 3: Deprecations in 1.x

As each SpeedySpec API that replaces an old one ships, deprecate the old one in a 1.x minor, naming the replacement.
Convert plugins with Claude: the project's own first, then the free, open-source plugins, as many as possible.

### Phase 4: SpeedySpec 2

Remove what 1.x deprecated. Drop the widget adapters, so widgets are blocks only. Change and migrate tables. Keep 1.x
following WordPress.

### Phase 5: SpeedySpec 3

Remove more of WordPress's old-style APIs, keeping SpeedySpec's. Change the SpeedySpec APIs that 2.x deprecated.

## What each major removes

Only SpeedySpec 1 is settled. The lists for 2 and 3 are the direction, not a schedule: nothing goes in a major unless
the previous one deprecated it.

- **SpeedySpec 1:** removes nothing. Departures are only those in the register.
- **SpeedySpec 2:** WordPress's deprecated functions and classes (`wp-includes/deprecated.php` and the like); the
  legacy widget system, so `WP_Widget`, `register_sidebar` widgets and the widget options give way to blocks; table
  changes and their migrations; the adapters 1.x deprecated.
- **SpeedySpec 3:** more of WordPress's old-style APIs, where a SpeedySpec API replaces them; SpeedySpec APIs that
  2.x deprecated, changed to their new form.

## SpeedySpec 1 departures register

Every way SpeedySpec 1 knowingly differs from WordPress in what a plugin, theme or site owner can observe, with how to
work with it. Gaps that are not intended (bugs and unbuilt parts) are not departures; they live in
[`deferred/`](../deferred/) and are fixed toward WordPress's behaviour. Paths starting `compat/`, `crates/` or `docs/`
are in speedyspec-server. Entries marked *planned* are designed but not built.

A departure whose workaround reads "none yet" still owes one under
[ADR-028](../knowledge/00_platform/adr/028-speedyspec-versions.md) rule 7: either a workaround, or a reason the
difference cannot matter.

### Hooks

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| Hook globals are proxies | `$wp_filter`, `$wp_actions` and `$wp_filters` are objects over the Rust hook store; reading, `isset`, iterating, counting and appending work | Use the globals as WordPress documents them | [ADR-020](../knowledge/00_platform/adr/020-rust-owns-hook-state.md); `docs/php.md` |
| Array functions on hook globals | `array_keys`, `array_key_exists`, `in_array`, `array_pop` and the like throw `TypeError` | Use `isset`, `foreach` and `has_filter()` | `compat/hooks/breaks.json` |
| Replacing hook globals wholesale | Assigning a new `$wp_filter` or `$wp_filters` is not seen; only `WP_Hook` objects may go in `$wp_filter[$name]` | Add and remove callbacks through the Hook API; hooks set before PHP runs are imported at request start | `compat/hooks/breaks.json` |
| `$wp_current_filter` is read-only in effect | Writing it does not change `current_filter()` | Run the code inside a real filter | `compat/formatting/breaks.json` |
| Core callbacks run in Rust | `wpautop` and the other `default-filters.php` callbacks have no PHP frame in `debug_backtrace()`; uopz or runkit overrides are not seen | Remove the core callback and add your own, rather than overriding the function | `compat/formatting/breaks.json` |
| Hook counts in three places | `did_filter('mime_types')` can undercount when nothing is hooked; `has_filter('query')` is false (no `remove_placeholder_escape`); a recurring cron event fires `pre_update_option_cron`/`update_option_cron` once, not twice | Hook a callback if you count `mime_types`; do not rely on core's own callbacks or on cron's write count | `compat/formatting/`, `compat/wp-db/`, `compat/cron/breaks.json` |

### Options, object cache and database

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| `object-cache.php` refused | `wp_cache_init()` throws while the file exists | Remove it; configure a Caching Store (SQL, Valkey or file) for persistence | [Options API port](../knowledge/00_platform/speedyspec-server/explorations/2026-11-19-options-api-port.md) |
| Object cache persists only when routed | Without engine configuration the cache lasts one request; a failing second level turns off for the rest of the request | Configure the cache engine | Options API port |
| Options stay in SQL | `wp_options` and transients never move to ReDB, Valkey or files | None needed; only the cache moves | Options API port; [database layer](../knowledge/00_platform/speedyspec-server/explorations/2026-11-19-database-layer.md) |
| `db.php` refused | Every request answers 500 while `wp-content/db.php` exists; HyperDB and similar are unsupported | Remove the file; SpeedySpec's native `wpdb` replaces it | Database layer; `docs/php.md` |
| `$wpdb->dbh` is not `mysqli` | `mysqli_*` calls on it fail | Use `$wpdb` methods | `compat/wp-db/breaks.json` |
| Every statement is parsed | SQL MySQL accepts but the parser rejects (such as a `CREATE PROCEDURE` body) fails, with the parser's message | None yet: rewrite the statement | Database layer; `compat/wp-db/breaks.json` |
| No silent translation on SQLite or PostgreSQL | A construct the dialect cannot express fails with an error naming it; SQLite reports duplicate keys as `UNIQUE constraint failed` and keeps UTF-8 in latin1 columns; PostgreSQL's `LIKE` is `ILIKE`, still accent-sensitive | Run on MySQL or MariaDB for unported plugin SQL | Database layer; `compat/wp-db/breaks.json` |
| Database settings from the environment | `DB_*` and `TABLE_PREFIX` come from the worker environment, not `wp-config.php`; an open transaction is rolled back at request end | Have `wp-config.php` read the same variables | `docs/php.md` |

### Users, passwords and pluggables

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| phpass rejected | `$P$`/`$H$` hashes never verify, including pre-6.8 reset keys, application passwords and recovery keys | Users reset their password; regenerate application passwords | `crates/speedyspec-wp-crypto` |
| Raw MD5 rejected | A hash of 32 characters or fewer fails before `check_password` runs | Users reset through "forgot password" | `crates/speedyspec-wp-password` |
| Other hash formats | argon2id is the default (WordPress: bcrypt); `$2x$`, md5-crypt and sha-crypt never verify; a hash above the site's cost ceiling never verifies | Bind your own `PasswordCostLimits` in `php.bindings` | `crates/speedyspec-wp-password-php` |
| Pluggables are ports | Declaring `wp_hash_password()` and the like before `pluggable.php` does nothing | Implement the port's PHP twin (such as `SpeedySpec\WpPassword\PasswordHasher`) and bind it in `app.<id>.php.bindings` | [ADR-017](../knowledge/00_platform/adr/017-php-twins-and-the-override-flag.md) |
| `signature_verification_unsupported` never returned | The runtime always has sodium and sha384 | None needed | `crates/speedyspec-wp-crypto-php` |

### PHP runtime and hosting

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| Runs only in the SpeedySpec runtime | No external php-fpm, Windows, ZTS or PHP threads | Run under the runtime | [ADR-016](../knowledge/00_platform/adr/016-rust-runs-as-an-embedded-php-extension.md) |
| Fixed extension set | PHP 8.5 with WordPress's extensions; `mcrypt` absent | Use a listed extension; a new one needs a runtime release | [ADR-024](../knowledge/00_platform/adr/024-static-php-with-every-wordpress-extension.md) |
| No Laravel | The container borrows Laravel's method names only; no `app()` or `resolve()` | `SpeedySpec\Container\Container::getInstance()` | [ADR-018](../knowledge/00_platform/adr/018-a-service-container-in-rust.md) |
| Stateful ports fixed per request | Rebinding a store such as `HookStore` after first use is refused | Declare it in `php.bindings` | ADR-017 |
| WordPress internals of ported areas are gone | `plugin.php`, `class-wp-hook.php` and ported functions in `option.php`, `cache.php`, `class-wpdb.php` and `cron.php` are absent or stripped | Depend only on public WordPress names | [ADR-014](../knowledge/00_platform/adr/014-port-wordpress-to-rust-and-delete-its-php.md); `docs/php.md` |
| `WP_CONTENT_DIR` is the site directory | Defined as `site_dir/wp-content` before app code; only it is served, never dot paths | Keep plugins, themes and uploads there | [ADR-026](../knowledge/00_platform/adr/026-a-writable-site-directory.md) |
| Filtered environment | Workers see only `PATH`, `HOME`, `USER`, `TMPDIR`, `TZ`, `LANG`, `LC_ALL` and the app's own env file | Put settings in `site_dir/.env` or the app's `env` | `docs/php.md` |
| One request per worker | No persistent state across requests; recycled by `max_requests` or `memory_limit`; timeout is 504, crash 502 | Keep state in options, the cache or the database | [ADR-025](../knowledge/00_platform/adr/025-supervised-php-worker-pools.md) |
| No loopback cron | The server calls the app's `php.cron` path; `/wp-cron.php` may 404 to visitors; multisite sites run without visits; five conflicting writes return `could_not_set` | Schedule with the Cron API as usual; report wake times with `\SpeedySpec\Runtime\report_next_wake()` | `docs/php.md`; [ADR-008](../knowledge/00_platform/adr/008-background-job-runner-in-the-runtime.md) |

### Text, kses and shortcodes

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| No PCRE backtrack limits | Where WordPress's regex gave up and returned `''`, the port returns the real result (`force_balance_tags`, `esc_xml`, `do_shortcode` on huge input) | None needed: output is what WordPress meant | `compat/formatting/`, `compat/shortcode/breaks.json` |
| East Asian charsets as single-byte | With Shift_JIS, BIG5 or EUC-JP, invalid sequences are escaped byte by byte, not blanked | Use UTF-8 | `compat/formatting/breaks.json` |
| Shortcodes in the C locale | After `setlocale(LC_CTYPE, …)`, `\w` does not match bytes ≥0x80 in shortcode names | Keep shortcode names ASCII | `compat/shortcode/breaks.json` |
| Lenient arguments | `null` is `''` with no deprecation; an array becomes `'Array'` with a warning; malformed texturize filters no longer fatal | Pass the documented types | `compat/formatting/breaks.json` |
| No named arguments | Native functions take positional arguments only | Pass by position | `compat/formatting/breaks.json` |
| kses fails closed | Malformed allowed-HTML rules strip attributes instead of throwing; a comment that never settles is dropped after 16 passes; nested work is capped | Return well-formed rules from kses filters | `compat/kses/breaks.json` |
| Theme JSON | Arrays equal with `==`, not `===`; overriding protected methods changes nothing; resolver statics do not exist; results are new objects | Use the public methods and filters | `crates/speedyspec-wp-theme-json/README.md` |

### Media

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| Wider MIME list | About 25 more formats upload, such as jxl, qoi, opus and caf | Remove them with the `upload_mimes` filter | `compat/media/DIFFERENCES.md` |
| Audio and video read by FFmpeg | Metadata from FFmpeg, not getID3: different tag merging and bitrates, no `encoder_options`; `.mka`/`.wma` labelled video by fileinfo are accepted | Read the documented metadata keys | `compat/media/DIFFERENCES.md` |
| More image types sized | `wp_getimagesize` answers for JPEG XL, QOI, PCX, PPM, SGI and HDR | None needed | `compat/media/DIFFERENCES.md` |
| *Planned:* sub-sizes after upload | Sub-sizes appear shortly after the upload returns | Use core's resumable sizing; srcset uses what is ready | [ADR-013](../knowledge/00_platform/adr/013-images-in-rust-on-the-job-runner.md) |

### Admin

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| Detached admin | `wp-admin` is a static site over `wp-admin/v1`; `add_menu_page()` screens show as the classic screen in a sandboxed frame | Register a native page with `register_admin_page()` | [Admin micro-frontends](2026-11-21-admin-micro-frontends.md) |
| Decisions made server-side | Notices, assets and conditions come from `wp-admin/v1`; REST writes fire no `admin_init`, `wp_ajax_*` or `load-*.php` | Move write handling to the plugin's own REST namespace | Admin micro-frontends |
| *Planned:* module rules | Precompiled Vue or JSX only; `react`, `vue` and `@wp-admin/*` cannot be remapped; one shared React; modules can be paused site-wide | Build modules against the shared kit | Admin micro-frontends |
| Gated admin and server maintenance | 401/403/404 alike for existing and missing paths; maintenance is the server's 503, and never covers the admin | None needed | [PHP runtime and admin gate](../knowledge/00_platform/speedyspec-server/explorations/2026-11-19-php-runtime-and-admin-gate.md) |

### Widgets and blocks

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| Widgets build blocks | Each widget renders through a block tree; `sidebars_widgets`, `widget_*` and the `/wp/v2` widget routes keep WordPress's shapes | `WP_Widget` subclasses keep working through the legacy-widget adapter; new code registers blocks | Widgets design, 2026-11-23 |

### Plugins replaced by native features

| Departure | What a plugin sees | How to work with it | Source |
|---|---|---|---|
| Native features replace plugins | Caching, firewall, redirects, sitemaps and image plugins' own APIs and options are unsupported; new hooks and options take the `speedyspec_` prefix | Use core hooks and the native features; importing settings is optional, not planned | [ADR-001](../knowledge/00_platform/adr/001-plugin-features-are-behaviour-of-ported-apis.md), [ADR-005](../knowledge/00_platform/adr/005-core-hooks-stay.md) |
| *Planned:* firewall and page cache | Firewall off by default, blocks XML-RPC and user enumeration; page cache bypassed for logged-in users, non-GET and `DONOTCACHEPAGE` | Application passwords exempt REST; filter the cache key | [ADR-010](../knowledge/00_platform/adr/010-firewall-and-login-security.md), [ADR-011](../knowledge/00_platform/adr/011-page-cache-coarse-purge-first.md) |

## Fleet Impact

- **speedyspec-php-runtime:** its major is the SpeedySpec version; reads the plugin header; serves the PHP APIs of its
  major.
- **speedyspec-server:** the WordPress port crates follow the runtime's rules; the server's own version does not.
- **speedyspec-cli:** outside the promise; its copied port crates follow the runtime's rules.
- **Admin REST API and blog API:** declare supported majors and honour their PHP APIs; endpoints version on their own.
- **Static admin:** outside the promise; Vue and React majors move only its own version.
- **SpeedySpec Guide:** shows deprecations and departures per symbol.
- **Plugins:** keep working on SpeedySpec 1; declare `Requires SpeedySpec:` to run on later majors.

## Open Questions

- How a converted plugin reaches sites: a fork in a SpeedySpec plugin directory, a change offered to its author, or
  both.
- How a conversion is proved: which recorded or upstream tests a converted plugin must pass on each major it claims.

## Success Criteria

- A site running SpeedySpec 1 upgrades to any 1.x without a plugin breaking.
- An existing WordPress install, with its database and `wp-content`, runs on SpeedySpec 1 apart from the register's
  departures.
- Every plugin-observable `breaks.json` entry appears in the register with a workaround.
- Each WordPress release reaches a 1.x minor.
- Every removal in SpeedySpec 2 names the 1.x minor that deprecated it.

## Non-Goals

- A compatibility promise for the CLI, the server, the static admin, the admin REST API or the blog API.
- A fixed end date for SpeedySpec 1.
- Running stock WordPress: SpeedySpec 1 is compatible with WordPress, but never runs WordPress's PHP for a ported area.

## Alternatives Considered

### Option: One version for every component

- **What:** all components ship together as 1.x, then 2.x.
- **Verdict:** rejected by the project owner. It would renumber the existing component versions, and every release
  would wait for the slowest component.

### Option: A platform number separate from the runtime's

- **What:** "SpeedySpec 1" as its own number, which components declare they implement.
- **Verdict:** dropped once the promise sat in the runtime alone; it would only rename the runtime's major.

### Option: SpeedySpec 1 frozen at WordPress 7.1.x

- **What:** fixes and additive SpeedySpec APIs only, no new WordPress releases.
- **Verdict:** rejected by the project owner. Drop-in compatibility would end as sites moved to newer WordPress.

### Option: SpeedySpec 1 follows WordPress for a fixed window after SpeedySpec 2

- **What:** for example two years of tracking, then security fixes only, as a deadline for converting plugins.
- **Verdict:** rejected by the project owner in favour of open-ended tracking.

## Dependencies

- [ADR-021](../knowledge/00_platform/adr/021-recorded-compatibility.md): recording each WordPress release.
- The block library and widget ports, before widgets can move to blocks only.

## References

- [ADR-028](../knowledge/00_platform/adr/028-speedyspec-versions.md): the rules.
- [ADR-014](../knowledge/00_platform/adr/014-port-wordpress-to-rust-and-delete-its-php.md): documented breaks.
- [ADR-021](../knowledge/00_platform/adr/021-recorded-compatibility.md): recorded compatibility.
- [Micro-frontends for the detached admin](2026-11-21-admin-micro-frontends.md): the admin's own SemVer.
- [Semantic Versioning 2.0.0](https://semver.org/)
