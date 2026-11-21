# SpeedySpec PHP Runtime and Admin Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move embedded PHP out of the server into a separately released `speedyspec-php-runtime` binary that the server supervises and upgrades, and give the server the admin gate: release-declared access rules, session answers from the admin API, revocation, and per-app maintenance.

**Architecture:** A contract crate (`speedyspec-runtime-contract`) holds everything the two binaries share: the worker spec, the per-request report, the contract major. A new app kind `runtime` is released like the server binary (one binary per target, manifest printed by the binary). The server talks to PHP apps only over FastCGI. The gate's questions, the revocation feed, health probes and migrations are ordinary internal requests to an app's pool, sent on a worker kept for them. The gate is an `AccessCheck` implementation that matches the release's access rules, then asks the gate app.

**Tech Stack:** Rust 1.99 (edition 2024), Salvo 1.2, tokio, `speedyspec-fastcgi` (FastCGI 1.0), serde/serde_json, globset (already used by `speedyspec-server-static` for cache rules), sha2.

**Spec:** `docs/superpowers/specs/2026-11-19-speedyspec-php-runtime-and-admin-gate-design.md`. Its "Replicas" section governs every cluster step below (amends `2026-11-17-speedyspec-server-core-design.md` and `2026-11-18-speedyspec-php-foundation-design.md`).

**Repository:** `packages/speedyspec-server` (its own git repository, `SpeedySpec/speedyspec-server`). Branch `feat/php-runtime-gate`, cut from `feat/php-foundation` (PR #4, not yet merged), because this plan moves code that PR adds. Rebase onto `main` once #4 merges.

## Global Constraints

- Rust 1.99, edition 2024. Unix only: Linux x86_64 and aarch64 (musl, static), macOS arm64 for development.
- TLS stays on `ring`: `cargo tree -i aws-lc-sys --target all` prints nothing.
- Contracts live in their own crate; implementations in separate crates. Crate names use the DDD layer names where a layer applies.
- The server binary builds and passes its suite with no static PHP present. Only `speedyspec-php-runtime` and the PHP crates need `.php-build`.
- The PHP foundation's pool behaviour is kept: one request per worker, poison and recycling, `prepare`/`activate`/`retire`, the site directory.
- A visitor never reaches an internal path: gate paths named `changes`, health probes, and the migration path answer 404 to visitors and are never forwarded.
- Status codes for gated paths: not signed in → 401; signed in without the page → 403; disabled conditional entry → 404; all three alike for paths that exist and paths that don't. Protected responses carry `Cache-Control: private, no-store`.
- Defaults, verbatim from the spec: `gate.session_ttl` 60 s, `gate.timeout` 5 s, `gate.site_ttl` 2 s, `gate.poll` 1 s, `gate.max_pending` 16, `health.interval` 2 s, `health.fail_after` 3, `health.pass_after` 3, `switch.migrate_timeout` 30 minutes, `runtime.trial_period` 2 minutes.
- Contract major is `1`. Gate contract version is `1`.
- Every feature works on independent replicas (no `[cluster]`). Where `[cluster]` is configured, it uses the existing cluster mode (`Cluster::is_single()` picks the branch). No cluster code is removed.
- Tests pass on macOS arm64 and Linux musl (`scripts/linux-php.sh` for anything that needs PHP).
- Conventional Commits, ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- After the last Linux run, delete that run's Linux build output (`target-linux*/` or the Docker target volume it created); keep `.php-build-linux*`.

## Review Focus

1. **A cookie header with several cookies, one of which the sign-out clears.** The remembered answer for the whole header must be forgotten. Pinned in Task 9 (`signout_clearing_one_of_several_cookies_forgets_the_header`).
2. **A visitor path that matches an internal path after normalisation** (`/admin-api/v1/gate//changes`, `%2e`, trailing slash). It must still get 404 and never reach PHP. Pinned in Task 6 (`internal_paths_match_after_normalisation`) and Task 8 (`internal_path_variants_get_404`).
3. **The gate app is not installed yet, or has no live release.** Gated mounts must fail closed (401), not 503 or open. Pinned in Task 8 (`no_gate_release_fails_closed`).
4. **A runtime switch while an app's migration is in flight.** It must be refused rather than started at the same time. Pinned in Task 11 (`runtime_switch_refused_during_migration`).
5. **A health probe answering slowly but within the timeout, alternating with failures.** Maintenance must not flap: the counters reset on the opposite result. Pinned in Task 6 (`alternating_results_never_flip`).

---

## File Structure

**New crates (workspace `packages/speedyspec-server/crates/`):**
- `speedyspec-runtime-contract/`: `lib.rs` (`CONTRACT_MAJOR`, re-exports), `spec.rs` (`WorkerSpec`), `report.rs` (`REPORT_PREFIX`, `WorkerReport`), `manifest.rs` (`RuntimeManifest` printed by `manifest`).
- `speedyspec-php-runtime/`: `main.rs` (`worker --spec`, `manifest`), `worker.rs` (moved from `speedyspec-server/src/php/worker.rs`), `tests/` (worker tests moved with it).
- `speedyspec-server-gate/`: the gate, framework-neutral. `rules.rs` (rule matching), `session.rs` (remembered answers, coalescing, cap), `revocation.rs` (changes poll, sign-out detection), `maintenance.rs` (per-app sources), `health.rs` (probe counters), `calls.rs` (the `GateCalls` port: site, session, changes, probe, migrate).

**Modified:**
- `speedyspec-server-domain/src/app.rs`: `AppKind::Runtime`, `AppKind::is_binary`.
- `speedyspec-server-domain/src/manifest.rs`: `PhpSection.runtime`, `.access`, `.gate`, `.health`, `.switch`; `StaticSection.access`, `.maintenance_page`; `ReleaseManifest.contracts`; `AccessRule`, `PublicWhen`, `GateSection`, `HealthSection`, `SwitchSection`, `MaintenanceScope`.
- `speedyspec-server-domain/src/access.rs`: `AccessDecision` gains `Forbidden`, `Disabled`, `Busy`; `AccessRequest` gains `app`, `rel_path`.
- `speedyspec-server-domain/src/state.rs`: `ReleaseState.maintenance`, `ReleaseState.runtime_trials`, `StagedSet.migrated`.
- `speedyspec-server-domain/src/cluster.rs`: `MemberReport.unhealthy`.
- `speedyspec-server-app/src/releases.rs`: binary kinds generalised (`is_binary`); the runtime requirement in the set check; migration step in switches; runtime trial and rollback.
- `speedyspec-server-app/src/compat.rs`: `check_set` takes the php-app → runtime-app map.
- `speedyspec-server-presentation/src/service.rs`: builds the extended `AccessRequest`; answers maintenance and the new decisions; 404 for internal paths.
- `speedyspec-server-presentation/src/handlers.rs`: removes `AdminApiPlaceholder` and `PhpPlaceholder`; `MaintenanceReply`.
- `speedyspec-server/src/config.rs`: `kind = "runtime"`, `app.<id>.runtime`, `app.<id>.php.env`/`env_file`, `[gate]`, `[health]`, `[runtime]`, `maintenance_page`; `AccessImpl::Gate`; removal of the built-in `/admin/api/` mount.
- `speedyspec-server/src/php/`: `pool.rs` spawns the runtime binary, keeps one internal worker; `runtime.rs` resolves the app's runtime binary; new `runtime_kind.rs` (the `runtime` kind's `AppRuntime`); new `calls.rs` (the FastCGI adapter of `GateCalls`). `worker.rs` leaves.
- `speedyspec-server/src/control.rs`, `main.rs`: `maintenance on|off|clear`, `release retry-migration`.
- `speedyspec-server/Cargo.toml`: the `php` feature and the PHP dependencies are removed.
- `.github/workflows/ci.yml`, `scripts/linux-php.sh`, `docs/php.md`, `docs/architecture.md`, `README.md`.

