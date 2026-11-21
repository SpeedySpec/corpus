# SpeedySpec server and PHP runtime release pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Merging a release pull request builds, tests, signs and publishes the server or the PHP runtime for three targets, and GitHub Pages serves each component's release index.

**Architecture:** Four small bash scripts hold every decision (pending, smoke, publish plan, index), each tested from Rust against fixtures. Thin workflows call them: a reusable per-target build, a release workflow (pending → build → release-please → publish → pages), and release-please with the `cargo-workspace` plugin.

**Tech Stack:** GitHub Actions, release-please v4 (`googleapis/release-please-action@v4`), bash + jq, minisign, `gh`, Rust 1.99.

**Spec:** `docs/superpowers/specs/2026-11-20-speedyspec-server-release-pipeline-design.md`

**Repository:** SpeedySpec/speedyspec-server, worktree `packages/speedyspec-server-runtime-gate`, branch from `origin/main`.

## Global Constraints

- Targets: `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl`, `aarch64-apple-darwin`. Runners: `ubuntu-latest` + `scripts/linux-php.sh` (`linux/amd64`), `ubuntu-24.04-arm` + `scripts/linux-php.sh` (`linux/arm64`), `macos-15`.
- Components → package, binary, app id, tag prefix: `speedyspec-server` → `speedyspec-server`, `server`, `speedyspec-server-v`; `speedyspec-php-runtime` → `speedyspec-php-runtime`, `php-runtime`, `speedyspec-php-runtime-v`.
- Asset names: `<binary>-<triple>`, `<binary>-<triple>.minisig`, `SHA256SUMS` (sorted, `sha256sum` format).
- Index URLs: `https://releases.speedyspec.com/server/index.json`, `…/php-runtime/index.json`.
- Secrets `MINISIGN_SECRET_KEY`, `MINISIGN_PASSWORD`; public key file `minisign.pub` at the repo root. Missing secrets or key: publish fails, nothing uploaded.
- A complete published release is never changed. An incomplete one has its assets replaced as a whole set.
- Rust 1.99; TLS stays on `ring`; Conventional Commits; commits end `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Scripts: `#!/usr/bin/env bash`, `set -euo pipefail`, usage line and exit 2 on bad arguments, GNU and BSD tools both (`sha256sum` or `shasum -a 256`).
- Never commit this plan or the spec.

## Review Focus

1. A release whose assets are partly uploaded (one target missing, or `SHA256SUMS` missing): it must never appear in an index, and a re-run must replace the set, not mix old and new binaries.
2. A dispatch for a complete release: it must change no asset (no `--clobber` path reachable) and only redeploy Pages.
3. A push to `main` with no merged release pull request (including before the first release, manifest `0.0.0`): nothing builds.
4. The runtime's `manifest` has no `app` field (only `version`, `contracts`, `php`); the smoke check must not require one for it.
5. More than 30 releases: the index must page through all of them (`gh api --paginate`).

Each has a test in the task that owns it (Tasks 1, 3, 4).

---

### Task 1: `scripts/release-pending.sh`

**Files:**
- Create: `scripts/release-pending.sh`, `crates/speedyspec-server-domain/tests/release_scripts.rs` (with a `support` helper `script(name) -> PathBuf` from `CARGO_MANIFEST_DIR/../../scripts`)

**Interfaces:**
- Produces: `scripts/release-pending.sh <component>` with `<component>` one of the two above; reads `crates/<component>/Cargo.toml`'s `version` and `.release-please-manifest.json`'s `"crates/<component>"` entry (jq); tags must be fetched. Prints `pending=true|false`, `tag=<component>-v<version>`, `version=<version>`. Pending iff crate version == manifest version, manifest version != `0.0.0`, and the tag does not exist. Exit 2 for an unknown component.
- `SPEEDYSPEC_REPO` overrides the repo root (tests).

