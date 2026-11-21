# SpeedySpec PHP Foundation design

Date: 2026-11-18
Status: draft for Jacob's review

This is project 3 of the server spec (`2026-11-17-speedyspec-server-core-design.md`): PHP workers. It also lays the
foundation that every later step of moving WordPress to Rust builds on: an architecture, a service container,
command and query buses, and the first WordPress area refactored onto them (the Hook API).

## Intent

Most of the WordPress clone is moving from PHP to Rust; in the end only plugins will be PHP. Two things make that
move possible step by step:

1. **The server runs PHP apps** (the front-end API, the blog, any `kind = "php"` app) in supervised worker processes,
   the way php-fpm does. A worker that crashes, hangs or leaks is killed and replaced, and no request reaches a
   poisoned worker. The PHP is static and embedded, with every extension WordPress uses, so the host needs no PHP.
2. **WordPress is ported to Rust**, with PHP embedded for the small part that remains. Its APIs are reimplemented in
   Rust and wired into the PHP engine as native functions and classes, so the remaining PHP and plugins keep calling
   the same names. WordPress's own PHP implementation of each ported area is then deleted: it is not kept as a package,
   a fallback, or anywhere else. Every piece of an API is a port with an adapter behind it,
   and any adapter can be replaced, in Rust or in PHP. Rust code calls Rust directly and never goes through PHP
   unless a PHP adapter has been plugged in.

Jacob's words, in order: the APIs are Rust and are wired into PHP through the engine; the Hook API is Rust, and Rust
code calls Rust functions and classes directly; PHP may still implement a contract and override the Rust one, behind
a flag, and with the flag unset the call short-circuits to Rust only; no DDD layering, but Clean Architecture and
Clean Code with the Repository and Service patterns; every WordPress function and method is a Use Case; CQRS for use
cases where possible; Ports and Adapters naming; WordPress's implementation is not kept anywhere, and compatibility
is proven by recording WordPress's outputs for a set of inputs and holding the port to them; Laravel is not supported,
and this is not a compatible library: WordPress is being ported to Rust, with PHP embedded for what remains.

## Goals

- **Architecture.** Ports and Adapters (hexagonal), with Clean Code, CQRS and the Repository pattern. It applies to
  everything this project writes and to every later WordPress area.
- **Service container.** In Rust, with the functionality of Laravel's service container, reachable from PHP through
  the engine. It wires every port to an adapter. It is not a Laravel-compatible library: Laravel is not supported.
- **Command and query buses.** In Rust, with optional middleware, and a direct path for hot operations.
- **The Hook API**, as the first area: every function of WordPress's plugin API and every method of `WP_Hook`, as
  ports and handlers in Rust, with a memory adapter, a PHP adapter and the WordPress function shims. It must
  reproduce WordPress's recorded behaviour exactly and pass WordPress's own hook tests; then WordPress's hook
  implementation is deleted.
