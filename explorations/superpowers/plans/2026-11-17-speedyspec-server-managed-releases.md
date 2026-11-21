# SpeedySpec Server Managed Releases Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** In managed mode on one host, `speedyspec-server` downloads, verifies, keeps, switches, rolls back and
prunes releases of its `static` and `php` apps on its own, with each switch atomic and every refusal explained. The
`release` and `status` commands reach the running server over a control socket.

**Architecture:** `-domain` gains the release index, the release state and the contracts `ReleaseSource`,
`ReleaseStore`, `StateStore`, `SignatureVerifier` and `AppRuntime`. Implementations live in new crates:
- `-infra-fs`: `state.json` and the release directories;
- `-infra-http`: the release index and resumable downloads;
- `-infra-minisign`: signature checks;
- `-infra-memory`: test doubles.

`-app` gets the `ReleaseManager`, which runs every lifecycle operation, one at a time. The presentation layer's app
handles become swappable (`ArcSwap`), so a static switch replaces a whole in-memory set in one store. The binary
wires the kind runtimes (static, and a php placeholder), the auto-update loop and the control socket.

**Tech Stack:** Plan 1's stack, plus arc-swap, ureq 3 (rustls on ring), sha2, minisign-verify, tar, ruzstd (pure-Rust
zstd: decoding in production, its encoder only in tests), humantime.

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-server-core-design.md`

**This is plan 2 of 4.** Plan 1 (foundation and immutable mode) is done and pushed to `SpeedySpec/speedyspec-server`.
What this plan leaves to later plans:
- **Plan 3 (keeper and socket handoff):**
  - server-kind releases (`install` / `switch` of the `server` app);
  - group switches that include the server;
  - trial and fall-back;
  - `reload`;
  - `keeper.sock`;
  - certificate reload on `SIGHUP`;
  - `Alt-Svc`.
- **Plan 4 (cluster):** leadership, member files, the two-step switch.

## Global Constraints

- Work in `packages/speedyspec-server/` (its own repo, remote `SpeedySpec/speedyspec-server`). Do not push without
  Jacob's go-ahead.
- Rust 1.99, edition 2024. Default rustfmt (K&R braces), matching `speedyspec-cli`; run `cargo fmt --all` before every
  commit.
- Salvo appears only in `-presentation` and the binary. `-app` depends only on `-domain` (plus `semver`, `sha2`,
  `tracing`); its tests may use `-infra-fs` and `-infra-memory` as dev-dependencies.
- Contracts in `-domain`, implementations in separate crates, each replaceable on its own.
- TLS stays on `ring`: `cargo tree -i aws-lc-sys` must print nothing after every task that adds a dependency.
- The core knows app *kinds*, never particular apps.
- Every refusal names what was refused and why. Exit codes: 0 success, 1 failure, 2 usage error. Logs are JSON lines
  on stderr. Command output goes to stdout.
- Immutable mode still never writes to disk. Its one exception is the control socket in `run_dir`: when `run_dir` is
  not writable, the server logs a warning and runs without it (Task 10).
- Conventional Commits, each ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- CI matrix stays Linux musl and macOS. Run Linux locally with
  `docker run --rm -v "$PWD":/w -v speedyspec-server-cargo:/usr/local/cargo/registry -v speedyspec-server-target:/w/target-linux -e CARGO_TARGET_DIR=/w/target-linux -w /w rust:1.99-alpine sh -c 'apk add -q build-base && cargo test --workspace'`.

Values the spec pins, used across tasks:

| Value | Where it is used |
|---|---|
| `keep` default 3 | Per app |
| `update` default `manual` for the server, `auto` for other apps | Per app |
| `check_interval` default 15 minutes | Per app |
| `release.static.fallback_grace` default 1 hour | Static apps |
| `limits.static_memory` (plan 1) | Static apps |
| `allow_unsigned = true` | Logs a warning on every start |
| An app with no `keys` and without `allow_unsigned` | Refuses to install anything |
| Pre-release versions | Ignored unless `prerelease = true` |
| 503 with `Retry-After: 30` | App with nothing installed |
| `admin 6.0.0 requires server ^4, active server is 3.2.1` | Compatibility failure message |

## Review Focus

Each line has a test in the task named in brackets.

1. **Hostile archives.** Entries with `../`, absolute paths, a symlink pointing outside the release, a hard link, a
   device file or FIFO, or more than 4 GiB once unpacked. Each is refused, nothing is written outside the release's
   `.partial` directory, and that `.partial` directory is removed. [Task 3]
2. **Downloads that go wrong.**
   - An interrupted download resumes from where it stopped.
   - A server that ignores `Range` (answers 200) restarts the file from zero instead of appending.
   - A truncated body or a checksum mismatch keeps nothing and removes the partial download.
   - An index that lists the same version twice is refused. [Task 1, Task 4, Task 5]
3. **A crash in the middle.** A leftover `state.json.tmp`, a leftover `<version>.partial/`, or a `state.json` that
   names a release directory that is gone. Start-up ignores the temp file, deletes partials, falls back to the newest
   kept release that loads, and says so in the log. [Task 3, Task 7]
4. **Two operations at once.**
   - Two `release switch` commands sent together.
   - An auto-update firing during a manual switch.

   They run one after the other, never interleaved, and both finish with a consistent `state.json`. [Task 6, Task 10]
5. **`keep` and version edge cases.**
   - `keep = 1`: rollback says there is no previous release kept.
   - `keep = 0` is a config error.
   - A pinned older version is never pruned.
   - `1.0.0-rc.1` sorts before `1.0.0`, and with `prerelease = false` it is never a candidate.
   - Build metadata (`1.0.0+abc`) is refused in an index, because SemVer gives it no order. [Task 5, Task 6, Task 9]

## File Structure

```
crates/speedyspec-server-domain/src/   release.rs (index), state.rs (release state), contracts.rs (new traits + errors)
crates/speedyspec-server-infra-fs/     src/lib.rs state.rs store.rs unpack.rs        tests/{state.rs,store.rs}
crates/speedyspec-server-infra-http/   src/lib.rs                                    tests/http.rs
crates/speedyspec-server-infra-minisign/ src/lib.rs                                  tests/fixtures/{key.pub,a.bin,a.bin.minisig}
crates/speedyspec-server-infra-memory/ src/lib.rs source.rs state.rs verifier.rs runtime.rs archive.rs
crates/speedyspec-server-app/src/      releases.rs (ReleaseManager), download.rs     tests/{install.rs,switch.rs,boot.rs}
crates/speedyspec-server-static/src/   build.rs (+ immutable_only)
crates/speedyspec-server-presentation/src/ live.rs (LiveApps), handlers.rs (StaticApp + fallback)
crates/speedyspec-server/src/          config.rs (new keys), runtime.rs (kind runtimes), managed.rs (managed boot,
                                       auto-update), control.rs (socket server + client), main.rs (commands)
