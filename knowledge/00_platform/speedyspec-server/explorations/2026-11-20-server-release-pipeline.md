# Exploration: Server and PHP runtime release pipeline

**Status:** built
**Created:** 2026-11-20
**Author:** Claude, for the project owner
**Context:** How the server and PHP runtime binaries get built, tested, signed, published and listed for managed mode.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

Managed mode can download, verify, keep, switch and roll back the server's own binary and the PHP runtime's binary.
But nothing published those binaries, so managed mode had nothing to install. Operators also had no stable URL for
each component's release index, and no signed artifacts to check against a public key. The server and the runtime
need separate versions, because the server should change rarely and the runtime often.

## Current State

- SpeedySpec/speedyspec-server already had CI (`ci.yml`) that built the static PHP with `scripts/build-php.sh` and
  ran the tests on Linux and macOS.
- The server already read a release index format and verified artifacts by `sha256` and minisign signature. The
  server refuses an index whose `app` does not match the configured app id.
- The crates sat at server 0.1.0 and runtime 1.0.0, never released.
- SpeedySpec's CLI repository already released with release-please and a `release-pending.sh` script, which served as
  prior art.

## Proposed Solution

A merged release pull request builds each bumped component on three targets, runs CI's tests there, smoke-checks the
binary, then tags, signs and publishes it. The indexes are rebuilt from the GitHub releases and deployed to Pages.

### Architecture

```
push to main
   |
   v
pending ----- which component has a version with no tag yet?
   |
   +--> build-server ----+    (reusable release-build.yml, per target:
   +--> build-runtime ---+     test -> build --locked -> smoke -> upload)
                         |
                         v
                  release-please ---- tags + GitHub releases + changelogs
                         |
                         v
                      publish ------- minisign, verify, SHA256SUMS, upload
                         |
                         v
                       pages -------- rebuild both indexes from releases, deploy
```

- Targets: `x86_64-unknown-linux-musl` and `aarch64-unknown-linux-musl` (static), and `aarch64-apple-darwin`
  (development).
- Each decision lives in a small bash script, tested from Rust against fixtures: `release-pending.sh`,
  `release-smoke.sh`, `release-publish-plan.sh` and `release-index.sh`. The workflows stay thin.
- `workflow_dispatch` with a tag rebuilds and republishes one release by hand, or only redeploys the indexes.

### Data Model

Each component's index on Pages:

```json
{ "app": "server",
  "releases": [ { "version": "0.2.0", "published": "...",
                  "artifacts": [ { "target": "...", "url": "...", "sha256": "...", "minisig": "..." } ] } ] }
```

Each GitHub release carries `<binary>-<triple>`, `<binary>-<triple>.minisig` per target, and one sorted
`SHA256SUMS`. Tags are `speedyspec-server-v<version>` and `speedyspec-php-runtime-v<version>`. The app ids are
`server` and `php-runtime`, served at `releases.speedyspec.com/server/index.json` and
`releases.speedyspec.com/php-runtime/index.json`.

### Interfaces

- `release-pending.sh <component>` prints `pending=`, `tag=` and `version=`.
- `release-smoke.sh <binary-path> <component> <version>` checks the binary's `manifest` output.
- `release-publish-plan.sh <binary> <assets.json>` prints `complete`, `empty` or `incomplete`.
- `release-index.sh <component> <app-id>` writes the index JSON.

## Design Decisions

1. **release-please with two released components, and Pages deployed by Actions.** Chosen by the project owner.
   Familiar from the CLI, gives changelogs, and ties tags to `Cargo.toml`.
   *Trade-off:* release-please's model of packages and paths has to be bent to fit a Cargo workspace (decision 2).
2. **Every workspace crate is a release-please package.** The plan listed extra paths per component, but
   release-please has no way to credit other paths' commits to a package. The project owner chose to register every
   crate. Only the server and runtime get GitHub releases; library crates get plain `<crate>-v<version>` tags after
   each release, so release-please knows where they last released. The `cargo-workspace` plugin bumps a component
   when a crate it is built from is bumped, and updates the root `Cargo.lock`, in one release pull request.
   *Trade-off:* many plain tags, and a crate no component is built from must be left out, or a library-only release
   pull request would stall release-please.
3. **The index is a Materialized View, rebuilt in full from the GitHub releases on every publish.** It can never drift
   from the releases.
   *Trade-off:* every publish reads every release (paged, since there can be more than 30).
4. **An incomplete release never appears in an index.** A release missing any target's binary, signature or checksum
   line is skipped with a warning.
   *Trade-off:* a part-failed release is invisible until fixed by hand.
