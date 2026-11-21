# SpeedySpec CLI Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A `speedyspec` binary with PHP linked in, a Rust hook registry exposed to PHP as `SpeedySpec\Hooks\Hook`,
plugin loading, and plugin-provided subcommands and apps, proven on macOS arm64 and Linux x86_64.

**Architecture:** A Cargo workspace of small crates. `speedyspec-hooks` is the hook contract (a trait and a value
type) and `speedyspec-hooks-memory` its in-memory backend, so a NATS or RabbitMQ backend can be added later as
another crate; `speedyspec-php` boots a statically linked PHP and bridges it to the hook library; `speedyspec-plugin`
finds and loads plugins; `speedyspec-cli` builds the command tree at run time from built-ins plus what plugins add.
The PHP contracts are a separate Composer package whose source is compiled into the binary.

**Tech Stack:** Rust 1.99, ext-php-rs (`embed` feature), static-php-cli (`spc`), clap (builder API), serde,
semver, PHP 8.5.x (pinned in one file), Pest for the PHP package.

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-cli-design.md`

**This is plan 1 of 4.** The spec covers four pieces that each ship working software. The later plans are written
once this one is done, because the outcome of Task 1 can change them:

1. **Foundation (this plan).**
2. Install flow, zip driver, `plugin install` / `plugin remove`.
3. Kubernetes driver and `app uninstall`.
4. Self-update, release builds, pinned contracts release.

## Global Constraints

- Two new git repositories, checked out under `packages/` and **not** tracked by `wordpress-develop`:
  `packages/speedyspec-cli/` (`SpeedySpec/speedyspec-cli`) and `packages/speedyspec-cli-contracts/`
  (`SpeedySpec/speedyspec-cli-contracts`). Create them locally with `git init -b main`. **Do not create GitHub
  repositories or push.** That needs Jacob's go-ahead.
- PHP is the latest stable release the binding and the static build both support. The exact version lives in
  `packages/speedyspec-cli/php-version` and nowhere else.
- No PHP, Composer, Helm or kubectl may be needed on the machine at run time. `php.ini` is never read from the
  machine.
- The CLI contains no knowledge of SpeedySpec or WordPress.
- PHP runs on the main thread only, non-thread-safe build.
- If the static link fails on a target, allowed fallbacks in order: dynamic libc on that target with PHP still
  linked in statically; patch the binding or build recipe; drop the target. A child-process PHP is **not** allowed.
  If macOS arm64 or Linux x86_64 cannot be made to work, stop and report to Jacob.
- Exit codes: 0 success, 1 failure, 2 usage error.
- PHP files follow `.claude/rules/coding-style.md`: 4 spaces, `declare(strict_types=1);`, opening braces on a new
  line, docblocks that say why, no `@return void`.
- Contracts namespace: `SpeedySpec\Cli\Contracts\`. Hook class: `SpeedySpec\Hooks\Hook`.
- Commit messages use Conventional Commits (`feat:`, `fix:`, `build:`, `ci:`, `test:`, `docs:`), as release-please
  expects.

## Review Focus

Each line has a test in the task named in brackets.

1. A plugin calls `exit()` or `die()` while loading or inside a command handler. The CLI must not vanish silently:
   on load the plugin is skipped and named; in a handler the exit status becomes the command's exit code. [Task 3,
   Task 7]
2. A plugin prints while loading. That output must go to stderr, never stdout, so `speedyspec --version` and any
   machine-read output stay clean. [Task 3, Task 7]
3. A hook callback adds or removes callbacks, or fires the same hook again, while that hook is running. No panic, no
   deadlock; changes apply to the next firing. This is in the conformance suite, so every backend must pass it.
   [Task 2]
4. A plugin's filter returns the wrong thing from `speedyspec_apps` or `speedyspec_commands` (null, a string, a list
   containing something that is not the contract class). The error names the plugin, the hook and the field, and
   built-in commands still work. [Task 7]
5. A plugin command is named like a built-in (`app`, `plugin`, `self`, `php`) or like another plugin's command. The
   built-in wins, the first plugin wins among plugins, and the collision is reported on stderr. [Task 7]

## File Structure

```
packages/speedyspec-cli-contracts/
  composer.json                     package speedyspec/cli-contracts, "version": "1.0.0"
  phpunit.xml
  src/App.php  AppPart.php  Command.php  CommandArgument.php  CommandOption.php
      InstallOption.php  InstallRequest.php  ZipSource.php  KubernetesResources.php  WaitFor.php
  tests/Pest.php  tests/ContractsTest.php