crates/speedyspec-server/tests/        managed.rs, control.rs, switch_under_load.rs
```

---

### Task 1: Domain: release index, release state and the lifecycle contracts

**Files:**
- Create: `crates/speedyspec-server-domain/src/release.rs`, `src/state.rs`, `src/contracts.rs`; modify `src/lib.rs`
- Test: unit tests in each new module

**Interfaces:**
- Produces, `release.rs`:
  - `pub struct ReleaseIndex { pub app: AppId, pub releases: Vec<IndexEntry> }`
  - `pub struct IndexEntry { pub version: Version, pub published: String, pub artifacts: Vec<Artifact> }`
  - `pub struct Artifact { pub target: String, pub url: String, pub sha256: String, pub minisig: Option<String> }`
  - `ReleaseIndex::parse(bytes: &[u8]) -> Result<ReleaseIndex, DomainError>`
  - `IndexEntry::artifact_for(&self, target: &str) -> Option<&Artifact>`
  - `ReleaseIndex::newest(&self, after: Option<&Version>, prerelease: bool) -> Option<&IndexEntry>`: the highest
    version strictly greater than `after`, skipping pre-releases unless `prerelease` is set.
- Produces, `state.rs`:
  - `pub struct ReleaseState { pub revision: u64, pub apps: BTreeMap<AppId, AppState> }`
  - `pub struct AppState { pub active: Option<Version>, pub previous: Option<Version>, pub kept: Vec<Version>, pub failed: BTreeMap<Version, String>, pub pinned: Option<Version> }`
    - `kept` is in install order, oldest first.
  - `ReleaseState::app_mut(&mut self, app: &AppId) -> &mut AppState` (inserts the default).
  - Both are serde, with `#[serde(default)]` on every field, so plan 3 and plan 4 can add fields without breaking
    existing state files.
- Produces, `contracts.rs` (all `Send + Sync`, sync methods; lifecycle work runs off the async runtime):
  - `pub trait ReleaseSource { fn index(&self, url: &str) -> Result<ReleaseIndex, SourceError>; fn download(&self, url: &str, dest: &Path) -> Result<(), SourceError>; }`
    - `download` resumes from `dest`'s current length, and restarts from zero when the server does not honour
      `Range`.
  - `pub trait SignatureVerifier { fn verify(&self, app: &AppId, artifact: &Path, signature: &Path) -> Result<(), VerifyError>; }`
  - `pub trait ReleaseStore`:
    - `fn partial_dir(&self, app: &AppId, version: &Version) -> PathBuf`
    - `fn unpack(&self, app: &AppId, version: &Version, archive: &Path) -> Result<ReleaseManifest, StoreError>`
    - `fn commit(&self, app: &AppId, version: &Version) -> Result<PathBuf, StoreError>`
    - `fn release_dir(&self, app: &AppId, version: &Version) -> PathBuf`
    - `fn manifest(&self, app: &AppId, version: &Version) -> Result<ReleaseManifest, StoreError>`
    - `fn discard_partial(&self, app: &AppId, version: &Version)`
    - `fn remove(&self, app: &AppId, version: &Version) -> Result<(), StoreError>`
    - `fn clean_partials(&self) -> Result<Vec<PathBuf>, StoreError>`
  - `pub trait StateStore { fn load(&self) -> Result<ReleaseState, StateError>; fn save(&self, state: &ReleaseState) -> Result<(), StateError>; }`
  - `pub struct Prepared { pub app: AppId, pub version: Version, pub payload: Box<dyn Any + Send> }`
  - `pub trait AppRuntime`:
    - `fn prepare(&self, app: &AppId, manifest: &ReleaseManifest, dir: &Path) -> Result<Prepared, String>`
    - `fn activate(&self, prepared: Prepared)`
    - `fn retire(&self, app: &AppId, version: &Version) {}` (default no-op)
    - `activate` is infallible and is the switch itself.
  - Error types (thiserror), each message naming what failed: `SourceError { url: String, message: String }`,
    `VerifyError(String)`, `StoreError { path: PathBuf, message: String }`, `StateError { path: PathBuf, message: String }`.

