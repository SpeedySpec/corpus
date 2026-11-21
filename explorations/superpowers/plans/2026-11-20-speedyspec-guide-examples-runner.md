# SpeedySpec Guide examples runner Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every example in the guide runs in CI on the sides it declares: WordPress 7.1.5 in the embedded PHP, and the SpeedySpec runtime's native port. The runtime's own PHP API (the functions and classes it registers) is listed through Reflection, with its SpeedySpec "added in" history.

**Architecture:** A Rust crate in the guide repo, `runner/` (`speedyspec-guide-examples`), is a plain executor. It boots the embedded PHP once per process, runs the PHP files in a manifest on one side, and prints what each file printed, returned and threw. It also prints what PHP has declared, through Reflection. The TypeScript toolchain from plan 1 owns everything else: example files and their expected results, the manifest, comparing results, the SpeedySpec history and the validation rules. Hexagonal as before: an `ExampleRunner` port in contracts, an adapter that spawns the binary, and use cases in `app`.

**Tech Stack:** Rust 1.99 (edition 2024) with path dependencies on a pinned speedyspec-server checkout (`speedyspec-php`, `speedyspec-php-ext`, `speedyspec-container`, `speedyspec-container-php`, `speedyspec-hooks`, `speedyspec-hooks-memory`, `speedyspec-wp-hooks`, `speedyspec-wp-hooks-memory`, `speedyspec-wp-hooks-php`), `serde`/`serde_json`; the static PHP built by speedyspec-server's `scripts/build-php.sh`; Bun/TypeScript for the rest.

**Spec:** `docs/superpowers/specs/2026-11-20-speedyspec-guide-design.md` (sections "Examples", "History", "Inventory").

Plan 2 of 4. It builds on plan 1, which is merged in `packages/speedyspec-guide` (`main`).

## Global Constraints

