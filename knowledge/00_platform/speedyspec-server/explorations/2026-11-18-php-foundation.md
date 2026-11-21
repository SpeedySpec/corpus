# Exploration: The PHP foundation

**Status:** superseded in part by the PHP runtime and admin gate design
**Created:** 2026-11-18
**Author:** Claude, for the project owner
**Context:** Running PHP apps under supervision, and the architecture every WordPress-to-Rust port follows, starting with the Hook API.
**Approved by:** the project owner on 2026-11-18 (recorded in the implementation plan; the spec file itself still reads "draft")
**Socialized with:** not socialized

---

## Problem Statement

The server's PHP mounts answered 503: nothing could run the front-end API or the blog. The wider goal, porting
WordPress to Rust area by area until only plugins are PHP, had no shared architecture, no way to wire ports to
adapters, and no method for proving a Rust port behaves as WordPress did before WordPress's PHP is deleted. Plugins
must keep calling the same function names throughout.

## Current State

- `speedyspec-server` (plans 1–4, merged through SpeedySpec/speedyspec-server#3) managed `php` releases but served
  them through a placeholder pool.
- `speedyspec-cli` already embedded a static PHP through `ext-php-rs` 0.16.5 and static-php-cli 2.8.6, with its own
  `SpeedySpec\Hooks\Hook` class for its plugins.
- In `wordpress-develop`, the hook API lived in `wp-plugin-legacy` (`functions/plugin.php`,
  `classes/class-wp-hook.php`) and in three PHP hook packages built in DDD layers.

## Proposed Solution

### Architecture

```
  speedyspec-server (Salvo)                         php-worker process (one request at a time)
  +-------------------------+   FastCGI 1.0 over   +------------------------------------------+
  | PhpRequestHandler       |   a Unix socket      | embedded static PHP 8.5                  |
  | EmbeddedPhpPool         |--------------------->|  + extension speedyspec_wordpress (Rust) |
  |  (min..max, recycling,  |                      |     native add_filter, WP_Hook, ...      |
  |   502/504 on failure)   |<---------------------|     Container, buses, HookStore          |
  +-------------------------+  stdout + report     +------------------------------------------+
          | prepare / activate / retire                  | reads release dir, site_dir/wp-content
```

Each WordPress area is a core crate (operations, handlers, outbound ports, domain services) plus one crate per adapter
technology; a port and its implementation never share a crate. See
[ADR-015](../../adr/015-ports-and-adapters-with-cqrs-use-cases.md).

### Data Model

- **Manifest `php` section:** `document_root` (default `.`), `entry` (default `index.php`), `ini`.
- **Per-app config `app.<id>.php`:** `workers` (default min 1, max 4), `max_requests` (1000), `request_timeout`
  (60 s), `memory_limit` (256 MiB), `idle_timeout` (10 s), `site_dir` (required in managed mode), `bindings`.
- **Recorded compatibility:** `compat/hooks/scenarios.json` (the catalogue) and a fixture per WordPress version, plus
  `breaks.json` listing documented breaks and known limits.

### Interfaces

- Hook commands: `AddFilter`, `RemoveFilter`, `RemoveAllFilters`, `ApplyFilters` (and `RefArray`, `Deprecated`),
  `DoAction` (same variants). Queries: `HasFilter`, `DidAction`, `DidFilter`, `DoingFilter`, `DoingAction`,
  `CurrentFilter`, `CurrentAction`.
- Outbound port `HookStore`; adapters `MemoryHookStore`, `PhpHookStore`, `WpHookFunctions`, `WpHookClass`.
- `#[php_port]` on a Rust trait generates its PHP twin and the `PhpPort<T>` adapter.
- `PhpPool` with the adapter `EmbeddedPhpPool`; `PhpRuntime` as the `php` kind's `AppRuntime`.

## Design Decisions

1. **Port WordPress to Rust and delete each ported area's PHP.** [ADR-014](../../adr/014-port-wordpress-to-rust-and-delete-its-php.md).
   *Trade-off:* no fallback once deleted.
2. **Ports and Adapters, with every function and method a CQRS command or query.** Chosen by the project owner over
   DDD layers and Clean Architecture rings. [ADR-015](../../adr/015-ports-and-adapters-with-cqrs-use-cases.md).
   *Trade-off:* hook firing is a listed CQRS exception.
3. **Rust runs as an extension compiled into the embedded PHP**, not over FFI. [ADR-016](../../adr/016-rust-runs-as-an-embedded-php-extension.md).
   *Trade-off:* ties the Rust to the embedded build.
4. **Every port has a PHP twin; binding it to PHP is the override.** Stateful ports are bound only in config, before
   PHP runs. [ADR-017](../../adr/017-php-twins-and-the-override-flag.md). *Trade-off:* a twin per port.
5. **A service container in Rust with Laravel's container's functions**, worker and request lifetimes, and no
   Laravel compatibility. [ADR-018](../../adr/018-a-service-container-in-rust.md).
6. **Command and query buses with a direct path for hot operations.** [ADR-019](../../adr/019-command-and-query-buses.md).
7. **Rust owns hook state**; the hook globals are proxies. [ADR-020](../../adr/020-rust-owns-hook-state.md).
   As built, `$wp_current_filter` stayed a real PHP array, updated in place; the other three are proxies.
   *Trade-off:* replacing a global wholesale is not seen, and array functions on the proxies throw.
8. **Recorded compatibility** before deletion. [ADR-021](../../adr/021-recorded-compatibility.md).
9. **FastCGI over a Unix socket** to the workers. [ADR-022](../../adr/022-fastcgi-over-a-unix-socket.md).
10. **Engine crates live in the server workspace and are copied into the CLI**, with a drift check.
    [ADR-023](../../adr/023-shared-crates-in-the-server-workspace.md).
11. **Static PHP 8.5 with every extension WordPress uses**; a missing extension stops the work.
    [ADR-024](../../adr/024-static-php-with-every-wordpress-extension.md).
12. **Supervised worker pools, one request per worker.** [ADR-025](../../adr/025-supervised-php-worker-pools.md).
    Superseded in part: the pool moved into its own binary.
13. **A writable site directory over read-only releases.** [ADR-026](../../adr/026-a-writable-site-directory.md).

## Implementation Phases

1. **Static PHP build** with the full extension list. *Delivers:* `every_wordpress_extension_loads`.
2. **Extension skeleton, PHP twins, boundary counter.** *Delivers:* a test that a Rust call never enters PHP.
3. **Container, then the container in PHP with the override.** *Delivers:* request-scoped PHP bindings.
4. **Command and query buses.**
5. **Recorder and replayer.** *Delivers:* WordPress 7.1.5's behaviour recorded for 89 scenarios.
6. **Hook core, native functions, `WP_Hook` and globals; WordPress's own hook tests.** *Delivers:* 182 upstream hook
   tests passing under PHPUnit 11.5.58 in the embedded PHP.
7. **Delete WordPress's hook implementation** in `wordpress-develop`, after the project owner's confirmation.
   Done in the `wordpress-develop` commit "refactor: remove WordPress's PHP Hook API; the Rust port replaces it".
8. **Worker pool over FastCGI.** *Delivers:* `php` apps served by supervised workers.
9. **Copies into the CLI.** Shipped in SpeedySpec/speedyspec-cli#11.
10. **CI and docs.**

Phases 1–8 and 10 shipped in SpeedySpec/speedyspec-server#4 (merged 2026-11-19). A follow-up,
SpeedySpec/speedyspec-server#5, made the hook path cheaper than the PHP it replaces (an empty `apply_filters` went
from one PHP crossing per call to none).