- [ ] **Step 1: Write the failing tests.**
  - `index_parses_spec_example` (the JSON from the spec's "Release index" section): `releases[0].version == 5.2.0`,
    `artifact_for("any").unwrap().minisig` is `Some`, and `artifact_for("x86_64-unknown-linux-musl")` is `None`.
  - `index_rejects`: a duplicate version, build metadata `1.0.0+abc`, an `app` that is not a valid id, and an
    artifact `sha256` that is not 64 lowercase hex characters.
  - `newest_respects_prerelease`: versions `1.0.0`, `1.1.0-rc.1`, `1.0.1`. `newest(Some(1.0.0), false)` is `1.0.1`;
    `newest(Some(1.0.1), true)` is `1.1.0-rc.1`; `newest(Some(1.0.1), false)` is `None`.
  - `state_round_trips_and_tolerates_unknown_and_missing_fields`: `{}` parses to the default; a field this version
    does not know is ignored.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-domain` → FAIL (types missing).
- [ ] **Step 3: Implement** the types and traits above. `ReleaseState` does *not* use `deny_unknown_fields`;
  `ReleaseIndex` does not either, because indexes are written by other tools.
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-domain` → PASS.
- [ ] **Step 5: Commit.** `feat(domain): release index, release state and lifecycle contracts`

---

### Task 2: `-infra-memory`: test doubles and an archive builder

**Files:**
- Create: `crates/speedyspec-server-infra-memory/` (deps: domain, tar, ruzstd, serde_json), `src/lib.rs`,
  `source.rs`, `state.rs`, `verifier.rs`, `runtime.rs`, `archive.rs`
- Test: unit tests in each module

**Interfaces:**
- Produces:
  - `archive::Entry`, an enum: `File(&str, &[u8])`, `Dir(&str)`, `Symlink(&str, &str)`, `HardLink(&str, &str)`,
    `Fifo(&str)`, `RawPath(&str, &[u8])`. `RawPath` writes the header name unchecked, for hostile paths.
  - `archive::build(entries: &[Entry]) -> Vec<u8>`: tar, then zstd via `ruzstd`'s encoder.
  - `archive::release(manifest_json: &str, files: &[(&str, &str)]) -> Vec<u8>`: a convenience wrapper.
  - `MemorySource`:
    - `new()`;
    - `add_index(url: &str, json: &str)`;
    - `add_file(url: &str, bytes: Vec<u8>)`;
    - `fail_next_download(url: &str, after_bytes: usize)` (writes that many bytes, then errors);
    - `downloads(&self) -> Vec<String>` (the URLs requested).
  - `MemoryStateStore`:
    - `new()`;
    - `fail_next_save()`;
    - `saves(&self) -> usize`.
  - `MemoryVerifier`: accepts a signature file whose content is exactly `good`, and rejects anything else.
  - `RecordingRuntime`:
    - `new()`;
    - `fail_prepare(app: &str, version: &str, reason: &str)`;
    - `events(&self) -> Vec<String>`, as `"prepare admin 1.0.0"`, `"activate admin 1.0.0"` and
      `"retire admin 1.0.0"`;
    - `active(&self, app: &str) -> Option<Version>`.

- [ ] **Step 1: Write the failing tests:**
  - `archive_round_trips` (decode it with `ruzstd` plus `tar` and list the entries);
  - `memory_source_resumes_like_http` (a pre-filled dest is appended to);
  - `recording_runtime_records_order`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-infra-memory` → FAIL.
- [ ] **Step 3: Implement** each double with `Mutex` interior state.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit.** `test(infra-memory): release doubles and a tar.zst archive builder`

---

### Task 3: `-infra-fs`: the state file and the release store

**Files:**
- Create: `crates/speedyspec-server-infra-fs/` (`Cargo.toml` deps: domain, tar, ruzstd, serde_json, thiserror,
  tracing; dev: tempfile, `-infra-memory` for `archive::build`), `src/lib.rs`, `src/state.rs`, `src/store.rs`,
  `src/unpack.rs`
- Test: `tests/state.rs`, `tests/store.rs`

**Interfaces:**
- Consumes: `StateStore`, `ReleaseStore`, `ReleaseState`, `ReleaseManifest`, `MANIFEST_FILE` (Task 1, plan 1).
- Consumes (tests only): `speedyspec_server_infra_memory::archive::build(entries: &[Entry]) -> Vec<u8>` (Task 2).
- Produces:
  - `FsStateStore::new(data_dir: &Path, run_dir: &Path) -> FsStateStore`
    - It writes `<data_dir>/state.json` and takes an exclusive `std::fs::File::lock` on `<run_dir>/state.lock`
      around every save.
  - `FsReleaseStore::new(data_dir: &Path) -> FsReleaseStore`
  - `pub const MAX_UNPACKED_BYTES: u64 = 4 * 1024 * 1024 * 1024;`

On-disk layout (spec "On disk"), with my calls inside `.partial`:

```
<data_dir>/releases/<app>/<version>/                  committed release
<data_dir>/releases/<app>/<version>.partial/artifact  the downloaded archive (download resumes into it)
<data_dir>/releases/<app>/<version>.partial/artifact.minisig
<data_dir>/releases/<app>/<version>.partial/unpacked/ unpack target; `commit` renames it to <version>/ and deletes .partial
```

- `save` writes `state.json.tmp`, fsyncs it, renames it over `state.json`, and fsyncs the directory.
- `load` of a missing `state.json` returns the default state.
- `load` never reads `state.json.tmp`.
- `unpack` checks that the manifest's `app` equals the directory's app, and that the manifest's version equals
  `version`.
- `clean_partials` deletes every `*.partial` directory and returns their paths.
- `remove` deletes `<version>/`.

Unpacking refuses, and then deletes the whole `.partial`:
- absolute paths;
- `..` components;
- symlinks whose target is absolute or resolves outside `unpacked/`;
- hard links;
- device files and FIFOs;
- more than `MAX_UNPACKED_BYTES` of file content (my call, against decompression bombs).

- [ ] **Step 1: Write the failing tests.**
  - `tests/state.rs`:
    - `missing_state_is_default`.
    - `save_then_load`.
    - `leftover_tmp_is_ignored`: write garbage to `state.json.tmp`, then `load` returns the saved state.
    - `save_is_atomic_on_replace`: after two saves, `state.json` parses, and no `.tmp` is left.
    - `save_takes_the_lock`: hold `File::lock` on `state.lock` from the test on another thread for 300 ms; `save`
      returns only after that, at ≥ 250 ms.
  - `tests/store.rs`, Review Focus 1 and 3:
    - `unpack_and_commit`: an archive with `speedyspec-release.json` and `dist/index.html` lands at
      `releases/admin/1.0.0/dist/index.html`, and `.partial` is gone.
    - `manifest_mismatch_refused` (wrong `app`, or wrong `version`).
    - One `rejects_*` test per hostile entry above (`rejects_parent_dir`, `rejects_absolute`,
      `rejects_escaping_symlink`, `rejects_hard_link`, `rejects_fifo`, `rejects_oversized`). Each asserts the error
      names the entry, nothing exists outside `.partial` (check the parent of `data_dir`), and `.partial` is removed.
    - `allows_internal_symlink` (`dist/latest -> v1`).
    - `clean_partials_removes_leftovers`.
    - `remove_deletes_release`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-infra-fs` → FAIL.
- [ ] **Step 3: Implement** `FsStateStore` and `FsReleaseStore`. Unpack with `tar::Archive` over
  `ruzstd::decoding::StreamingDecoder`. Check every entry's header before writing it, and count bytes as they are
  written. `rejects_oversized` uses a store built with `FsReleaseStore::with_limit(dir, 1024)`, a `#[doc(hidden)]`
  constructor for tests.
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-infra-fs` → PASS.
- [ ] **Step 5: Commit.** `feat(infra-fs): atomic state file and a release store that refuses hostile archives`

---

### Task 4: `-infra-http` and `-infra-minisign`

**Files:**
- Create: `crates/speedyspec-server-infra-http/` (deps: domain, ureq 3 with rustls, thiserror, url), `src/lib.rs`,
  `tests/http.rs`
- Create: `crates/speedyspec-server-infra-minisign/` (deps: domain, minisign-verify), `src/lib.rs`,
  `tests/fixtures/{key.pub,a.bin,a.bin.minisig,other.pub}`
- Test: the two test files above, plus unit tests in the minisign `lib.rs`

**Interfaces:**
- Consumes: `ReleaseSource`, `SignatureVerifier`, `ReleaseIndex::parse` (Task 1).
- Produces:
  - `HttpSource::new() -> HttpSource`. URLs must be `https://`, or `http://` with a loopback host (my call, so tests
    and local mirrors work); any other URL is a `SourceError` naming it.
  - `MinisignVerifier::new(keys: BTreeMap<AppId, Vec<String>>) -> Result<MinisignVerifier, VerifyError>`. Keys are the
    base64 public-key strings from config. A bad key is an error at construction, naming the app.

