# SpeedySpec PHP Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the foundation that ports WordPress to Rust:
- a service container and command and query buses, in Rust;
- WordPress's Hook API as the first ported area, wired into an embedded static PHP (with every extension WordPress
  uses) as native functions and classes, proven against WordPress's recorded behaviour, after which WordPress's own
  hook implementation is deleted;
- supervised PHP worker processes serving `kind = "php"` apps.

The container and the Hook API are also copied into the CLI.

**Architecture:**
- **Ports and Adapters.**
  - Every WordPress function and method is an inbound port: a CQRS command or query, with a handler in the area's
    core.
  - Outbound ports (repositories, stores) are what the core needs.
  - Adapters (Rust or PHP) implement or call ports, and the container connects each port to one adapter.
  - A PHP adapter is the override. With none bound, a call stays in Rust and never enters PHP.
- **The embedded PHP** carries a Rust extension, `speedyspec_wordpress`, which registers the native WordPress
  functions and classes.
- **The server** gains a `php-worker` role that speaks FastCGI, and a pool that supervises it per release.

**Tech Stack:**
- The stack of plans 1–4.
- `ext-php-rs` 0.16.5 (`embed`), as `packages/speedyspec-cli` pins it.
- static-php-cli 2.8.6 and PHP 8.5.12, from the CLI's recipe.
- A FastCGI 1.0 codec, written here (my call; it is small, and no maintained crate covers both sides).
- `serde_json` for the scenario and fixture formats.

**Spec:** `docs/superpowers/specs/2026-11-18-speedyspec-php-foundation-design.md` (approved 2026-11-18), and the
server spec it builds on, `docs/superpowers/specs/2026-11-17-speedyspec-server-core-design.md`.