packages/speedyspec-cli/
  Cargo.toml                        workspace
  php-version                       e.g. "8.5.12"
  craft.yml                         static-php-cli recipe (generated from php-version by the script)
  scripts/build-php.sh              builds the static embed library into .php-build/buildroot
  scripts/linux.sh                  runs a command in a Linux container, for local Linux checks
  docs/php-embed.md                 what Task 1 learned: versions, flags, fallbacks taken
  .github/workflows/ci.yml          build PHP (cached) and run tests on macOS arm64 and Linux x86_64
  crates/speedyspec-hooks/          src/lib.rs  src/value.rs  src/contract.rs  src/conformance.rs
  crates/speedyspec-hooks-memory/   src/lib.rs  src/registry.rs  tests/conformance.rs
  crates/speedyspec-php/            build.rs  src/lib.rs  src/runtime.rs  src/convert.rs  src/hook_class.rs
                                    tests/php.rs (harness = false)  tests/fixtures/*.php
  crates/speedyspec-driver/         src/lib.rs  src/contracts.rs
  crates/speedyspec-plugin/         src/lib.rs  src/manifest.rs  src/discover.rs  src/load.rs
  crates/speedyspec-cli/            build.rs  src/main.rs  src/boot.rs  src/commands.rs  src/app.rs
                                    tests/cli.rs  tests/fixtures/plugins/<name>/
```

Each crate's `lib.rs` only re-exports. `speedyspec-hooks`, `speedyspec-hooks-memory` and `speedyspec-driver` must
not depend on `speedyspec-php` or ext-php-rs. Only `speedyspec-cli` (and tests) may name `MemoryHooks`; every other
crate holds `SharedHooks`.

---

### Task 1: Prove PHP links statically and can call Rust

This is a proof. Its job is to find out whether the chosen approach works and to write down exactly how.

**Files:**
- Create: `Cargo.toml`, `php-version`, `craft.yml`, `scripts/build-php.sh`, `scripts/linux.sh`, `docs/php-embed.md`,
  `.github/workflows/ci.yml`, `.gitignore` (ignores `target/`, `.php-build/`)
- Create: `crates/speedyspec-php/Cargo.toml`, `build.rs`, `src/lib.rs`, `src/runtime.rs`
- Test: `crates/speedyspec-php/tests/php.rs`, `tests/fixtures/calls_rust.php`

**Interfaces:**
- Produces: `speedyspec_php::Php::boot() -> Result<Php, PhpError>`,
  `Php::eval(&self, label: &str, code: &str) -> Result<(), PhpError>`,
  `Php::run_file(&self, path: &Path, argv: &[String]) -> Result<i32, PhpError>`,
  `speedyspec_php::PHP_VERSION: &str`. Task 3 extends these.
- Produces: env var `SPEEDYSPEC_PHP_PREFIX` (default `<workspace>/.php-build/buildroot`) read by `build.rs`.

- [ ] **Step 1: `git init -b main` in `packages/speedyspec-cli/` and create the workspace.**
  `Cargo.toml` lists `crates/*` as members, edition 2024, `rust-version = "1.99"`.

- [ ] **Step 2: Write `php-version` and `scripts/build-php.sh`.**
  Put the newest stable PHP 8.5 patch in `php-version`. The script downloads the `spc` binary for the host from
  `dl.static-php.dev` into `.php-build/`, verifies it against a SHA-256 pinned in the script, writes `craft.yml`
  with `php-version` from the file, `sapi: embed`, and extensions
  `ctype,curl,filter,mbstring,openssl,phar,tokenizer,zip,zlib`, then runs `spc craft`. Non-thread-safe. It exits
  non-zero unless `.php-build/buildroot/lib/libphp.a` exists afterwards.
  static-php-cli's README documents `craft.yml` with `php-version`, `extensions` and `sapi`; confirm the embed value
  and any NTS flag against its docs, and record the exact recipe in `docs/php-embed.md`.

- [ ] **Step 3: Run `scripts/build-php.sh` on macOS arm64.**
  Expected: `.php-build/buildroot/lib/libphp.a` exists and `.php-build/buildroot/bin/php-config --version` prints
  the version in `php-version`.

- [ ] **Step 4: Write the failing test in `tests/php.rs`.**
  `[[test]] name = "php", harness = false` in `Cargo.toml`: PHP is process-global and must stay on one thread, so
  this file has its own `main()` that boots once, runs every case in order, prints `ok` / `FAILED: <name>: <why>`
  per case, and exits 1 if any failed. First cases:

```rust
fn case_version(php: &Php) -> Result<(), String> {
    let out = capture_stdout(|| php.eval("t", "echo PHP_VERSION;"))?;
    eq(out.as_str(), speedyspec_php::PHP_VERSION)
}

fn case_php_calls_rust(php: &Php) -> Result<(), String> {
    // fixtures/calls_rust.php: <?php echo speedyspec_proof_add(2, 40);
    let out = capture_stdout(|| php.run_file(&fixture("calls_rust.php"), &[]))?;
    eq(out.as_str(), "42")
}

fn case_ini_not_read(php: &Php) -> Result<(), String> {
    let out = capture_stdout(|| php.eval("t", "var_export(php_ini_loaded_file());"))?;
    eq(out.as_str(), "false")
}
```

- [ ] **Step 5: Run it.** `cargo test -p speedyspec-php --test php`. Expected: fails to compile (`Php` not defined).

- [ ] **Step 6: Implement `build.rs` and `runtime.rs`.**
  `build.rs` reads `SPEEDYSPEC_PHP_PREFIX`, points ext-php-rs at that prefix's `php-config`, links `libphp.a` and
  the static libraries `php-config --libs` reports, and sets `PHP_VERSION` from `php-version`. `Php::boot` starts
  the embed SAPI with ini scanning disabled and registers one native function, `speedyspec_proof_add(int, int):
  int`, implemented in Rust. How a native function is registered with an embedded interpreter in ext-php-rs is the
  thing this task has to establish; record the mechanism in `docs/php-embed.md`.

- [ ] **Step 7: Run the test on macOS arm64.** Expected: three `ok` lines, exit 0. Then
  `otool -L target/debug/deps/php-*` shows no `libphp` dynamic library.

- [ ] **Step 8: Write `scripts/linux.sh` and run the same on Linux x86_64.**
  `scripts/linux.sh <command...>` runs the command in a `rust:1.99-alpine` container (musl) with the repo mounted
  and its own `.php-build-linux/` and `target-linux/`. Run `scripts/linux.sh scripts/build-php.sh` then
  `scripts/linux.sh cargo test -p speedyspec-php --test php`. Expected: three `ok` lines, and `ldd` on the test
  binary reports it is not a dynamic executable. If musl will not link, apply the fallbacks from Global Constraints
  in order and record which one was taken.

- [ ] **Step 9: Write `.github/workflows/ci.yml`.**
  Matrix `macos-latest` (arm64) and `ubuntu-latest` (via `scripts/linux.sh`). Steps: checkout, cache `.php-build*`
  keyed on `php-version` and the script's hash, `scripts/build-php.sh`, `cargo test --workspace`. It cannot run
  until the repository exists on GitHub; say so in `docs/php-embed.md`.

- [ ] **Step 10: Write `docs/php-embed.md`.** PHP version pinned and why, `spc` version and checksum, exact recipe,
  link flags per platform, how native functions and classes are registered, build time and binary size, any
  fallback taken.

- [ ] **Step 11: Commit.** `git add -A && git commit -m "build: prove static PHP embed on macOS and Linux"`

**Stop here and report to Jacob if either platform failed after the allowed fallbacks.**

---

### Task 2: The hook contract and the memory backend

Two crates. `speedyspec-hooks` is the contract: a trait, the value type, and a conformance suite. It contains no
storage. `speedyspec-hooks-memory` is the backend the CLI uses. Later backends (NATS, RabbitMQ) are separate crates
that implement the same trait and pass the same suite; none is built in this plan.

Standalone Rust, no PHP. Rust callers are first-class: firing a hook with only Rust callbacks never touches PHP.

**Files:**
- Create: `crates/speedyspec-hooks/Cargo.toml` (feature `conformance`), `src/lib.rs`, `src/value.rs`,
  `src/contract.rs`, `src/conformance.rs`
- Create: `crates/speedyspec-hooks-memory/Cargo.toml`, `src/lib.rs`, `src/registry.rs`
- Test: unit tests in `speedyspec-hooks/src/value.rs`; `crates/speedyspec-hooks-memory/tests/conformance.rs`

**Interfaces:**
- Produces, in `speedyspec-hooks`:

```rust
pub enum Key { Int(i64), Str(String) }

pub enum Value {
    Null, Bool(bool), Int(i64), Float(f64),
    Str(String), Bytes(Vec<u8>),          // Bytes only for strings that are not valid UTF-8
    Array(Vec<(Key, Value)>),             // ordered, like a PHP array
    Opaque(Rc<dyn Any>),                  // carried, never looked inside; cannot leave the process
}
impl Clone for Value                       // Opaque clones the Rc, not the contents
impl serde::Serialize for Value            // Opaque is a serialisation error, at any depth
impl<'de> serde::Deserialize<'de> for Value
impl Value { pub fn is_portable(&self) -> bool; }   // false if an Opaque is anywhere inside

pub struct CallbackError(pub String);
pub type Callback = dyn Fn(&[Value]) -> Result<Value, CallbackError>;

pub struct AddOptions {
    pub priority: i32,                     // default 10
    pub accepted_args: usize,              // default usize::MAX
    pub key: Option<String>,               // caller-chosen identity, for removal by identity
    pub owner: Option<String>,             // who registered it, for error messages
}
impl Default for AddOptions

#[derive(Clone, Copy, PartialEq, Eq, Hash)] pub struct CallbackId(pub u64);

pub enum HookErrorKind {
    Callback,          // a callback returned an error
    NotPortable,       // the backend has to move a value out of the process and it holds an Opaque
    Backend,           // the backend itself failed (connection, timeout, ...)
}
pub struct HookError { pub kind: HookErrorKind, pub hook: String, pub owner: Option<String>, pub message: String }

pub trait Hooks {
    fn add(&self, hook: &str, callback: Rc<Callback>, options: AddOptions) -> Result<CallbackId, HookError>;
    fn remove(&self, hook: &str, id: CallbackId) -> Result<bool, HookError>;
    fn remove_keyed(&self, hook: &str, key: &str, priority: i32) -> Result<bool, HookError>;
    fn has(&self, hook: &str) -> Result<bool, HookError>;
    fn has_keyed(&self, hook: &str, key: &str) -> Result<bool, HookError>;
    fn run(&self, hook: &str, args: &[Value]) -> Result<(), HookError>;
    fn filter(&self, hook: &str, value: Value, args: &[Value]) -> Result<Value, HookError>;
    fn ran(&self, hook: &str) -> Result<u64, HookError>;
}
pub type SharedHooks = Rc<dyn Hooks>;      // what every consumer holds

#[cfg(feature = "conformance")]
pub mod conformance {
    pub struct Capabilities { pub carries_opaque: bool }
    /// Runs every case against a fresh backend from `make`; panics with the case name on the first failure.
    pub fn run_all(make: &dyn Fn() -> SharedHooks, capabilities: Capabilities);
}
```

- Produces, in `speedyspec-hooks-memory`: `pub struct MemoryHooks; impl MemoryHooks { pub fn new() -> MemoryHooks }`,
  `impl Hooks for MemoryHooks`. It never returns `NotPortable` or `Backend`.
- Every method is fallible and takes `&self`, because a broker backend can fail on any of them and may be shared.
  The trait is synchronous; a broker backend blocks inside the call.
- Semantics every backend must have: lower priority runs first; insertion order within a priority. `run` passes
  `args` and ignores return values. `filter` passes `[value, args...]`, and each callback's return value becomes
  `value` for the next. Both truncate what a callback sees to its `accepted_args`. `ran` counts `run` calls for a
  hook name, whether or not anything was registered. The first callback error stops the firing and is returned with
  kind `Callback` and that callback's `owner`.
- Re-entrancy every backend must have: a callback may add, remove or fire hooks, including the one running; such
  changes apply from the next firing.
- Memory backend storage: hook names are interned to an integer id on first use; each hook holds one `Vec` of
  entries kept sorted by `(priority, insertion sequence)` at insert time, so firing is a straight walk. Firing
  takes a snapshot of the entries first, which is what makes re-entrancy safe.

- [ ] **Step 1: Write the conformance cases in `speedyspec-hooks/src/conformance.rs`.** Each is a function taking
  `&dyn Fn() -> SharedHooks`; `run_all` calls them in this order:

```rust
runs_in_priority_then_insertion_order   // add p20 "c", p10 "a", p10 "b"; run; log == ["a","b","c"]
filter_threads_value_through_callbacks  // +1 at p10, *2 at p20; filter(Int(3)) == Int(8)
filter_with_no_callbacks_returns_input  // filter("none", Int(3), &[]) == Int(3)
accepted_args_truncates                 // accepted_args 1; run with 3 args; callback saw len 1
remove_by_id_and_by_key                 // both return Ok(true) once, Ok(false) the second time
remove_keyed_needs_matching_priority    // added at 10; remove_keyed(.., 20) == Ok(false); has_keyed Ok(true)
has_reflects_registration               // false, add, true, remove, false
ran_counts_runs_even_when_empty         // run("x") twice with nothing registered; ran("x") == Ok(2)
error_stops_firing_and_names_owner      // 2nd of 3 errs with owner "acme/p"; 3rd not called; kind Callback,
                                        // err.hook == "h", err.owner == Some("acme/p")
callback_may_add_during_run             // callback adds another to same hook; first run calls 1, second 2
callback_may_remove_itself_during_run   // removes itself; run completes; next run calls 0
callback_may_fire_same_hook_recursively // recurses to depth 3 via a counter; no panic; ran == 3
array_value_round_trips                 // nested Array with Int and Str keys through an identity filter; equal
opaque_value                            // carries_opaque: identity filter returns the same Rc (Rc::ptr_eq).
                                        // otherwise: filter with an Opaque is Err with kind NotPortable
```

- [ ] **Step 2: Write the failing tests.** In `speedyspec-hooks/src/value.rs`:

```rust
#[test] fn portable_value_serialises_and_back()   // nested Array via serde_json round trip is equal
#[test] fn opaque_refuses_to_serialise()          // serde_json::to_string(&Opaque) is Err
#[test] fn is_portable_sees_nested_opaque()       // Array containing an Opaque -> false
```

  In `speedyspec-hooks-memory/tests/conformance.rs`:

```rust
#[test] fn memory_backend_conforms() {
    run_all(&|| Rc::new(MemoryHooks::new()) as SharedHooks, Capabilities { carries_opaque: true });
}
```

- [ ] **Step 3: Run.** `cargo test -p speedyspec-hooks -p speedyspec-hooks-memory`. Expected: does not compile.
- [ ] **Step 4: Implement `value.rs` and `contract.rs`, then `speedyspec-hooks-memory/src/registry.rs`.**
- [ ] **Step 5: Run.** Same command. Expected: 4 passed.
- [ ] **Step 6: Commit.** `git commit -am "feat: add hook contract and in-memory backend"`

---

### Task 3: PHP bridge and `SpeedySpec\Hooks\Hook`

**Files:**
- Modify: `crates/speedyspec-php/src/runtime.rs`, `src/lib.rs`, `Cargo.toml` (depends on `speedyspec-hooks`)
- Create: `crates/speedyspec-php/src/convert.rs`, `src/hook_class.rs`
- Delete: the `speedyspec_proof_add` function and `tests/fixtures/calls_rust.php`
- Test: `crates/speedyspec-php/tests/php.rs`, `tests/fixtures/*.php`

**Interfaces:**
- Consumes: `SharedHooks`, `Value`, `Key`, `AddOptions`, `HookError` from Task 2. The test harness uses
  `MemoryHooks` as a dev-dependency; the crate itself does not depend on it.
- Produces:

```rust
pub struct BootOptions<'a> {
    pub hooks: SharedHooks,
    pub preload: &'a [(&'a str, &'a str)],   // (file name, PHP source) evaluated at boot, before anything else
}
pub enum PhpError {
    Boot(String),
    Exception { class: String, message: String, file: String, line: u32 },
    Fatal { message: String, file: String, line: u32 },
    Exit(i32),                                // the script called exit()/die()
}
impl Php {
    pub fn boot(options: BootOptions) -> Result<Php, PhpError>;          // replaces Task 1's boot()
    pub fn eval(&self, label: &str, code: &str) -> Result<Value, PhpError>;
    pub fn require(&self, path: &Path) -> Result<Value, PhpError>;       // exceptions, fatals and exit() are caught
    pub fn run_file(&self, path: &Path, argv: &[String]) -> Result<i32, PhpError>; // Exit(n) becomes Ok(n)
    pub fn call(&self, callable: &Value, args: &[Value]) -> Result<Value, PhpError>;
    pub fn class_of(&self, value: &Value) -> Option<String>;             // None unless an object
    pub fn properties_of(&self, value: &Value) -> Option<Vec<(String, Value)>>; // public properties, in order
    pub fn set_owner(&self, owner: Option<&str>);  // stamped on every Hook::add until changed
    pub fn redirect_output(&self, to: Output);     // Output::Stdout (default) | Output::Stderr
}
```

- Conversion (`convert.rs`): null, bool, int, float map directly. A PHP string is `Value::Str` when valid UTF-8,
  else `Value::Bytes`. A PHP array is `Value::Array` with keys and order kept. Objects, including closures, are
  `Value::Opaque` wrapping a reference-counted handle to the same PHP object, so the object that comes out is the
  object that went in. Converting back reverses each rule.
- PHP surface (`hook_class.rs`), a final class with static methods and no constructor:

```php
namespace SpeedySpec\Hooks;
final class Hook
{
    public static function add(string $hook, callable $callback, int $priority = 10, int $acceptedArgs = 1): true;
    public static function remove(string $hook, callable $callback, int $priority = 10): bool;
    public static function has(string $hook, ?callable $callback = null): bool;
    public static function run(string $hook, mixed ...$args): void;
    public static function filter(string $hook, mixed $value, mixed ...$args): mixed;
    public static function ran(string $hook): int;
}
```

- Global aliases with the same parameters: `add_action`, `add_filter` → `Hook::add`; `remove_action`,
  `remove_filter` → `Hook::remove`; `has_action`, `has_filter` → `Hook::has`; `do_action` → `Hook::run`;
  `apply_filters` → `Hook::filter`; `did_action` → `Hook::ran`.
- A PHP callable's `key` for `AddOptions` is: the function name for a string; `Class::method` for a static pair or
  `"Class::method"` string; `spl_object_id` of the object plus `::method` for an instance pair; `spl_object_id` for
  a closure or invokable. `remove` and `has` with a callback use `remove_keyed` / `has_keyed`.
- An exception thrown by a PHP callback becomes that callback's `CallbackError`; when the firing was started from
  PHP, the original exception is rethrown to the PHP caller unchanged. Any other `HookError` reaching PHP is thrown
  as `SpeedySpec\Hooks\HookException` (extends `RuntimeException`) with the error's message.

- [ ] **Step 1: Write the failing cases in `tests/php.rs`** (same harness as Task 1; each fixture is a PHP file that
  prints, the case compares stdout):

```rust
case_hook_add_and_run            // Hook::add('h', fn() => print('a')); Hook::run('h');            -> "a"
case_filter_from_php             // add fn($v) => $v + 1; echo Hook::filter('f', 41);              -> "42"
case_priority_order              // p20 prints 'b', p5 prints 'a'; run                             -> "ab"
case_accepted_args               // acceptedArgs 2; callback prints func_num_args(); run with 3    -> "2"
case_aliases_share_registry      // add_filter('f', ...); echo Hook::has('f') ? 'y' : 'n';
                                 // add_action('a', ...); do_action('a'); echo did_action('a');    -> "y1"
case_remove_closure_by_identity  // $c = fn() => print('x'); add; remove($c) is true; run prints nothing -> "1"
case_object_identity_survives    // two filters pass an stdClass through; spl_object_id equal      -> "same"
case_rust_callback_seen_by_php   // Rust adds a filter that appends "!" to a string; PHP filter('r','hi') -> "hi!"
case_rust_fires_into_php         // PHP adds filter on 'p' doubling an int; Rust hooks.filter("p", Int(21))
                                 // == Int(42)
case_rust_only_hook_skips_php    // Rust adds and fires a hook with a Rust callback before any PHP code has run
                                 // since boot; result correct. (No PHP callback is registered, so none is called.)
case_exception_reaches_php       // callback throws RuntimeException('boom'); caller catches it; prints message -> "boom"
case_exception_reaches_rust      // same callback fired from Rust; Err with message containing "boom" and the
                                 // owner set via set_owner("acme/p")
case_exit_in_require_is_caught   // require() of a file that calls exit(3) -> Err(PhpError::Exit(3)); harness
                                 // process still running afterwards
case_exit_in_run_file            // run_file of the same file -> Ok(3)
case_fatal_is_caught             // require() of a file calling an undefined function -> Err(Fatal or Exception)
                                 // whose message names the function; next case still runs
case_array_round_trip            // ['a' => 1, 5 => 'x', 'n' => [true, null]] through an identity Rust filter;
                                 // var_export equal
case_preload_is_available        // boot with preload [("T.php", "<?php final class PreT {}")];
                                 // class_exists('PreT')                                            -> "1"
case_output_redirect             // redirect_output(Stderr); echo 'x'; captured stdout empty, stderr "x"
case_backend_error_is_exception  // boot with a stub Hooks whose filter() returns kind Backend "down"; PHP
                                 // catches SpeedySpec\Hooks\HookException; prints message          -> "down"
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-php --test php`. Expected: does not compile.
- [ ] **Step 3: Implement `convert.rs`.**
- [ ] **Step 4: Implement `hook_class.rs`**: the class and the aliases, registered the way `docs/php-embed.md`
  describes.
- [ ] **Step 5: Extend `runtime.rs`** to the interface above. `require`, `call` and `eval` run inside PHP's bailout
  guard so a fatal error or `exit()` returns an error instead of unwinding through Rust.
- [ ] **Step 6: Run on macOS.** Expected: every case `ok`.
- [ ] **Step 7: Run on Linux.** `scripts/linux.sh cargo test -p speedyspec-php --test php`. Expected: every case `ok`.
- [ ] **Step 8: Commit.** `git commit -am "feat: expose hook registry to PHP as SpeedySpec\\Hooks\\Hook"`

---

### Task 4: The contracts package (`speedyspec/cli-contracts` 1.0.0)

**Files (in `packages/speedyspec-cli-contracts/`, a new repository: `git init -b main`):**
- Create: `composer.json`, `phpunit.xml`, `LICENSE` (BSD-3-Clause, as the sibling packages), `README.md`,
  `release-please-config.json` (copy the shape of `packages/speedyspec-support/release-please-config.json`)
- Create: `src/` one file per class below
- Test: `tests/Pest.php`, `tests/ContractsTest.php`

**Interfaces:**
- Produces: these final readonly classes in `SpeedySpec\Cli\Contracts`, constructor-promoted public properties,
  no methods, no validation (Rust is the validator), no inheritance, no interfaces:

```php
AppPart(string $id, string $label, bool $default = true)
App(string $id, string $label, array $parts, array $drivers)            // AppPart[], string[]
CommandArgument(string $name, string $description = '', bool $required = true)
CommandOption(string $name, string $description = '', bool $takesValue = true, ?string $default = null)
Command(string $name, string $description, \Closure $handler, array $arguments = [], array $options = [])
InstallOption(string $name, string $label, ?string $default = null, bool $required = false, bool $secret = false)
InstallRequest(string $app, array $parts, string $driver, array $options, array $settings)
ZipSource(string $url, string $destination, ?string $sha256 = null, int $stripComponents = 0)
WaitFor(string $kind, string $name, ?string $namespace = null, ?string $condition = null)
KubernetesResources(array $resources, ?array $waitFor = null)           // array[], WaitFor[]|null
```

- A command handler is `fn(array $arguments, array $options): int`. It receives values keyed by name, writes with
  `echo`, and returns the exit code.
- `composer.json`: name `speedyspec/cli-contracts`, `"version": "1.0.0"`, `"php": ">=8.4"` (so the tests run on the
  local PHP 8.4; the classes use nothing newer), PSR-4 `SpeedySpec\\Cli\\Contracts\\` → `src/`, dev dependency
  `pestphp/pest: ^3.0|^4.0`.
- Every file must be loadable in any order, because the CLI evaluates them all at boot without an autoloader. That
  is why there is no inheritance and no interface.

- [ ] **Step 1: Write the failing tests in `tests/ContractsTest.php`:**

```php
it('holds what it is given', function () {
    $part = new AppPart('admin', 'Admin');
    $app = new App('speedyspec', 'SpeedySpec', [$part], ['zip']);
    expect($part->default)->toBeTrue()
        ->and($app->parts)->toBe([$part])
        ->and($app->drivers)->toBe(['zip']);
});

it('defaults the optional zip fields', function () {
    $zip = new ZipSource('https://example.com/a.zip', 'site');
    expect($zip->sha256)->toBeNull()->and($zip->stripComponents)->toBe(0);
});

it('treats an omitted waitFor as null, not an empty list', function () {
    expect((new KubernetesResources([]))->waitFor)->toBeNull();
});

it('is readonly', function (string $class, array $args, string $property) {
    $object = new $class(...$args);
    expect(fn () => $object->{$property} = 'x')->toThrow(Error::class);
})->with([/* one row per class */]);