`download`:
- If `dest` has `n > 0` bytes, it sends `Range: bytes=n-`.
- `206` → append.
- `200` → truncate and write from zero.
- `416` with `n > 0` → treat the file as complete.
- Any other status → `SourceError` with the status.
- A body that ends early → `SourceError`; what was written stays, so the next call resumes.
- Timeouts (my call): connect 10 s, and no progress for 60 s.

- [ ] **Step 1: Write the failing tests.** `tests/http.rs` runs a tiny threaded `std::net::TcpListener` HTTP/1.1
  server on `127.0.0.1:0`, scripted per test.
  - `index_fetched_and_parsed`.
  - `rejects_plain_http_off_loopback` (`http://example.com/x`; no request is made).
  - Review Focus 2:
    - `resumes_with_range`: a pre-filled 10-byte `dest`; the server checks `Range: bytes=10-` and answers 206; the
      final file equals the full body.
    - `restarts_when_range_ignored`: the server answers 200 with the full body; the file equals the body, not
      10 bytes plus the body.
    - `truncated_body_is_an_error`: `Content-Length: 100`, then the server closes after 40 bytes; the result is
      `Err`, and `dest` holds 40 bytes.
    - `status_error_names_url`: 404 → the error contains the URL and `404`.
  - minisign:
    - `verifies_good_signature`.
    - `rejects_wrong_key` (`other.pub`).
    - `rejects_tampered_file` (one byte flipped).
    - `rejects_unknown_app`.
    - `bad_key_rejected_at_construction`.
  - Generate the fixtures once with the `minisign` CLI or the `minisign` crate in a throwaway script, commit them,
    and note in `tests/fixtures/README.md` how they were made. The test key's secret is not committed.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-infra-http -p speedyspec-server-infra-minisign` → FAIL.
- [ ] **Step 3: Implement** both.
- [ ] **Step 4: Run** the same command → PASS; `cargo tree -i aws-lc-sys` → nothing.
- [ ] **Step 5: Commit.** `feat(infra): HTTPS release source with resumable downloads, and minisign verification`

---

### Task 5: `ReleaseManager`: check and install

**Files:**
- Create: `crates/speedyspec-server-app/src/releases.rs`, `src/download.rs`; modify `src/lib.rs` and `Cargo.toml`
  (add `sha2`; dev: `-infra-fs`, `-infra-memory`, `tempfile`)
- Test: `crates/speedyspec-server-app/tests/install.rs`