- **The container and the Hook API copied into the CLI** (Jacob's request), so WordPress code the CLI runs keeps its
  hook API once WordPress's PHP one is deleted, and the CLI wires its ports the same way the server does.
- **Recorded compatibility**, the method every ported area uses: run WordPress's original implementation over a
  catalogue of inputs, record its outputs, and hold the Rust port to them.
- **The PHP worker pool** in the server, replacing the 503 placeholder: supervised workers per release, a new pool on
  `prepare`, the old pool draining on `retire`, and poisoned workers replaced.
- **The PHP build**: static PHP 8.5 with every extension WordPress's Site Health checks for, plus what WordPress
  otherwise relies on (the list below).

## Non-goals

- Routing through Salvo and the `WP_Rewrite` refactor. They are the next plans, built on this foundation (see "What
  comes next").
- Refactoring any WordPress area other than hooks.
- Moving the CLI onto the buses or the worker pool. The CLI gets the container and the Hook API (see "The container
  and the Hook API in the CLI").
- Laravel compatibility: the container borrows Laravel's container's functionality and, where they fit, its method
  names, but nothing promises Laravel code would run against it.
- Running WordPress code under an external php-fpm. Without the Rust extension it has no WordPress API.
- A PHP worker mode that keeps state between requests (FrankenPHP-style). Every request starts clean, as with
  php-fpm, because WordPress expects that.
- Windows, ZTS PHP, and threads running PHP.

## Anti-goals

- Rust code reaching a Rust adapter through PHP, or converting values to PHP when no PHP adapter is plugged in.
- A request served by a worker the pool already knows is poisoned.
- A PHP object outliving the PHP request that made it.
- A WordPress function whose behaviour plugins can observe changing (order of callbacks, the `$wp_filter` global,
  `current_filter()`, the `all` hook, return values), except where this design says so.
- `Service`, `Contract`, `Interface` or `Manager` as a name suffix.

## Approaches

### Architecture

- DDD with layer packages (`-domain`, `-app`, `-infra-*`, `-legacy`), as the existing `wp-*` packages do. Rejected by
  Jacob: the layering is replaced.
- Clean Architecture rings with "services". Jacob asked for Clean Architecture and Clean Code first; Ports and
  Adapters then replaced the ring naming, because it names each kind of "service" precisely.
- **Ports and Adapters, with use cases as CQRS commands and queries (chosen by Jacob).** The core holds the
  operations and their shared logic. Ports are interfaces the core defines. Adapters, in Rust or PHP, implement or
  call them, and the container connects each port to one adapter.

### Where the Rust runs

- **A PHP extension written in Rust, compiled into the embedded PHP (chosen).** Native functions and classes run Rust
  directly. The CLI already does this with `ext-php-rs` (its `SpeedySpec\Hooks\Hook` class).
- A PHP library calling Rust through FFI. Rejected: PHP code in between, which is what Jacob ruled out.

### Server to worker protocol

- **FastCGI over a Unix socket (chosen, my call).** A known standard (FastCGI 1.0), with a request format every tool
  understands, so the worker can be tested and debugged on its own.
- A private protocol over a socketpair, like the keeper's. Smaller, but nothing else could speak it.
- Running PHP in the server process. Rejected: one fatal error or leak would take the server down; isolation is the
  point of workers.

### Where the code lives (my call)

- **In the `speedyspec-server` workspace, as crates with no server dependency**, so the CLI can depend on them later
  (by git), and nothing new needs a repository or a push.
- A new `speedyspec-engine` repository shared by the server and the CLI. Cleaner, but needs a new repository and
  cross-repository CI now, for no gain until the CLI moves.

## Design

### The architecture, as rules

Every WordPress area becomes:

- **Inbound ports: the operations.** One per WordPress function and one per public method, as a CQRS **command**
  (it changes state) or **query** (it only reads). A command returns nothing, or only what WordPress's signature
  needs (`add_filter` returns `true`). A query has no side effects. An operation that is necessarily both is a
  command carrying a result. Every such exception is listed in the area's section; for hooks, that is `ApplyFilters`
  and `DoAction` and their variants (they run callbacks that may change state).
- **Handlers: the use cases,** in the core, one per inbound port (`AddFilterHandler`). A handler depends only on
  ports, which the container injects. It never reads globals.
- **Outbound ports: what the core needs**, named for the capability as a noun (`HookStore`). Repositories are
  outbound ports, split into a read side and a write side where reads and writes differ.
- **Domain services: pure shared logic** in the core, named for what they compute (`PriorityOrder`). They are not
  ports and have no adapters.
- **Driven adapters** implement outbound ports, named technology first (`MemoryHookStore`, `PhpHookStore`).
- **Driving adapters** call inbound ports, named entry point first (`WpHookFunctions`, the native `add_filter()` and
  the rest). WordPress's global functions stay, as these shims, so plugins and themes keep working.
- **Requests and results are values**: every command and query is a struct of its inputs, with a result type, so
  Rust and PHP callers pass the same thing.
- **Crates:** one per area holding its core and ports (`speedyspec-wp-hooks`), and one per adapter technology
  (`speedyspec-wp-hooks-memory`, `speedyspec-wp-hooks-php`). Ports and their implementations never share a crate. No crate
  is named `-domain`, `-app` or `-infra`.

### Ports across Rust and PHP

A port is a Rust trait, and has a PHP twin: an interface in PHP with the same methods, generated from the trait so
the two cannot drift. A **PHP adapter** of a port is a Rust struct that implements the trait by calling a PHP object
that implements the twin, converting values at that boundary only.

### The service container (`speedyspec-container`)

The functionality of Laravel's service container, in Rust. Where Laravel's method names fit, they are reused, because
they are well known; nothing promises Laravel compatibility:

| Laravel | Here |
|---|---|
| `bind`, `bindIf`, `singleton`, `singletonIf`, `scoped`, `scopedIf` | the same: a factory per key, shared or not; `scoped` is shared for one PHP request |
| `instance` | the same |
| `make`, `makeWith`, `get` (PSR-11), `has`/`bound`, `resolved` | the same |
| `alias`, `getAlias`, `isAlias` | the same |
| `extend` | the same: decorates a binding's result |
| `when(..)->needs(..)->give(..)` | contextual binding, the same |
| `tag`, `tagged` | the same |
| `resolving`, `afterResolving`, `beforeResolving` | the same callbacks |
| `call` | the same, for PHP callables; in Rust, functions taking the container |
| `factory`, `flush`, `forgetInstance`, `forgetInstances`, `forgetScopedInstances` | the same |
| Autowiring `make(Foo::class)` | PHP classes: constructor parameters by type, through PHP's reflection; Rust types: registered constructors only |

- **Keys** are strings: a port's name (`SpeedySpec\Hooks\HookStore`) or any PHP class or alias. Rust callers use
  typed helpers (`container.make::<dyn HookStore>()`), which map the trait to its key.
- **Two lifetimes.**
  - *Worker lifetime*: bindings, aliases, tags, contextual bindings, extenders, callbacks registered from Rust, and
    instances of Rust adapters.
  - *Request lifetime*: anything PHP made (PHP instances, PHP closures, PHP factories, `scoped` instances), and any
    binding PHP registered. All of it is cleared at PHP request shutdown, which puts every port back on its
    worker-lifetime binding. A PHP object therefore never outlives its request.
- **Overriding a stateful port** (my call). A port that holds state (a store, such as `HookStore`) is bound to a PHP
  adapter in the app's config (`php.bindings`, port to PHP class), and the binding is installed at request startup,
  before any PHP code runs, so the PHP store sees every registration from the start. Binding such a port from PHP code
  after it has been used in the request is refused, naming the port. Stateless ports (handlers, domain services) may
  be rebound from PHP at any point in the request.
- **The override flag.** Binding a port to a PHP adapter is the override. The container records, per key, whether
  the current binding is PHP. Resolving a Rust binding never enters PHP; resolving a PHP binding wraps the PHP object
  in its PHP adapter. Hot ports cache their resolved adapter, and the cache is dropped when the binding changes.
- **From PHP**: `SpeedySpec\Container\Container` (with `getInstance()`) is a native class, so the remaining PHP
  code reaches the same container. No `app()` or `resolve()` global helpers (my call: WordPress has neither, and they
  would only be there for Laravel).

### Command and query buses (`speedyspec-cqrs`)

- `Command` and `Query` traits, each with a result type, and `CommandHandler` and `QueryHandler` traits. Handlers
  are resolved from the container by the command's or query's key.
- `CommandBus` and `QueryBus`, with ordered middleware (logging and transactions are the obvious first ones).
- **Direct dispatch for hot paths**: a driving adapter may resolve a handler once and call it with no middleware.
  WordPress's hook functions do this; `apply_filters` runs thousands of times per request.
- PHP can dispatch commands and queries, and register handlers, through native classes. A PHP handler is a PHP
  adapter like any other.

### The Hook API (`speedyspec-wp-hooks`, `-memory`, `-php`)

The first area. It must reproduce WordPress's recorded behaviour (see "Recorded compatibility") and pass WordPress's
own hook tests (`tests/phpunit/tests/actions.php`, `filters.php` and `hooks/`). Then `wp-plugin-legacy`'s
`functions/plugin.php` and `classes/class-wp-hook.php` are deleted, and so are the PHP hook packages
(`speedyspec-wp-hook-domain`, `-infra-memory`, `-infra-wp`).

- **Commands:**
  - `AddFilter` (and `AddAction`, the same with no return);
  - `RemoveFilter`, `RemoveAllFilters`;
  - `ApplyFilters`, `ApplyFiltersRefArray`, `ApplyFiltersDeprecated`;
  - `DoAction`, `DoActionRefArray`, `DoActionDeprecated`.

  The `Apply*` and `Do*` commands carry a result: they run callbacks that may change state, which is the CQRS
  exception this area lists.
- **Queries:**
  - `HasFilter`, `DidAction`, `DidFilter`;
  - `DoingFilter`, `DoingAction`;
  - `CurrentFilter`, `CurrentAction`.
- **Rust owns hook state** (Jacob's choice). `$wp_filter`, `$wp_current_filter`, `$wp_actions` and `$wp_filters`
  are native proxy objects over the Rust store, so reading, indexing, iterating and appending to them works as in
  WordPress, and a Rust caller never reads PHP. **The documented break:** replacing one of those globals wholesale
  (for example `$wp_current_filter = array()`, or WordPress's test harness swapping `$GLOBALS['wp_filter']`) replaces
  the proxy and is not seen by the Rust store. The conformance harness's hook backup and restore are adapted to save
  and restore the Rust store instead, and `build_preinitialized_hooks` (hooks set before the API loads) is supported
  by importing that array when the extension starts a request.
- **`WP_Hook`'s public methods** map to the same commands and queries on one hook name (`add_filter`,
  `remove_filter`, `has_filter`, `has_filters`, `remove_all_filters`, `apply_filters`, `do_action`, `do_all_hook`,
  `current_priority`). The class stays, as a native class, with `ArrayAccess`, `Iterator` and `Countable`, so code
  reading `$wp_filter` directly keeps working.
- **Outbound port:** `HookStore`, with a read side (callbacks by hook and priority, counts) and a write side.
- **Domain services:**
  - `PriorityOrder`: ascending priority, then insertion order.
  - `CallbackIdentity`: WordPress's `_wp_filter_build_unique_id`, for functions, static methods, object methods and
    closures.
  - `HookIteration`: WordPress's exact behaviour when callbacks are added or removed while a hook runs, and when a
    hook runs inside itself.
- **Adapters:**
  - `MemoryHookStore`, the default;
  - `PhpHookStore`, a PHP implementation of the store's twin;
  - `WpHookFunctions`, the native WordPress functions;
  - `WpHookClass`, the native `WP_Hook`.
- **Callbacks** are either Rust (worker lifetime) or PHP (request lifetime, cleared at shutdown, like every PHP
  value). One store holds both, in one priority order. Calling a Rust callback never enters PHP.
- **The `all` hook**, `accepted_args` and priorities that are not integers behave as WordPress does.
- **WordPress's own hook tests** (my call on the harness) run in the embedded PHP through PHPUnit, with a small
  bootstrap and a stub `WP_UnitTestCase` providing the hook backup and restore the tests rely on (over the Rust store).
  They need no database and no full WordPress. They stay after WordPress's implementation is deleted.
- The server's extension is named `speedyspec_wordpress` (my call), so it never collides with the CLI's `speedyspec`
  module.

### The container and the Hook API in the CLI

Jacob asked for the Rust Hook API and the container to be copied into the CLI (`packages/speedyspec-cli`).

- **What is copied:** `speedyspec-container`, `speedyspec-wp-hooks`, `speedyspec-wp-hooks-memory` and
  `speedyspec-wp-hooks-php`, into the CLI's workspace, wired into the CLI's embedded PHP. WordPress code the CLI runs
  (its PHP plugins) calls the native `add_filter` and the rest, and reaches the same native
  `SpeedySpec\Container\Container`, exactly as in the server.
- **Naming** (my call): the WordPress hook area is `speedyspec-wp-hooks` in both repositories, because the CLI
  already has a crate named `speedyspec-hooks`: the hook contract for its own plugins. That crate and the CLI's
  plugin hooks are unchanged.
- **Wiring:** the CLI wires its ports through the container, as the server does, so a PHP binding overrides a port in
  the CLI the same way. The hook crates themselves still do not depend on the container; each program's start-up
  binds them.
- **Keeping the copies in step** (my call): the server repository is the source. The CLI's copies record the commit
  they were copied from, and a CI check in the CLI compares them with that commit, so drift shows up as a failing
  check rather than silently.
- **Tests:** the recorded hook scenarios, WordPress's hook tests and the container's tests also run in the CLI.
- **The CLI's PHP build** gains the same extension list as the server's (the shared recipe), so WordPress code
  behaves the same under both.

### Recorded compatibility

How every ported area proves it behaves as WordPress did, before WordPress's implementation is deleted:

1. **A catalogue of scenarios**, as data: each is a sequence of calls to WordPress functions and methods, with their
   arguments, written in a small format both runtimes read (my call: JSON). Callbacks are named fixtures (for
   example "append `-a`", "return the second argument", "remove myself", "add another callback at priority 5"), so
   a scenario needs no PHP code. The hook catalogue covers every function and method, every priority case, nested
   and recursive hooks, callbacks added or removed mid-run, the `all` hook, `accepted_args`, deprecated hooks, and the
   globals' read, index, iterate and append behaviour.
2. **A recorder**: it runs each scenario against WordPress's original implementation, in the same static PHP build,
   and writes what came back. That is every return value, every callback invocation in order with its arguments,
   and the observable globals after each step. The output goes into a fixture file per area, committed. Each fixture
   names the WordPress version it was recorded from. The recorder fetches that exact upstream release when
   re-recording, rather than relying on a local copy.
3. **A replayer**: it runs the same scenarios against the port and compares results to the fixture, exactly. Any
   difference fails the test, unless the area's section of its design lists it as a documented break (for hooks:
   replacing a hook global wholesale).
4. **Deletion**: once the replayer and WordPress's own tests pass, the area's WordPress implementation is deleted.
   The fixtures and the replayer stay as the compatibility tests. Re-recording needs the original release, which the
   recorder fetches.

### The PHP build

Static PHP 8.5, non-thread-safe, through static-php-cli, from the CLI's pinned recipe (`scripts/build-php.sh`,
`php-version`), copied into the server repository and extended. It has the `embed` SAPI, plus `cli` for the build
only.

- **What Site Health checks for:**
  - required: `hash`, `json`, `pcre` (built in);
  - recommended: `curl`, `dom`, `exif`, `fileinfo`, `imagick`, `mbstring`, `mysqli` with `mysqlnd`, `sodium`,
    `openssl`, `libxml`/`xml`, `xmlreader`, `simplexml`, `zip`, `filter`, `gd` (JPEG, PNG, WebP, AVIF, FreeType),
    `iconv`, `intl`, `zlib`.
- **What WordPress otherwise relies on:** `opcache`, `ctype`, `tokenizer`, `xmlwriter`, `bcmath`, `session`, `pdo`
  with `pdo_mysql`, `pdo_sqlite` and `sqlite3` (the SQLite database plugin), and `phar`.
- **Not included:** `mcrypt`. PHP removed it in 7.2; Site Health lists it only as a fallback for `sodium`.
- **Builds:** macOS arm64 and Linux musl (x86_64 and aarch64). The first task proves `imagick` and `intl` build and
  link statically on both, before anything depends on it. The list is a hard requirement (my call, from "the embed
  should include all extensions that WordPress requires"): an extension that will not build stops the plan and comes
  back to Jacob, rather than being dropped quietly.

### The PHP worker pool

- **The worker** is a role of the server binary (`speedyspec-server php-worker`, hidden), built with the `php` Cargo
  feature. It is on by default; without it the binary builds and runs as today, with the placeholder.
- **One request per worker at a time**, as php-fpm does.
  - Each request runs a full PHP request startup and shutdown, with the request's `$_SERVER` from the FastCGI
    parameters (the CGI/1.1 variables plus `SCRIPT_FILENAME`, `DOCUMENT_ROOT`, `PATH_INFO`, `HTTPS`), and its body
    on `php://input`.
  - The response's status and headers come from `header()` and `http_response_code()`; its body is the output,
    streamed.
  - The container's request-lifetime half and the hook store's PHP callbacks are cleared at shutdown.
- **What a release declares**, in its manifest's `php` section (it has been stored unread until now):
  - `document_root` (default `.`);
  - `entry`: the front controller every request without a matching file goes to (default `index.php`);
  - `ini`: per-app INI values.

  A request for a `.php` file that exists under the document root runs that file; any other existing file is served
  as static, from the release directory.
- **The site directory** (Jacob's choice). Releases stay read-only, so each PHP app gets a writable directory of its
  own, in the config (`site_dir`), that survives every release switch. It holds:
  - the site's configuration (`wp-config.php`, or values the worker passes as environment variables, so secrets and
    database credentials are never in a release);
  - uploads (`wp-content/uploads`);
  - installed plugins and themes.

  The worker exposes it to the app: the app sees the release directory with the site directory's `wp-content`
  layered over it (my call: the worker sets `WP_CONTENT_DIR` and `WP_CONTENT_URL` to the site directory, and
  `wp-content` paths are served from there). Static files are served from the site directory first, then the
  release. Installing plugins through the admin is a later plan; here they are put in place by hand or by the CLI.
- **Pool settings**, per app in the config: `workers` (min, max), `max_requests`, `request_timeout`, `memory_limit`,
  `idle_timeout`. php-fpm's `dynamic` mode: start `min`, grow to `max` on demand, shrink after `idle_timeout`.
- **Poison and recycling.** A worker is killed and replaced when:
  - it exits, or dies on a signal (the request gets 502);
  - its request runs past `request_timeout` (SIGKILL; the request gets 504);
  - its memory passes `memory_limit` after a request;
  - it has served `max_requests`;
  - the engine itself reports it poisoned (`speedyspec-php`'s `poisoned()`).

  Each case is logged (`php_worker_exited`, `php_request_timeout`, `php_worker_recycled`), and a replacement is
  started at once.
- **Releases.** The pool is the `php` kind's `AppRuntime`:
  - `prepare` starts the release's pool and waits for its first worker to answer a FastCGI management request (a
    failure here leaves the old release live, as the lifecycle requires);
  - `activate` swaps the mount to the new pool in one step;
  - `retire` drains the old pool: no new requests, the open ones finish, then its workers exit.
- **The ports:**
  - `PhpPool`, an outbound port, with the adapter `EmbeddedPhpPool` (the workers above);
  - `PhpRequestHandler`, the Salvo handler that sends requests to the mounted pool.
- **Handoff and cluster.** Pools belong to the generation, so a server handoff starts the new generation's pools
  before it reports ready, and the old generation's pools drain with it. Nothing new is needed in plan 3's handoff
  or plan 4's cluster.

### Errors and logging

As in the server spec: every refusal names what and why; JSON log lines; every pool and worker event its own event.
PHP's own diagnostics keep PHP's form and go to the worker's stderr, which the server forwards into its log with the
app and worker id.

### Testing

- **Container:** the behaviours of Laravel's container test suite, written against this API (bind, singleton,
  scoped, contextual binding, tags, extenders, callbacks, aliases, autowiring), plus the two lifetimes and the
  override flag. The flag
  test checks that resolving a Rust binding never calls into PHP, with a counter on the PHP boundary.
- **Buses:** middleware order; direct dispatch skips middleware; PHP handlers.
- **Hooks:**
  - unit tests of the handlers against `MemoryHookStore`;
  - the recorded scenarios, replayed against the port (see "Recorded compatibility");
  - WordPress's own hook tests (`actions.php`, `filters.php`, `hooks/`), run in the embedded PHP with the extension
    loaded;
  - a test that a Rust caller firing a hook with only Rust callbacks never enters PHP.
- **Worker pool:** a fixture PHP app, testing:
  - `$_SERVER` and a body round trip;
  - streaming;
  - a worker that crashes (502, then replaced), hangs (504, then replaced) or leaks (recycled);
  - `max_requests`;
  - a release switch under load (no failed request; old workers exit).
- **Build:** every extension listed above loads (`extension_loaded`), checked in CI on macOS arm64 and Linux musl.

## What comes next

- **Routing**, next plan. A Rust routing area (`speedyspec-route`): PHP apps' routes are matched in the server
  through Salvo. Each worker reports its app's route table when it starts, and after a rewrite flush. Whole-path regex
  rules get a regex route type beside Salvo's segment matching. Jacob chose matching in the server.
- **`WP_Rewrite`**, the plan after: `add_rewrite_rule`, `flush_rewrite_rules`, `WP::parse_request` and the rest, as
  commands and queries over the routing area. The `rewrite_rules_array` and `generate_rewrite_rules` filters fire
  through the Rust hook API directly.
- **The rest of WordPress**, area by area, the same way.

## Dependencies

- `ext-php-rs` 0.16.5 with `embed`, as the CLI pins it.
- static-php-cli 2.8.6, as the CLI pins it, extended for the libraries `imagick`, `intl` and `gd` need.

## Left to the builder

- How the PHP twin of a port is generated from its trait (a build script, or a macro), as long as the two cannot
  drift.
- The exact FastCGI parameters beyond the ones named above, and the FastCGI management request a pool uses to check
  a worker is ready.
- How `$wp_filter`'s `WP_Hook` objects are made lazily, as long as code reading and iterating `$wp_filter` behaves as
  in WordPress.
- The default pool settings, and their config keys' validation messages.
- Which middleware ship first, beyond logging.
