# SpeedySpec CLI — Self-update and Release Builds Implementation Plan (plan 4)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `speedyspec self update [--check]` replaces the running binary with the latest GitHub release, verified
against `SHA256SUMS`; CI builds the four release binaries and release-please publishes them; the binary embeds a
pinned contracts release instead of whatever checkout happens to be next to it.

**Architecture:** A new crate `speedyspec-selfupdate` finds the latest release through the GitHub API, downloads the
asset for the running platform into a temporary file *in the executable's own directory*, verifies size and hash,
runs it once (`--no-plugins --version`), and renames it over the real executable (symlinks resolved). HTTP comes from
a new crate `speedyspec-fetch`, which is the zip driver's existing `Fetch`/`HttpFetch` moved out so the updater does
not depend on a driver. The product version moves from the crate's `Cargo.toml` to `version.txt`, which
release-please (`simple` type) owns. One reusable workflow builds, tests and smoke-runs all four targets; CI calls it
on every pull request, and the release workflow calls it for a new tag and uploads the binaries plus `SHA256SUMS`.
The contracts are fetched by `scripts/fetch-contracts.sh` from a pinned GitHub tag archive (version, URL and
SHA-256 in `contracts-version`) into `.contracts/<version>/`; `build.rs` stays offline and reads that directory, or
`SPEEDYSPEC_CONTRACTS_DIR` for local development.

**Tech Stack:** Rust 1.99 (edition 2024), ureq 3 (rustls, compiled roots) via `speedyspec-fetch`, `sha2` 0.11,
`semver` 1, `tempfile` 3, clap 4; GitHub Actions (`googleapis/release-please-action@v4`, `actions/upload-artifact@v4`,
`actions/download-artifact@v4`), `gh` on the runner; static-php-cli as before; `rhysd/actionlint` in Docker to check
workflows locally.

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-cli-design.md` — sections "Packages", "Commands",
"Contracts" (versioning), "Self-update and distribution", "Errors". Plan 3 (`2026-11-17-speedyspec-cli-kubernetes.md`,
"What this plan leaves for later plans") hands this plan its scope.

## Global Constraints

- Repositories: `packages/speedyspec-cli` (`SpeedySpec/speedyspec-cli`, base `main`) for Tasks 1–6 and 8;
  `packages/speedyspec-cli-contracts` (`SpeedySpec/speedyspec-cli-contracts`, base `main`) for Task 7. Work on a
  branch `feat/self-update` (CLI) and `ci/release` (contracts), cut from an up-to-date `main`.
- **Nothing outward-facing without Jacob's go-ahead:** no push, no pull request, no tag, no GitHub release, no
  workflow dispatch. Read-only `gh` calls (`gh release view`, `gh api` GETs) are allowed.
- **Tests never contact GitHub.** Every test that resolves a release sets `SPEEDYSPEC_GITHUB_API` to a loopback
  `speedyspec_testkit::Server` and removes `GITHUB_TOKEN`. Download URLs in fake release descriptions point at the
  same loopback server.
- **Self-update only ever replaces copies.** Tests and manual checks copy the built binary into a temporary directory
  and run the copy. Never run `self update` (without `--check`) on `target/**/speedyspec`, on anything on `PATH`, or
  against the real GitHub API.
- Nothing installed on the host. Docker images may be pulled (`rust:1.99-alpine` via `scripts/linux.sh`,
  `rhysd/actionlint:1.7.7`). Never set `SPEEDYSPEC_ALLOW_HOST_INSTALL` on the host. Downloading the pinned spc
  binary and the contracts archive to compute their SHA-256 is allowed (the project's own build inputs).
- Asset names, exactly: `speedyspec-linux-x86_64`, `speedyspec-linux-aarch64`, `speedyspec-macos-x86_64`,
  `speedyspec-macos-aarch64` (`speedyspec-{std::env::consts::OS}-{std::env::consts::ARCH}`), plus `SHA256SUMS`
  in `sha256sum` format (`<64 lowercase hex>␠␠<name>\n`, sorted by name). Raw executables, not archives.
- Stable download URL: `https://github.com/SpeedySpec/speedyspec-cli/releases/latest/download/<asset>`.
- Product version: `version.txt` at the CLI repository root, starting at `0.0.0`, owned by release-please
  (`release-type: simple`, manifest `{".": "0.0.0"}`, so the first release is `0.1.0`). Crate versions stay `0.1.0`
  and `Cargo.lock` never changes for a release.
- `MACOSX_DEPLOYMENT_TARGET=13.0` for every macOS build: static PHP and Rust alike (my call; see Review Focus 5).
- Self-update is Unix only (Windows is a spec non-goal). Replacement is a `rename(2)` of a file in the same
  directory: never write into or copy over the existing executable (macOS kills a process whose signed binary was
  modified in place, and Linux refuses to exec a file still open for writing, `ETXTBSY`).