- [ ] **Step 1: Write the failing tests** in `release_scripts.rs`, each building a temp git repo with `crates/<c>/Cargo.toml` (`version = "…"`), `.release-please-manifest.json` and tags:
  - `pending_after_a_merged_release_pr`: crate 0.2.0, manifest 0.2.0, no tag → `pending=true`, `tag=speedyspec-server-v0.2.0`.
  - `not_pending_once_tagged`: same plus tag `speedyspec-server-v0.2.0` → `pending=false`.
  - `never_released_sentinel_is_not_pending`: crate 0.1.0, manifest 0.0.0 → `pending=false`.
  - `open_release_pr_is_not_pending`: crate 0.1.0, manifest 0.2.0 (impossible mid-state, still false) → `pending=false`.
  - `components_are_independent`: runtime pending, server not, in one repo.
  - `unknown_component_exits_2`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-domain --test release_scripts` Expected: FAIL (script missing).
- [ ] **Step 3: Implement** the script.
- [ ] **Step 4: Run** the same Expected: PASS.
- [ ] **Step 5: Commit** `ci(release): say which component has a merged, untagged release`.

### Task 2: Release profile and `scripts/release-smoke.sh`

**Files:**
- Create: `scripts/release-smoke.sh`, `crates/speedyspec-server/tests/release_smoke.rs`
- Modify: `Cargo.toml` (workspace root)

**Interfaces:**
- Produces: `scripts/release-smoke.sh <binary-path> <component> <version>`. Runs `<binary-path> manifest`, parses it with jq. Server: `.app == "server"` and `.version == <version>`. Runtime: `.version == <version>` and `.contracts | length > 0` (no `app` field). On Linux, `file -b <binary-path>` contains `statically linked` or `static-pie linked`. Exit 1 naming the failed check; exit 2 on usage.
- `[profile.release]`: `strip = "symbols"`, `lto = "thin"`, `codegen-units = 1`.

- [ ] **Step 1: Write the failing tests** in `release_smoke.rs` using `env!("CARGO_BIN_EXE_speedyspec-server")`:
  - `smoke_accepts_the_server_at_its_version` → exit 0 with `env!("CARGO_PKG_VERSION")`.
  - `smoke_refuses_a_wrong_version` → exit 1, stderr names `version`.
  - `smoke_runtime_needs_no_app`: a fake runtime script printing `{"version":"1.0.0","contracts":[1],"php":"8.5.12"}` passes with `speedyspec-php-runtime 1.0.0`, and one printing `"contracts":[]` fails.
  (On Linux the server test binary is a debug build; the static check runs there too and must pass, since the musl build is static.)
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --features test-hooks --test release_smoke` Expected: FAIL.
- [ ] **Step 3: Implement** the script and the profile.
- [ ] **Step 4: Run** the tests Expected: PASS. Then `cargo build --release --locked -p speedyspec-server` and `scripts/release-smoke.sh target/release/speedyspec-server speedyspec-server 0.1.0` Expected: exit 0; record the stripped size in the ledger.
- [ ] **Step 5: Commit** `build: a release profile, and a smoke check for release binaries`.

### Task 3: `scripts/release-publish-plan.sh`

**Files:**
- Create: `scripts/release-publish-plan.sh`
- Modify: `crates/speedyspec-server-domain/tests/release_scripts.rs`

**Interfaces:**
- Produces: `scripts/release-publish-plan.sh <binary> <assets.json>` where `assets.json` is `gh release view <tag> --json assets`'s output. Prints one word: `complete` (every `<binary>-<triple>` for the three triples, each `.minisig`, and `SHA256SUMS` present), `empty` (none of those present), or `incomplete`. The workflow: `complete` → upload nothing; `empty` → upload; `incomplete` → `gh release delete-asset` each present one of those names, then upload. Other asset names are ignored.