it('is final and has no parent or interface', function (string $class) {
    $r = new ReflectionClass($class);
    expect($r->isFinal())->toBeTrue()
        ->and($r->getParentClass())->toBeFalse()
        ->and($r->getInterfaceNames())->toBe([]);
})->with([/* all ten class names */]);
```

- [ ] **Step 2: Run.** `composer install && vendor/bin/pest`. Expected: fails, classes not found.
- [ ] **Step 3: Write the ten classes.** Each docblock says what the CLI does with the value, not what the fields are.
- [ ] **Step 4: Run.** `vendor/bin/pest`. Expected: all pass.
- [ ] **Step 5: Commit.** `git add -A && git commit -m "feat: add CLI contracts 1.0.0"`

---

### Task 5: Rust mirror of the contracts (`speedyspec-driver`)

Only the contracts this plan uses: `App`, `AppPart`, `Command`, `CommandArgument`, `CommandOption`. The driver
trait and the remaining contracts arrive with the plans that use them.

**Files:**
- Create: `crates/speedyspec-driver/Cargo.toml` (depends on `speedyspec-hooks` only), `src/lib.rs`,
  `src/contracts.rs`
- Test: unit tests in `src/contracts.rs`

**Interfaces:**
- Consumes: `Value`, `Key` from Task 2.
- Produces:

```rust
pub trait ObjectReader {                       // speedyspec-php's Php implements this in Task 7
    fn class_of(&self, value: &Value) -> Option<String>;
    fn properties_of(&self, value: &Value) -> Option<Vec<(String, Value)>>;
}
pub struct ContractError { pub contract: &'static str, pub field: String, pub problem: String }

pub struct AppPart { pub id: String, pub label: String, pub default: bool }
pub struct App { pub id: String, pub label: String, pub parts: Vec<AppPart>, pub drivers: Vec<String> }
pub struct CommandArgument { pub name: String, pub description: String, pub required: bool }
pub struct CommandOption { pub name: String, pub description: String, pub takes_value: bool,
                           pub default: Option<String> }
pub struct Command { pub name: String, pub description: String, pub handler: Value,
                     pub arguments: Vec<CommandArgument>, pub options: Vec<CommandOption> }

pub fn decode_apps(value: &Value, reader: &dyn ObjectReader) -> Result<Vec<App>, ContractError>;
pub fn decode_commands(value: &Value, reader: &dyn ObjectReader) -> Result<Vec<Command>, ContractError>;
```

- Rules: the top-level value must be a list. Each element must be an object whose class is exactly
  `SpeedySpec\Cli\Contracts\App` (or `...\Command`). `id` and `name` match `^[a-z][a-z0-9-]*$`. `label` is
  non-empty. Ids are unique within their list (apps, parts within an app, commands, arguments and options within a
  command). `field` is a path such as `[1].parts[0].id`.

- [ ] **Step 1: Write the failing tests** with a fake `ObjectReader` backed by a map from an `Rc` pointer to
  `(class, properties)`:

```rust
#[test] fn decodes_an_app_with_parts_and_drivers()
#[test] fn rejects_non_list()                 // Value::Null -> field "", problem contains "list"
#[test] fn rejects_wrong_class()              // class "stdClass" -> field "[0]", problem names the expected class
#[test] fn rejects_bad_id()                   // id "Word Press" -> field "[0].id"
#[test] fn rejects_duplicate_app_ids()        // two "wp" -> field "[1].id"
#[test] fn rejects_nested_part_error_with_path() // part label "" -> field "[0].parts[0].label"
#[test] fn decodes_a_command_and_keeps_handler() // handler Opaque is the same Rc
#[test] fn rejects_command_without_callable_handler() // handler Value::Null -> field "[0].handler"
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-driver`. Expected: does not compile.
- [ ] **Step 3: Implement `contracts.rs`.**
- [ ] **Step 4: Run.** Expected: 8 passed.
- [ ] **Step 5: Commit.** `git commit -am "feat: add Rust mirror of app and command contracts"`

---

### Task 6: Plugin discovery and loading (`speedyspec-plugin`)

**Files:**
- Create: `crates/speedyspec-plugin/Cargo.toml`, `src/lib.rs`, `src/manifest.rs`, `src/discover.rs`, `src/load.rs`
- Test: unit tests in each file. Loading uses a fake `Loader` here; real PHP loading is tested through the binary
  in Task 7.

**Interfaces:**
- Consumes: `semver::{Version, VersionReq}`.
- Produces:

```rust
pub struct Manifest { pub name: String, pub version: Version, pub contracts: VersionReq, pub entry: PathBuf }
pub struct Plugin { pub manifest: Manifest, pub dir: PathBuf }
pub enum PluginError {
    Manifest { dir: PathBuf, problem: String },
    Contracts { name: String, needs: String, bundled: String },
    Load { name: String, problem: String },
}
pub fn parse_manifest(dir: &Path) -> Result<Manifest, PluginError>;
pub fn user_plugin_dir() -> PathBuf;     // $SPEEDYSPEC_HOME/plugins; SPEEDYSPEC_HOME defaults to the platform
                                         // per-user data directory + "speedyspec"
pub fn discover(dirs: &[PathBuf]) -> (Vec<Plugin>, Vec<PluginError>);

pub trait Loader {                       // Php implements this in Task 7; tests use a fake
    fn load(&self, plugin: &Plugin) -> Result<(), String>;
}
pub struct LoadReport { pub loaded: Vec<Plugin>, pub skipped: Vec<PluginError> }
pub fn load_all(plugins: Vec<Plugin>, bundled_contracts: &Version, loader: &dyn Loader) -> LoadReport;
```

- Manifest file is `speedyspec-plugin.json` with `name`, `version`, `contracts`, `entry`. `name` matches
  `^[a-z0-9-]+/[a-z0-9-]+$`. `contracts` uses Composer's caret/tilde syntax, which `semver::VersionReq` accepts for
  the forms used here. `entry` is relative, must stay inside the plugin directory after resolving `..` and
  symlinks, and must exist.
- `discover` reads each directory in the order given; a plugin is any immediate subdirectory containing a manifest,
  and a directory that itself contains a manifest counts as one plugin. Within a directory, plugins are ordered by
  name. When two plugins share a name, the one from the later directory wins and the earlier is dropped silently.
  A missing directory is not an error.
- `load_all` checks the contract range first, then calls the loader. A failure of one plugin never stops the others.

- [ ] **Step 1: Write the failing tests:**

```rust
#[test] fn parses_a_valid_manifest()
#[test] fn rejects_missing_field()                 // no "entry" -> problem names "entry"
#[test] fn rejects_bad_name()                      // "WordPress" -> problem names "name"
#[test] fn rejects_entry_outside_plugin_dir()      // "../evil.php" -> problem contains "outside"
#[test] fn rejects_entry_symlink_escaping_dir()    // entry is a symlink to a file outside
#[test] fn rejects_missing_entry_file()
#[test] fn discovers_subdirectories_in_name_order()
#[test] fn directory_with_manifest_is_one_plugin() // for --plugin-dir pointing straight at a plugin
#[test] fn later_directory_wins_on_same_name()
#[test] fn missing_directory_is_not_an_error()
#[test] fn skips_plugin_whose_contract_range_is_not_met()
    // bundled 1.0.0, plugin "^2.0" -> Contracts { needs: "^2.0", bundled: "1.0.0" }; loader not called for it
#[test] fn one_failing_plugin_does_not_stop_others() // fake loader fails "a/a"; "b/b" is loaded; skipped has 1
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-plugin`. Expected: does not compile.
- [ ] **Step 3: Implement `manifest.rs`, `discover.rs`, `load.rs`.**
- [ ] **Step 4: Run.** Expected: 12 passed.
- [ ] **Step 5: Commit.** `git commit -am "feat: add plugin manifest, discovery and loading"`

---

### Task 7: The `speedyspec` binary

**Files:**
- Create: `crates/speedyspec-cli/Cargo.toml`, `build.rs`, `src/main.rs`, `src/boot.rs`, `src/commands.rs`,
  `src/app.rs`
- Test: `crates/speedyspec-cli/tests/cli.rs` (runs the built binary), `tests/fixtures/plugins/`

**Interfaces:**
- Consumes: everything above. `Php` gets `impl ObjectReader` and `impl Loader` here (in `boot.rs`, as thin wrappers,
  so neither `speedyspec-driver` nor `speedyspec-plugin` depends on PHP).
- Produces: the `speedyspec` executable.

- `build.rs` reads the contracts source from `SPEEDYSPEC_CONTRACTS_DIR` (default `../../../speedyspec-cli-contracts`,
  the sibling checkout), embeds every file under `src/`, and exposes the `version` from its `composer.json` as
  `CONTRACTS_VERSION`. It fails the build with a clear message if the directory is missing. (Fetching a pinned
  release instead is plan 4.)
- Start-up order in `boot.rs`:
  1. Pre-scan the arguments for `--no-plugins`, `--no-interaction` and every `--plugin-dir <path>`, without clap,
     because plugins must load before the command tree exists.
  2. `Rc::new(MemoryHooks::new())` as `SharedHooks` (the only place a backend is named), then `Php::boot` with
     the contracts as `preload`.
  3. Unless `--no-plugins`: `discover([user_plugin_dir(), ...plugin_dirs])`, then `load_all`. While each plugin
     loads, `set_owner(name)` and `redirect_output(Stderr)`; afterwards restore both. Each skipped plugin prints one
     line to stderr: `speedyspec: plugin <name> skipped: <why>`.
  4. `hooks.run("speedyspec_init", &[])`.
  5. `hooks.filter("speedyspec_commands", empty list, &[])` → `decode_commands`;
     `hooks.filter("speedyspec_apps", empty list, &[])` → `decode_apps`. A hook or contract error prints
     `speedyspec: <owner or "a plugin">: <hook>: <field>: <problem>` to stderr and that list is treated as empty.
  6. Build the clap tree: built-ins first, then plugin commands. A plugin command whose name equals a built-in or
     an earlier plugin command is dropped with `speedyspec: command "<name>" from a plugin ignored: name already
     taken` on stderr.
  7. Parse and dispatch. A clap usage error exits 2.
- Built-ins in this plan:
  - `speedyspec --version` prints exactly three lines: `speedyspec <cargo version>`, `php <PHP_VERSION>`,
    `contracts <CONTRACTS_VERSION>`.
  - `speedyspec php <file> [args...]` runs the file; exit code is the script's.
  - `speedyspec app list` prints one line per app, `<id>\t<label>\t<drivers joined by ",">`. A listed driver that
    the binary does not implement is shown with a trailing `?`; in this plan that is every driver. With no apps it
    prints `No apps. Install a plugin that provides one.` to stderr and exits 0.
  - `speedyspec plugin list` prints `<name>\t<version>\t<dir>` per loaded plugin.
  - `app`, `plugin`, `self` and `php` are reserved names even where their subcommands arrive in later plans.
- A plugin command runs its handler through `Php::call` with `[arguments, options]` as two string-keyed arrays; the
  integer it returns is the exit code; a non-integer return is exit 0; an exception prints
  `speedyspec: <command>: <class>: <message>` to stderr and exits 1; `exit(n)` in the handler exits `n`.

- [ ] **Step 1: Create the fixture plugins** under `tests/fixtures/plugins/`, each a directory with a manifest and
  `plugin.php`:

| Fixture | What its `plugin.php` does |
|---|---|
| `good` | Adds app `demo` ("Demo", parts `web` and `worker`, drivers `zip`) and command `hello` with argument `name` and option `--shout`; handler echoes `hello <name>` (upper-cased with `--shout`) and returns 0. Uses `Hook::add` for one and `add_filter` for the other. |
| `noisy` | `echo "loading noisy\n";` then adds nothing. |
| `exits` | Calls `exit(7);` at load. |
| `fatal` | Calls `undefined_function_xyz();` at load. |
| `needs-v2` | Manifest `"contracts": "^2.0"`. |
| `bad-apps` | `speedyspec_apps` filter returns `'nope'`. |
| `bad-app-field` | `speedyspec_apps` filter appends `new App('Bad Id', 'X', [], [])`. |
| `clash` | Adds commands `app` and `hello`. |
| `exit-handler` | Command `quit` whose handler calls `exit(5);`. |
| `throws` | Command `boom` whose handler throws `RuntimeException('kaboom')`. |

- [ ] **Step 2: Write the failing tests in `tests/cli.rs`.** Each runs the binary with `SPEEDYSPEC_HOME` set to an
  empty temp directory and the named fixtures passed by `--plugin-dir`:

```rust
#[test] fn version_prints_three_lines()            // lines start "speedyspec ", "php 8.", "contracts 1.0.0"; exit 0
#[test] fn runs_with_no_plugins_at_all()           // `app list` -> exit 0, stdout empty, stderr has "No apps"
#[test] fn php_command_runs_a_file_with_hooks()    // file: add_filter + echo apply_filters -> stdout "42"; exit 0
#[test] fn php_command_exit_code_is_scripts()      // file calls exit(4) -> exit code 4
#[test] fn plugin_app_is_listed()                  // good -> stdout "demo\tDemo\tzip?\n"
#[test] fn plugin_command_runs()                   // good; `hello Ada` -> "hello Ada"; `hello Ada --shout` -> "HELLO ADA"
#[test] fn plugin_command_appears_in_help()        // good; `--help` stdout contains "hello"
#[test] fn plugin_list_shows_loaded()              // good -> stdout has "speedyspec-fixture/good\t1.0.0\t"
#[test] fn no_plugins_flag_skips_loading()         // good + --no-plugins; `hello Ada` -> exit 2
#[test] fn load_output_goes_to_stderr()            // noisy; `--version` stdout is exactly the three lines;
                                                   // stderr contains "loading noisy"
#[test] fn exit_at_load_skips_and_names_plugin()   // exits + good; `hello Ada` still works; stderr has
                                                   // "plugin speedyspec-fixture/exits skipped"
#[test] fn fatal_at_load_skips_and_names_plugin()  // fatal + good; same, stderr names "fatal" and
                                                   // "undefined_function_xyz"
#[test] fn contract_range_mismatch_is_reported()   // needs-v2; stderr has "^2.0" and "1.0.0"
#[test] fn wrong_filter_return_is_reported()       // bad-apps; `--version` exit 0;
                                                   // stderr has "speedyspec-fixture/bad-apps" and "speedyspec_apps"
#[test] fn bad_field_names_plugin_hook_and_field() // bad-app-field; stderr has "speedyspec_apps" and "[0].id"
#[test] fn builtin_wins_over_plugin_command()      // clash; `app list` still the built-in; stderr has
                                                   // `command "app" from a plugin ignored`
#[test] fn first_plugin_wins_duplicate_command()   // good then clash; `hello Ada` -> "hello Ada"; stderr reports "hello"
#[test] fn exit_in_handler_is_exit_code()          // exit-handler; `quit` -> exit code 5
#[test] fn exception_in_handler_is_reported()      // throws; `boom` -> exit 1; stderr has "RuntimeException: kaboom"
#[test] fn unknown_command_is_usage_error()        // `nope` -> exit 2
#[test] fn working_directory_plugins_are_not_loaded()
    // cwd contains .speedyspec/plugins/good; no --plugin-dir; `hello Ada` -> exit 2
```

- [ ] **Step 3: Run.** `cargo test -p speedyspec-cli`. Expected: does not compile.
- [ ] **Step 4: Implement `build.rs` and `boot.rs`.**
- [ ] **Step 5: Implement `commands.rs`** (clap tree from built-ins plus decoded commands, dispatch, exit codes) **and
  `app.rs`** (`app list`).
- [ ] **Step 6: Run on macOS.** `cargo test --workspace`. Expected: all pass.
- [ ] **Step 7: Run on Linux.** `scripts/linux.sh cargo test --workspace`. Expected: all pass.
- [ ] **Step 8: Check the binary is self-contained.** `cargo build --release -p speedyspec-cli`, then run
  `env -i ./target/release/speedyspec --version` (empty environment, so no system PHP on `PATH`). Expected: the
  three version lines. Record the binary size in `docs/php-embed.md`.
- [ ] **Step 9: Write `README.md`**: what the CLI is, how to build it (`scripts/build-php.sh`, `cargo build`), the
  manifest format, a ten-line example plugin using `Hook::add`, and the hooks this plan fires.
- [ ] **Step 10: Commit.** `git add -A && git commit -m "feat: add speedyspec binary with plugin commands and app list"`

---

## What this plan leaves for later plans

- `app install`, `speedyspec_install_options`, `speedyspec_install_before` / `_after`, prompts, `--no-interaction`
  behaviour beyond being accepted, the zip driver, `plugin install` / `plugin remove` (plan 2).
- The Kubernetes driver and `app uninstall` (plan 3).
- Hook backends other than memory (NATS, RabbitMQ). Not in any of the four plans; the contract and conformance
  suite from Task 2 are what they build on.
- `self update`, release binaries for all four targets, fetching a pinned contracts release at build time (plan 4).