- Size caps: `SHA256SUMS` at most 64 KiB; an asset at most its declared `size` and never more than 512 MiB.
- Exit codes as the spec: 0 success (including "already the latest"), 1 failure, 2 usage error.
- The contracts package's `src/` and `composer.json` do not change; Task 7 adds only release tooling.
- Rust 1.99, edition 2024; `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --all --check`
  clean after every task. Stage explicit paths, never `git add -A`. Conventional Commits; commit messages end with
  `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; PR bodies end with
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.

## Review Focus

Each line has a test in the task named in brackets.

1. **The install location is not writable** (`/usr/local/bin` owned by root, a read-only file system). The command
   says so before downloading anything, names the directory, and gives both fixes (`sudo speedyspec self update`, or
   the stable download URL); nothing is left behind. [Task 3, Task 4]
2. **A download that is bad in any way**: cut short, not matching `SHA256SUMS`, not listed in it, larger than the
   release declares, or a binary that will not start on this machine. The running executable is untouched, no
   temporary file remains in its directory, and the message says which check failed. [Task 3]
3. **The latest release is not ready or not reachable**: no release yet, a release whose assets are still being
   uploaded (the minutes after release-please publishes it), no build for this platform, or GitHub rate limiting
   anonymous calls (HTTP 403/429). Each gets its own message saying what to do; exit 1; nothing changes. [Task 2,
   Task 4]
4. **Run through a symlink** (`~/bin/speedyspec -> /opt/speedyspec/speedyspec`). The file the link points to is
   replaced and the link keeps working; the link is never replaced by a regular file. [Task 3, Task 4]
5. **Release binaries that fail on users' machines**: a macOS binary built on a newer runner that demands that
   runner's macOS, or a binary embedding a contracts version other than the pin (a stray `SPEEDYSPEC_CONTRACTS_DIR` or
   sibling checkout). The build workflow checks `minos` and the `--version` lines of every binary it publishes.
   [Task 5, Task 8]

## File Structure

```
packages/speedyspec-cli/
  Cargo.toml                              [profile.release] strip = "symbols"
  .cargo/config.toml                      [env] MACOSX_DEPLOYMENT_TARGET = "13.0"
  version.txt                             the product version (release-please)
  contracts-version                       version=, url=, sha256= of the pinned contracts archive (Task 8)
  release-please-config.json, .release-please-manifest.json
  .github/workflows/build.yml             reusable: build, test, smoke-run, upload one artifact per target
  .github/workflows/ci.yml                calls build.yml; keeps kubernetes-e2e
  .github/workflows/release.yml           release-please, then build.yml at the tag, then upload to the release
  scripts/build-php.sh                    + Darwin-x86_64 spc, deployment target in the craft stamp
  scripts/linux.sh                        mounts contracts only when SPEEDYSPEC_CONTRACTS_DIR is set (Task 8)
  scripts/release-sums.sh                 writes SHA256SUMS for dist/speedyspec-*
  scripts/fetch-contracts.sh              downloads, verifies and unpacks the pinned contracts (Task 8)
  crates/speedyspec-fetch/                Fetch, FetchError, HttpFetch (moved), github_api_base()
  crates/speedyspec-selfupdate/
    src/lib.rs                            re-exports
    src/release.rs                        asset names, the GitHub release, version comparison, SHA256SUMS
    src/replace.rs                        target executable, writability, download + verify + smoke-run + rename
    tests/update.rs                       against a loopback server and temporary directories
  crates/speedyspec-driver-zip/           re-exports Fetch types from speedyspec-fetch
  crates/speedyspec-cli/
    build.rs                              + SPEEDYSPEC_VERSION from version.txt; contracts from the pin (Task 8)
    src/self_cmd.rs                       `self update [--check]`
    src/commands.rs                       clap tree and dispatch for `self`
    tests/self_update.rs                  end to end: a copy of the binary updates itself
    tests/contracts_pin.rs                fetch-contracts.sh against a local archive (Task 8)
  README.md, docs/php-embed.md
packages/speedyspec-cli-contracts/
  .github/workflows/php-release.yml, .release-please-manifest.json
```

---

### Task 1: Move HTTP fetching into `speedyspec-fetch`

The updater needs the same checked HTTPS downloads the zip driver has, without depending on a driver.

**Files:**
- Create: `crates/speedyspec-fetch/{Cargo.toml,src/lib.rs}` (the contents of `crates/speedyspec-driver-zip/src/fetch.rs`, its tests included)
- Delete: `crates/speedyspec-driver-zip/src/fetch.rs`
- Modify: `crates/speedyspec-driver-zip/{Cargo.toml,src/lib.rs}`, `crates/speedyspec-cli/{Cargo.toml,src/plugin_cmd.rs}`

**Interfaces:**
- Produces: crate `speedyspec_fetch` with `pub trait Fetch`, `pub enum FetchError`, `pub struct HttpFetch` exactly as
  they are today (same variants, `Display`, `new`, `with_timeouts`, GitHub token only to `https://api.github.com`), and
  `pub fn github_api_base() -> Result<String, String>`: `SPEEDYSPEC_GITHUB_API` if set (must pass
  `speedyspec_driver::is_fetchable_url`, else the existing message from `plugin_cmd.rs`), otherwise
  `"https://api.github.com"`. Dependencies: `speedyspec-driver` (for `is_fetchable_url`), `ureq` as in the zip driver.
- `speedyspec_driver_zip` keeps `pub use speedyspec_fetch::{Fetch, FetchError, HttpFetch};` so no caller changes.