- [ ] **Step 1: Write the failing tests**: `complete_release_is_left_alone` (all 7 names plus an unrelated `notes.txt` → `complete`), `no_assets_is_empty`, `one_missing_signature_is_incomplete`, `missing_sums_is_incomplete`, `another_binarys_assets_do_not_count` (the runtime's names for the server → `empty`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-domain --test release_scripts publish` Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** Expected: PASS.
- [ ] **Step 5: Commit** `ci(release): decide whether a release's assets are complete`.

### Task 4: `scripts/release-index.sh`

**Files:**
- Create: `scripts/release-index.sh`, fixtures under `crates/speedyspec-server-domain/tests/fixtures/releases/`
- Modify: `crates/speedyspec-server-domain/tests/release_scripts.rs`

**Interfaces:**
- Produces: `scripts/release-index.sh <component> <app-id>`, writing the index JSON on stdout. Releases come from `gh api --paginate "repos/$GITHUB_REPOSITORY/releases"` (concatenated pages merged with `jq -s add`), or from `$RELEASE_INDEX_RELEASES` (a file) in tests. A release's `SHA256SUMS` comes from `gh release download <tag> -p SHA256SUMS -O -`, or from `$RELEASE_INDEX_SUMS_DIR/<tag>.SHA256SUMS` in tests. Includes a release iff its tag is `<component>-v<semver>`, it is not a draft or pre-release, and every target has its binary, its `.minisig` and a line in `SHA256SUMS`; others are skipped with `release-index: skipping <tag>: <why>` on stderr. Entry: `version` (tag minus prefix), `published` (`published_at`), `artifacts` per target in the triple order above, `url`/`minisig` = the assets' `browser_download_url`, `sha256` lowercase. Releases sorted by version, newest first.

- [ ] **Step 1: Write the failing tests** (fixtures: a releases file with a complete server 0.1.0 and 0.2.0, a complete runtime 1.0.0, a draft, a pre-release, an incomplete server 0.3.0 with no aarch64 signature, and an unrelated tag `v9.9.9`):
  - `index_lists_complete_releases_and_parses`: output passes `ReleaseIndex::parse`; `app == "server"`; versions `[0.2.0, 0.1.0]`; each has three artifacts with the fixture URLs and checksums.
  - `index_skips_incomplete_drafts_prereleases_and_other_components`: 0.3.0, the draft, the pre-release, the runtime and `v9.9.9` are absent; stderr names 0.3.0's missing signature.
  - `runtime_index_is_separate`: `speedyspec-php-runtime php-runtime` → only 1.0.0, `app == "php-runtime"`.
  - `paged_releases_are_merged`: the releases file as two concatenated JSON arrays (as `--paginate` prints) → both pages' releases listed.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-domain --test release_scripts index` Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** Expected: PASS; also run the test file in the Linux container when Docker is available (GNU tools), else ledger that CI's test job covers it.
- [ ] **Step 5: Commit** `ci(release): build a component's release index from its GitHub releases`.

### Task 5: release-please configuration

**Files:**
- Create: `release-please-config.json`, `.release-please-manifest.json`, `CHANGELOG.md` files are left to release-please
- Modify: none

**Interfaces:**
- `release-please-config.json`: `pull-request-title-pattern: "chore: release ${component} ${version}"`, `separate-pull-requests: true`, `include-component-in-tag: true`, `plugins: [{"type": "cargo-workspace", "merge": false}]`, `bootstrap-sha`: the commit before this task's (the pipeline's base on `main`, filled in when the PR merges — use `origin/main`'s SHA at branch time), `changelog-sections` as `speedyspec-cli`'s, and packages:
  - `crates/speedyspec-server`: `release-type: rust`, `component: speedyspec-server`, `release-as: "0.1.0"`, `bump-minor-pre-major: true`, `additional-paths`: every `crates/speedyspec-server-*` directory, `crates/speedyspec-runtime-contract`, `crates/speedyspec-fastcgi`, `Cargo.toml`, `Cargo.lock`.
  - `crates/speedyspec-php-runtime`: `release-type: rust`, `component: speedyspec-php-runtime`, `release-as: "1.0.0"`, `additional-paths`: `crates/speedyspec-php`, `crates/speedyspec-php-ext`, `crates/speedyspec-php-port-macros`, `crates/speedyspec-container`, `crates/speedyspec-container-php`, `crates/speedyspec-cqrs`, `crates/speedyspec-cqrs-php`, `crates/speedyspec-hooks`, `crates/speedyspec-hooks-memory`, `crates/speedyspec-wp-hooks`, `crates/speedyspec-wp-hooks-memory`, `crates/speedyspec-wp-hooks-php`, `crates/speedyspec-compat`, `crates/speedyspec-runtime-contract`, `crates/speedyspec-fastcgi`, `php-version`, `scripts/build-php.sh`, `Cargo.toml`, `Cargo.lock`.
- `.release-please-manifest.json`: `{"crates/speedyspec-server": "0.0.0", "crates/speedyspec-php-runtime": "0.0.0"}`.

- [ ] **Step 1: Write the failing test** `release_config_covers_every_dependency` in `release_scripts.rs`: parse `cargo metadata --format-version 1 --no-deps` and the config; every workspace path dependency (transitively) of each package is in that package's path or `additional-paths`; the manifest has both packages at `0.0.0`; `release-pending.sh` says `pending=false` for both against the real repo.
- [ ] **Step 2: Run** Expected: FAIL (config missing).
- [ ] **Step 3: Write** both files.
- [ ] **Step 4: Run** Expected: PASS. Validate the config against release-please's schema: `npx --yes release-please@16 release-pr --dry-run --token "$(gh auth token)" --repo-url SpeedySpec/speedyspec-server --target-branch main --config-file release-please-config.json --manifest-file .release-please-manifest.json` Expected: two proposed PRs, server 0.1.0 and runtime 1.0.0, each with a `Cargo.lock` change. (Read-only: `--dry-run` opens nothing.)
- [ ] **Step 5: Commit** `ci(release): release-please for the server and the PHP runtime`.

### Task 6: Workflows

**Files:**
- Create: `.github/workflows/release-build.yml`, `.github/workflows/release.yml`

**Interfaces:**
- Consumes: Tasks 1–4's scripts.
- `release-build.yml` (`on: workflow_call`, inputs `ref`, `component`, `version`; `permissions: contents: read`): matrix of the three rows (`triple`, `os`, `run`, `platform`, `php-build`). Steps: checkout `ref`; Rust 1.99 (macOS); static PHP cache and `scripts/build-php.sh` exactly as `ci.yml`'s php job (same cache key); tests — server: `cargo build -p speedyspec-server --features test-hooks`, `scripts/check-no-php-in-server.sh`, `cargo test --workspace --features speedyspec-server/test-hooks`; runtime: `cargo test -p speedyspec-php --test extensions`, `cargo test --workspace --features speedyspec-server/test-hooks`; `cargo build --release --locked -p <component> --target <triple>` (through `run` on Linux); `scripts/release-smoke.sh <path> <component> <version>`; copy to `dist/<component>-<triple>`; `upload-artifact` name `<component>-<triple>`, `if-no-files-found: error`, retention 7 days.
- `release.yml` (`on: push: branches [main]`, `workflow_dispatch` input `tag`; `concurrency: {group: release, cancel-in-progress: false}`; top-level `permissions: contents: read`):
  1. `pending` (push): `git fetch --tags`; `release-pending.sh` for each component → outputs `server`, `server_version`, `runtime`, `runtime_version`. On dispatch, a `resolve` job maps the tag to component and version instead.
  2. `build-server`, `build-runtime`: `uses: ./.github/workflows/release-build.yml` when that component is pending or dispatched.
  3. `release-please` (push, only if every pending build succeeded or was skipped): `permissions: contents: write, pull-requests: write`.
  4. `publish` (per built component, after release-please or on dispatch; `permissions: contents: write`): download artifacts; require exactly three files; fail if `MINISIGN_SECRET_KEY` is empty or `minisign.pub` is missing; install minisign; write the key to `$RUNNER_TEMP/minisign.key` (umask 077) and delete it in an `always()` step; `minisign -S -s <key> -m <file> -t "<component> <version> <triple>"` with the password on stdin; `minisign -V -p minisign.pub -m <file>` for each; write sorted `SHA256SUMS`; `gh release view <tag> --json assets > assets.json`; act on `release-publish-plan.sh` (complete: nothing; empty: `gh release upload`; incomplete: delete those assets, then upload).
  5. `pages` (after publish, or on dispatch with no build; `permissions: pages: write, id-token: write, contents: read`; `environment: github-pages`): `release-index.sh` for both components into `site/server/index.json` and `site/php-runtime/index.json`, a short `site/index.html` linking both and `minisign.pub` (copied to `site/minisign.pub`), `actions/configure-pages@v5`, `actions/upload-pages-artifact@v3` (`path: site`), `actions/deploy-pages@v4`.

- [ ] **Step 1: Write** both workflows.
- [ ] **Step 2: Lint** `docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.7 -color` Expected: no findings. (If Docker is unavailable, `brew install actionlint` is not allowed without asking; ledger that and lint in CI by adding the same command as a step to `ci.yml`'s `test` job on `ubuntu-latest`.)
- [ ] **Step 3: Commit** `ci(release): build, sign and publish releases, and deploy the indexes`.

### Task 7: Documentation

**Files:**
- Modify: `README.md`, `docs/php.md` (the runtime's `source`)

- [ ] **Step 1:** README example config: `[app.server]` and `[app.php-runtime]` `source` set to the two index URLs and `keys` to the contents of `minisign.pub` (a placeholder line `RW…` until Jacob adds the key, with a note). "Releasing" section: merge the component's release pull request; its binaries and both indexes follow; re-run with *Run workflow* and the tag (fills in an incomplete release, never replaces a complete one); first-time setup (minisign key pair, the two secrets, `minisign.pub`, Pages from Actions); remove `release-as` from both packages after the first two releases. "Verifying a download": `minisign -Vm <file> -p minisign.pub` and `sha256sum -c`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --features test-hooks --test serve readme` Expected: PASS (the README's config still parses).
- [ ] **Step 3: Commit** `docs: releasing, and the published sources and key`.