---

### Task 1: The contract crate and the runtime binary

**Files:**
- Create: `crates/speedyspec-runtime-contract/{Cargo.toml,src/lib.rs,src/spec.rs,src/report.rs,src/manifest.rs}`
- Create: `crates/speedyspec-php-runtime/{Cargo.toml,src/main.rs,src/worker.rs}`; move `crates/speedyspec-server/src/php/worker.rs` → `crates/speedyspec-php-runtime/src/worker.rs`
- Modify: `crates/speedyspec-server/src/php/{mod.rs,pool.rs,runtime.rs}`, `crates/speedyspec-server/src/main.rs` (remove the hidden `php-worker` role), workspace `Cargo.toml`
- Test: `crates/speedyspec-runtime-contract/src/*.rs` (unit), `crates/speedyspec-php-runtime/tests/worker.rs`, `crates/speedyspec-server/tests/support/runtime.rs`

**Interfaces:**
- Produces:
  - `speedyspec_runtime_contract::CONTRACT_MAJOR: u32 = 1`
  - `WorkerSpec { contract: u32, app: String, socket: PathBuf, release_dir: PathBuf, site_dir: Option<PathBuf>, ini: BTreeMap<String,String>, bindings: Vec<(String,String)>, env: Vec<(String,String)> }`, serde, with `contract` required.
  - `WorkerSpec::check(&self) -> Result<(), String>`: `Err` naming both majors when `contract != CONTRACT_MAJOR`.
  - `REPORT_PREFIX: &[u8] = b"\0speedyspec:"`
  - `WorkerReport { memory: u64, requests: u64, poisoned: bool }` with `fn encode(&self) -> Vec<u8>` and `fn decode(record: &[u8]) -> Option<WorkerReport>`. The format stays `memory=<n> requests=<n> poisoned=<0|1>` after the prefix.
  - `RuntimeManifest { version: semver::Version, contracts: Vec<u32>, php: String }`: what `speedyspec-php-runtime manifest` prints as JSON. `php` is the embedded PHP version.
  - The binary `speedyspec-php-runtime` with subcommands `worker --spec <json>` and `manifest`.
  - `EmbeddedPhpPool::start(exe: PathBuf, spec: WorkerSpec, dir, settings, version)`: unchanged signature; `exe` is now the runtime binary, spawned as `exe worker --spec <json>`.
  - Test helper `support::runtime::binary() -> PathBuf`: builds `speedyspec-php-runtime` once per test process with `$CARGO build -p speedyspec-php-runtime` (same profile and target as the test) and returns its path.

- [ ] **Step 1: Write the failing contract tests** in `spec.rs` and `report.rs`:

```rust
#[test]
fn a_spec_of_another_major_is_refused_naming_both() {
    let spec = WorkerSpec { contract: 2, ..sample() };
    let err = spec.check().unwrap_err();
    assert!(err.contains("contract 2") && err.contains("contract 1"), "{err}");
}

#[test]
fn a_spec_without_contract_does_not_parse() {
    let json = r#"{"app":"a","socket":"/s","release_dir":"/r","site_dir":null,"ini":{},"bindings":[],"env":[]}"#;
    assert!(serde_json::from_str::<WorkerSpec>(json).is_err());
}

#[test]
fn unknown_spec_fields_are_ignored() {
    // Adding a field is a minor contract change: the other side ignores it.
    let mut value = serde_json::to_value(sample()).unwrap();
    value["later_field"] = serde_json::json!(true);
    assert!(serde_json::from_value::<WorkerSpec>(value).is_ok());
}

#[test]
fn report_round_trips_and_rejects_other_records() {
    let report = WorkerReport { memory: 1234, requests: 7, poisoned: true };
    assert_eq!(WorkerReport::decode(&report.encode()), Some(report));
    assert_eq!(WorkerReport::decode(b"PHP Warning: x"), None);
}
```

- [ ] **Step 2: Run them.** `cargo test -p speedyspec-runtime-contract`. Expected: FAIL to compile (crate empty).
- [ ] **Step 3: Implement the contract crate.** `WorkerSpec`, `WorkerReport` and `REPORT_PREFIX` move here from `speedyspec-server/src/php/{worker.rs,mod.rs}`. The pool's report parsing switches to `WorkerReport::decode`. The new `env` field replaces the worker's own `.env` reading: the server fills it (Task 5), and the worker applies it before PHP starts.
- [ ] **Step 4: Run them.** Expected: PASS.
- [ ] **Step 5: Write the failing runtime binary test** `crates/speedyspec-php-runtime/tests/worker.rs`:
  - `manifest_prints_contract_majors`: runs `env!("CARGO_BIN_EXE_speedyspec-php-runtime") manifest`, parses `RuntimeManifest`, asserts `contracts == [1]` and `version == env!("CARGO_PKG_VERSION")`.
  - `worker_refuses_another_major`: runs `worker --spec` with `contract: 9`, asserts exit status 2 and that stderr contains `contract 9`.
  - Move the existing worker unit tests (`read_env`, `quote`, preload) with the file.
- [ ] **Step 6: Run it.** `cargo test -p speedyspec-php-runtime`. Expected: FAIL (no binary).
- [ ] **Step 7: Implement `speedyspec-php-runtime`.**
  - `main.rs`: clap with `worker --spec` (calls `worker::run_worker`, which first calls `spec.check()` and exits 2 on `Err`) and `manifest`.
  - The crate depends on the PHP crates (`speedyspec-php`, `-php-ext`, `-container(-php)`, `-hooks(-memory)`, `-wp-hooks(-memory,-php)`, `speedyspec-fastcgi`, `speedyspec-runtime-contract`).
  - Its version starts at `1.0.0`.
- [ ] **Step 8: Point the server's pool at the runtime binary.** `PhpRuntime::new` takes the runtime binary path in place of `current_exe` (Task 3 makes it per app). The spawn becomes `exe worker --spec <json>`. Remove the `php-worker` subcommand from `speedyspec-server/src/main.rs`. In the server's PHP tests (`tests/php.rs`, `tests/serve.rs`), pass `support::runtime::binary()`.
- [ ] **Step 9: Run both suites.** `cargo test -p speedyspec-php-runtime && cargo test -p speedyspec-server --features php --test php --test serve`. Expected: PASS, same counts as before plus the new tests.
- [ ] **Step 10: Commit.** `feat: the PHP runtime as its own binary, over a versioned worker contract`.

### Task 2: The `runtime` app kind

**Files:**
- Modify: `crates/speedyspec-server-domain/src/{app.rs,manifest.rs,error.rs}`, `crates/speedyspec-server-app/src/{releases.rs,compat.rs}`, `crates/speedyspec-server/src/{config.rs,boot.rs,managed.rs}`, `crates/speedyspec-server-infra-*/` where `AppKind::Server` is matched for binaries
- Test: `crates/speedyspec-server-domain/src/manifest.rs` (unit), `crates/speedyspec-server-app/tests/runtime_releases.rs`, `crates/speedyspec-server/src/config.rs` (unit)

**Interfaces:**
- Consumes: `RuntimeManifest` (Task 1).
- Produces:
  - `AppKind::Runtime` (JSON `"runtime"`) and `AppKind::is_binary(self) -> bool` (true for `Server` and `Runtime`).
  - `ReleaseManifest.contracts: Vec<u32>`: only for kind `runtime`, required and non-empty there.
  - `PhpSection.runtime: Option<semver::VersionReq>`.
  - `AppConfig.runtime: Option<AppId>`: for `php` apps; defaults to the only `runtime` app, and is an error when there are none or several.
  - `check_set(set: &BTreeMap<AppId, ReleaseManifest>, runtimes: &BTreeMap<AppId, AppId>) -> Result<(), Vec<Incompatibility>>`: `runtimes` maps each php app to its runtime app.
  - `RuntimeInspector` (the adapter of `BinaryInspector` for kind `runtime`): runs `<binary> manifest` and maps `RuntimeManifest` to `ReleaseManifest { kind: Runtime, app, version, contracts, requires: {} }`.