5. **A complete published release is never changed.** Builds are not reproducible, so a new binary under an old URL
   would fail every server's checksum, and a server marks a version that fails verification as failed for good. An
   incomplete release has its assets replaced as a whole set, which is safe because no server could have seen it.
   The designer's call.
   *Trade-off:* a bad release is fixed only by the next version.
6. **Nothing publishes unless every target builds, tests and smoke-checks.** The binary's `manifest` must report the
   version being released, and on Linux it must be static. The runtime's manifest has no `app` field, so its check
   asks for a non-empty `contracts` list instead.
   *Trade-off:* one flaky test on one target blocks the whole release.
7. **Signing fails closed.** The minisign key comes from a secret, lives in a 0600 file only for the signing step and
   is deleted after. Every signature is checked against the committed `minisign.pub` before upload.
   *Trade-off:* without the secrets nothing publishes at all.
8. **A stripped, thin-LTO, single-codegen-unit release profile.** The designer's call. The server came out at 14 MB
   stripped on macOS.
   *Trade-off:* slower release builds.
9. **Release builds test only the crates a component is built from.** Added in
   [SpeedySpec/speedyspec-server#24](https://github.com/SpeedySpec/speedyspec-server/pull/24) after a flaky test in a
   crate neither component uses blocked the first release. CI still tests the whole workspace.
   *Trade-off:* a release no longer re-proves the whole workspace on each target.
10. **release-please and the library-tag push use a `RELEASE_TOKEN` secret with the workflows permission.** Added in
    [SpeedySpec/speedyspec-server#27](https://github.com/SpeedySpec/speedyspec-server/pull/27); see Spike Findings.
    It falls back to `GITHUB_TOKEN` when unset.
    *Trade-off:* a long-lived token must be made and kept by hand.

## Implementation Phases

1. **Pending check.** `release-pending.sh` and its tests. *Delivers:* builds only for a merged, untagged release.
2. **Release profile and smoke check.** *Delivers:* binaries that report the right version and are static on Linux.
3. **Publish plan.** *Delivers:* the complete, empty or incomplete decision for re-runs.
4. **Index script.** *Delivers:* indexes rebuilt from GitHub releases, paged.
5. **release-please configuration.** *Delivers:* one release pull request that bumps components and the lock file.
6. **Workflows.** `release-build.yml` and `release.yml`. *Delivers:* the whole pipeline on push and by dispatch.
7. **Documentation.** *Delivers:* README sections on published sources, the public key, releasing and verifying.

Phases 1 to 7 shipped together in
[SpeedySpec/speedyspec-server#11](https://github.com/SpeedySpec/speedyspec-server/pull/11), merged 2026-11-21.
Follow-ups: #24 (narrower release tests, and the release build reads its scripts from the run's own commit) and #27
(`RELEASE_TOKEN`). The first release pull request,
[SpeedySpec/speedyspec-server#21](https://github.com/SpeedySpec/speedyspec-server/pull/21), proposed server 0.1.0 and
runtime 1.0.1. After #27 and the secret, the release run on 2026-11-21 published `speedyspec-server-v0.1.0` and
`speedyspec-php-runtime-v1.0.1`, each with three binaries, three signatures and `SHA256SUMS`, and the server index on
Pages lists 0.1.0.

## Fleet Impact

- **Server and PHP runtime:** now released as signed binaries with their own versions. Confirmed in
  SpeedySpec/speedyspec-server#11.
- **Managed-mode servers:** can point `source` at the two Pages indexes and `keys` at `minisign.pub`. The README's
  example config uses those URLs, and the server index is live. Not yet checked end to end with a managed
  server.
- **Admin site, admin API, front-end API, blog:** unchanged. Their own repositories publish `.tar.zst` archives with
  their own indexes.
- **CLI:** unchanged; it was only prior art.

## Open Questions

1. **Does a managed server install, verify and switch to both published components?** Settled by the end-to-end
   check the spec calls for, now that the first releases are published.
2. **Should `release-pending.sh` be shared with the CLI or kept as a copy?** Left to the builder.
3. **What about known flaky tests still inside the components' crates?** #24 names two; a failure means re-running
   the failed jobs. Settled by fixing or isolating those tests.
4. **Pre-release channels, macOS notarization, Windows and container images.** Out of scope for now.

## Spike Findings

- **Settles:** why release-please was refused with "Resource not accessible by integration" when creating the
  releases from #21, even though its token had `contents: write`.
- **Built:** a throwaway workflow on a probe branch (since deleted) that used `GITHUB_TOKEN` to create one tag at the
  old release commit and one at main's head.
- **Established:** the tag at the old commit was refused with HTTP 403; the tag at main's head was created, then
  deleted. GitHub refuses `GITHUB_TOKEN` a tag at a commit whose workflow files differ from main's, because that
  needs the workflows permission, which `GITHUB_TOKEN` can't have. The first release's builds had failed, so the
  retry tagged a commit that #23 and #24 had since left behind in `ci.yml` and `release-build.yml`.
- **Does not establish:** whether other ref types, or other kinds of workflow-file difference, behave the same.
- **Consequence:** [SpeedySpec/speedyspec-server#27](https://github.com/SpeedySpec/speedyspec-server/pull/27) moved
  release-please and the library-tag push to a `RELEASE_TOKEN` secret with the workflows permission. Without it,
  releases still work whenever the release commit is main's head. Once #27 merged and `RELEASE_TOKEN` was added as
  an organisation secret, the Release run for `b6534ef` on 2026-11-21 passed every job. release-please created the
  two releases #21 prepared (server v0.1.0 and runtime v1.0.1), and the publish and pages jobs passed.

## Success Criteria

- [x] A push with no merged release pull request builds nothing, including before the first release. Checked by the
  `release-pending.sh` tests.
- [x] A partly uploaded release never appears in an index. Checked by the `release-index.sh` fixture tests.
- [x] A dispatch for a complete release changes no asset. Checked by the `release-publish-plan.sh` tests.
- [x] The index pages through more than 30 releases. Checked by `paged_releases_are_merged`.
- [x] The runtime's smoke check needs no `app` field. Checked by `smoke_runtime_needs_no_app`.
- [x] The scripts work with BSD and GNU tools. Checked on macOS and in a Linux arm64 container (#11).
- [x] Every workflow passes actionlint. Checked in #11.
- [x] Merging the release pull request publishes signed binaries for all three targets. Checked on the GitHub
  releases for server 0.1.0 and runtime 1.0.1, and in the Pages server index, on 2026-11-21.
- [x] release-please can tag a release whose commit is behind main's. Checked by the Release run for `b6534ef` on
  2026-11-21, which tagged #21's commit with `RELEASE_TOKEN` and passed every job.
- [ ] A managed server installs and switches to both published components. Not yet run.

---

## Non-Goals

- **Release pipelines for the admin site, admin API, front-end API and blog** — their own repositories own them.
- **Pre-release channels** — every published release is a full release.
- **macOS notarization, Windows and container images** — not needed yet.
- **Changing the index format or the server's verification** — the pipeline fits what managed mode already reads.

## Alternatives Considered

### Option: release-please with Pages deployed by Actions
- **Pros:** Changelogs, versions tied to `Cargo.toml`, index always rebuildable from releases, familiar from the CLI.
- **Cons:** Needs workarounds for workspace crates and, as it turned out, a token with the workflows permission.
- **Verdict:** Chosen by the project owner.

### Option: Same builds, index committed to a `gh-pages` branch
- **Pros:** No Pages deployment step.
- **Cons:** The index can drift from the releases after a delete or re-upload, and the workflow needs push access.
- **Verdict:** Rejected.

### Option: Hand-pushed tags, no release-please
- **Pros:** Simplest.
- **Cons:** No changelogs, and nothing ties the tag to `Cargo.toml`.
- **Verdict:** Rejected.

## Dependencies

- A minisign key pair made by the project owner, with the `MINISIGN_SECRET_KEY` and `MINISIGN_PASSWORD` secrets.
- Pages set to deploy from GitHub Actions, and Actions allowed to create pull requests.
- The `RELEASE_TOKEN` secret, added by hand as an organisation secret after #27.
- CI's static PHP build and its cache key, reused by the release builds.

## References

- [SpeedySpec/speedyspec-server#11](https://github.com/SpeedySpec/speedyspec-server/pull/11), the pipeline
- [SpeedySpec/speedyspec-server#21](https://github.com/SpeedySpec/speedyspec-server/pull/21), the first release pull
  request
- [SpeedySpec/speedyspec-server#24](https://github.com/SpeedySpec/speedyspec-server/pull/24), narrower release tests
- [SpeedySpec/speedyspec-server#27](https://github.com/SpeedySpec/speedyspec-server/pull/27), `RELEASE_TOKEN`
- [Core server design exploration](2026-11-17-speedyspec-server-core.md)
- Source spec: `2026-11-20-speedyspec-server-release-pipeline-design.md`
- Source plan: `2026-11-20-speedyspec-server-release-pipeline.md`