- Never edit speedyspec-server. The runner uses its crates as path dependencies through the symlink `runner/server`, which points at the pinned checkout (`sources.json` `server.commit`, today `8fc498313cd9b6cd381fed1fc68e2e106a86b3b3`) or at `GUIDE_SERVER_DIR`.
- PHP is the server's static build: `<server>/.php-build/buildroot`, made by `<server>/scripts/build-php.sh`. `PHP` and `PHP_CONFIG` must point there (ext-php-rs refuses anything else); `MACOSX_DEPLOYMENT_TARGET = "12.0"` as in the server's `.cargo/config.toml`.
- `Php::boot` works once per process. Each side, and each Reflection mode, is its own process.
- The runtime side boots the same native areas as `crates/speedyspec-php-runtime/src/worker.rs` at the pin, except `RequestHooksArea` (private to the binary; `BootOptions.hooks` gets a `MemoryHooks` instead, as the server's own tests do).
- An example runs on `runtime` only when its seam is `ported: true`.
- The SpeedySpec API version is the PHP runtime's version, tagged `speedyspec-php-runtime-v<version>`. No tag exists yet, so everything is `unreleased` until the first.
- The design principles apply: contracts apart from implementations, plain domain objects, adapters at the edges.
- Commit locally only. Don't commit `docs/superpowers` plans or specs.

## Review Focus

- An example that never returns: `require` gives back `1`, which must be what `return` reports (`'1'`), not a missing value. Task 2 covers it.
- An example that echoes and then throws must report both the output so far and the error; output must not leak into the next example. Task 2 covers it.
- A fatal error (calling an undefined function) must become that example's error, and later examples in the same run must still run. Task 2 covers it.
- An `.expect.json` naming `runtime` in an unported seam, or naming an unknown side, must be a validation failure, not a silent skip. Task 6 covers it.
- Reflection of a function with a default value that is a constant expression (`$priority = 10`, `$flags = PHP_INT_MAX`) must print the default, not crash. Task 3 covers it.

---

## File Structure

```
packages/speedyspec-guide/
  runner/
    Cargo.toml                 own [workspace]; path deps under server/crates/
    .cargo/config.toml         PHP, PHP_CONFIG, MACOSX_DEPLOYMENT_TARGET
    server -> <server checkout> (symlink, gitignored)
    src/main.rs                CLI: run, reflect
    src/manifest.rs            the manifest and result JSON types
    src/sides.rs               booting each side (WordPress files, or the runtime's native areas)
    src/driver.php             speedyspec_guide_example(string $file): string
    src/reflect.php            speedyspec_guide_reflect(): string
    tests/run.rs  tests/reflect.rs  tests/areas.rs
    tests/fixtures/            a tiny WordPress root and example files
  scripts/link-server.ts       makes runner/server
  seams/hooks/seam.yaml        + wordpress: [wp-includes/plugin.php]
  seams/hooks/examples/add_filter/change-a-value.{php,expect.json}
  packages/domain/src/example.ts
  packages/contracts/src/index.ts             + ExampleRunner, ExampleSource
  packages/adapter-example-files/             reads seams/*/examples
  packages/adapter-runner-crate/              spawns the runner binary
  packages/app/src/examples.ts  runtime-history.ts  validate/rules.ts (+2 rules)
  packages/adapter-git-tag-history/           + tag prefix
  .github/workflows/ci.yml  nightly.yml
```

---

### Task 1: Runner crate that links the server's PHP

**Files:**
- Create: `runner/Cargo.toml`, `runner/.cargo/config.toml`, `runner/src/main.rs`, `runner/tests/cli.rs`, `scripts/link-server.ts`
- Modify: `.gitignore` (`runner/server`, `runner/target`), `package.json` (scripts `runner:link`, `runner:build`, `runner:test`)

**Interfaces:**
- Produces: binary `speedyspec-guide-examples` with `run` and `reflect` subcommands (the bodies come in Tasks 2 and 3). With no arguments or an unknown one it prints `usage: speedyspec-guide-examples run --side <wordpress|runtime> --manifest <file> | reflect --mode <baseline|runtime>` to stderr and exits 2.
- Produces: `bun scripts/link-server.ts` creates or replaces `runner/server` as a symlink to `GUIDE_SERVER_DIR`, or else to the checkout `packages/cli/src/checkout.ts` makes for `sources.json` `server`. It exits 1 with a message naming `scripts/build-php.sh` when `<server>/.php-build/buildroot/bin/php-config` is missing.

- [ ] **Step 1:** `Cargo.toml`: package `speedyspec-guide-examples`, edition 2024, `rust-version = "1.99"`, an empty `[workspace]`, the path dependencies from the Tech Stack line as `{ path = "server/crates/<name>" }`, `serde` (derive) and `serde_json`. `.cargo/config.toml` `[env]`: `PHP = { value = "server/.php-build/buildroot/bin/php", relative = true }`, `PHP_CONFIG` the same with `php-config`, `MACOSX_DEPLOYMENT_TARGET = "12.0"`, `SPEEDYSPEC_PHP_PREFIX = { value = "server/.php-build/buildroot", relative = true }`.
- [ ] **Step 2: Failing test** `tests/cli.rs`: `usage_without_arguments` runs `env!("CARGO_BIN_EXE_speedyspec-guide-examples")` with no arguments and asserts exit code 2 and stderr containing `usage: speedyspec-guide-examples run`; `unknown_side_is_refused` runs `run --side elsewhere --manifest x.json` and asserts exit 2 and stderr containing `--side must be wordpress or runtime`.
- [ ] **Step 3:** `bun run runner:link && cd runner && cargo test --test cli`. Expected: compile error (no `main`) or FAIL.
- [ ] **Step 4:** Implement `main.rs` argument parsing (no clap; a `match` on the argument slice), `link-server.ts`, and the `package.json` scripts (`runner:build` = `cargo build --release --manifest-path runner/Cargo.toml`).
- [ ] **Step 5:** Run `cd runner && cargo test --test cli`. Expected: 2 passed; the build links `libphp.a` from the server's build root.
- [ ] **Step 6:** Commit `feat: the examples runner crate, linked to the server's static PHP`.

### Task 2: Running examples on each side

**Files:**
- Create: `runner/src/manifest.rs`, `runner/src/sides.rs`, `runner/src/driver.php`, `runner/tests/run.rs`, fixtures `runner/tests/fixtures/wordpress/wp-includes/plugin.php` (defines `guide_fixture_hello(): string` returning `'hello from wordpress'`), `runner/tests/fixtures/examples/{returns.php,echoes.php,no-return.php,throws.php,fatal.php,native-hooks.php,wordpress-only.php}`
- Modify: `runner/src/main.rs`

**Interfaces:**
- Consumes: `speedyspec_php::{Php, BootOptions, NativeArea, PhpError}`, `speedyspec_hooks::Value`, `speedyspec_hooks_memory::MemoryHooks`; for the runtime side `speedyspec_php_ext::twins_area()`, `speedyspec_container::Container`, `speedyspec_container_php::ContainerArea::new("guide", container).port::<dyn HookStore>(HOOK_STORE, |object| Rc::new(PhpHookStore::new(object))).stateful(HOOK_STORE)`, `speedyspec_wp_hooks_php::{WpHooksArea, HOOK_STORE, PhpHookStore}`, `speedyspec_wp_hooks_memory::MemoryHookStore`, `speedyspec_wp_hooks::HookStore` (as `worker.rs` builds them).
- Produces: the manifest, read from `--manifest <file>`:
  `{ "wordpressDir": string, "examples": [ { "id": string, "file": string (absolute), "load": string[] (WordPress files relative to wordpressDir, WordPress side only) } ] }`
- Produces: on stdout, one JSON array, one entry per manifest example in order:
  `{ "id": string, "side": "wordpress" | "runtime", "output": string, "return": string | null, "error": string | null }`
  where `return` is PHP's `var_export()` of the file's return value, and `null` when the example failed before returning.
- Behaviour: every example runs in a fresh request (`Php::restart_request`). WordPress side: before the example, `define('ABSPATH', <wordpressDir>/); define('WPINC', 'wp-includes');` and `require ABSPATH . <file>` for each `load` entry, as the compat recorder does. Runtime side: nothing is loaded; the native areas are booted once. The driver `speedyspec_guide_example(string $file): string` runs the file inside `ob_start()`/`ob_get_clean()` and a `try`/`catch (\Throwable $e)`, and returns JSON `{output, return, error}` with `error` as `<class>: <message>`. A `PhpError` from `Php::call` (a fatal error) becomes `error: "fatal: <message>"` with the output so far. An example's output never reaches the runner's own stdout.

- [ ] **Step 1: Failing tests** in `tests/run.rs`, each writing a manifest to a temp file and running the binary:
  - `returns_a_value`: `returns.php` (`<?php return 20;`) on `wordpress` → `{"output":"","return":"20","error":null}`.
  - `echoes_and_returns`: `echoes.php` (`echo "a"; return [1];`) → output `"a"`, return `"array (\n  0 => 1,\n)"`.
  - `no_return_is_one`: `no-return.php` (`echo "x";`) → return `"1"`.
  - `throws_keeps_output`: `throws.php` (`echo "before"; throw new RuntimeException("nope");`) → output `"before"`, error `"RuntimeException: nope"`, return `null`.
  - `fatal_does_not_stop_the_run`: manifest `[fatal.php (calls undefined_function()), returns.php]` → first has `error` starting `fatal:` or `Error: Call to undefined function`, second returns `"20"`.
  - `wordpress_files_are_loaded`: `wordpress-only.php` (`return guide_fixture_hello();`) with `load: ["wp-includes/plugin.php"]` on `wordpress` → return `"'hello from wordpress'"`; the same on `runtime` → error mentioning `guide_fixture_hello`.
  - `native_hooks_on_the_runtime`: `native-hooks.php` (`add_filter('x', fn($v) => $v . '!'); return apply_filters('x', 'a');`) on `runtime` → return `"'a!'"`.
  - `requests_are_fresh`: two examples, the first `define('GUIDE_ONCE', 1); return 1;`, the second `return defined('GUIDE_ONCE');` → second returns `"false"`.
- [ ] **Step 2:** Run `cd runner && cargo test --test run`. Expected: FAIL (`run` not implemented).
- [ ] **Step 3:** Implement `manifest.rs` (`Manifest`, `ManifestExample`, `RunResult` with serde), `sides.rs` (`fn boot(side: Side) -> Result<Php, String>`, `fn prepare(php: &Php, side: Side, manifest: &Manifest, example: &ManifestExample) -> Result<(), String>`), the driver, and `run` in `main.rs`. The driver is evaluated after `prepare` in every request.
- [ ] **Step 4:** Run `cargo test --test run`. Expected: 8 passed.
- [ ] **Step 5:** Commit `feat: run examples on WordPress and on the runtime`.

### Task 3: Reflecting the runtime's PHP API

**Files:** Create `runner/src/reflect.php`, `runner/tests/reflect.rs`; modify `runner/src/main.rs`.

**Interfaces:**
- Produces: `reflect --mode baseline` (PHP booted with no native areas) and `reflect --mode runtime` (the runtime's areas) print one JSON array of plain symbols, the same shape as plan 1's `GuideSymbol`:
  `{ "id", "kind": "function"|"class"|"method"|"constant", "name", "file": "native", "line": 0, "signature" }`
  for every declared function, class, interface, trait and enum and every public or protected method. Signatures follow the extractor's format: functions `name($a, $b = 10)`, with types where declared and a `: type` return; methods with `public|protected [abstract] [final] [static]` first; classes `[final|abstract] class Name [extends X] [implements A, B]`.
- A default is printed as PHP itself writes it in `ReflectionParameter::__toString()` (the text after ` = ` up to the closing ` ]`), so constant expressions keep their source form (`ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML401`) for builtins and native functions alike.

- [ ] **Step 1: Failing tests:** `runtime_adds_the_hook_api` runs both modes and asserts `function:add_filter` and `function:apply_filters` are in `runtime` and not in `baseline`, and `function:strlen` is in both; `signatures_print_defaults` asserts the runtime's `function:add_filter` signature contains `$priority = 10` and `$accepted_args = 1`; `constant_defaults_keep_their_source` asserts the baseline's `function:htmlspecialchars` signature contains `$flags = ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML401`.
- [ ] **Step 2:** `cargo test --test reflect`. Expected: FAIL.
- [ ] **Step 3:** Implement `reflect.php` and the `reflect` subcommand (boot by mode, `restart_request`, eval the driver, call it, print its JSON).
- [ ] **Step 4:** `cargo test --test reflect`. Expected: 3 passed.
- [ ] **Step 5:** Commit `feat: list the runtime's PHP API through Reflection`.

### Task 4: The runtime side keeps up with the real runtime

**Files:** Create `runner/src/areas_check.rs` (in the crate, `pub fn missing_areas(worker_rs: &str, sides_rs: &str) -> Vec<String>`), `runner/tests/areas.rs`; modify `runner/src/main.rs` (`mod areas_check;`, and a `lib.rs` exposing it if the test needs it).

**Interfaces:** `missing_areas` takes the text between `let areas: Vec<Rc<dyn NativeArea>> = vec![` and the next `];` in `worker_rs`; collects each `Rc::new(<Type>(`, each `Rc::new(<var>)` resolved to `<Type>` through `let <var> = <Type>::new` in the same text, and each `twins_area()`; drops `RequestHooksArea`; and returns the names not found in `sides_rs`, sorted.

- [ ] **Step 1: Failing tests:** `finds_a_missing_area`: a worker text booting `twins_area()`, `Rc::new(container_area)` (with `let container_area = ContainerArea::new(`), `Rc::new(hooks)` (with `let hooks = WpHooksArea::new(`), `Rc::new(RequestHooksArea(legacy.clone()))` and `Rc::new(OptionsArea::new())`, against a sides text mentioning only `twins_area`, `ContainerArea` and `WpHooksArea` → `["OptionsArea"]`. `the_runtime_side_boots_every_worker_area`: the real `runner/server/crates/speedyspec-php-runtime/src/worker.rs` and `runner/src/sides.rs` → `[]`.
- [ ] **Step 2:** `cargo test --test areas`. Expected: FAIL (`missing_areas` not defined).
- [ ] **Step 3:** Implement `missing_areas`.
- [ ] **Step 4:** `cargo test --test areas`. Expected: 2 passed.
- [ ] **Step 5:** Commit `test: fail when the runtime boots an area the runner doesn't`.

### Task 5: Example domain and ports

**Files:** Create `packages/domain/src/example.ts`, `packages/domain/test/example.test.ts`; modify `packages/domain/src/index.ts`, `packages/contracts/src/index.ts`.

**Interfaces:**
- `type Side = 'wordpress' | 'runtime'`
- `interface ExampleOutcome { output: string; return: string | null; error: string | null }`
- `interface Example { id: string; seam: string; symbol: string; name: string; file: string; sides: Side[]; expect: ExampleOutcome }` — `id` is `<seam>/<symbol folder>/<name>`; `symbol` is the folder name (`add_filter`, `WP_Hook--apply_filters` for a method).
- `interface ExampleRun extends ExampleOutcome { id: string; side: Side }`
- `interface ExampleResult { id: string; side: Side; passed: boolean; actual: ExampleOutcome; expected: ExampleOutcome }`
- `compareExample(expected: ExampleOutcome, run: ExampleRun): ExampleResult` — passed when all three fields are equal; an `expected.error` ending in `*` matches any actual error with that prefix.
- Contracts: `ExampleSource { all(): Promise<{ examples: Example[]; problems: { where: string; message: string }[] }> }`; `ExampleRunner { run(side: Side, examples: { id: string; file: string; load: string[] }[], wordpressDir: string): Promise<ExampleRun[]>; reflect(mode: 'baseline' | 'runtime'): Promise<GuideSymbol[]> }`.

- [ ] **Step 1: Failing tests:** equal outcomes pass; a different `return` fails; `error: "Error: Call to undefined function*"` matches `"Error: Call to undefined function x()"`; a `*` anywhere but the end is literal.
- [ ] **Step 2:** `bun test packages/domain`. Expected: FAIL. **Step 3:** Implement. **Step 4:** PASS with `bun run typecheck`.
- [ ] **Step 5:** Commit `feat: example domain and ports`.

### Task 6: Example files

**Files:** Create `packages/adapter-example-files/{package.json,src/index.ts,test/examples.test.ts}`, fixtures `tests/fixtures/examples-seams/{hooks,options}/…`; modify `seams/hooks/seam.yaml` (add `wordpress: [wp-includes/plugin.php]`), `packages/adapter-seam-files/src/index.ts` and its test (read `wordpress`, default `[]`), `packages/domain/src/seam.ts` (`wordpress: string[]`); create `seams/hooks/examples/add_filter/change-a-value.php` (`add_filter( 'excerpt_length', fn() => 20, 999 ); return apply_filters( 'excerpt_length', 55 );`) and `change-a-value.expect.json` (`{ "sides": ["wordpress", "runtime"], "output": "", "return": "20", "error": null }`).

**Interfaces:** `class ExampleFiles implements ExampleSource { constructor(seamsDir: string, seams: Seam[]) }` reads `<seamsDir>/<seam>/examples/<symbol>/<name>.php` with its sibling `<name>.expect.json`.

Problems it reports (never skips silently): a `.php` without `.expect.json` and the reverse; an `.expect.json` missing `sides`, `output`, `return` or `error`; a side other than `wordpress`/`runtime`; `runtime` in a seam that isn't `ported`.

- [ ] **Step 1: Failing tests** with the fixture seams (`hooks` ported, `options` not): a good example is read with id `hooks/add_filter/a`, its absolute file, sides and expect; each problem above appears once with the file it concerns; `SeamFiles` reads `wordpress` and defaults it to `[]`.
- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat: read examples and their expected results`.

### Task 7: Runner adapter and the examples, reflect and runtime-history stages

**Files:** Create `packages/adapter-runner-crate/{package.json,src/index.ts,test/runner.test.ts}`, `packages/app/src/{examples,runtime-history}.ts`, tests `packages/app/test/{examples,runtime-history}.test.ts`; modify `packages/app/src/pipeline.ts`, `packages/cli/src/main.ts`, `packages/cli/test/pipeline.test.ts`, `README.md`.

**Interfaces:**
- `class RunnerCrate implements ExampleRunner { constructor(binary: string) }` writes the manifest to a temp file, runs `<binary> run --side <side> --manifest <file>`, parses stdout; `reflect` runs `<binary> reflect --mode <mode>`. A non-zero exit rejects with the binary's stderr.
- `runExamples(source: ExampleSource, runner: ExampleRunner, seams: Seam[], wordpressDir: string): Promise<{ results: ExampleResult[]; problems: { where: string; message: string }[] }>` — one runner call per side; each example's `load` is its seam's `wordpress` list.
- `runtimeApi(runner: ExampleRunner): Promise<GuideSymbol[]>` — the `runtime` symbols whose id is not in `baseline`.
- `updateRuntimeHistory(current: GuideSymbol[], version: string, released: boolean, file: RuntimeHistoryFile): RuntimeHistoryFile` with `interface RuntimeHistoryFile { versions: string[]; symbols: Record<string, { since: string; removed?: string }> }`. A symbol first seen gets `since: released ? version : 'unreleased'`; an `unreleased` since becomes `version` once `released`; a symbol gone gets `removed` (released versions only). `versions` lists the released versions recorded.
- Stages (Pipes and Filters, as plan 1): `examples` → `generated/examples.json` (`{ results, problems }`); `reflect` → `generated/inventory-runtime.json`; `runtime-history` → `generated/speedyspec-history.json` (committed, like `wordpress-history.json`). The runtime's version is read from `<server>/crates/speedyspec-php-runtime/Cargo.toml` `version`; `released` is whether `speedyspec-php-runtime-v<version>` is a tag of the server remote (`false` with `--offline`, which only ever leaves a symbol `unreleased` until an online run). `all` runs `extract, fork, assign, graph, history, examples, reflect, runtime-history, validate`.
- CLI: `GUIDE_RUNNER` overrides the binary path (default `runner/target/release/speedyspec-guide-examples`). The `examples`, `reflect` and `runtime-history` stages are skipped with a line on stderr when the binary doesn't exist and `--offline` is given; without `--offline` a missing binary fails with `bun run runner:build`.

- [ ] **Step 1: Failing tests:** `runner.test.ts` with a fake binary (a small `#!/usr/bin/env bun` script written to a temp dir that echoes canned JSON and records its arguments): `run` passes the side and a manifest whose examples carry `load`; a non-zero exit rejects with its stderr. `examples.test.ts` with a fake `ExampleRunner`: two sides call it twice, results compare against expect, a seam's `wordpress` list becomes `load` on the WordPress side and `[]` on the runtime side, source problems pass through. `runtime-history.test.ts`: first unreleased run gives `since: 'unreleased'`; then `1.0.0` released turns it into `1.0.0` and records `versions: ['1.0.0']`; a symbol missing at `1.1.0` gets `removed: '1.1.0'`; an unreleased run never sets `removed`. `pipeline.test.ts`: `all --offline` with no runner binary still passes as before and writes no `examples.json`; with `GUIDE_RUNNER` pointing at the fake binary it writes `examples.json` and `inventory-runtime.json`.
- [ ] **Step 2:** `bun test`. Expected: the new tests FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** `bun test && bun run typecheck`. Expected: all pass.
- [ ] **Step 5: Real run:** `bun run runner:link && bun run runner:build`, then `bun run guide examples && bun run guide reflect && bun run guide runtime-history`. Expected: `examples.json` has `hooks/add_filter/change-a-value` passing on both sides with no problems; `inventory-runtime.json` contains `function:add_filter` and `class:WP_Hook`; `speedyspec-history.json` gives them `since: 'unreleased'`. Record the runtime API's symbol count in the README.
- [ ] **Step 6:** Commit `feat: examples, reflect and runtime-history stages`.

### Task 8: Validation for examples and SpeedySpec versions

**Files:** Modify `packages/app/src/validate/{rule,rules,run}.ts`, `packages/app/test/validate.test.ts`, `packages/adapter-git-tag-history/src/index.ts` and its test, `packages/app/src/pipeline.ts`.

**Interfaces:**
- `GitTagHistory` gains an optional third constructor argument `prefix: string` (default `''`): `tags()` keeps only tags starting with it and returns them without it.
- `ValidationContext` gains `examples: { results: ExampleResult[]; problems: { where: string; message: string }[] } | null` and `runtimeTags: string[]`.
- Rule `ExampleHasRun`: every problem is a failure (`where` = its file); every example × declared side without a passing result is a failure naming the side and, when it ran, the expected and actual `return`/`error`; skipped entirely when `examples` is null (offline without a runner), with no failure.
- `VersionHasTag` also checks `ss` versions against `runtimeTags`.

- [ ] **Step 1: Failing tests:** a problem becomes an `ExampleHasRun` failure; a failed result names `runtime` and shows `expected return 20, got 55`; a missing result for a declared side fails; `examples: null` gives no failure; a page naming SpeedySpec `1.2.0` with `runtimeTags: ['1.0.0']` fails `VersionHasTag`; `GitTagHistory` with prefix `speedyspec-php-runtime-v` on a fixture repo tagged `speedyspec-php-runtime-v1.0.0` and `speedyspec-server-v0.1.0` returns `['1.0.0']`.
- [ ] **Step 2:** FAIL. **Step 3:** Implement, and wire `runtimeTags` (from the server remote, prefix `speedyspec-php-runtime-v`; `[]` with `--offline`) and `examples` (from `generated/examples.json`, or null) into the validate stage. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat: validate examples and SpeedySpec versions`.

### Task 9: CI

**Files:** Modify `.github/workflows/ci.yml`; create `.github/workflows/nightly.yml`.

- [ ] **Step 1:** `ci.yml` gains a job `examples` (needs `test`): check out speedyspec-server at the pinned commit into `.cache/speedyspec-server` (`actions/checkout` with `repository: SpeedySpec/speedyspec-server`, `ref` from `sources.json`); cache `.cache/speedyspec-server/.php-build/buildroot` keyed by the hash of the server's `php-version` and `craft.yml`; run `scripts/build-php.sh` there on a cache miss; install Rust 1.99; cache `runner/target`; `GUIDE_SERVER_DIR=.cache/speedyspec-server bun run runner:link`; `cd runner && cargo test`; `bun run runner:build`; `bun run guide examples && bun run guide reflect` (neither needs the fork; WordPress comes from the release cache). The gated `Build` step of the `test` job (`GUIDE_FORK_PUSHED`) moves into this job after these steps and becomes `bun run guide all`, so validation sees the example results.
- [ ] **Step 2:** `nightly.yml`, daily and on `workflow_dispatch`: the same as the `examples` job but with the server at `main` (`ref: main`); on failure, `gh issue create --title "Examples fail against speedyspec-server main" --body <the failing stage's output>` unless an open issue with that title exists (`gh issue list --search`). Permissions: `contents: read`, `issues: write`.
- [ ] **Step 3:** Lint both with `actionlint` if available, else `bunx yaml-lint`. Expected: no errors.
- [ ] **Step 4:** Commit `ci: run the examples on every pull request and against the server's main nightly`.