- [ ] **Step 1: Write failing tests.**
  - `manifest.rs`:
    - `runtime_manifest_requires_contracts` (kind `runtime` with empty `contracts` is `DomainError::MissingSection { kind: "runtime", section: "contracts" }`);
    - `contracts_only_on_runtime`;
    - `php_section_parses_runtime_requirement` (`"runtime": "^1.4"`).
  - `compat.rs`: `php_runtime_requirement_uses_the_configured_runtime`. The blog requires `^1.4`, its runtime app is `rt`, and `rt` is `1.3.0`, so the result is an `Incompatibility` naming `blog` and `rt`. With `rt` at `1.4.2` it is `Ok`. An entry under `requires` naming an unconfigured app is still ignored.
  - `config.rs`:
    - `php_app_defaults_to_the_only_runtime`;
    - `php_app_with_two_runtimes_must_name_one` (error text contains `app.blog.runtime`);
    - `php_app_with_no_runtime_is_an_error`.
  - `runtime_releases.rs` (app crate, with the in-memory infra):
    - `runtime_release_installs_the_target_artifact`: index with `aarch64-unknown-linux-musl` and `x86_64-unknown-linux-musl` artifacts; the manager's target is picked; the inspector's manifest is checked.
    - `runtime_artifact_whose_manifest_differs_is_refused`.
  - `config.rs`: `immutable_runtime_from_path` (`mode = "immutable"`, `app.rt.kind = "runtime"`, `path = "/opt/rt/1.0.0"` parses; `source` with `path` is the existing error).
  - In the server's immutable boot (`boot.rs`), the runtime's binary is `<path>/speedyspec-php-runtime`, and its manifest is read with `RuntimeInspector`.
- [ ] **Step 2: Run them.** `cargo test -p speedyspec-server-domain -p speedyspec-server-app -p speedyspec-server --lib`. Expected: FAIL to compile.
- [ ] **Step 3: Implement.**
  - Replace every `kind == AppKind::Server` that means "a binary artifact" with `kind.is_binary()`; the code at `releases.rs:406`, `:472`, `:515`, `:1010`, `:1432` decides this. Leave the uses that mean "the server itself" (handoff, `ServerSwitchViaKeeper`, `server_app`) alone.
  - `ReleaseManager` gets one inspector per binary kind: `inspectors: HashMap<AppKind, Arc<dyn BinaryInspector>>`, replacing `inspector`.
  - Add `runtimes: BTreeMap<AppId, AppId>` to `ReleaseManager`, passed by `new`. Every `check_set` call passes it.
- [ ] **Step 4: Run them.** Expected: PASS. Run the app and server suites (`cargo test -p speedyspec-server-app -p speedyspec-server --features test-hooks`). Expected: PASS.
- [ ] **Step 5: Commit.** `feat: the runtime app kind, released like the server binary`.

### Task 3: Runtime switches move every PHP app to new pools

**Files:**
- Create: `crates/speedyspec-server/src/php/runtime_kind.rs`
- Modify: `crates/speedyspec-server/src/php/runtime.rs`, `crates/speedyspec-server/src/boot.rs`, `crates/speedyspec-server/src/serving.rs`
- Test: `crates/speedyspec-server/tests/runtime_switch.rs`

**Interfaces:**
- Consumes: `AppKind::Runtime`, `AppConfig.runtime`, `RuntimeManifest`, `support::runtime::binary()`.
- Produces:
  - `RuntimeBinaries` (`Arc`-shared): `fn active(&self, runtime: &AppId) -> Option<(Version, PathBuf, Vec<u32>)>` and `fn set(&self, runtime: &AppId, version: Version, binary: PathBuf, contracts: Vec<u32>)`.
  - `PhpRuntime::prepare` resolves the binary through `RuntimeBinaries::active(config.runtime)`. It fails with `"<app>: runtime <id> speaks contract <list>, this server speaks contract 1"` when `CONTRACT_MAJOR` isn't in the runtime's `contracts`, and with `"<app>: runtime <id> has no active release"` when there is none.
  - `RuntimeKindRuntime` implements `AppRuntime` for kind `Runtime`:
    - `prepare(app, manifest, dir)` checks contracts, then calls `PhpRuntime::prepare_with(php_app, binary)` for every php app whose `runtime == app` and that has an active release. Its `Prepared.payload` holds the binary and every app's prepared pool. If any one fails, every pool already started is drained and it returns `Err`.
    - `activate` calls `RuntimeBinaries::set`, then activates every prepared pool in one pass.
    - `retire(app, version)` drains the pools started from `version`'s binary.
  - `PhpRuntime::prepare_with(&self, app: &AppId, manifest: &ReleaseManifest, dir: &Path, binary: &Path) -> Result<Prepared, String>`.