**Interfaces:**
- Consumes: every contract from Task 1, plus `check_set` and `Incompatibility` (plan 1).
- Produces:
  - `pub struct AppPolicy { pub kind: AppKind, pub source: Option<String>, pub keep: u32, pub prerelease: bool, pub signed: bool }`
    - `signed` is false only with `allow_unsigned` and no keys.
  - `pub struct ReleaseManager`, built by `ReleaseManager::new(apps: BTreeMap<AppId, AppPolicy>, server: ReleaseManifest, source: Arc<dyn ReleaseSource>, verifier: Arc<dyn SignatureVerifier>, store: Arc<dyn ReleaseStore>, state: Arc<dyn StateStore>, runtimes: HashMap<AppKind, Arc<dyn AppRuntime>>) -> Result<ReleaseManager, ReleaseError>`.
    - It loads the state once and keeps it, behind one `Mutex` that also serializes operations.
  - `ReleaseManager::check(&self, app: &AppId) -> Result<Option<Version>, ReleaseError>`: `None` when pinned, up to
    date, or the only newer versions are failed ones.
  - `ReleaseManager::install(&self, app: &AppId, version: Option<Version>) -> Result<Version, ReleaseError>`: download,
    verify, unpack, commit, then record the version in `kept`. It does not switch. With `None` it installs
    `check`'s candidate.
  - `pub enum ReleaseError` (thiserror), variants:
    - `UnknownApp(AppId)`;
    - `NoSource(AppId)`;
    - `ServerNeedsKeeper` (message: `server releases need the keeper, which is not in this build yet`);
    - `NothingNewer(AppId)`;
    - `NotInIndex(AppId, Version)`;
    - `NoArtifact(AppId, Version)`;
    - `Checksum { app, version, expected, actual }`;
    - `Signature(String)`;
    - `Unsigned(AppId)` (message names `keys` and `allow_unsigned`);
    - `Source(SourceError)`;
    - `Store(StoreError)`;
    - `State(StateError)`;
    - `Incompatible(Vec<Incompatibility>)`;
    - `NotKept(AppId, Version)`;
    - `NoPrevious(AppId)`;
    - `Prepare { app, version, reason }`;
    - `KeepTooSmall(AppId)`.

Install, step by step (spec lifecycle steps 1–6, my order: the compatibility check moves to `switch`, so a kept
release can wait until its requirements are met):
1. Refuse `server`-kind apps with `ServerNeedsKeeper`.
2. Fetch the index from `policy.source`, check `index.app == app`, and pick the entry: the given version, or the
   candidate.
3. Take the `any` artifact.
4. `source.download(artifact.url, partial/artifact)`.
5. Compare the SHA-256 against the index. On a mismatch, `store.discard_partial`, then `Checksum`.
6. When `signed`, download `minisig` to `partial/artifact.minisig` and verify; on failure, discard the partial. When
   not signed, skip both and log a warning.
7. `store.unpack`; its manifest `kind` must match the policy.
8. `store.commit`.
9. Push the version onto `kept`, remove it from `failed`, bump `revision`, and save.
10. Log `{"event":"installed","app":…,"version":…}`.