- [ ] **Step 1: Add a test for `github_api_base` in `crates/speedyspec-fetch/src/lib.rs`** (environment-mutating, so
  one test that sets, checks and restores, in this order):
  ```rust
  #[test]
  fn github_api_base_defaults_and_overrides() {
      // SAFETY: the only test in this crate that touches the variable.
      unsafe { std::env::remove_var("SPEEDYSPEC_GITHUB_API") };
      assert_eq!(github_api_base().unwrap(), "https://api.github.com");
      unsafe { std::env::set_var("SPEEDYSPEC_GITHUB_API", "http://127.0.0.1:9") };
      assert_eq!(github_api_base().unwrap(), "http://127.0.0.1:9");
      unsafe { std::env::set_var("SPEEDYSPEC_GITHUB_API", "http://example.com") };
      assert!(github_api_base().unwrap_err().contains("SPEEDYSPEC_GITHUB_API must be an https:// URL"));
      unsafe { std::env::remove_var("SPEEDYSPEC_GITHUB_API") };
  }
  ```
- [ ] **Step 2: Run it.** `cargo test -p speedyspec-fetch` Expected: fails to compile (`github_api_base` not found).
- [ ] **Step 3: Move the code.** `git mv` the file into the new crate's `src/lib.rs`; add `github_api_base`; make
  `plugin_cmd.rs` call it instead of reading the variable itself; re-export from the zip driver.
- [ ] **Step 4: Run.** `cargo test --workspace` Expected: all pass (389 + 1). `cargo clippy --workspace --all-targets -- -D warnings`, `cargo fmt --all --check` clean.
- [ ] **Step 5: Commit** `refactor: move HTTP fetching into speedyspec-fetch`.

---

### Task 2: Finding the latest release

**Files:**
- Create: `crates/speedyspec-selfupdate/{Cargo.toml,src/lib.rs,src/release.rs}`
- Test: unit tests in `src/release.rs`

**Interfaces:**
- Consumes: `speedyspec_fetch::{Fetch, FetchError}`.
- Produces (`speedyspec_selfupdate`):
  - `pub const REPOSITORY: &str = "SpeedySpec/speedyspec-cli";` `pub const SUMS: &str = "SHA256SUMS";`
  - `pub fn asset_name(os: &str, arch: &str) -> Option<String>` — `Some("speedyspec-<os>-<arch>")` for `os` in
    `linux`, `macos` and `arch` in `x86_64`, `aarch64`; `None` otherwise. `pub fn current_asset() -> Option<String>`.
  - `pub struct Asset { pub name: String, pub url: String, pub size: u64 }`
  - `pub struct Release { pub tag: String, pub version: semver::Version, pub assets: Vec<Asset> }` with
    `pub fn asset(&self, name: &str) -> Option<&Asset>`.
  - `pub struct UpdateError(pub String)` — `Display` prints the string; every failure in this crate is one.
  - `pub fn latest_release(fetch: &dyn Fetch, api_base: &str) -> Result<Release, UpdateError>` — GET
    `{api_base}/repos/SpeedySpec/speedyspec-cli/releases/latest` (body capped at 1 MiB); `tag_name` with one leading
    `v` stripped must parse as semver; assets from `name`, `browser_download_url`, `size`.
  - `pub enum Check { UpToDate(Version), Ahead { latest: Version }, Newer(Release) }` and
    `pub fn check(fetch: &dyn Fetch, api_base: &str, current: &Version) -> Result<Check, UpdateError>`.
  - `pub fn parse_sums(text: &str) -> Result<BTreeMap<String, String>, String>` — lines `<64 hex> <space> <space or *><name>`;
    hex lowercased; blank lines ignored; a malformed line or a repeated name is an error naming the line number.