- [ ] **Step 1: Write the failing tests** in `tests/runtime_switch.rs`. The fixture runtime release directories each hold a copy of the built runtime binary, plus a renamed copy whose `manifest` prints version `1.1.0` (the copy is made by a test-only `SPEEDYSPEC_RUNTIME_VERSION` env var the binary reads under `cfg(debug_assertions)`; my call, the same mechanism as the server's `test-hooks` version file):
  - `runtime_switch_moves_every_php_app`: two php apps (`blog`, `frontend-api`) on runtime `rt` 1.0.0. Switch `rt` to 1.1.0 while a client loops requests to both. Afterwards each app's `X-Runtime-Pid` fixture header comes from a process whose binary is the 1.1.0 copy, and the loop saw no failed request.
  - `runtime_that_fails_to_prepare_switches_nothing`: the 1.1.0 binary exits at once. The switch returns `Err` naming the app that failed, both apps still answer from 1.0.0, and `failed` records 1.1.0.
  - `contract_mismatch_is_refused_naming_both`: the runtime's `manifest` prints `contracts: [2]`. `Err` contains `contract 2` and `contract 1`.
- [ ] **Step 2: Run them.** `cargo test -p speedyspec-server --features php --test runtime_switch`. Expected: FAIL to compile.
- [ ] **Step 3: Implement** `RuntimeBinaries`, `RuntimeKindRuntime` and `PhpRuntime::prepare_with`. Register `RuntimeKindRuntime` in `boot.rs` for `AppKind::Runtime`. At boot, runtime apps are activated before php apps (my call: the php pools need a binary), so `ReleaseManager::boot` orders kinds `Static, Runtime, Php`.
- [ ] **Step 4: Run them.** Expected: PASS. Then run the server suite. Expected: PASS.
- [ ] **Step 5: Commit.** `feat: switching the runtime moves every PHP app to new pools in one step`.

### Task 4: The server binary without PHP

**Files:**
- Modify: `crates/speedyspec-server/Cargo.toml` (drop the `php` feature and the PHP dependencies), `crates/speedyspec-server/src/php/` (no PHP crates used), `crates/speedyspec-server-presentation/src/handlers.rs` (drop `PhpPlaceholder`), `.github/workflows/ci.yml`, `scripts/linux-php.sh`
- Test: `scripts/check-no-php-in-server.sh`, CI

**Interfaces:**
- Produces: the server crate depends on `speedyspec-fastcgi` and `speedyspec-runtime-contract` only, among the PHP-related crates. `php` apps always use the pool; there is no placeholder.

- [ ] **Step 1: Write the failing check** `scripts/check-no-php-in-server.sh`:

```bash
#!/usr/bin/env bash
# The server binary must not depend on the embedded PHP.
set -euo pipefail
deps=$(cargo tree -p speedyspec-server -e normal --prefix none | awk '{print $1}' | sort -u)
for crate in speedyspec-php speedyspec-php-ext ext-php-rs speedyspec-wp-hooks-php speedyspec-container-php; do
    if grep -qx "$crate" <<<"$deps"; then echo "speedyspec-server depends on $crate" >&2; exit 1; fi
done
echo "no PHP in speedyspec-server"
```

- [ ] **Step 2: Run it.** Expected: FAIL (`speedyspec-server depends on speedyspec-php`).
- [ ] **Step 3: Remove the feature.** Everything under `cfg(feature = "php")` in the server becomes unconditional, except the PHP crates' use, which is gone after Task 1. Move `tests/php.rs` and `tests/serve.rs` out from behind `required-features`.
- [ ] **Step 4: Prove it builds with no PHP.** Run `mv .php-build .php-build.away && cargo test -p speedyspec-server --features test-hooks --no-run; mv .php-build.away .php-build`. Expected: builds. Then run `scripts/check-no-php-in-server.sh`. Expected: `no PHP in speedyspec-server`.
- [ ] **Step 5: Update CI.**
  - The `test` job runs `cargo test --workspace --exclude speedyspec-php-runtime --exclude <PHP crates>` with no PHP build, plus the check script.
  - The `php` job builds PHP and runs `cargo test --workspace --features speedyspec-server/test-hooks`.
  - `scripts/linux-php.sh` is unchanged.
- [ ] **Step 6: Commit.** `build: the server binary holds no PHP`.

### Task 5: Per-app environment and the internal worker

**Files:**
- Modify: `crates/speedyspec-server/src/config.rs`, `crates/speedyspec-server/src/php/{pool.rs,runtime.rs}`
- Test: `crates/speedyspec-server/src/php/pool.rs` (unit, with the test spawner), `crates/speedyspec-server/src/config.rs` (unit), `crates/speedyspec-server/tests/php.rs`

**Interfaces:**
- Produces:
  - `PhpConfig.env: Vec<(String,String)>` from `app.<id>.php.env` (a TOML table).
  - `PhpConfig.env_file: Option<PathBuf>`.
  - `WorkerSpec.env` is filled from three sources, later ones winning: the site directory's `.env`, then `env_file`, then `env`. The server reads the files; the worker no longer does.
  - `EmbeddedPhpPool::acquire_internal(&self) -> impl Future<Output = Result<Lease, String>>`: leases the pool's internal worker, which is started with the pool, never counted against `workers.max`, and never used by `acquire`. It waits when busy. `release(lease, fate)` handles both kinds, and a killed internal worker is replaced.
  - `PhpRequestHandler::internal(&self, method: Method, path: &str, headers: HeaderMap, body: Bytes, timeout: Duration) -> impl Future<Output = Result<http::Response<Bytes>, InternalError>>`, where `InternalError = Timeout | Unavailable(String)`. It sends one internal FastCGI request through `acquire_internal`. Parameters are built as for a visitor request to the app's first mount, with `REMOTE_ADDR` empty and `SERVER_NAME` the mount's host or `localhost`.

- [ ] **Step 1: Write failing tests.**
  - `pool.rs`: `internal_worker_is_kept_apart`. With `workers.max = 1` and one visitor lease held, `acquire_internal` returns at once. A second `acquire` waits, and `counts()` reports one visitor worker.
  - `pool.rs`: `killed_internal_worker_is_replaced`.
  - `config.rs`: `env_sources_merge_in_order` (site `.env` `A=1 B=1`, `env_file` `B=2 C=2`, `env` `C=3` → `A=1 B=2 C=3`).
  - `tests/php.rs`: `internal_request_runs_while_visitors_fill_the_pool`. A fixture script sleeps 2 s, and `max = 1` with one visitor request in it. `internal(GET, "/health.php")` answers 200 within 1 s.
- [ ] **Step 2: Run them.** Expected: FAIL.
- [ ] **Step 3: Implement** as in Interfaces.
- [ ] **Step 4: Run them, then the server suite.** Expected: PASS.
- [ ] **Step 5: Commit.** `feat: per-app environments and a worker kept for the server's own calls`.

### Task 6: Health probes and per-app maintenance

**Files:**
- Create: `crates/speedyspec-server-gate/{Cargo.toml,src/lib.rs,src/health.rs,src/maintenance.rs,src/calls.rs,src/internal.rs}`
- Create: `crates/speedyspec-server/src/php/calls.rs` (the FastCGI `GateCalls` adapter over `PhpRequestHandler::internal`)
- Modify: `crates/speedyspec-server-domain/src/manifest.rs` (`HealthSection`, `StaticSection.maintenance_page`), `crates/speedyspec-server/src/config.rs` (`[health]`, `app.<id>.maintenance_page`), `crates/speedyspec-server-presentation/src/{service.rs,handlers.rs}`, `crates/speedyspec-server/src/{serving.rs,cluster.rs}` (`/readyz`)
- Test: `crates/speedyspec-server-gate/src/{health,maintenance}.rs` (unit), `crates/speedyspec-server/tests/maintenance.rs`

**Interfaces:**
- Produces:
  - `HealthSection { live: String, ready: String }` under `php.health`. Both are paths starting with `/`. When absent, the app has no probes and never enters maintenance by health.
  - `StaticSection.maintenance_page: Option<String>`: a file under `root`, checked to exist when the release is prepared.
  - `trait GateCalls: Send + Sync` in `speedyspec-server-gate::calls`, with these methods, each returning a boxed future:
    - `fn probe(&self, app: &AppId, path: &str) -> BoxFuture<'_, Result<u16, CallError>>`
    - `fn site(&self, app: &AppId, path: &str) -> BoxFuture<'_, Result<SiteState, CallError>>`
    - `fn session(&self, app: &AppId, path: &str, cookie: &HeaderValue) -> BoxFuture<'_, Result<SessionAnswer, CallError>>`
    - `fn changes(&self, app: &AppId, path: &str, since: &str) -> BoxFuture<'_, Result<Changes, CallError>>`
    - `fn migrate(&self, app: &AppId, version: &Version, path: &str, timeout: Duration) -> BoxFuture<'_, Result<u16, CallError>>`

    Its types: `CallError = Timeout | Unavailable(String) | Malformed(String)`; `SiteState { installed: bool, maintenance: bool }`; `SessionAnswer = SignedIn { user: u64, pages: BTreeSet<String> } | SignedOut`; `Changes { cursor: String, change: ChangeSet }`; `ChangeSet = Users(Vec<u64>) | Reset`. `site`, `session` and `changes` are used from Task 8 on.
  - `HealthCounter::new(fail_after: u32, pass_after: u32)` with `fn observe(&mut self, ok: bool) -> Option<bool>`. It returns `Some(true)` when the app turns unhealthy and `Some(false)` when it recovers. A result opposite to the current streak resets the streak.
  - `MaintenanceSet` (`Arc`-shared):
    - `fn set(&self, app: &AppId, source: MaintenanceSource, on: bool)`;
    - `fn is_on(&self, app: &AppId) -> bool`;
    - `fn sources(&self, app: &AppId) -> BTreeSet<MaintenanceSource>`;
    - `MaintenanceSource = Site | Upgrade | Operator | Health`.
  - `HealthMonitor::spawn(apps: Vec<(AppId, HealthSection)>, calls: Arc<dyn GateCalls>, set: Arc<MaintenanceSet>, interval: Duration, timeout: Duration, fail_after: u32, pass_after: u32, on_change: Arc<dyn Fn(&AppId, bool) + Send + Sync>)`. A probe counts as failed unless both `live` and `ready` answer 200 within `timeout`.
  - `ServeState.maintenance: Arc<MaintenanceSet>` and `ServeState.maintenance_pages: HashMap<AppId, Bytes>`, keyed by the app that shows the page.
  - The gate answers for an app in maintenance:
    - a `static` or `php` app with a configured `maintenance_page`: the page bytes, `text/html; charset=utf-8`, status 503, `Retry-After: 30`, `Cache-Control: no-store`;
    - any other app: status 503, `Retry-After: 30`, body `{"error":"maintenance"}` (`application/json`).
  - `fn is_internal(rel_path: &str, internal: &[String]) -> bool` in `speedyspec-server-gate::internal`. Each internal path is written with its leading `/`; the comparison is between normalised mount-relative paths, with a trailing `/` ignored.
  - `LiveApps::internal_paths(&self, app: &AppId) -> Arc<[String]>`, swapped with the handle on activate. In this task it holds the release's health paths; Task 8 adds `gate.changes` and Task 11 adds `switch.migrate`.
  - The presentation answers 404, before any access check or handler, when `is_internal(rel_path, live.internal_paths(app))`.
  - `/readyz` answers 503 while any app is in maintenance by `Health`.

- [ ] **Step 1: Write failing unit tests.**
  - `health.rs`:

```rust
#[test]
fn turns_unhealthy_after_three_failures_and_back_after_three_passes() {
    let mut c = HealthCounter::new(3, 3);
    assert_eq!([c.observe(false), c.observe(false), c.observe(false)], [None, None, Some(true)]);
    assert_eq!([c.observe(true), c.observe(true), c.observe(true)], [None, None, Some(false)]);
}

#[test]
fn alternating_results_never_flip() {
    let mut c = HealthCounter::new(3, 3);
    for i in 0..20 { assert_eq!(c.observe(i % 2 == 0), None); }
}
```

  - `internal.rs`: `internal_paths_match_after_normalisation`. With `internal = ["/v1/gate/changes"]`, both `v1/gate/changes` and `v1/gate/changes/` are internal, and `v1/gate/changesx` is not. (`%2e` and `//` are removed by the presentation layer's `normalize` before this point; Task 8's `internal_path_variants_get_404` covers those spellings end to end.)
  - `maintenance.rs`: `on_while_any_source_is_on` (Operator on and Health on → on; Health off → still on; Operator off → off) and `apps_are_independent`.
- [ ] **Step 2: Run them.** `cargo test -p speedyspec-server-gate`. Expected: FAIL to compile.
- [ ] **Step 3: Implement** `HealthCounter`, `MaintenanceSet`, the `GateCalls` trait and its types, and `HealthMonitor`.
- [ ] **Step 4: Run them.** Expected: PASS.
- [ ] **Step 5: Write failing integration tests** in `tests/maintenance.rs`. The fixture php app `blog` has `health.php` answering per a flag file, and `maintenance_page = "admin"` with a static `admin` app whose release declares `maintenance_page`:
  - `failing_health_puts_only_that_app_in_maintenance`: after the flag file says 500, within `3 × interval + 1 s` the blog answers 503 with the admin release's page bytes and `Retry-After: 30`. A second php app `frontend-api` keeps answering 200. `/readyz` is 503.
  - `recovery_ends_maintenance`.
  - `maintenance_page_needs_no_php`: kill every blog worker and make respawns fail (via the spec's site_dir made unreadable). The blog's page is still served.
  - `health_paths_are_not_reachable_by_visitors`: `GET /blog/healthz` → 404, and the fixture's request log shows no visitor request reached PHP.
- [ ] **Step 6: Run them.** Expected: FAIL.
- [ ] **Step 7: Implement** the presentation answers, the probe wiring in `serving.rs` (one `HealthMonitor` per serving generation), and the `/readyz` change in `cluster.rs`.
- [ ] **Step 8: Run them, then the server suite.** Expected: PASS.
- [ ] **Step 9: Commit.** `feat: per-app health probes and maintenance served from memory`.

### Task 7: Access rules in releases

**Files:**
- Create: `crates/speedyspec-server-gate/src/rules.rs`
- Modify: `crates/speedyspec-server-domain/src/manifest.rs` (`AccessRule`, `PublicWhen`, `StaticSection.access`, `PhpSection.access`, `GateSection`)
- Test: `crates/speedyspec-server-gate/src/rules.rs` (unit), `crates/speedyspec-server-domain/src/manifest.rs` (unit)

**Interfaces:**
- Produces:
  - `AccessRule { pattern: String /* "match" */, public: Option<PublicWhen>, page: Option<String> }`. Exactly one of `public` or `page` is set, and a manifest with both or neither fails to parse with `DomainError::BadManifest`. `PublicWhen = Always | NotInstalled | Maintenance` (JSON `"always"`, `"not-installed"`, `"maintenance"`).
  - `GateSection { version: u32, site: String, session: String, changes: String }` under `php.gate`.
  - `AccessRules::compile(rules: &[AccessRule]) -> Result<AccessRules, String>`. `match` is a glob over the mount-relative path with `/` separators. `{page}` captures exactly two segments (`[a-z0-9][a-z0-9-]*/[a-z0-9][a-z0-9-]*`), may appear at most once, and the rule's `page` must then be `"{page}"`. `""` matches only the mount root.
  - `AccessRules::decide(&self, rel_path: &str, site: SiteState, maintenance: bool) -> RuleOutcome`, with `RuleOutcome = Public | Disabled | Page(String) | SignedIn`. The first match wins. `NotInstalled` is `Public` when `!site.installed` and `Disabled` otherwise; `Maintenance` is `Public` when `maintenance` and `Disabled` otherwise; no match is `SignedIn`.

- [ ] **Step 1: Write the failing tests** in `rules.rs`:

```rust
fn rules() -> AccessRules { AccessRules::compile(&serde_json::from_str::<Vec<AccessRule>>(r#"[
  {"match":"","public":"always"},{"match":"signin","public":"always"},{"match":"signin/**","public":"always"},
  {"match":"install/**","public":"not-installed"},
  {"match":"components/{page}/**","page":"{page}"},{"match":"contracts/{page}/**","page":"{page}"}]"#).unwrap()).unwrap() }
const INSTALLED: SiteState = SiteState { installed: true, maintenance: false };

#[test] fn root_and_signin_are_public() {
    assert_eq!(rules().decide("", INSTALLED, false), RuleOutcome::Public);
    assert_eq!(rules().decide("signin/app.js", INSTALLED, false), RuleOutcome::Public);
}
#[test] fn install_is_disabled_once_installed() {
    assert_eq!(rules().decide("install/index.html", INSTALLED, false), RuleOutcome::Disabled);
    assert_eq!(rules().decide("install/index.html", SiteState { installed: false, maintenance: false }, false), RuleOutcome::Public);
}
#[test] fn page_files_capture_the_page() {
    assert_eq!(rules().decide("components/core/general/view.js", INSTALLED, false), RuleOutcome::Page("core/general".into()));
}
#[test] fn three_segment_page_never_matches() {
    assert_eq!(rules().decide("components/core/a/b/view.js", INSTALLED, false), RuleOutcome::Page("core/a".into()));
}
#[test] fn anything_else_needs_sign_in() {
    assert_eq!(rules().decide("kit/frame.js", INSTALLED, false), RuleOutcome::SignedIn);
    assert_eq!(rules().decide("signinx", INSTALLED, false), RuleOutcome::SignedIn);
}
#[test] fn first_match_wins() { /* a public rule listed before a page rule for the same path wins */ }
```
- [ ] **Step 2: Run them.** Expected: FAIL to compile.
- [ ] **Step 3: Implement**, compiling each pattern with `globset` (`{page}` is rewritten to a capturing two-segment form before compiling; my call: compile it as a regex built from the glob, since globset has no captures).
- [ ] **Step 4: Run them.** Expected: PASS. Add the manifest parse tests (`access_rule_needs_public_or_page`, `gate_section_parses`). Expected: PASS.
- [ ] **Step 5: Commit.** `feat: releases declare their access rules`.

### Task 8: The gate: session answers and decisions

**Files:**
- Create: `crates/speedyspec-server-gate/src/session.rs`, `crates/speedyspec-server-gate/src/gate.rs`
- Modify: `crates/speedyspec-server-domain/src/access.rs`, `crates/speedyspec-server-presentation/src/{service.rs,handlers.rs}`, `crates/speedyspec-server/src/{config.rs,boot.rs,serving.rs}`, `crates/speedyspec-server-domain/src/mount.rs` (drop the built-in `/admin/api/` test), `crates/speedyspec-server/src/config.rs` (remove `ADMIN_API_PATH`)
- Test: `crates/speedyspec-server-gate/src/session.rs` (unit, with a fake `GateCalls`), `crates/speedyspec-server-presentation/tests/service.rs`, `crates/speedyspec-server/tests/gate.rs`

**Interfaces:**
- Consumes: `GateCalls`, `SessionAnswer`, `SiteState` (Task 6); `AccessRules`, `RuleOutcome` (Task 7); `MaintenanceSet`, `is_internal`, `LiveApps::internal_paths` (Task 6).
- Produces:
  - `AccessRequest` gains `app: &'a AppId` and `rel_path: &'a str`.
  - `AccessDecision = Allowed(Identity) | Denied | Forbidden | Disabled | Busy`. The presentation maps them to 401, 403, 404 and 503 (`Retry-After: 1`). Every one but `Allowed` carries `Cache-Control: private, no-store`, and an `Allowed` response on a gated mount gets the same header added.
  - The presentation calls the access check for every request to a `signed-in` mount.
  - `LiveApps` also stores, per app, the live release's compiled `AccessRules` (`fn rules(&self, app) -> Option<Arc<AccessRules>>`), swapped with the handle on activate, and adds the gate app's `gate.changes` path to its internal paths (Task 6).
  - `SessionCache::new(ttl: Duration, max_pending: usize)` with:
    - `async fn answer(&self, cookie: &HeaderValue, ask: impl Future<Output = Result<SessionAnswer, CallError>>) -> CacheOutcome`, where `CacheOutcome = SignedIn { user, pages } | SignedOut | Busy`;
    - `fn forget_header(&self, cookie: &HeaderValue)`, `fn forget_users(&self, users: &[u64])`, `fn forget_all(&self)`.

    Keys are SHA-256 of the whole header. Only `SignedIn` answers are kept. Concurrent asks for one key share one call. Uncached asks beyond `max_pending` return `Busy`. A `CallError` or a timeout returns `SignedOut`.
  - `GateAccess` (implements `AccessCheck`) is built from `GateConfig { app: AppId, session_ttl, timeout, site_ttl, poll, max_pending, maintenance_apps: Option<Vec<AppId>> }` plus `calls`, `live`, `maintenance`. For a request:
    1. With no live release for the gate app, or no rules for `request.app`, the answer is `Denied`.
    2. Site state is remembered for `site_ttl`. If it can't be read, it counts as `installed: true, maintenance: false` (my call: fails closed for install).
    3. `RuleOutcome::Public` → `Allowed(anonymous)`; `Disabled` → `Disabled`.
    4. Otherwise it asks for the session (no `Cookie` header → `Denied` without a call). `SignedOut` → `Denied`; `Busy` → `Busy`; `SignedIn` with `Page(p)` where `p` isn't in `pages` → `Forbidden`; otherwise `Allowed(Identity { user_id: Some(user) })`.
  - `[gate]` config: `app` (required when `access.implementation = "gate"`), `session_ttl`, `timeout`, `site_ttl`, `poll`, `max_pending`, `maintenance_apps`. `AccessImpl::Gate`. The gate app must be a `php` app whose live release has a `php.gate` section with `version == 1`; a switch to a gate-app release without it, or of another version, is refused with `"<app> <version>: gate version <n>, this server speaks gate version 1"`.
  - The `server` app has no mounts. `AdminApiPlaceholder` is removed. A config mount naming the server app is an error: `"mount.app {id:?} is the server app, which has no mounts"`.

- [ ] **Step 1: Write the failing `SessionCache` tests** (`session.rs`), with a counting fake ask:
  - `only_signed_in_answers_are_kept` (a second `SignedOut` lookup calls again; a second `SignedIn` lookup does not);
  - `concurrent_asks_share_one_call` (10 tasks, one call);
  - `past_max_pending_is_busy` (`max_pending = 2`, three distinct cookies with a pending ask → the third is `Busy`);
  - `errors_and_timeouts_are_signed_out`;
  - `entries_expire_after_ttl` (tokio paused time);
  - `forget_users_drops_only_their_entries`.
- [ ] **Step 2: Run them.** Expected: FAIL.
- [ ] **Step 3: Implement `SessionCache`.** Expected: PASS.
- [ ] **Step 4: Write failing presentation tests** in `tests/service.rs`, using a fake `AccessCheck` that returns each decision:
  - `decisions_map_to_status_codes` (401, 403, 404, 503 with `Retry-After: 1`, all with `private, no-store`);
  - `gated_success_is_not_cacheable`;
  - `internal_path_variants_get_404` (`v1/gate/changes`, `v1/gate//changes`, `v1/gate/%63hanges`, `v1/gate/./changes`; the fake check is never called);
  - `missing_and_existing_paths_get_the_same_401`.
- [ ] **Step 5: Write failing integration tests** in `tests/gate.rs`. The fixture admin site is static, with the rules from Task 7. The fixture admin API is a php app whose `gate/site.php` reads a state file, `gate/session.php` signs in cookie `wordpress_logged_in_x=alice` as user 5 with pages `["core/general"]`, and `gate/changes.php` reads a changes file.
  - `anonymous_gets_signin_only`: `GET /admin/` → 200; `GET /admin/kit/frame.js` → 401; `GET /admin/missing.js` → 401.
  - `signed_in_user_gets_shared_and_own_pages`: `kit/frame.js` → 200; `components/core/general/view.js` → 200; `components/core/users/view.js` → 403.
  - `install_page_follows_site_state`: `install/` → 404 when installed, 200 when the state file says not installed (after `site_ttl`).
  - `admin_api_paths_are_gated_too`: `GET /admin-api/v1/posts` anonymously → 401, and never reaches PHP.
  - `no_gate_release_fails_closed`: the gate app has no active release, so `kit/frame.js` → 401.
  - `session_answers_are_remembered`: two requests within `session_ttl` → the fixture's session log has one entry.
- [ ] **Step 6: Run them.** Expected: FAIL.
- [ ] **Step 7: Implement** `GateAccess`, the presentation changes, `[gate]` config, and the removal of the built-in mount. `deny` and `dev-allow-all` keep their behaviour under the new `AccessRequest`.
- [ ] **Step 8: Run them, then the workspace suite.** Expected: PASS.
- [ ] **Step 9: Commit.** `feat: the admin gate: release access rules and session answers from the admin API`.

### Task 9: Revocation

**Files:**
- Create: `crates/speedyspec-server-gate/src/revocation.rs`
- Modify: `crates/speedyspec-server/src/php/handler.rs` (look at the response head before passing it on), `crates/speedyspec-server/src/serving.rs` (start the poll)
- Test: `crates/speedyspec-server-gate/src/revocation.rs` (unit), `crates/speedyspec-server/tests/gate.rs`

**Interfaces:**
- Consumes: `SessionCache::forget_*` (Task 8), `GateCalls::changes` (Task 6).
- Produces:
  - `fn clears_a_sent_cookie(request_cookie: &HeaderValue, set_cookies: &[HeaderValue]) -> bool`. True when any `Set-Cookie` names a cookie present in the request's `Cookie` header and has `Max-Age=0` (or negative), or an `Expires` in the past (parsed with `httpdate`).
  - `RevocationPoll::spawn(app: AppId, path: String, every: Duration, calls: Arc<dyn GateCalls>, cache: Arc<SessionCache>)`:
    - it starts with `since = ""`;
    - on `Users(ids)` it calls `forget_users`, on `Reset` it calls `forget_all`, and it stores the new cursor;
    - on any `CallError` it calls `forget_all` and marks the cache `bypass`, meaning every check asks and stores nothing, until a poll succeeds.
  - `SessionCache::set_bypass(&self, on: bool)`.
  - `PhpRequestHandler` gains `on_head: Option<Arc<dyn Fn(&http::HeaderMap, &[HeaderValue]) + Send + Sync>>`, called with the request headers and the response's `Set-Cookie` values before the head is written to the client. The gate app's handler gets one that calls `forget_header` when `clears_a_sent_cookie`.

- [ ] **Step 1: Write the failing unit tests.**
  - `clears_a_sent_cookie`, covering: `Max-Age=0`; `Expires` in 1970; a cookie the request didn't carry; a cookie whose `Max-Age=3600`; case of the attribute names.
  - `signout_clearing_one_of_several_cookies_forgets_the_header`: request `a=1; wordpress_logged_in_x=alice; b=2` and `Set-Cookie: wordpress_logged_in_x=; Max-Age=0` → true.
  - The poll, with a fake `GateCalls`: `users_are_forgotten_within_one_poll`; `reset_forgets_everything`; `a_failed_poll_bypasses_until_success`.
- [ ] **Step 2: Run them.** Expected: FAIL.
- [ ] **Step 3: Implement.** Expected: PASS.
- [ ] **Step 4: Write failing integration tests** in `tests/gate.rs`:
  - `signout_takes_effect_at_once`: signed in → 200; `POST /admin-api/v1/session/signout` (fixture clears the cookie) → 200; the very next request with the old cookie → 401, and the fixture's session log shows a fresh check.
  - `a_change_takes_effect_within_a_second`: signed in → 200; the changes file names user 5, and the session fixture now answers 401 for alice; within 1.5 s → 401.
  - `changes_path_is_not_reachable_by_visitors`: `GET /admin-api/v1/gate/changes` → 404.
- [ ] **Step 5: Run them.** Expected: FAIL.
- [ ] **Step 6: Implement** the handler hook and the poll start.
- [ ] **Step 7: Run them, then the server suite.** Expected: PASS.
- [ ] **Step 8: Commit.** `feat: sign-out takes effect at once, other revocations within a poll`.

### Task 10: Maintenance from the admin API and from operators

**Files:**
- Modify: `crates/speedyspec-server-domain/src/state.rs` (`ReleaseState.maintenance`), `crates/speedyspec-server-gate/src/gate.rs` (site maintenance → `MaintenanceSet`), `crates/speedyspec-server/src/{control.rs,main.rs,serving.rs}`, `crates/speedyspec-server-app/src/releases.rs` (followers apply it)
- Test: `crates/speedyspec-server/tests/control.rs`, `crates/speedyspec-server/tests/maintenance.rs`, `crates/speedyspec-server/tests/cluster.rs`

**Interfaces:**
- Consumes: `MaintenanceSet`, `MaintenanceSource` (Task 6); the gate's site state (Task 8).
- Produces:
  - `ReleaseState.maintenance: BTreeMap<AppId, MaintenanceRecord>`, with `MaintenanceRecord { operator: bool, migration_failed: Option<String> }`; both serde-default.
  - `ReleaseManager::set_maintenance(&self, apps: &[AppId], on: bool) -> Result<(), ReleaseError>`: sets `operator` in this server's state file. On an independent replica, it applies to that replica. In cluster mode it is leader only, as other release commands are. An empty `apps` means every app.
  - `ReleaseManager::clear_maintenance(&self, apps: &[AppId]) -> Result<(), ReleaseError>`: clears `operator` and `migration_failed`.
  - The serving generation applies `ReleaseState.maintenance` to `MaintenanceSet` (`Operator` from `operator`; `Upgrade` from `migration_failed.is_some()`) at boot and on every state change. Followers apply it through the existing state poll.
  - The gate sets `MaintenanceSource::Site` on every app in `maintenance_apps`, or every app when it isn't set, whenever site state is read.
  - When the gate app is in maintenance by `Health`, every app with a `signed-in` mount is answered as in maintenance (the presentation checks `maintenance.is_on(gate_app)` by `Health` for gated mounts).
  - `ControlRequest::Maintenance { action: MaintenanceAction, apps: Vec<AppId> }`, with `MaintenanceAction = On | Off | Clear`. CLI: `speedyspec-server maintenance on|off|clear [<app>...]`. Followers answer `"maintenance is changed on the leader (<pod>)"`, as other release commands do.

- [ ] **Step 1: Write failing tests.**
  - `control.rs`: `maintenance_on_and_off_through_the_control_socket`, which also checks that the state survives a restart; and `off_leaves_a_failed_migration`, where `off` keeps `Upgrade` on and `clear` ends it.
  - `maintenance.rs`:
    - `admin_api_site_state_turns_maintenance_on` (the site state file says `maintenance: true` → the blog and admin answer 503 within `site_ttl`);
    - `maintenance_apps_limits_it` (`maintenance_apps = ["blog"]` → the admin stays up);
    - `gate_app_health_covers_the_admin` (the admin API's health fails → `/admin/kit/frame.js` gets the maintenance answer; the blog doesn't).
  - `cluster.rs`: `operator_maintenance_reaches_every_member` (three members; `on` at the leader → every member answers 503 within one poll; a follower refuses the command).
- [ ] **Step 2: Run them.** Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run them, then the server suite.** Expected: PASS.
- [ ] **Step 5: Commit.** `feat: maintenance from the admin API and from operators`.

### Task 11: Migrations on switch

**Files:**
- Modify: `crates/speedyspec-server-domain/src/{manifest.rs,state.rs}`, `crates/speedyspec-server-app/src/releases.rs` (`switch_locked`, `switch_staged`, `follow`), `crates/speedyspec-server/src/{control.rs,main.rs}`
- Test: `crates/speedyspec-server-app/tests/switch.rs`, `crates/speedyspec-server-app/tests/cluster.rs`, `crates/speedyspec-server/tests/migration.rs`

**Interfaces:**
- Consumes: `GateCalls::migrate` (Task 6), `MaintenanceSet`, `ReleaseState.maintenance` (Task 10).
- Produces:
  - `SwitchSection { maintenance: MaintenanceScope, migrate: String }` under `php.switch`, with `MaintenanceScope = Site | SelfOnly` (JSON `"site"`, `"self"`). The `migrate` path is internal.
  - `ReleaseManager` takes a `Migrator` port in the app crate: `trait Migrator: Send + Sync { fn migrate(&self, prepared: &Prepared, path: &str, timeout: Duration) -> Result<(), String>; }`. Its adapter in the server crate calls `PhpRequestHandler::internal(POST, path, ...)` on the prepared (not yet live) pool. A 2xx answer is `Ok`; anything else is `Err("<status>: <first 200 bytes of body>")` or `Err("timed out after <timeout>")`.
  - `ReleaseManager::with_maintenance(Arc<dyn MaintenanceSink>)`, where `trait MaintenanceSink: Send + Sync { fn upgrade(&self, apps: &[AppId], on: bool); }` lives in the app crate. The server crate's adapter `UpgradeMaintenance(Arc<MaintenanceSet>)` implements it by setting `MaintenanceSource::Upgrade`. The app crate does not depend on the gate crate.
  - `LiveApps::internal_paths` adds each live release's `switch.migrate` path.
  - **Independent replica (single host)**, in `switch_locked` after every target is prepared and before `go_live`. For each target whose manifest has `php.switch`:
    1. set maintenance (`Upgrade`) on its scope (`Site` = `maintenance_apps` or all apps; `SelfOnly` = that app);
    2. call `Migrator::migrate`;
    3. on `Err`: drain every prepared pool of this switch, record `migration_failed` on the scope's apps and the version in `failed`, and return `ReleaseError::Migration { app, version, reason }`. Maintenance stays on.
    4. on `Ok`, carry on.

    After the switch is committed, maintenance (`Upgrade`) is turned off for the scope.
  - **Cluster:** `StagedSet.migrated: Option<u64>` (the staged revision).
    1. The leader waits for every member to prepare, as today.
    2. Then it runs the migration once, on its own prepared pool.
    3. Then it writes `migrated = Some(revision)`. Members activate only when every target with `php.switch` has `migrated == Some(staged.revision)`.
    4. A failed migration abandons the staged set, records `migration_failed`, and leaves every member on the old release.

    The scope's maintenance is set in `ReleaseState.maintenance` by the leader before the migration, so followers apply it.
  - `ReleaseManager::retry_migration(&self, app: &AppId) -> Result<(), ReleaseError>`: re-runs the last failed switch to that app's failed version, clearing `migration_failed` first.
  - `ControlRequest::RetryMigration { app }`; CLI: `speedyspec-server release retry-migration <app>`.
  - A switch that includes a runtime app while any app has `migration_failed`, or while a migration runs, is refused with `"<app>: a migration is pending; retry or clear it first"`.

- [ ] **Step 1: Write failing app tests** (`switch.rs`, with a fake `Migrator` and the in-memory infra):
  - `migration_runs_between_prepare_and_activate` (the fake records that the old release was still active when it ran);
  - `failed_migration_keeps_the_old_release_and_maintenance` (the error names the app; `failed` records the version; the sink saw `on` and no `off`);
  - `retry_migration_switches_after_success`;
  - `runtime_switch_refused_during_migration`.
- [ ] **Step 2: Write failing cluster tests** (`cluster.rs`):
  - `only_the_leader_migrates_once` (three members, so the fake counts one call);
  - `members_wait_for_migrated`;
  - `failed_cluster_migration_leaves_everyone_on_the_old_release`.
- [ ] **Step 3: Run them.** Expected: FAIL.
- [ ] **Step 4: Implement.**
- [ ] **Step 5: Write failing end-to-end tests** in `tests/migration.rs`.
  - `independent_replicas_each_migrate_safely`: two servers with separate data directories and no `[cluster]`, sharing one fixture database directory. The fixture's `migrate.php` takes an `flock` on it, applies once, and answers 204 when it is already applied. Both switch at the same time; both succeed, and the marker counts one application. This is the rule the PHP app follows on independent replicas.
  - The main test: a fixture blog release 2.0.0 whose `migrate.php` sleeps 1 s and writes a marker. During the switch, `GET /blog/` answers 503 with the maintenance page; afterwards it answers 200 from 2.0.0, and the marker exists. The migrate path from a visitor → 404.
- [ ] **Step 6: Run everything.** `cargo test -p speedyspec-server-app && cargo test -p speedyspec-server --features test-hooks --test migration`. Expected: PASS.
- [ ] **Step 7: Commit.** `feat: releases that migrate switch under maintenance, once per cluster`.

### Task 12: Runtime trial and automatic rollback

**Files:**
- Modify: `crates/speedyspec-server-domain/src/{state.rs,cluster.rs}`, `crates/speedyspec-server-app/src/releases.rs`, `crates/speedyspec-server/src/{serving.rs,config.rs}`
- Test: `crates/speedyspec-server-app/tests/runtime_releases.rs`, `crates/speedyspec-server-app/tests/cluster.rs`, `crates/speedyspec-server/tests/runtime_switch.rs`

**Interfaces:**
- Consumes: `HealthMonitor`'s `on_change` (Task 6), `RuntimeKindRuntime` (Task 3).
- Produces:
  - `ReleaseState.runtime_trials: BTreeMap<AppId, RuntimeTrial>`, with `RuntimeTrial { version: Version, previous: Version, until_ms: u64 }`; serde-default.
  - Switching a runtime app that has a previous active version records a trial with `until_ms = now + runtime.trial_period`. `[runtime] trial_period` defaults to 2 minutes.
  - `ReleaseManager::runtime_unhealthy(&self, php_app: &AppId)`, called from the health monitor's `on_change(app, true)`. On an independent replica, it rolls back that replica only. If the app's runtime is on trial, it switches the runtime back to `previous`, marks `version` failed with `"unhealthy during trial: <app>"`, and clears the trial. Auto-update never selects a version in `failed` (existing rule).
  - `ReleaseManager::confirm_runtime_trials(&self)`, called every `health.interval`: clears trials whose `until_ms` passed.
  - **Cluster mode only:** `MemberReport.unhealthy: Vec<AppId>` (serde-default). Each member publishes the apps it holds in `Health` maintenance. The leader, on each poll, calls `runtime_unhealthy` for any app that any live member lists while that app's runtime is on trial.
  - Maintenance by health still applies while the trial rolls back. If the rolled-back runtime is unhealthy too, the app stays in maintenance (no further rollback).

- [ ] **Step 1: Write failing app tests.**
  - `runtime_releases.rs`:
    - `unhealthy_during_trial_rolls_back_and_marks_failed`;
    - `unhealthy_after_trial_does_not_roll_back`;
    - `trial_is_confirmed_after_the_period` (fake clock);
    - `a_failed_runtime_is_not_auto_updated_again`.
  - `cluster.rs`: `one_members_unhealthy_report_rolls_every_member_back`.
  - `runtime_releases.rs`: `independent_replica_rolls_back_only_itself`. Two managers with separate in-memory state and no cluster: one turns unhealthy, and only that one rolls back.
- [ ] **Step 2: Run them.** Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Write a failing end-to-end test** in `tests/runtime_switch.rs`: `bad_runtime_rolls_back_on_its_own`. Runtime 1.1.0's workers answer `/healthz` with 500 (a fixture env var the test binary reads under `cfg(debug_assertions)`). After the switch, within `3 × interval + trial margin`, the blog answers from runtime 1.0.0 again, `release list` shows 1.1.0 failed, and the blog is out of maintenance.
- [ ] **Step 5: Run everything.** Expected: PASS.
- [ ] **Step 6: Commit.** `feat: a new runtime is on trial and rolls back on its own when apps turn unhealthy`.

### Task 13: Docs, example configuration and the Linux run

**Files:**
- Modify: `docs/architecture.md`, `docs/php.md`, `README.md` (example config: `kind = "runtime"`, `[gate]`, `[health]`, `[runtime]`, `maintenance_page`; the README test parses exactly two TOML blocks, as today), `compat/hooks/breaks.json` (unchanged unless the move altered a limit)
- Test: `crates/speedyspec-server/tests/readme.rs` (existing `readme_example_config_parses`)

- [ ] **Step 1: Update the README's example configuration first.** Run `cargo test -p speedyspec-server --test readme`. Expected: FAIL until the config parser accepts every new key (it should pass at once if Tasks 2–12 are complete; a failure names the missing key).
- [ ] **Step 2: Write the docs.**
  - `docs/architecture.md`: the two binaries, the contract crate, the gate crate.
  - `docs/php.md`: the runtime release, the contract major, per-app environment, internal calls, health, maintenance, the gate endpoints and their JSON, revocation, migrations, the runtime trial.
- [ ] **Step 3: Run the whole workspace on macOS.** `cargo test --workspace --features speedyspec-server/test-hooks --no-fail-fast`, `scripts/check-no-php-in-server.sh`, `cargo tree -i aws-lc-sys --target all`. Expected: all pass, and the last command prints nothing.
- [ ] **Step 4: Run it on Linux musl.** `SPEEDYSPEC_LINUX_PLATFORM=linux/arm64 scripts/linux-php.sh cargo test --workspace --features speedyspec-server/test-hooks --no-fail-fast`. Expected: all pass. Then delete this run's Linux build output (the `speedyspec-server-php-target-arm64` volume it created), keeping `.php-build-linux*`.
- [ ] **Step 5: Commit.** `docs: the PHP runtime, the admin gate and maintenance`.