- [ ] **Step 1: Write the failing tests** in `tests/install.rs`. Use `FsReleaseStore` and `MemoryStateStore` on a temp
  dir, `MemorySource`, `MemoryVerifier`, and `archive::release` for the artifacts, with `sha256` computed in the test.
  - `installs_newest_and_keeps_it`: the state's `kept == [1.1.0]`, `active` is unchanged (`None`), and the release
    dir exists.
  - `install_named_version`.
  - `nothing_newer` when active is the newest.
  - `check_skips_pinned_and_failed`.
  - Review Focus 2:
    - `checksum_mismatch_keeps_nothing`: `Checksum` names expected and actual, the `.partial` is gone, and the state
      is unchanged.
    - `interrupted_download_resumes`: `fail_next_download(url, 100)`; the first install errors; the second succeeds
      and requested the URL twice.
  - `bad_signature_refused` (`.minisig` content `bad`).
  - `unsigned_refused_without_allow_unsigned`.
  - `unsigned_allowed_skips_signature` (no `.minisig` download in `downloads()`).
  - `index_for_other_app_refused`.
  - `kind_mismatch_refused`.
  - `server_kind_refused` (the error message contains `keeper`).
  - Review Focus 5, `prerelease_never_candidate_unless_enabled`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-app --test install` → FAIL.
- [ ] **Step 3: Implement** `check`, `install` and `download::sha256_file(path) -> io::Result<String>`. Use
  lowercase hex, and stream the file in 64 KiB chunks.
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-app` → PASS (plan 1's compat tests too).
- [ ] **Step 5: Commit.** `feat(app): check and install releases: download, verify, unpack, keep`

---

### Task 6: `ReleaseManager`: switch, group switch, rollback, pin, prune

**Files:**
- Modify: `crates/speedyspec-server-app/src/releases.rs`
- Test: `crates/speedyspec-server-app/tests/switch.rs`

**Interfaces:**
- Consumes: Task 5.
- Produces:
  - `ReleaseManager::switch(&self, targets: &[(AppId, Version)]) -> Result<(), ReleaseError>`
  - `ReleaseManager::rollback(&self, app: &AppId, version: Option<Version>) -> Result<Version, ReleaseError>`
    (with `None`, the previous version)
  - `ReleaseManager::pin(&self, app: &AppId, version: Version) -> Result<(), ReleaseError>` (the version must be kept)
  - `ReleaseManager::unpin(&self, app: &AppId) -> Result<(), ReleaseError>`
  - `ReleaseManager::status(&self) -> ReleaseState` (a clone)
  - `ReleaseManager::active_manifests(&self) -> BTreeMap<AppId, ReleaseManifest>` (the server's included)

Switch (spec lifecycle steps 5 and 7–10, and "Switching several apps together"):
1. Every target must be kept (`NotKept`) and not a `server` kind (`ServerNeedsKeeper`).
2. Run `check_set` over the active manifests with every target substituted (`Incompatible`).
3. `prepare` every target through its kind's runtime. If any fails, record each failed version in `failed` with the
   reason, save, and return `Prepare` for the first failure; nothing is activated.
4. `activate` all of them in target order.
5. For each app: `previous = old active`, `active = target`, bump `revision`, save once.
6. `retire` each old active.
7. Prune each target app: remove kept versions beyond `keep`, oldest first, never the active, pinned or previous
   version. Each is removed with `store.remove` and dropped from `kept` and `failed`. Save.

Other rules:
- Switching to the version that is already active is a no-op that returns `Ok`.
- Rollback is `switch` with one target. With no previous version, or a previous version that has been pruned, it
  returns `NoPrevious`.
- Every lifecycle method takes the manager's one `Mutex` for its whole run, so operations never interleave.

- [ ] **Step 1: Write the failing tests** in `tests/switch.rs`, with `RecordingRuntime`:
  - `switch_activates_and_records_previous`: the events are `prepare`, then `activate`, then `retire` of the old
    version; the state's `active`/`previous` are set and `revision` has increased.
  - `switch_refuses_unkept`.
  - `switch_refuses_incompatible`: the message equals `admin 2.0.0 requires server ^9, active server is <server version>`;
    nothing is prepared.
  - `group_switch_all_or_nothing`: two apps, the second's prepare fails; neither is activated, and `failed` holds the
    second with its reason.
  - `group_switch_satisfies_mutual_requirements`: `a@2` requires `b ^2`, and `b@2` requires `a ^2`; switching either
    alone is refused, and switching both together succeeds.
  - `rollback_to_previous` and `rollback_named`.
  - Review Focus 5: `rollback_without_previous_with_keep_1` → `NoPrevious`.
  - `prune_keeps_n_and_never_active_pinned_or_previous`: `keep = 2`, install four, pin the oldest, and switch to the
    newest; the kept set is pinned, previous and active, and the rest are removed from disk.
  - `pin_requires_kept`, and `unpin`.
  - Review Focus 4, `operations_serialize`: two threads call `switch` at once, with `RecordingRuntime` sleeping 200 ms
    in `prepare`; the events never interleave (a whole prepare/activate pair, then the next), and the final state is
    the second switch's.
  - `save_failure_does_not_activate`: `MemoryStateStore::fail_next_save()` on the step 5 save → `Err`. The runtime
    has activated, so to keep memory and disk in step the manager re-activates the old prepared release. Re-preparing
    the old version must succeed in the test; assert that `active()` is the old version again.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-app --test switch` → FAIL.
- [ ] **Step 3: Implement.** For the save-failure rule: when the step 5 save fails, prepare the old active versions
  again, activate them, and return `State`. If that re-prepare fails too, log an error event
  `memory_and_state_differ` naming the apps (my call; the next start reads `state.json` again).
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-app` → PASS.
- [ ] **Step 5: Commit.** `feat(app): switch, group switch, rollback, pin and prune`

---

### Task 7: `ReleaseManager`: start-up recovery

**Files:**
- Modify: `crates/speedyspec-server-app/src/releases.rs`
- Test: `crates/speedyspec-server-app/tests/boot.rs`

**Interfaces:**
- Produces:
  - `ReleaseManager::boot(&self) -> BootReport`
  - `pub struct BootReport { pub cleaned: Vec<PathBuf>, pub fallbacks: Vec<(AppId, Option<Version>, Option<Version>)>, pub not_installed: Vec<AppId>, pub incompatible: Vec<Incompatibility> }`
    - `fallbacks` holds (app, the version the state named, the version used).
  - `ReleaseManager::last_boot(&self) -> Option<BootReport>`: the manager keeps the report, and `status` (Task 10)
    prints its incompatibilities.

`boot`:
1. `store.clean_partials()`.
2. For each non-server app in the state, drop from `kept` any version whose `manifest` fails to load.
3. If `active` is missing or does not load, fall back to the newest remaining kept version, by version order (my
   call: install order may reflect a rollback). Record the fallback and log `{"event":"active_fallback",…}`.
4. Prepare and activate every active release. If one fails to prepare, try the next newest kept version, and so on.
5. Apps with nothing that loads go into `not_installed`.
6. Save if anything changed.
7. Run `check_set` over the active set. Incompatibilities are reported and logged as an error event
   `incompatible_set`, and the server keeps serving (my call: refusing to start would leave no running server to fix
   it through). `status` shows them.

- [ ] **Step 1: Write the failing tests:**
  - Review Focus 3:
    - `boot_cleans_partials`.
    - `boot_falls_back_when_active_missing`: the state says `active 1.2.0`, whose directory was deleted, and `kept`
      holds `1.0.0` and `1.1.0`, so it activates `1.1.0`.
    - `boot_falls_back_when_prepare_fails`.
    - `boot_reports_not_installed`.
  - `boot_reports_incompatible_but_activates`.
  - `boot_is_quiet_when_consistent` (no save when nothing changed: `MemoryStateStore::saves() == 0`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-app --test boot` → FAIL.
- [ ] **Step 3: Implement** `boot`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-app` → PASS.
- [ ] **Step 5: Commit.** `feat(app): start-up recovery from partials, missing releases and failed prepares`

---

### Task 8: Swappable app handles and the static fall-back grace

**Files:**
- Modify: `crates/speedyspec-server-static/src/build.rs` (add `immutable_only`)
- Create: `crates/speedyspec-server-presentation/src/live.rs`
- Modify: `src/handlers.rs`, `src/service.rs`, `src/lib.rs`, `Cargo.toml` (add arc-swap)
- Test: `crates/speedyspec-server-static/tests/build.rs`, `crates/speedyspec-server-presentation/tests/service.rs`, and
  unit tests in `live.rs`

**Interfaces:**
- Consumes: `StaticSet` (plan 1), `ServeState`/`AppHandle` (plan 1).
- Produces:
  - `StaticSet::immutable_only(&self) -> StaticSet`: only the files whose `cache_control` contains `immutable`, and no
    `not_found`.
  - `pub struct StaticApp { pub current: Arc<StaticSet>, pub fallback: Option<Fallback> }`
  - `pub struct Fallback { pub set: Arc<StaticSet>, pub until: Instant }`
  - `AppHandle::Static(StaticApp)` replaces `AppHandle::Static(Arc<StaticSet>)`; the `NotInstalled`, `PhpPlaceholder`
    and `AdminApiPlaceholder` variants are unchanged.
  - `pub struct LiveApps`:
    - `new(initial: HashMap<AppId, AppHandle>) -> LiveApps`;
    - `get(&self, app: &AppId) -> Option<Arc<AppHandle>>`;
    - `swap(&self, app: &AppId, handle: AppHandle) -> Option<Arc<AppHandle>>` (returns the old handle). The app set
      is fixed at construction, and `swap` on an unknown app panics.
  - `ServeState.apps: LiveApps`; `ServeState::handle(&self, app) -> Option<Arc<AppHandle>>`.

How `StaticApp` answers:
- It answers from `current`.
- When that response is a 404, a fallback exists, `Instant::now() < until`, and the fallback set has the file, it
  answers from the fallback.
- An expired fallback is ignored. Dropping it is the runtime's job, in Task 9.

- [ ] **Step 1: Write the failing tests.**
  - `immutable_only_keeps_immutable_files` (static crate).
  - `live_swap_returns_old_and_serves_new`: while a cloned `Arc` of the old handle is held, `get` already returns the
    new one, so no request ever sees a mix.
  - In `tests/service.rs`:
    - `fallback_serves_old_hashed_asset_during_grace`: the current set lacks `_astro/old.js`, which a fallback with a
      future `until` has → 200 with the old bytes.
    - `fallback_ignored_after_grace` (`until` in the past) → 404.
    - `fallback_never_serves_html` (an `index.html` only in the old set → 404, because it is not immutable).
  - Update plan 1's tests to build `AppHandle::Static(StaticApp { current, fallback: None })` and
    `ServeState { apps: LiveApps::new(…), … }`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-static -p speedyspec-server-presentation` → FAIL.
- [ ] **Step 3: Implement.** `LiveApps` is a `HashMap<AppId, ArcSwap<AppHandle>>`.
- [ ] **Step 4: Run** `cargo test --workspace` → PASS. The binary's `boot.rs` is updated for the new types; immutable
  mode is unchanged.
- [ ] **Step 5: Commit.** `feat(presentation): swappable app handles and a grace period for old static assets`

---

### Task 9: Managed mode in the binary: config, kind runtimes, start-up and auto-update

**Files:**
- Modify: `crates/speedyspec-server/src/config.rs`, `src/main.rs`, `src/boot.rs`
- Create: `crates/speedyspec-server/src/runtime.rs`, `src/managed.rs`
- Test: unit tests in `config.rs` and `runtime.rs`; `crates/speedyspec-server/tests/managed.rs`

**Interfaces:**
- Consumes: `ReleaseManager` (Tasks 5–7), `LiveApps`/`StaticApp`/`Fallback` (Task 8), and every infra crate.
- Produces:
  - Config:
    - `AppConfig.keep: u32` (default 3; 0 is an error naming `app.<id>.keep`).
    - `AppConfig.update: UpdatePolicy` (default `Manual` for the server kind, `Auto` otherwise).
    - `AppConfig.check_interval: Duration` (humantime, default `15m`; unparsable is an error naming the value).
    - `Config.static_fallback_grace: Duration` (`[release.static] fallback_grace`, humantime, default `1h`).
    - In managed mode, a non-server app without `source` is an error naming `app.<id>.source`.
  - `runtime::StaticRuntime::new(live: Arc<LiveApps>, limit: u64, grace: Duration)`, implementing `AppRuntime`.
    - `prepare` builds the `StaticSet`.
    - `activate` swaps in `StaticApp { current: new, fallback: Some(Fallback { set: old.current.immutable_only(), until: now + grace }) }`,
      with no fallback when there was no old static set.
    - A background tokio task in `managed.rs` drops expired fallbacks every minute, by swapping in the same `current`
      with `fallback: None`.
  - `runtime::PhpPlaceholderRuntime`: `prepare` checks the directory exists; `activate` swaps in `PhpPlaceholder`.
  - `managed::start(config: &Config, live: Arc<LiveApps>) -> Result<Arc<ReleaseManager>, BootError>`.
    - It creates `data_dir` and the releases directory if they are missing, builds the manager, and runs `boot`.
    - It logs one `{"event":"unsigned","app":…}` warning per app with `allow_unsigned`.
    - It returns the manager.
  - `managed::auto_update(manager: Arc<ReleaseManager>, apps: Vec<(AppId, Duration)>)`, a spawned task per `auto`
    app.
    - Each check runs `check` then `install` and `switch` through `spawn_blocking`.
    - It runs at once for apps with nothing installed, then every interval.
    - Errors are logged as `{"event":"auto_update_failed","app":…,"error":…}` and never stop the loop.

`main.rs`:
- Managed mode no longer exits 1 (plan 1's refusal is removed, along with its test `refuses_managed_for_now`).
- In managed mode, `prepare` starts every non-server app as `NotInstalled`, then `managed::start` activates what
  `boot` finds.
- The listener binding and `serve` are unchanged.

- [ ] **Step 1: Write the failing tests.**
  - Config:
    - `parses_release_keys_and_defaults`.
    - `rejects_keep_zero` (Review Focus 5).
    - `rejects_bad_interval`.
    - `rejects_managed_without_source`.
    - `parses_fallback_grace`.
  - `runtime.rs`:
    - `static_activate_sets_fallback_from_old_set`.
    - `first_activate_has_no_fallback`.
  - `tests/managed.rs` (the real binary, a temp `data_dir`/`run_dir`, and a local HTTP release host as in Task 4,
    serving an index, two admin releases with real minisign signatures from the Task 4 fixture key, and their
    checksums):
    - `first_start_installs_auto_app`: `GET /` is a 503 (`Retry-After: 30`) until the first install, then a 200 with
      the release's body, within 10 s.
    - `manual_app_waits` (`update = "manual"` → stays at 503, and the release host saw no download).
    - `restart_serves_last_active` (stop, then start again with the release host stopped → the same body).
    - `unsigned_warning_logged`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `cargo test --workspace` → PASS.
- [ ] **Step 5: Commit.** `feat: managed mode installs, switches and auto-updates releases on one host`

---

### Task 10: Control socket and the `release` / `status` commands

**Files:**
- Create: `crates/speedyspec-server/src/control.rs`; modify `src/main.rs`
- Test: `crates/speedyspec-server/tests/control.rs`, `tests/switch_under_load.rs`

**Interfaces:**
- Consumes: `ReleaseManager` (Tasks 5–7), the immutable manifests (plan 1's `prepare`).
- Produces:
  - `control::serve(path: &Path, backend: Arc<dyn ControlBackend>)`, a tokio task.
  - `pub trait ControlBackend: Send + Sync { fn handle(&self, request: ControlRequest) -> ControlResponse; }`
    - Implementations: `ManagedBackend(Arc<ReleaseManager>)` and `ImmutableBackend(BTreeMap<AppId, ReleaseManifest>)`.
  - `#[derive(Serialize, Deserialize)] pub enum ControlRequest`, variants:
    - `List { app: Option<AppId> }`
    - `Check { app: Option<AppId> }`
    - `Install { app: AppId, version: Option<Version> }`
    - `Switch { targets: Vec<(AppId, Version)> }`
    - `Rollback { app: AppId, version: Option<Version> }`
    - `Pin { app: AppId, version: Version }`
    - `Unpin { app: AppId }`
    - `Status`
  - `pub enum ControlResponse { Ok(serde_json::Value), Error(String) }`
  - `control::request(path: &Path, request: &ControlRequest) -> Result<ControlResponse, String>` (the client).

Protocol (my call; the spec leaves it to the builder):
- One connection per command, one JSON request line, one JSON response line.
- The socket is created at `<run_dir>/control.sock` with mode 0600, after removing a stale one.
- The managed backend runs each request through `spawn_blocking`.
- Plan 3 moves clients to `keeper.sock`, and the keeper forwards to this socket.

Commands (spec "Control"), each taking `--config <path>` (default `DEFAULT_CONFIG`) to find `run_dir`:
- `release list [<app>]`;
- `release check [<app>]`;
- `release install <app> [<version>]`;
- `release switch <app>@<version>...`;
- `release rollback <app> [<version>]`;
- `release pin <app> <version>` and `release unpin <app>`;
- `status`.

Output:
- `list` and `status` print one line per app, tab-separated: `app`, `kind`, `active` (or `-`), kept versions joined
  with `,`, `pinned` (or `-`), and failed versions joined with `,`.
- `status` adds a first line `server<TAB><version><TAB><mode>`, and any incompatibility lines from boot as
  `incompatible<TAB><message>`.
- Mutating commands print the result in one line (`switched admin 1.1.0`).
- Errors print `speedyspec-server: <message>` and exit 1.
- When the socket cannot be reached: `speedyspec-server: cannot reach the server at <path>: <os error>; is it running?`,
  exit 1.
- In immutable mode, `install`/`switch`/`rollback`/`pin`/`unpin` answer
  `mode = "immutable": releases are baked into the image; upgrade with the speedyspec CLI`.
- If `run_dir` is not writable in immutable mode, the server logs `{"event":"control_unavailable",…}` and serves
  without the socket.

- [ ] **Step 1: Write the failing tests.**
  - `tests/control.rs`, managed, with the Task 9 release host:
    - `install_then_switch_then_rollback` through the commands, checking stdout and `GET /` bodies at each step.
    - `list_and_status_output` (exact tab-separated lines).
    - `switch_incompatible_refused_with_message`.
    - `pin_stops_auto_update`.
    - `unreachable_socket_message` (the server not running → exit 1 and the message).
    - `socket_is_owner_only` (mode 0600).
  - Immutable:
    - `immutable_list_reports_baked_versions`.
    - `immutable_refuses_changes` (the message names the speedyspec CLI).
    - `immutable_without_writable_run_dir_still_serves` (plan 1's `writes_nothing` still passes, and now also checks
      the `control_unavailable` event).
  - Review Focus 4, `concurrent_switches_serialize`: two `release switch` processes started together (different
    versions); both exit 0; `status` afterwards shows the active version from whichever finished last, the other as
    `previous` in `state.json`; and `state.json` parses.
  - `tests/switch_under_load.rs` (spec testing: "Switch under load"):
    - Load: one task each over HTTP/1.1, h2c, HTTPS/h2 and h3 (the plan 1 `support::h3` helper), requesting `/` and
      `/_astro/app.js` in a loop for 5 s.
    - Meanwhile, `release switch` alternates between admin 1.0.0 and 1.1.0 every 250 ms.
    - Each fixture release stamps its version into both files (`admin-1.0.0`).
    - Assert zero failed requests, and that every response body carries exactly one version stamp, so no response
      mixes releases.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --test control --test switch_under_load` → FAIL.
- [ ] **Step 3: Implement** `control.rs` and the commands. Clap subcommand `release` takes the actions above;
  `<app>@<version>` parses into `(AppId, Version)` with a usage error (exit 2) when malformed.
- [ ] **Step 4: Run** `cargo test --workspace` → PASS on macOS, then the Linux Docker command → PASS.
- [ ] **Step 5: Commit.** `feat: control socket with release and status commands`