## Fleet Impact

- **Server:** gained the `php` feature, the `php-worker` role and the pool (confirmed, PR #4). The PHP runtime and
  admin gate design then moved all PHP out of the server binary.
- **PHP runtime:** these crates became the core of `speedyspec-php-runtime` the next day.
- **CLI:** received the container and Hook API copies, the shared PHP recipe and a copy-drift check (confirmed,
  CLI PR #11). The copies drift whenever the server's crates change, as PR #5 noted.
- **wordpress-develop:** lost its PHP hook implementation and three hook packages (confirmed by commit).
- **Front-end API and blog:** can now run as `php` apps with a site directory; whether their releases use it yet is
  unverified.

## Open Questions

1. **Linux x86_64 coverage** — the PR left x86_64 untested because ICU crashed under emulation. Later traced to an
   export-dynamic relocation bug and fixed in SpeedySpec/speedyspec-server#9; CI's x86_64 PHP job is the remaining
   check.
2. **Array functions on hook globals** — `array_keys` and similar throw a `TypeError` on the proxies. Recorded as a
   known limit; real plugin usage would show whether it matters.
3. **Routing in the server and `WP_Rewrite`** — named as the next plans; not designed here.
4. **Installing plugins through the admin** — deferred; plugins go in by hand or by the CLI.
5. **Which bus middleware ship first beyond logging** — left to the builder.

## Spike Findings

**Static extensions**
- **Settles:** whether `imagick`, `intl` and `gd` with AVIF build and link statically on macOS and Linux musl.
- **Built:** the shared build recipe and a test that every listed extension loads and each Site Health probe answers.
- **Established:** PHP 8.5.12 builds with all 30 extensions on macOS arm64 and Linux musl arm64.
- **Does not establish:** Linux x86_64, where `intl` crashed (see Open Questions 1).
- **Consequence:** no extension was dropped.

## Success Criteria

- [x] Every WordPress extension loads — `every_wordpress_extension_loads`, in CI.
- [x] A Rust caller with Rust bindings never enters PHP — boundary-counter tests per area.
- [x] The port matches WordPress's recorded behaviour — the replayer over 89 scenarios, with breaks listed.
- [x] WordPress's own hook tests pass — 182 tests in the embedded PHP.
- [x] A PHP override is gone on the next request, even after a fatal error — container-php tests.
- [x] Crashes give 502, hangs 504, leaks and `max_requests` recycle — pool tests.
- [x] The server still builds and passes without PHP — the two-build rule.
- [ ] Linux x86_64 passes the PHP suite — not covered by PR #4.

---

## Non-Goals

- **Refactoring any area other than hooks** — later work, area by area.
- **Laravel compatibility** — the container borrows names only.
- **Running under an external php-fpm** — without the extension there is no WordPress API.
- **A persistent worker mode** — WordPress expects a clean start.
- **Windows, ZTS PHP, threads running PHP** — out of scope.

## Alternatives Considered

### Option: DDD layer packages, as the existing PHP packages used
- **Pros:** already in use.
- **Cons:** the project owner wanted the layering replaced.
- **Verdict:** rejected by the project owner.

### Option: A PHP library calling Rust through FFI
- **Pros:** no custom PHP build.
- **Cons:** PHP sits between callers and Rust.
- **Verdict:** rejected by the project owner.

### Option: A private socketpair protocol, or PHP inside the server process
- **Pros:** smaller; no hop.
- **Cons:** nothing else speaks the protocol; in-process PHP lets one fatal error take the server down.
- **Verdict:** rejected by the designer in favour of FastCGI.

### Option: A separate `speedyspec-engine` repository
- **Pros:** cleaner sharing.
- **Cons:** a new repository and cross-repository CI for no gain yet.
- **Verdict:** rejected by the designer.

## Dependencies

- The server core plans 1–4, merged.
- `ext-php-rs` 0.16.5 and static-php-cli 2.8.6, as the CLI pins them.
- The project owner's confirmation before deleting WordPress code.

## References

- ADRs: [0014](../../adr/014-port-wordpress-to-rust-and-delete-its-php.md),
  [0015](../../adr/015-ports-and-adapters-with-cqrs-use-cases.md),
  [0016](../../adr/016-rust-runs-as-an-embedded-php-extension.md),
  [0017](../../adr/017-php-twins-and-the-override-flag.md),
  [0018](../../adr/018-a-service-container-in-rust.md),
  [0019](../../adr/019-command-and-query-buses.md),
  [0020](../../adr/020-rust-owns-hook-state.md),
  [0021](../../adr/021-recorded-compatibility.md),
  [0022](../../adr/022-fastcgi-over-a-unix-socket.md),
  [0023](../../adr/023-shared-crates-in-the-server-workspace.md),
  [0024](../../adr/024-static-php-with-every-wordpress-extension.md),
  [0025](../../adr/025-supervised-php-worker-pools.md),
  [0026](../../adr/026-a-writable-site-directory.md)
- [SpeedySpec/speedyspec-server#4](https://github.com/SpeedySpec/speedyspec-server/pull/4),
  [#5](https://github.com/SpeedySpec/speedyspec-server/pull/5),
  [#9](https://github.com/SpeedySpec/speedyspec-server/pull/9)
- [SpeedySpec/speedyspec-cli#11](https://github.com/SpeedySpec/speedyspec-cli/pull/11)
- [The server core](2026-11-17-speedyspec-server-core.md)
- [The PHP runtime and admin gate](2026-11-19-php-runtime-and-admin-gate.md)
- Source spec: `2026-11-18-speedyspec-php-foundation-design.md`
- Source plan: `2026-11-18-speedyspec-php-foundation.md`