- [ ] **Step 1: Write the failing tests** (a `FakeApi` implementing `Fetch` that answers one body or one status, as in
  `plugin_cmd.rs`'s tests):
  ```rust
  #[test] fn asset_names() {
      assert_eq!(asset_name("linux", "x86_64").as_deref(), Some("speedyspec-linux-x86_64"));
      assert_eq!(asset_name("macos", "aarch64").as_deref(), Some("speedyspec-macos-aarch64"));
      assert_eq!(asset_name("windows", "x86_64"), None);
      assert_eq!(asset_name("linux", "riscv64"), None);
  }
  #[test] fn reads_the_latest_release() { /* body {"tag_name":"v0.3.0","assets":[{"name":"SHA256SUMS",
      "browser_download_url":"https://x/SHA256SUMS","size":400}, {"name":"speedyspec-linux-x86_64", ...,"size":123}]}
      → version 0.3.0, tag "v0.3.0", asset("speedyspec-linux-x86_64").size == 123; the one request was
      "https://api.github.com/repos/SpeedySpec/speedyspec-cli/releases/latest" */ }
  #[test] fn compares_versions() { /* current 0.2.0: latest 0.3.0 → Newer, 0.2.0 → UpToDate(0.2.0),
      0.1.9 → Ahead { latest: 0.1.9 } */ }
  #[test] fn no_release_yet() { /* 404 → "no release of speedyspec has been published yet" */ }
  #[test] fn rate_limited() { /* 403 and 429 → "GitHub refused the request (HTTP 403); it limits anonymous
      requests, so set GITHUB_TOKEN and try again" (429 says 429) */ }
  #[test] fn unreadable_tag() { /* tag "nightly" → "the latest release has the tag \"nightly\", which is not a
      version" */ }
  #[test] fn sums_parse() {
      let text = format!("{}  speedyspec-linux-x86_64\n{} *speedyspec-macos-aarch64\n\n", "A".repeat(64), "b".repeat(64));
      let sums = parse_sums(&text).unwrap();
      assert_eq!(sums["speedyspec-linux-x86_64"], "a".repeat(64));
      assert_eq!(sums.len(), 2);
      assert_eq!(parse_sums("abc  x\n").unwrap_err(), "line 1 of SHA256SUMS is not \"<sha256>  <name>\"");
      let twice = format!("{0}  x\n{0}  x\n", "a".repeat(64));
      assert_eq!(parse_sums(&twice).unwrap_err(), "line 2 of SHA256SUMS lists x again");
  }
  ```
- [ ] **Step 2: Run.** `cargo test -p speedyspec-selfupdate` Expected: does not compile (crate items missing).
- [ ] **Step 3: Implement** `src/release.rs` and the crate (`Cargo.toml` deps: `speedyspec-fetch`, `semver`,
  `serde_json`, `sha2`, `tempfile`; dev: `speedyspec-testkit`).
- [ ] **Step 4: Run.** `cargo test -p speedyspec-selfupdate` Expected: 7 passed.
- [ ] **Step 5: Commit** `feat(selfupdate): find the latest release and read SHA256SUMS`.

---

### Task 3: Downloading, verifying and swapping in the new binary

**Files:**
- Create: `crates/speedyspec-selfupdate/src/replace.rs`, `crates/speedyspec-selfupdate/tests/update.rs`

**Interfaces:**
- Consumes: Task 2's `Release`, `Asset`, `parse_sums`, `SUMS`, `UpdateError`.
- Produces:
  - `pub fn target_executable(invoked: &Path) -> Result<PathBuf, UpdateError>` — `invoked` canonicalised (the real
    file behind any symlink). The CLI passes `std::env::current_exe()`.
  - `pub fn writable_problem(exe: &Path, asset: &str) -> Option<String>` — tries to create (and removes) a temporary
    file in `exe`'s directory; on failure returns exactly:
    `cannot replace <exe>: <dir> is not writable by you. Run \`sudo speedyspec self update\`, or download https://github.com/SpeedySpec/speedyspec-cli/releases/latest/download/<asset> yourself`
  - `pub fn install(fetch: &dyn Fetch, release: &Release, asset: &str, exe: &Path, progress: &mut dyn FnMut(&str)) -> Result<(), UpdateError>`
    — in order: `writable_problem` (before any download); the release must have `asset` and `SHA256SUMS`; fetch the
    sums (≤ 64 KiB) and find `asset`; stream the asset into `tempfile::Builder::new().prefix(".speedyspec-update-")`
    in `exe`'s directory through a writer that hashes, counts and fails past `min(asset.size, 512 MiB)`; compare the
    hash; close the file (`into_temp_path`), set mode `0o755`; run it with `--no-plugins --version`, killing it after
    30 s, and require exit 0 and a first line `speedyspec <release.version>`; then `TempPath::persist(exe)`. Any
    failure before the rename drops the temporary path, which deletes it.
  - Messages (the release's tag is `v0.3.0`, the asset `speedyspec-linux-x86_64` in these examples):
    - no asset: `release v0.3.0 has no speedyspec-linux-x86_64 yet; if it was published in the last few minutes its files are still being uploaded, so try again shortly`
    - no sums: `release v0.3.0 has no SHA256SUMS, so its download cannot be verified; nothing was changed`
    - not listed: `SHA256SUMS of release v0.3.0 does not list speedyspec-linux-x86_64; nothing was changed`
    - mismatch: `the download of speedyspec-linux-x86_64 does not match SHA256SUMS (expected <hex>, got <hex>); nothing was changed`
    - too large: `speedyspec-linux-x86_64 is larger than release v0.3.0 says (<size> bytes); nothing was changed`
    - cut short / connection: `could not download speedyspec-linux-x86_64: <FetchError>; nothing was changed`
    - will not start: `the downloaded speedyspec-linux-x86_64 did not run (<reason>); nothing was changed`, where
      `<reason>` is `exit status 3`, `it printed "<first line>"`, `it did not finish within 30s`, or the spawn error.

- [ ] **Step 1: Write the failing tests** in `tests/update.rs`. Fixtures: a `Server` with routes `/sums` and `/asset`;
  a release built in the test with asset URLs `{server}/asset`, `{server}/sums`; the "new binary" is a shell script
  `#!/bin/sh\necho "speedyspec 0.3.0"\n` (a `bad` variant prints `speedyspec 0.2.9`, a `failing` one `exit 3`); the
  "old binary" is a file `bin/speedyspec` containing `old` in a `tempfile::TempDir`. Helper `leftovers(dir)` lists names
  starting with `.speedyspec-update-`.
  - `replaces_the_executable` — after `install`, `bin/speedyspec` has the script's bytes and mode `0o755`; leftovers
    empty; progress saw `downloading speedyspec-linux-x86_64 (v0.3.0)` then `verifying`.
  - `follows_a_symlink` — `link/speedyspec -> ../bin/speedyspec`; `install(…, &target_executable(link)?)`;
    `bin/speedyspec` replaced; `link/speedyspec` is still a symlink (`symlink_metadata().file_type().is_symlink()`).
  - `checksum_mismatch_changes_nothing`, `unlisted_asset_changes_nothing`, `missing_sums_changes_nothing`,
    `missing_asset_says_to_wait` — exact messages above; `bin/speedyspec` still reads `old`; leftovers empty.
  - `larger_than_declared_changes_nothing` — declared size 10, served 40 bytes → the too-large message.
  - `cut_short_changes_nothing` — `Route::truncated(script, 5)` → message starts
    `could not download speedyspec-linux-x86_64:`; unchanged; leftovers empty.
  - `a_binary_that_does_not_start_is_not_installed` — `failing` → `… did not run (exit status 3); nothing was changed`;
    `bad` → `… did not run (it printed "speedyspec 0.2.9"); …`; unchanged; leftovers empty.
  - `unwritable_directory_is_reported_before_downloading` — `bin` mode `0o555`. If a file can still be created there
    (running as root, as in `scripts/linux.sh`), print `skipped: running as root` and return. Otherwise: the exact
    writable message; the server received no request; mode restored at the end.
- [ ] **Step 2: Run.** `cargo test -p speedyspec-selfupdate --test update` Expected: does not compile.
- [ ] **Step 3: Implement `src/replace.rs`.**
- [ ] **Step 4: Run.** `cargo test -p speedyspec-selfupdate` Expected: 7 unit + 10 integration passed.
  `scripts/linux.sh cargo test -p speedyspec-selfupdate` Expected: the same, with the unwritable test printing
  `skipped: running as root` (Linux exec of a just-closed file proves no `ETXTBSY`).
- [ ] **Step 5: Commit** `feat(selfupdate): download, verify and swap in a release binary`.

---

### Task 4: `speedyspec self update [--check]`

**Files:**
- Create: `version.txt` (`0.0.0\n`), `crates/speedyspec-cli/src/self_cmd.rs`, `crates/speedyspec-cli/tests/self_update.rs`
- Modify: `crates/speedyspec-cli/{build.rs,Cargo.toml,src/commands.rs,src/main.rs}`, `crates/speedyspec-cli/tests/common/mod.rs`, `README.md`

**Interfaces:**
- Consumes: `speedyspec_selfupdate::{check, Check, install, current_asset, target_executable, latest_release}`,
  `speedyspec_fetch::{HttpFetch, github_api_base}`.
- Produces: `build.rs` sets `cargo:rustc-env=SPEEDYSPEC_VERSION=<version.txt trimmed>` (must parse as semver, else
  the build fails naming the file) and `rerun-if-changed` on it; `--version`'s first line and the Kubernetes record's
  `cli_version` use `env!("SPEEDYSPEC_VERSION")` instead of `CARGO_PKG_VERSION`. `HttpFetch`'s user agent stays the
  crate version (no behaviour depends on it). `common::Speedyspec::at(exe: &Path, home: &Path) -> Speedyspec` runs
  another executable with the same environment rules.
- Output:
  - `--check`, newer: stdout `speedyspec 0.3.0 is available (this is 0.0.0); run \`speedyspec self update\`` — exit 0.
  - `--check` or update, same version: `speedyspec 0.0.0 is the latest release` — exit 0.
  - ahead: `speedyspec 0.0.0 is newer than the latest release, <v>` — exit 0.
  - update: progress lines on stderr prefixed `speedyspec: `; stdout `updated <real path> from 0.0.0 to 0.3.0` — exit 0.
  - unsupported platform: `there is no release build for <os> <arch>; build speedyspec from source` — exit 1.
  - every `UpdateError` on stderr as `speedyspec: <message>` — exit 1.
  - `self update` runs no plugin code (it is handled before plugins are consulted for anything; it works with
    `--no-plugins` and with a broken plugin installed).

- [ ] **Step 1: Write the failing tests** in `tests/self_update.rs`. Each copies `env!("CARGO_BIN_EXE_speedyspec")` to
  `<tmp>/bin/speedyspec` and runs the copy through `Speedyspec::at` with `SPEEDYSPEC_GITHUB_API={server}`. The release
  JSON names tag `v9.9.9`, an asset `speedyspec-{OS}-{ARCH}` (from `std::env::consts`) at `{server}/download/asset`
  and `SHA256SUMS` at `{server}/download/sums`; the asset is the script `#!/bin/sh\necho "speedyspec 9.9.9"\n`.
  - `check_reports_a_newer_release` — `["self", "update", "--check"]` → code 0, stdout exactly
    `speedyspec 9.9.9 is available (this is 0.0.0); run \`speedyspec self update\`\n`; the copy is unchanged; the server
    saw only the `releases/latest` request.
  - `updates_the_copy` — `["self", "update"]` → code 0; stdout `updated <canonical copy path> from 0.0.0 to 9.9.9\n`;
    running the copy with `--version` now prints `speedyspec 9.9.9`.
  - `updates_through_a_symlink` — run `<tmp>/link/speedyspec -> ../bin/speedyspec`; afterwards the link is a link and
    `bin/speedyspec` prints `speedyspec 9.9.9`.
  - `up_to_date_is_not_an_error` — release `v0.0.0` → code 0, `speedyspec 0.0.0 is the latest release\n`; no download.
  - `no_release_yet` — 404 → code 1, stderr `speedyspec: no release of speedyspec has been published yet`.
  - `assets_not_uploaded_yet` — release with no assets → code 1, the "try again shortly" message; copy unchanged.
  - `works_with_a_broken_plugin` — with the `fatal` fixture plugin loaded (as `cli.rs`'s
    `fatal_at_load_skips_and_names_plugin` loads it), `self update --check` still exits 0 with the available line.
  - `version_comes_from_version_txt` — `--version` first line is `speedyspec 0.0.0`.
- [ ] **Step 2: Run.** `cargo test -p speedyspec-cli --test self_update` Expected: fails (no `self` subcommand).
- [ ] **Step 3: Implement** `version.txt`, `build.rs`, `self_cmd.rs`, the clap subtree (`self` → `update` with
  `--check`, `subcommand_required(true)`), and dispatch; update existing tests that assert `0.1.0` as the CLI version.
- [ ] **Step 4: Run.** `cargo test --workspace` Expected: all pass. Manual, read-only, on a copy:
  `cp target/debug/speedyspec "$TMP/speedyspec" && "$TMP/speedyspec" self update --check` against the real API
  Expected: exit 1, `speedyspec: no release of speedyspec has been published yet` (no release exists yet).
- [ ] **Step 5: README.** Document `self update [--check]`, what it verifies, the not-writable fixes, that it needs no
  credentials (`GITHUB_TOKEN` only lifts GitHub's rate limit), and that it runs no plugin code.
- [ ] **Step 6: Commit** `feat: speedyspec self update` (explicit paths).

---

### Task 5: Release binaries for four targets

**Files:**
- Create: `.cargo/config.toml`, `.github/workflows/build.yml`
- Modify: `Cargo.toml`, `scripts/build-php.sh`, `.github/workflows/ci.yml`, `docs/php-embed.md`

**Interfaces:**
- Produces `build.yml` (`on: workflow_call`, inputs `ref: string` (default `""`, the checkout ref) and
  `contracts-ref: string` (default `main`)), job `build` with this matrix — `asset`, `runs-on`, `run`, `php-build`,
  `binary`:
  | asset | runs-on | run | php-build | binary |
  |---|---|---|---|---|
  | speedyspec-macos-aarch64 | macos-15 | `""` | .php-build | target/release/speedyspec |
  | speedyspec-macos-x86_64 | macos-15-intel | `""` | .php-build | target/release/speedyspec |
  | speedyspec-linux-x86_64 | ubuntu-latest | scripts/linux.sh | .php-build-linux | target-linux/x86_64-unknown-linux-musl/release/speedyspec |
  | speedyspec-linux-aarch64 | ubuntu-24.04-arm | scripts/linux.sh | .php-build-linux-arm64 | target-linux-arm64/aarch64-unknown-linux-musl/release/speedyspec |

  (the aarch64 Linux row sets `SPEEDYSPEC_LINUX_PLATFORM: linux/arm64`). Steps per row: checkout both repositories as
  `ci.yml` does today; Rust 1.99 on macOS; PHP cache keyed by asset name; `build-php.sh`; `cargo test --workspace`;
  `cargo build --release --locked -p speedyspec-cli`; **smoke** — `"$binary" --version` must print
  `speedyspec $(cat version.txt)` on line 1 and a `contracts ` line; `"$binary" --no-plugins php
  examples/install-wordpress/plugin.php` must exit 0 (any file that only registers hooks); on macOS
  `otool -l "$binary"` must show `minos 13.0` under `LC_BUILD_VERSION`; copy to `dist/<asset>`; `upload-artifact`
  named `<asset>`, `if-no-files-found: error`, retention 7 days. `permissions: contents: read`.
- `ci.yml`: the `test` job becomes `uses: ./.github/workflows/build.yml`; `kubernetes-e2e` unchanged.

- [ ] **Step 1: Deployment target.** `.cargo/config.toml` `[env] MACOSX_DEPLOYMENT_TARGET = "13.0"`; `build-php.sh`
  exports `MACOSX_DEPLOYMENT_TARGET="${MACOSX_DEPLOYMENT_TARGET:-13.0}"` on Darwin and writes it into the craft stamp
  so a change rebuilds PHP; add `Darwin-x86_64) spc_name="spc-macos-x86_64"; spc_sha256=<computed>` with the hash
  from `curl -fsSL https://dl.static-php.dev/static-php-cli/spc-bin/nightly/spc-macos-x86_64 | shasum -a 256`.
- [ ] **Step 2: Check it fails first.** `otool -l target/release/speedyspec | grep -A4 LC_BUILD_VERSION` on today's
  build Expected: `minos` is the host's macOS (not 13.0) — the check below would fail.
- [ ] **Step 3: Rebuild.** `scripts/build-php.sh && cargo build --release -p speedyspec-cli`, then the same `otool`
  check, and `otool -l .php-build/buildroot/lib/libphp.a | grep minos | sort -u` Expected: only `minos 13.0` (or
  lower). If spc's objects still carry the host version, pass `-mmacosx-version-min=13.0` through `CFLAGS`/
  `LDFLAGS` in `build-php.sh` and ledger a ruling.
- [ ] **Step 4: Strip.** `[profile.release] strip = "symbols"` in the workspace `Cargo.toml`; rebuild; the smoke
  commands pass; record the macOS arm64 size before and after.
- [ ] **Step 5: Linux aarch64 locally** (native on Apple silicon):
  `SPEEDYSPEC_LINUX_PLATFORM=linux/arm64 scripts/linux.sh scripts/build-php.sh`, then
  `SPEEDYSPEC_LINUX_PLATFORM=linux/arm64 scripts/linux.sh cargo test --workspace` Expected: all pass;
  then the release build and the smoke commands inside the container. Linux x86_64: release build and smoke as in
  plan 3. Record all three sizes in `docs/php-embed.md`; macOS x86_64 is "built in CI only".
- [ ] **Step 6: Workflows.** Write `build.yml`, change `ci.yml`. Check:
  `docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.7 -color` Expected: no findings.
- [ ] **Step 7: Run everything.** `cargo test --workspace`, clippy, fmt; `scripts/linux.sh cargo test --workspace`.
- [ ] **Step 8: Commit** `ci: build, test and smoke-run release binaries for four targets`.

---

### Task 6: Releases with release-please

**Files:**
- Create: `release-please-config.json`, `.release-please-manifest.json`, `.github/workflows/release.yml`, `scripts/release-sums.sh`
- Modify: `crates/speedyspec-selfupdate/tests/update.rs`, `README.md`

**Interfaces:**
- `release-please-config.json`: the same shape as `speedyspec-support`'s (title pattern `chore: release ${version}`,
  same changelog sections, `bump-minor-pre-major`, `bump-patch-for-minor-pre-major`), with `release-type: simple`
  and no `extra-files`. Manifest `{".": "0.0.0"}`.
- `scripts/release-sums.sh <dir>`: writes `<dir>/SHA256SUMS` for every `<dir>/speedyspec-*` regular file, sorted by
  name, in `sha256sum` format (uses `sha256sum`, or `shasum -a 256` where that is missing); exit 2 with a usage line
  without an argument; exit 1 if there is no `speedyspec-*` file.
- `release.yml`: `on: push: branches [main]` and `workflow_dispatch: inputs: tag (required)`. `concurrency: release`.
  Jobs:
  1. `release-please` (push only; `permissions: contents: write, pull-requests: write`) — outputs `release_created`,
     `tag_name`.
  2. `build` — `if: needs.release-please.outputs.release_created == 'true' || github.event_name == 'workflow_dispatch'`
     (with `always() && !cancelled()` so it runs when job 1 is skipped); `uses: ./.github/workflows/build.yml` with
     `ref:` the tag; `permissions: contents: read`.
  3. `publish` — `permissions: contents: write`; `download-artifact` with `pattern: speedyspec-*`,
     `merge-multiple: true` into `dist/`; require exactly four files; `scripts/release-sums.sh dist`;
     `gh release upload "$TAG" dist/* --clobber --repo "$GITHUB_REPOSITORY"` with `GH_TOKEN: ${{ github.token }}`.
  The dispatch input is the way to rebuild and re-upload the assets of an existing release (a failed build, or the
  first release); it uploads to a release that must already exist.

- [ ] **Step 1: Write the failing test** in `crates/speedyspec-selfupdate/tests/update.rs`:
  `release_sums_script_output_is_read_back` — writes `speedyspec-linux-x86_64` (`a`) and `speedyspec-macos-aarch64`
  (`bb`) and an unrelated `notes.txt` into a temp dir; runs `../../scripts/release-sums.sh <dir>` (path from
  `CARGO_MANIFEST_DIR`); `parse_sums` of the written file has exactly the two names, each equal to the SHA-256 the
  test computes with `sha2`; the file's lines are sorted.
- [ ] **Step 2: Run.** `cargo test -p speedyspec-selfupdate --test update release_sums` Expected: fails (script missing).
- [ ] **Step 3: Implement** the script, the config, the manifest and `release.yml`.
- [ ] **Step 4: Run.** The test passes; `scripts/linux.sh cargo test -p speedyspec-selfupdate` passes (GNU
  `sha256sum` path); actionlint as in Task 5 Expected: no findings.
- [ ] **Step 5: README.** A "Download" section: a table of the four stable URLs; `curl -fsSLO` of the asset and of
  `SHA256SUMS`, `grep " speedyspec-linux-x86_64$" SHA256SUMS | sha256sum -c` (macOS: `shasum -a 256 -c`), then
  `chmod +x` and rename; that browser downloads on macOS are quarantined (`xattr -d com.apple.quarantine`) because the
  binaries are not notarized. A "Releasing" section for maintainers: merge release-please's PR; the binaries follow
  within about half an hour; until then `self update` and the stable URLs say or return that the files are missing;
  re-run with *Run workflow* and the tag.
- [ ] **Step 6: Commit** `ci: publish release binaries and SHA256SUMS with release-please`.

---

### Task 7: A release workflow for the contracts repository

In `packages/speedyspec-cli-contracts`, branch `ci/release`.

**Files:**
- Create: `.github/workflows/php-release.yml`, `.release-please-manifest.json`

**Interfaces:**
- `php-release.yml` is `speedyspec-support`'s `php-release.yml` with `php-version: '8.4'` (the package requires
  `>=8.4`) and the same jobs (`test`, `release-please`, `packagist`). Manifest `{".": "1.0.0"}`, so release-please
  continues from a `v1.0.0` release once one exists (Task 8's gate).

- [ ] **Step 1: Check the tests run as CI will.** `composer install --prefer-dist --no-progress && vendor/bin/pest`
  Expected: passes (host PHP 8.4).
- [ ] **Step 2: Write the files**; actionlint (Task 5's command, run in this repository) Expected: no findings.
- [ ] **Step 3: Commit** `ci: release with release-please` (contracts repository, explicit paths).

---

### Task 8: Embed a pinned contracts release

**Gate.** This task needs the tag `v1.0.0` of `SpeedySpec/speedyspec-cli-contracts`.

- [ ] **Step 1: Check the gate.** `gh release view v1.0.0 --repo SpeedySpec/speedyspec-cli-contracts --json tagName`
  Expected: `{"tagName":"v1.0.0"}`. **If it is missing, stop and ask Jacob** to publish it — commit `d3d1ab2` is
  `feat: add CLI contracts 1.0.0`:
  `gh release create v1.0.0 --repo SpeedySpec/speedyspec-cli-contracts --target d3d1ab2 --title v1.0.0 --notes "First release."`
  Do not create it yourself.

**Files:**
- Create: `contracts-version`, `scripts/fetch-contracts.sh`, `crates/speedyspec-cli/tests/contracts_pin.rs`
- Modify: `crates/speedyspec-cli/build.rs`, `scripts/linux.sh`, `.gitignore` (`/.contracts/`),
  `.github/workflows/{build.yml,ci.yml}`, `README.md`

**Interfaces:**
- `contracts-version` (three `key=value` lines):
  `version=1.0.0`, `url=https://github.com/SpeedySpec/speedyspec-cli-contracts/archive/refs/tags/v1.0.0.tar.gz`,
  `sha256=<computed once with curl -fsSL <url> | shasum -a 256>`.
- `scripts/fetch-contracts.sh`: reads the pin (`SPEEDYSPEC_CONTRACTS_PIN` overrides its path, for tests); if
  `.contracts/<version>/.sha256` already holds the pinned hash, prints `fetch-contracts.sh: .contracts/<version> is up
  to date` and exits 0; otherwise downloads to a temporary file, verifies (mismatch: exit 1,
  `fetch-contracts.sh: checksum mismatch for <url>` with expected and actual, nothing left in `.contracts`), unpacks
  with `--strip-components=1` into `.contracts/<version>.tmp`, writes `.sha256`, and renames into place.
- `build.rs` contracts source, in order: `SPEEDYSPEC_CONTRACTS_DIR` if set (a `cargo:warning` when its
  `composer.json` version differs from the pin: `building with contracts <v> from <dir>, not the pinned <pin>`);
  else `.contracts/<pinned version>` at the repository root, whose `composer.json` version must equal the pin; else
  fail with `run scripts/fetch-contracts.sh, or set SPEEDYSPEC_CONTRACTS_DIR to a checkout of speedyspec-cli-contracts`.
  The sibling checkout is no longer used implicitly. `rerun-if-changed` on `contracts-version` and the chosen dir.
- `scripts/linux.sh` mounts and passes a contracts directory only when `SPEEDYSPEC_CONTRACTS_DIR` is set.
- Workflows: `build.yml` and `kubernetes-e2e` drop the contracts checkout; `build.yml` runs
  `scripts/fetch-contracts.sh` before `build-php.sh`, and its smoke check also requires the `--version` line
  `contracts <version from contracts-version>`; the `contracts-ref` input is removed.

- [ ] **Step 2: Write the failing tests** in `tests/contracts_pin.rs`. Each builds a `.tar.gz` with `tar -czf` of a
  directory `speedyspec-cli-contracts-1.0.0/{composer.json,src/X.php}`, serves it from a `Server`, writes a pin file
  naming it, and runs the script with `SPEEDYSPEC_CONTRACTS_PIN=<pin>` from a temporary copy of the repository's
  `scripts/` so `.contracts` lands in the temp dir:
  - `fetches_and_unpacks_the_pinned_archive` — exit 0; `.contracts/1.0.0/composer.json` and `src/X.php` exist.
  - `a_second_run_downloads_nothing` — second run exit 0, prints `is up to date`, the server saw one request.
  - `checksum_mismatch_leaves_nothing` — wrong `sha256` → exit 1, stderr contains `checksum mismatch`,
    `.contracts` has no `1.0.0` and no `1.0.0.tmp`.
- [ ] **Step 3: Run.** `cargo test -p speedyspec-cli --test contracts_pin` Expected: fail (script missing).
- [ ] **Step 4: Implement** the pin (hash computed now), the script, `build.rs`, `linux.sh`, `.gitignore`.
- [ ] **Step 5: Run.** The three tests pass. Then, with `SPEEDYSPEC_CONTRACTS_DIR` unset:
  `rm -rf .contracts && cargo build -p speedyspec-cli` Expected: fails with the `run scripts/fetch-contracts.sh` line;
  `scripts/fetch-contracts.sh && cargo build -p speedyspec-cli && target/debug/speedyspec --version` Expected: line 3
  `contracts 1.0.0`; `SPEEDYSPEC_CONTRACTS_DIR=../speedyspec-cli-contracts cargo build -p speedyspec-cli` Expected:
  builds without a warning (same version).
- [ ] **Step 6: Workflows and docs**: change `build.yml`/`ci.yml`; actionlint Expected: no findings; README
  "Building" section: `scripts/fetch-contracts.sh` before the first build, `SPEEDYSPEC_CONTRACTS_DIR` for working on
  the contracts, how to move the pin.
- [ ] **Step 7: Run everything.** `cargo test --workspace`, clippy, fmt, `scripts/linux.sh cargo test --workspace`
  (with `SPEEDYSPEC_CONTRACTS_DIR` unset, using `.contracts`).
- [ ] **Step 8: Commit** `build: embed the pinned contracts release`.

---

## What this plan leaves for later

- Apple notarization and Developer ID signing (needs Jacob's Apple account); build provenance attestations;
  verifying a signature, not only `SHA256SUMS`, in `self update`.
- `self update <version>`, rolling back to the previous binary, update notices from other commands.
- An install script (`curl … | sh`); package-manager formulas.
- Windows (a spec non-goal); (S)FTP, Docker, Composer, Functions and AWS drivers; broker hook backends.
- The minors deferred in plans 2 and 3.