**Repositories:**
- `packages/speedyspec-server`: branch `feat/php-foundation` from `main`. Plan 4,
  [SpeedySpec/speedyspec-server#3](https://github.com/SpeedySpec/speedyspec-server/pull/3), is merged.
- `packages/speedyspec-cli`, for Task 12 only: branch `feat/wp-hooks` from that repository's default branch. It is on
  `feat/app-foundation` now; rebase onto it instead if that has merged by then.
- `wordpress-develop` (the outer repository), for Task 10's deletions only.

## Global Constraints

- **Do not push or open a PR** without Jacob's go-ahead. Task 10 deletes WordPress code: confirm with Jacob before
  running its deletion step.
- **Formatting and commits:**
  - Rust 1.99, edition 2024, default rustfmt; `cargo fmt --all` before every commit.
  - Conventional Commits, each ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Naming (the design's rules):**
  - inbound ports are operation names (`AddFilter`), and their handlers are `…Handler`;
  - outbound ports are capability nouns (`HookStore`);
  - driven adapters are technology first (`MemoryHookStore`, `PhpHookStore`);
  - driving adapters are entry point first (`WpHookFunctions`);
  - domain services are named for what they compute (`PriorityOrder`);
  - no name ends in `Service`, `Contract`, `Interface` or `Manager`;
  - no crate is named `-domain`, `-app` or `-infra`.
- **Crates:**
  - each area's core and ports are one crate, and each adapter technology its own crate;
  - a port and its implementation never share a crate;
  - `kube`, Salvo and `ext-php-rs` appear only in adapter crates and the binary.
- **Never enter PHP needlessly.** A Rust caller resolving or calling a Rust binding must never enter PHP. Every area
  has a test that counts PHP-boundary crossings and asserts zero.
- **TLS stays on `ring`.** `cargo tree -i aws-lc-sys --target all` prints nothing.
- **Platforms.** Unix only, and every test passes on macOS arm64 and on Linux musl (plan 3's Docker command, with the
  static PHP built for musl).
- **Two builds, both green:**
  - Tests run with `cargo test --workspace --features speedyspec-server/test-hooks,speedyspec-server/php`.
  - The server still builds and passes its plan 1–4 tests without `php`.
- **Errors and logs.** Every refusal names what was refused and why. Logs are JSON lines. Each pool and worker event
  is its own event: `php_pool_started`, `php_worker_started`, `php_worker_exited`, `php_request_timeout`,
  `php_worker_recycled` and `php_pool_drained`.
- **The extension list is a hard requirement.** If an extension will not build, stop and bring it to Jacob; never
  drop one quietly.

Values used across tasks (the pool defaults are my calls; the design left them to the builder):

| Value | Key | Default |
|---|---|---|
| Workers per app | `app.<id>.php.workers` | `{ min = 1, max = 4 }` |
| Requests before a worker is recycled | `app.<id>.php.max_requests` | 1000 |
| Request timeout | `app.<id>.php.request_timeout` | 60 s |
| Memory limit (per worker, checked after each request) | `app.<id>.php.memory_limit` | 256 MiB |
| Idle worker exits after | `app.<id>.php.idle_timeout` | 10 s |
| Site directory | `app.<id>.php.site_dir` | required for a `php` app in managed mode |
| PHP bindings (stateful-port overrides) | `app.<id>.php.bindings` | none |
| Extension name | fixed | `speedyspec_wordpress` |

## Review Focus

Each line has a test in the task named in brackets.

1. **A PHP request that dies mid-way** (fatal error, `exit()`, an uncaught exception, a segfault in an extension).
   The response is 500, or 502 for a crashed worker. PHP's message reaches the log. The next request on that worker,
   or on its replacement, starts clean: no request-lifetime binding, callback or global survives. [Tasks 4 and 11]
2. **Changing a hook while it runs.** A callback adds or removes callbacks on the hook that is running, at the
   current, an earlier and a later priority. `do_action` also runs inside itself. Both must behave exactly as
   WordPress does, which is the easiest thing to get subtly wrong. [Tasks 7 and 9]
3. **A slow client** (a large upload trickling in, a response read slowly). It holds a worker no longer than
   `request_timeout`. The response is streamed with backpressure, not buffered whole. [Task 11]
4. **A PHP override of a stateful port.** The next request on the same worker resolves the Rust adapter again: the
   flag is reset at request shutdown, even after a fatal error. [Tasks 4 and 9]
5. **A release switch while a long PHP request runs.** The request finishes on the old release's pool, never with
   files from two releases, and the old pool's workers exit after it. [Task 11]

## File Structure

```
packages/speedyspec-server/
  php-version, scripts/build-php.sh            the shared static PHP recipe, full WordPress extension list
  crates/speedyspec-php/                       the embed runtime, copied from the CLI's (boundary, SAPI, convert)
  crates/speedyspec-php-ext/                   the speedyspec_wordpress extension: module registry, PHP twins, boundary counter
  crates/speedyspec-container/                 the container (core)
  crates/speedyspec-container-php/             Container's native PHP class, PHP bindings, the override flag
  crates/speedyspec-cqrs/                      Command/Query, handlers, buses, middleware
  crates/speedyspec-cqrs-php/                  PHP dispatch and PHP handlers
  crates/speedyspec-wp-hooks/                  hook ports, handlers, HookStore, domain services
  crates/speedyspec-wp-hooks-memory/           MemoryHookStore
  crates/speedyspec-wp-hooks-php/              WpHookFunctions, WpHookClass, hook globals, PhpHookStore
  crates/speedyspec-compat/                    scenario format, recorder, replayer
  crates/speedyspec-fastcgi/                   FastCGI 1.0 records, both sides
  crates/speedyspec-server/src/php/            php-worker role, EmbeddedPhpPool, PhpRuntime, PhpRequestHandler
  compat/hooks/scenarios.json, compat/hooks/wordpress-<version>.json   the hook catalogue and its recording
  tests/php/                                   PHPUnit harness for WordPress's hook tests
packages/speedyspec-cli/crates/                copies of speedyspec-container, speedyspec-wp-hooks(-memory,-php)
```

---

### Task 1: The static PHP build with every WordPress extension

**Files:**
- Create: `php-version`, `scripts/build-php.sh`, `scripts/linux-php.sh`
- Create: `crates/speedyspec-php/` (copied from `packages/speedyspec-cli/crates/speedyspec-php`, with `ext-php-rs`
  pinned the same way)
- Test: `crates/speedyspec-php/tests/extensions.rs`

**Interfaces:**
- Produces:
  - `scripts/build-php.sh`: the CLI's recipe, with the design's extension list in `craft.yml`:
    `bcmath,ctype,curl,dom,exif,fileinfo,filter,gd,iconv,imagick,intl,mbstring,mysqli,mysqlnd,opcache,openssl,pdo,pdo_mysql,pdo_sqlite,phar,session,simplexml,sodium,sqlite3,tokenizer,xml,xmlreader,xmlwriter,zip,zlib`.
    `gd` is built with JPEG, PNG, WebP, AVIF and FreeType.
  - The `.php-build/` output and `link-flags.txt` are as the CLI's, cached the same way.
  - `speedyspec-php`: the CLI's `Php`, `BootOptions`, `PhpError`, `Output` and the boundary and SAPI modules,
    unchanged except the crate's place.
- [ ] **Step 1: Write the failing test** `every_wordpress_extension_loads`. It boots PHP, and for each name in the
  list asserts `extension_loaded(name)`. It also asserts the Site Health probes WordPress uses:
  - functions: `curl_version`, `exif_read_data`, `finfo_file`, `hash`, `json_last_error`, `mb_check_encoding`,
    `mysqli_connect`, `openssl_encrypt`, `preg_match`, `filter_list`, `iconv`;
  - classes: `DOMNode`, `ZipArchive`;
  - the constant `SODIUM_LIBRARY_VERSION`;
  - `gd_info()` reporting `JPEG`, `PNG`, `WebP`, `AVIF` and `FreeType`.
- [ ] **Step 2: Run** `scripts/build-php.sh`, then the test, on macOS arm64. Expected: the build succeeds, then the
  test passes. If `imagick`, `intl` or `gd`'s AVIF will not build, stop and report it (Global Constraints).
- [ ] **Step 3: Run** `scripts/linux-php.sh`, which does the same inside the musl container. Expected: PASS.
- [ ] **Step 4: Commit.** `build: static PHP 8.5 with every extension WordPress uses`

---

### Task 2: The extension skeleton, PHP twins of ports, and the boundary counter

**Files:**
- Create: `crates/speedyspec-php-ext/`
- Test: `crates/speedyspec-php-ext/tests/ext.rs`

**Interfaces:**
- Consumes: Task 1.
- Produces:
  - **`ModuleRegistry`:** areas register their native functions, classes and request hooks (startup and shutdown)
    with it before PHP boots. The module is named `speedyspec_wordpress`.
  - **`#[php_port]`, an attribute macro** on a Rust port trait. It generates the trait's PHP twin, an interface with
    the same methods and argument and return types mapped through `convert` (my call: a macro, not a build script,
    so the twin is compiled from the trait and cannot drift). It also generates `PhpPort<T>`, the PHP adapter that
    implements the trait by calling a PHP object implementing the twin.
  - **`BoundaryCounter`:** a per-thread count of entries into PHP from Rust. It is incremented by every `call_php`
    and every zval conversion done for an adapter, and read by tests through `boundary_crossings()`.
  - **The request cycle:** each area registers `on_request_start` and `on_request_end` with the registry. The end
    hooks run on every request end, including after a fatal error or `exit()`.
- [ ] **Step 1: Write the failing tests:**
  - `module_name_is_speedyspec_wordpress`;
  - `php_port_twin_round_trips`: a sample trait `Greeter { fn greet(&self, name: String) -> String; }`, whose PHP
    twin is implemented by a PHP class, then called from Rust through `PhpPort<dyn Greeter>`;
  - `rust_call_never_crosses_boundary`: a Rust implementation of `Greeter`, called through the same dispatch, has
    `boundary_crossings() == 0`;
  - `request_end_runs_after_fatal_error` (Review Focus 1).
- [ ] **Step 2: Run** `cargo test -p speedyspec-php-ext`. Expected: FAIL (the crate is missing).
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the same command. Expected: PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat(php-ext): the speedyspec_wordpress module, PHP twins of ports, and a boundary counter`

---

### Task 3: The container

**Files:**
- Create: `crates/speedyspec-container/`
- Test: `crates/speedyspec-container/tests/container.rs`

**Interfaces:**
- Produces: `Container`, with the design's table. The Rust signatures:
  - **Binding:**
    - `bind(key, factory)`, `bind_if`;
    - `singleton`, `singleton_if`;
    - `scoped`, `scoped_if`;
    - `instance(key, value)`.

    A factory is `Fn(&Container, &Params) -> Resolved`.
  - **Resolving:**
    - `make(key) -> Result<Resolved, ContainerError>`, `make_with(key, params)`;
    - `has` (alias `bound`), `resolved`;
    - typed `make_port::<dyn P>()`.
  - **Aliases and decoration:** `alias`, `get_alias`, `is_alias`; `extend(key, Fn(Resolved, &Container) -> Resolved)`.
  - **Contextual binding:** `when(consumer).needs(key).give(factory | key)`.
  - **Tags:** `tag(keys, tag)`, `tagged(tag) -> Vec<Resolved>`.
  - **Callbacks:** `before_resolving`, `resolving`, `after_resolving`, per key or global.
  - **Clearing:** `factory(key) -> impl Fn() -> Result<Resolved>`; `flush`, `forget_instance`, `forget_instances`,
    `forget_scoped_instances`.
  - `Resolved` is `Arc<dyn Any + Send + Sync>` for Rust values, or a `ForeignValue` handle that a language bridge
    (Task 4) supplies.
  - **Keys** are strings. `PortKey::of::<dyn P>()` gives a port's key, from its PHP twin's name.
  - **Lifetimes:** every registration carries `Lifetime::Worker` or `Lifetime::Request`, and `end_request()` removes
    every `Request` registration, instance and scoped instance.
  - **The override flag:** `is_foreign(key) -> bool`, true when the current binding came from a bridge.
  - **Autowiring:** for Rust, registered constructors only. For other languages, through an `Autowirer` port that a
    bridge implements (PHP's reflection, in Task 4).
  - **Errors:** `ContainerError::{Unresolvable(key, why), Circular(Vec<key>)}`, each naming the key.
- [ ] **Step 1: Write the failing tests**, ported from the behaviours of Laravel's `ContainerTest`,
  `ContainerExtendTest`, `ContextualBindingTest`, `ContainerTaggingTest` and `ResolvingCallbackTest`. Write one test
  per behaviour, named for it:
  - `singleton_is_shared`, `bind_is_not_shared`, `scoped_is_shared_until_end_request`;
  - `instance_is_returned_as_is`, `bind_if_keeps_the_first`;
  - `alias_resolves_its_target`, `circular_alias_is_refused`;
  - `extend_decorates_and_applies_to_existing_singleton`;
  - `contextual_binding_gives_per_consumer`, `tagged_returns_in_tag_order`;
  - `resolving_callbacks_run_in_order_before_after`;
  - `circular_dependency_names_the_chain`;
  - `forget_scoped_instances_keeps_singletons`;
  - `end_request_removes_request_lifetime_everything`;
  - `is_foreign_tracks_the_current_binding`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-container`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the same command. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(container): a service container with request and worker lifetimes`

---

### Task 4: The container in PHP, and the override

**Files:**
- Create: `crates/speedyspec-container-php/`
- Test: `crates/speedyspec-container-php/tests/php.rs`

**Interfaces:**
- Consumes: Tasks 2 and 3.
- Produces:
  - **The native class `SpeedySpec\Container\Container`**, with `getInstance()` and the PHP spellings of Task 3's
    methods: `bind`, `singleton`, `scoped`, `instance`, `make`, `makeWith`, `has`, `bound`, `alias`, `extend`,
    `when()->needs()->give()`, `tag`, `tagged`, `resolving`, `afterResolving`, `beforeResolving`, `call`,
    `factory`, `flush`. There are no `app()` or `resolve()` globals.
  - Every PHP registration is `Lifetime::Request`. `on_request_end` calls `end_request()`.
  - **Making PHP values:** a PHP closure, object or class-string bound to a key resolves to a `ForeignValue`. When
    such a key is a port with a PHP twin, `make_port::<dyn P>()` wraps the object in `PhpPort<P>`.
  - **Autowiring:** `PhpAutowirer` reads constructor parameter types through `ReflectionClass` and resolves each one
    from the container.
  - **Stateful ports:** `StatefulPorts`, a registry of keys that may only be bound before first use in a request.
    Binding one after its first resolution in the request fails with "<key> holds state and was already used in this
    request; bind it in app.<id>.php.bindings instead".
  - **`install_php_bindings(&[(key, class)])`** runs at request start, before any PHP code, for
    `app.<id>.php.bindings`.
- [ ] **Step 1: Write the failing tests:**
  - `php_binds_and_makes`;
  - `php_autowires_constructor_dependencies`;
  - `php_binding_overrides_a_port_for_rust_callers`;
  - `rust_binding_resolves_without_entering_php` (`boundary_crossings() == 0`);
  - `override_is_gone_next_request` (Review Focus 4), plus the same after a fatal error in the overriding request
    (Review Focus 1);
  - `stateful_port_bound_after_use_is_refused`;
  - `php_bindings_installed_before_php_runs`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-container-php`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the same command. Expected: PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat(container-php): the container from PHP, with PHP bindings as the override`

---

### Task 5: Command and query buses

**Files:**
- Create: `crates/speedyspec-cqrs/`, `crates/speedyspec-cqrs-php/`
- Test: in-crate tests, and `crates/speedyspec-cqrs-php/tests/php.rs`

**Interfaces:**
- Consumes: Tasks 3 and 4.
- Produces:
  - **`speedyspec-cqrs`:**
    - `trait Command { type Result; const KEY: &str; }` and `trait Query { type Result; const KEY: &str; }`;
    - `trait CommandHandler<C: Command> { fn handle(&self, command: C) -> C::Result; }`, and `QueryHandler` the same;
    - `CommandBus::dispatch(command)` and `QueryBus::ask(query)`, which resolve the handler from the container by
      `KEY` and run the middleware in registration order;
    - `trait Middleware { fn around(&self, message: &dyn Message, next: Next) -> Outcome; }`, with a logging
      middleware;
    - **direct dispatch:** `HandlerRef<C>`, resolved once and cached, which calls the handler with no middleware; its
      cache is dropped when the handler's binding changes.
  - **`speedyspec-cqrs-php`:** `SpeedySpec\Cqrs\CommandBus` and `QueryBus`, native. A PHP class bound to a handler's
    key is a PHP handler.
- [ ] **Step 1: Write the failing tests:**
  - `middleware_runs_in_order`;
  - `direct_dispatch_skips_middleware`;
  - `handler_ref_follows_rebinding`;
  - `query_bus_refuses_command`, a type-level test (`trybuild`);
  - `php_handler_overrides_rust_handler`;
  - `rust_handler_never_enters_php`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-cqrs -p speedyspec-cqrs-php`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the same command. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(cqrs): command and query buses with middleware and direct dispatch`

---

### Task 6: Recorded compatibility: the scenario format, the recorder and the replayer

**Files:**
- Create: `crates/speedyspec-compat/`, `compat/hooks/scenarios.json`
- Create: `compat/hooks/wordpress-<version>.json`, written by the recorder
- Test: `crates/speedyspec-compat/tests/format.rs`

**Interfaces:**
- Consumes: Task 1 (the static PHP).
- Produces:
  - **The scenario format** (JSON): `{ "area": "hooks", "scenarios": [{ "name", "steps": [ { "call": "<function or
    Class::method>", "args": [...] } ] }] }`.
    - An argument is a JSON value, or a callback fixture `{ "callback": "<fixture>", "with": {...} }`.
    - The hook fixtures: `append`, `return_arg`, `count`, `remove_self`, `add_callback`, `remove_callback`,
      `do_action`, `apply_filters`, `throw`, `record_current_filter`, and the four callback kinds (named function,
      static method, object method, closure).
  - **The fixture format:** `{ "wordpress": "<version>", "results": { "<scenario>": [ { "returned", "invocations":
    [ { "fixture", "args" } ], "globals": { "wp_filter", "wp_actions", "wp_filters", "wp_current_filter" } } ] } }`.
  - **The recorder**, `cargo run -p speedyspec-compat -- record hooks --wordpress <version>`:
    - it downloads the pinned WordPress release tarball, checks its SHA-256 (pinned in `compat/hooks/source.json`),
      and extracts it into a temporary directory;
    - it loads the original `wp-includes/plugin.php` in the static PHP, runs every scenario, and writes the fixture.
  - **The replayer**, `speedyspec_compat::replay(area, runner) -> Vec<Mismatch>`:
    - it runs the scenarios through a `ScenarioRunner` (the port's side) and compares each step to the fixture,
      exactly;
    - a documented break is listed in `compat/<area>/breaks.json`, and a step it names may differ.
  - **The hook catalogue** covers:
    - every hook function and every `WP_Hook` method;
    - priorities: equal, negative, string-numeric, and `PHP_INT_MIN`/`MAX`;
    - each of the four callback kinds and identity;
    - nesting and recursion; adding or removing callbacks while a hook runs (current, earlier and later priority;
      Review Focus 2);
    - the `all` hook, `accepted_args` (0, 1, more than given), the deprecated variants, and `did_*`/`doing_*`/
      `current_*`;
    - reading, indexing, iterating and appending to `$wp_filter`, `$wp_actions`, `$wp_filters` and
      `$wp_current_filter`.
- [ ] **Step 1: Write the failing tests:**
  - `scenario_format_round_trips`;
  - `replay_reports_first_mismatch_with_scenario_and_step`;
  - `listed_break_is_tolerated_and_unlisted_is_not`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-compat`. Expected: FAIL.
- [ ] **Step 3: Implement**, then write `compat/hooks/scenarios.json` and run the recorder. Expected: a fixture with a
  result for every scenario and no recorder errors.
- [ ] **Step 4: Run** the tests. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(compat): record WordPress's behaviour for a catalogue of inputs, and replay it`

---

### Task 7: The Hook API core and the memory store

**Files:**
- Create: `crates/speedyspec-wp-hooks/`, `crates/speedyspec-wp-hooks-memory/`
- Test: in-crate tests, and `crates/speedyspec-wp-hooks-memory/tests/handlers.rs`

**Interfaces:**
- Consumes: Tasks 3 and 5.
- Produces:
  - **Commands** (Task 5's traits): `AddFilter { hook, callback, priority, accepted_args }` (result `bool`),
    `AddAction`, `RemoveFilter` (result `bool`), `RemoveAction`, `RemoveAllFilters`, `RemoveAllActions`,
    `ApplyFilters { hook, value, args }`, `ApplyFiltersRefArray`, `ApplyFiltersDeprecated { …, version, replacement,
    message }`, `DoAction`, `DoActionRefArray`, `DoActionDeprecated`.
  - **Queries:**
    - `HasFilter { hook, callback: Option, priority: Option }`, whose result is `HasFilterResult::{No, Any, At(priority)}`
      (WordPress's `int|bool`);
    - `HasAction`, `DidAction`, `DidFilter`;
    - `DoingFilter { hook: Option }`, `DoingAction`;
    - `CurrentFilter`, `CurrentAction`, `CurrentPriority`.
  - **Callbacks:**
    - `Callback::{Rust(Arc<dyn HookCallback>), Foreign(ForeignValue)}`;
    - `HookCallback::call(&self, args: &mut HookArgs) -> HookValue`;
    - `HookValue` is a value enum convertible to and from PHP in the `-php` crate only.
  - **Outbound port `HookStore`:**
    - read side: `callbacks(hook) -> PrioritySlice`, `has_hook`, `counts`, `current_stack`;
    - write side: `insert`, `remove`, `clear`, `push_current`, `pop_current`, `increment_did`.
  - **Domain services:**
    - `PriorityOrder`;
    - `CallbackIdentity`, WordPress's `_wp_filter_build_unique_id`: the same ids for functions,
      `Class::method` strings, `[class, method]`, `[object, method]` (by object id) and closures;
    - `HookIteration`, WordPress's iterator rules when the running hook changes: the priority list is re-read after
      every priority, new callbacks at the current priority run, and removal of a not-yet-run priority skips it.
  - **The `all` hook**, as `_wp_call_all_hook` does.
  - **`MemoryHookStore`**, with worker-lifetime and request-lifetime partitions; `end_request()` drops the request
    partition.
- [ ] **Step 1: Write the failing tests** against `MemoryHookStore` with Rust callbacks:
  - every command and query on its own;
  - `HookIteration`'s cases (Review Focus 2):
    - `add_at_current_priority_runs`, `add_at_earlier_priority_waits`, `add_at_later_priority_runs`;
    - `remove_later_priority_skips_it`, `remove_self_mid_run`;
    - `recursive_do_action_keeps_outer_position`;
  - `all_hook_sees_every_hook`;
  - `accepted_args_truncates`;
  - `callback_identity_matches_wordpress`, against ids taken from the fixture;
  - `request_partition_cleared_keeps_worker_callbacks`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-hooks -p speedyspec-wp-hooks-memory`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the same command. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(wp-hooks): WordPress's Hook API as commands and queries over a hook store`

---

### Task 8: WordPress's hook functions, `WP_Hook` and the hook globals, native

**Files:**
- Create: `crates/speedyspec-wp-hooks-php/`
- Test: `crates/speedyspec-wp-hooks-php/tests/php.rs`

**Interfaces:**
- Consumes: Tasks 2, 4, 5 and 7.
- Produces:
  - **`WpHookFunctions`:** native PHP functions with WordPress's exact signatures and defaults:
    - `add_filter`, `apply_filters`, `apply_filters_ref_array`, `has_filter`, `remove_filter`,
      `remove_all_filters`;
    - `current_filter`, `doing_filter`, `did_filter`;
    - `add_action`, `do_action`, `do_action_ref_array`, `has_action`, `remove_action`, `remove_all_actions`;
    - `current_action`, `doing_action`, `did_action`;
    - `apply_filters_deprecated`, `do_action_deprecated`, `_wp_call_all_hook`, `_wp_filter_build_unique_id`.

    Each resolves its handler once (Task 5's `HandlerRef`) and calls it directly.
  - **`WpHookClass`:** a native `WP_Hook` with the same public properties and methods: `callbacks`, `add_filter`,
    `remove_filter`, `has_filter`, `has_filters`, `remove_all_filters`, `apply_filters`, `do_action`, `do_all_hook`,
    `current_priority`, `build_preinitialized_hooks`, `ArrayAccess`, `Iterator` and `Countable`. Writes to `callbacks`
    (including `unset`) go to the store.
  - **The hook globals:** `$wp_filter`, `$wp_actions`, `$wp_filters` and `$wp_current_filter` are native proxy
    objects over the store, made at request start (my call: `$wp_filter` holds `WP_Hook` proxies made lazily per hook
    on access).
    - Replacing one of them wholesale is the design's documented break, listed in `compat/hooks/breaks.json`.
    - An array of hooks present at request start is imported (`build_preinitialized_hooks`).
  - **`PhpHookStore`:** a `PhpPort<dyn HookStore>`, bindable only through `php.bindings` (stateful).
  - **Callbacks:** `Callback::Foreign` holds the PHP callable for the request, and calling it is a counted boundary
    crossing.
  - **`HooksScenarioRunner`**, which runs Task 6's scenarios through these functions.
- [ ] **Step 1: Write the failing tests:**
  - `replayed_scenarios_match_wordpress`: Task 6's replayer with `HooksScenarioRunner`. Expected: no mismatch, except
    the listed break.
  - `rust_caller_with_rust_callbacks_never_enters_php`: a Rust `ApplyFilters` with two Rust callbacks has
    `boundary_crossings() == 0`.
  - `php_and_rust_callbacks_share_one_priority_order`.
  - `php_callbacks_cleared_at_request_end_rust_kept`.
  - `wp_filter_reads_and_iterates_like_wordpress`.
  - `php_hook_store_binding_takes_every_registration` (Review Focus 4).
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-hooks-php`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the same command. Expected: PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat(wp-hooks-php): WordPress's hook functions, WP_Hook and its globals, native and in Rust`

---

### Task 9: WordPress's own hook tests in the embedded PHP

**Files:**
- Create: `tests/php/bootstrap.php`, `tests/php/WP_UnitTestCase.php`, `tests/php/phpunit.xml`
- Create: `crates/speedyspec-wp-hooks-php/tests/wordpress.rs`, which runs PHPUnit in the embedded PHP
- Copy: `tests/phpunit/tests/actions.php`, `filters.php` and `hooks/*` from `wordpress-develop`, into
  `tests/php/wordpress/`, with their WordPress licence header kept

**Interfaces:**
- Consumes: Task 8.
- Produces:
  - **The harness:** PHPUnit, run from a pinned `phpunit.phar` checked by SHA-256, inside the embedded PHP with
    `speedyspec_wordpress` loaded.
  - **A stub `WP_UnitTestCase`:** the assertions those tests use, and `_backup_hooks`/`_restore_hooks` implemented
    over the Rust store (`SpeedySpec\Hooks\Testing::snapshot()`/`restore()`, native, test builds only). It needs no
    database.
- [ ] **Step 1: Write** `wordpress_hook_tests_pass`, which runs PHPUnit and asserts the suite's result.
- [ ] **Step 2: Run** `cargo test -p speedyspec-wp-hooks-php --test wordpress`. Expected: only the tests that
  reassign a hook global wholesale fail. List each in the ledger with its reason, and mark it in the copied file with
  `@group documented-break`.
- [ ] **Step 3:** exclude `documented-break` from the run, and add a test asserting each is still listed in
  `compat/hooks/breaks.json`.
- [ ] **Step 4: Run.** Expected: PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `test(wp-hooks): WordPress's own hook tests pass against the Rust Hook API`

---

### Task 10: Delete WordPress's hook implementation

**Files (outer `wordpress-develop` repository):**
- Modify: `packages/wp-plugin/wp-plugin-legacy/functions/plugin.php`. Remove the 23 hook functions (`add_filter`
  through `do_action_deprecated`, `_wp_call_all_hook` and `_wp_filter_build_unique_id`). Keep `plugin_basename`,
  `wp_register_plugin_realpath`, `plugin_dir_path`, `plugin_dir_url` and the `register_*_hook` functions, which now
  call the native `add_action`.
- Delete: `packages/wp-plugin/wp-plugin-legacy/classes/class-wp-hook.php`, `packages/speedyspec-wp-hook-domain`,
  `packages/speedyspec-wp-hook-infra-memory` and `packages/speedyspec-wp-hook-infra-wp`

**Interfaces:**
- Consumes: Tasks 6, 8 and 9 all passing. The fixture is the record of WordPress's behaviour, and it stays.
- [ ] **Step 1: Confirm with Jacob** (Global Constraints): show him the exact list above, and wait for his yes.
- [ ] **Step 2: Check** that nothing else uses the deleted code. `grep` the outer repository for `WP_Hook`, the 23
  function names' definitions, and the three packages' namespaces and Composer names. Expected: only callers of the
  functions, no other definitions or package dependencies. Any other user goes back to Jacob before deleting.
- [ ] **Step 3: Delete**, and run the remaining PHP of `plugin.php` in the embedded PHP, calling
  `register_activation_hook` and checking it reaches the native `add_action`. Expected: PASS.
- [ ] **Step 4: Run** the server's whole suite. Expected: PASS (the fixture and WordPress's tests still pass).
- [ ] **Step 5: Commit**, in the outer repository:
  `refactor: remove WordPress's PHP Hook API; the Rust port replaces it`

---

### Task 11: The PHP worker pool

**Files:**
- Create: `crates/speedyspec-fastcgi/`, and `crates/speedyspec-server/src/php/{worker,pool,runtime,handler}.rs`
- Modify:
  - `crates/speedyspec-server/{Cargo.toml (feature php), src/config.rs, src/main.rs, src/managed.rs,
    src/runtime.rs, src/boot.rs}`;
  - `crates/speedyspec-server-domain/src/manifest.rs` (the `php` section).
- Test: `crates/speedyspec-fastcgi/tests/codec.rs`, `crates/speedyspec-server/tests/php.rs`, and
  `tests/fixtures/php-app/`

**Interfaces:**
- Consumes: Tasks 1, 2, 4 and 8.
- Produces:
  - **`speedyspec-fastcgi`:** FastCGI 1.0 records (`BEGIN_REQUEST`, `PARAMS`, `STDIN`, `STDOUT`, `STDERR`,
    `END_REQUEST`, `GET_VALUES`), with name-value pair encoding, a `Responder` (the worker's side) and a `Client`
    (the server's side). It is streaming in both directions, with backpressure.
  - **The manifest's `php` section:** `PhpSection { document_root (default "."), entry (default "index.php"), ini:
    BTreeMap<String,String> }`, validated at unpack. A missing `entry` file is refused, naming it.
  - **Config:** `app.<id>.php` with `workers`, `max_requests`, `request_timeout`, `memory_limit`, `idle_timeout`,
    `site_dir` and `bindings`. The defaults are in the table above, and `site_dir` is required in managed mode.
  - **The `php-worker` role** (`speedyspec-server php-worker`, hidden, feature `php`):
    - it boots PHP once with the release's INI and the extension, then answers FastCGI on the socket it was given;
    - per request:
      1. PHP request startup;
      2. `install_php_bindings` with the app's `php.bindings`, before any PHP runs;
      3. `$_SERVER` from the parameters, and the body on `php://input`;
      4. the status and headers from `header()`/`http_response_code()`, and the output streamed as `STDOUT`;
      5. request shutdown, which runs every `on_request_end`;
    - it sets `WP_CONTENT_DIR` and `WP_CONTENT_URL` to `site_dir`'s `wp-content`, and passes environment values from
      `site_dir/.env` if present (my call: `.env` lines `KEY=value`, nothing evaluated);
    - after each request it reports its memory and request count.
  - **`EmbeddedPhpPool`** (a `PhpPool` adapter):
    - per release, a socket per worker under `run_dir/php/<app>/<version>/`;
    - dynamic sizing between `min` and `max`;
    - one request per worker at a time; requests wait while every worker is busy;
    - replacement on exit or signal (502), on `request_timeout` (SIGKILL, 504), on `memory_limit` or `max_requests`
      (recycled), and when the engine reports itself poisoned;
    - `drain()` stops new requests, waits for the open ones, then stops the workers;
    - each event is logged.
  - **`PhpRuntime`**, an `AppRuntime` replacing `PhpPlaceholderRuntime` when `php` is on:
    - `prepare` starts the pool and waits for one worker to answer `GET_VALUES`;
    - `activate` swaps the mount's handle;
    - `retire` drains the old pool.
  - **`PhpRequestHandler`** (Salvo):
    - it maps the URL to `SCRIPT_FILENAME` (an existing `.php` file under `document_root`, else `entry`), with
      `PATH_INFO` and the CGI/1.1 parameters;
    - it serves other existing files as static, from `site_dir` first, then the release;
    - it streams both bodies.
- [ ] **Step 1: Write the failing tests:**
  - FastCGI: `records_round_trip`, `name_value_pairs_long_and_short`, `stdin_streams_with_backpressure`.
  - The pool and handler, against `tests/fixtures/php-app/` (scripts that echo `$_SERVER`, stream, sleep, crash,
    leak, `exit()`, throw, and call `apply_filters`):
    - `server_vars_and_body_round_trip`;
    - `front_controller_gets_unmatched_paths`;
    - `php_file_runs_directly`;
    - `static_file_from_site_dir_then_release`;
    - `response_streams`;
    - `fatal_error_is_500_and_next_request_clean` (Review Focus 1);
    - `crash_is_502_and_worker_replaced` (Review Focus 1);
    - `hang_is_504_and_worker_killed`;
    - `slow_upload_bounded_by_request_timeout` (Review Focus 3);
    - `slow_reader_gets_streamed_response` (Review Focus 3);
    - `leak_recycles_worker`;
    - `max_requests_recycles_worker`;
    - `grows_to_max_and_shrinks_after_idle`;
    - `release_switch_finishes_long_request_on_old_pool` (Review Focus 5);
    - `hooks_work_in_a_request`, where `add_filter` then `apply_filters` happen inside a request and nothing is left
      on the next;
    - `without_php_feature_placeholder_still_answers_503` (built without `php`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-fastcgi -p speedyspec-server --features test-hooks,php --test php`.
  Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the whole suite, with and without `php`. Expected: PASS on macOS and Linux musl.
- [ ] **Step 5: Commit.** `feat: supervised PHP workers for php apps, over FastCGI`

---

### Task 12: The container and the Hook API in the CLI

**Files (`packages/speedyspec-cli`, branch `feat/wp-hooks`):**
- Create, by copying from the server at Task 11's commit: `crates/speedyspec-container`, `crates/speedyspec-container-php`,
  `crates/speedyspec-wp-hooks`, `crates/speedyspec-wp-hooks-memory`, `crates/speedyspec-wp-hooks-php`,
  `crates/speedyspec-php-ext`; and `compat/hooks/`
- Create: `copied-from.json` (the server commit, and each copied path's tree hash), `scripts/check-copies.sh`
- Modify:
  - the CLI's PHP boot, to register `speedyspec_wordpress` and bind the hook adapters through the container;
  - the CLI's `scripts/build-php.sh` and `php-version`, to the server's recipe and extension list;
  - `.github/workflows/ci.yml`.

**Interfaces:**
- Consumes: Tasks 2–8.
- Produces:
  - In the CLI's PHP, WordPress code calls the native hook functions, `WP_Hook` and
    `SpeedySpec\Container\Container`.
  - The CLI's own `speedyspec-hooks` crate and its plugin API (`SpeedySpec\Hooks\Hook`) are unchanged.
  - `speedyspec-php-ext` is copied too, because the PHP adapter crates need it. It builds on the CLI's existing
    `speedyspec-php`, which the server's copy came from in Task 1, so the API is the same. Any difference found here
    is a ruling.
  - `scripts/check-copies.sh` fails when a copied path's tree differs from the hash in `copied-from.json`, naming it.
- [ ] **Step 1: Write the failing tests** in the CLI:
  - `wordpress_hook_functions_available`;
  - `cli_plugin_hooks_unchanged` (the existing plugin tests);
  - the replayed hook scenarios;
  - the container tests;
  - `check_copies_detects_drift`.
- [ ] **Step 2: Run** the CLI's suite. Expected: FAIL.
- [ ] **Step 3: Copy, wire and build.**
- [ ] **Step 4: Run** the CLI's suite on macOS and Linux musl (`scripts/linux.sh`). Expected: PASS.
- [ ] **Step 5: Commit**, in the CLI repository:
  `feat: the Rust container and WordPress Hook API in the CLI, copied from speedyspec-server`

---

### Task 13: CI and docs

**Files:**
- Modify: `.github/workflows/ci.yml`, `README.md`
- Create: `docs/architecture.md`, `docs/php.md`

**Interfaces:** none.

- [ ] **Step 1: CI.**
  - A `php` job on macOS arm64 and Linux musl builds the static PHP (cached on `php-version` plus the recipe) and runs
    the suite with `php`.
  - The existing jobs keep running without `php`.
  - An `extensions` step runs `every_wordpress_extension_loads`.
- [ ] **Step 2: Docs.**
  - `docs/architecture.md`: the Ports and Adapters rules, the naming table, CQRS, the container's lifetimes and the
    override, and recorded compatibility, as the pattern for every later WordPress area.
  - `docs/php.md`: the build and its extension list, the worker, the pool settings, the site directory, and
    `php.bindings`.
  - A README section on `kind = "php"` apps.
- [ ] **Step 3: Run** the whole suite with and without `php`, on macOS and Linux musl. Expected: PASS.
- [ ] **Step 4: Commit.** `docs: the architecture, the PHP workers and their CI`
