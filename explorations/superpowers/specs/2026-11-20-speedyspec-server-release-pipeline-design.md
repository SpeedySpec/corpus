# SpeedySpec server and PHP runtime release pipeline design

**Repository:** SpeedySpec/speedyspec-server (public). **Date:** 2026-11-20.

## Intent

Managed mode downloads, verifies, keeps, switches and rolls back the server's own binary and the PHP runtime's binary.
Until now nothing publishes them, so managed mode has nothing to install. This pipeline builds, tests, signs and
publishes both, and serves the release index each app's `source` points at.

The server and the runtime are released apart, with their own versions (today 0.1.0 and 1.0.0): the server should
change rarely, the runtime often.

## Goals

- Merging a release pull request publishes that component's release: one binary per target, its minisign signature,
  and a `SHA256SUMS` file, on a GitHub release.
- Each component's index is served by GitHub Pages at a URL that never changes, in the format managed mode already
  reads (core design, "Release artifacts"), and lists every published release of that component.
- A binary is published only after it passes the same tests as CI on its own target.
- Nothing is published if any target fails to build or test, or signing fails.
- A release whose publishing failed part-way can be built and published again by hand from its tag.

## Non-goals

- Release pipelines of the admin site, admin API, front-end API and blog. Other repositories own them; they publish
  `.tar.zst` archives with their own indexes.
- Pre-release channels. Every published release is a full release (`prerelease = true` in a server's config would
  see none).
- macOS notarization, Windows, and container images.
- Changing the index format or the server's verification.

## Anti-goals

- A published index entry whose artifact is missing, unsigned, or fails its `sha256`.
- An index that only the pipeline's own history can rebuild: it must always be rebuildable from the GitHub releases
  alone.
- A release whose binary reports a different version than its tag.
- The private signing key in the repository, in logs, or in any artifact.

## Constraints

- Rust 1.99. Targets: `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl` (static) and `aarch64-apple-darwin`
  (development).
- TLS stays on `ring`.
- The runtime embeds the static PHP that `scripts/build-php.sh` builds; the server binary contains no PHP
  (`scripts/check-no-php-in-server.sh`).
- The server refuses an index whose `app` is not the configured app id, so the published ids fix the names operators
  use: `server` and `php-runtime`, as the README's example config already names them.
- Conventional Commits drive versions and changelogs.

## Approaches

- **A. release-please with two components, Pages deployed by Actions (chosen by Jacob).** release-please keeps one
  release pull request per component. Merging one builds, tests, signs and publishes that component, then rebuilds
  both indexes from every published GitHub release and deploys them to Pages. Familiar from `speedyspec-cli`; the
  index is always rebuildable from the releases.
- **B. The same builds, with the index committed to a `gh-pages` branch.** Each publish appends to the index in git.
  Rejected: the index can drift from the releases (a deleted or re-uploaded release), and the workflow needs push
  access to a branch.
- **C. Hand-pushed tags, no release-please.** Simplest, but no changelogs and nothing ties the tag to `Cargo.toml`.
  Rejected.

## Patterns

- **Pipes and Filters** for the release: build → test → smoke → sign → publish → index, each stage consuming the last
  one's artifacts and stopping the pipeline on failure.
- **Template Method** in a reusable workflow: one `release-build.yml` holds the per-target steps; each component fills
  in its package, binary name and whether it needs the static PHP.
- **Materialized View** for the indexes: derived data, rebuilt in full from the GitHub releases (the source of truth)
  on every publish, never edited in place.

## Design

### Versions and tags

- `release-please-config.json` registers **every workspace crate** as a package (Jacob, 2026-11-20; release-please has
  no way to attribute other paths' commits to a package, so the design's `additional-paths` does not exist):
  - `crates/speedyspec-server` and `crates/speedyspec-php-runtime` publish GitHub releases, tags and changelogs;
  - every other crate has `skip-github-release` and `skip-changelog`. After a release, the workflow tags each one at
    its released version (`<crate>-v<version>`, a plain tag), because release-please finds where a package was last
    released only from a GitHub release or a tag;
  - the `cargo-workspace` plugin bumps the server or the runtime whenever a crate it is built from is bumped, and
    updates the root `Cargo.lock`. It merges everything into **one release pull request**;
  - files outside the crates (`php-version`, `scripts/build-php.sh`) release nothing by themselves: a PHP update is
    committed together with a change to the runtime crate.
- Tags: `speedyspec-server-v<version>` and `speedyspec-php-runtime-v<version>`.
- The lock file is the workspace's root `Cargo.lock`, which the `rust` type alone does not update: the config adds
  release-please's `cargo-workspace` plugin, so a release pull request bumps the crate's `Cargo.toml` and the root
  `Cargo.lock` together and `cargo build --locked` passes at the release commit. The binary's `CARGO_PKG_VERSION`,
  which is what `manifest` prints and managed mode reads, always matches the tag.
- **The first releases** (my calls) are the crates' current versions, server 0.1.0 and runtime 1.0.0, published
  through release-please like every later one:
  - `.release-please-manifest.json` starts at `0.0.0` for both, a sentinel meaning "never released";
  - no `release-as` (final review): it applies only to a package with new commits, and left in place it proposes
    the same version again. The server's first release is release-please's first Rust version, 0.1.0; the runtime's
    1.0.0 is asked for once with a `Release-As: 1.0.0` footer on a commit that touches only its crate. The config
    sets `bootstrap-sha` to the commit the pipeline branches from, so the first changelog does not replay history;
  - a crate no component is built from (`speedyspec-cqrs-php`) is not a package: a library-only release pull
    request would never be marked released and would stall release-please;
  - a component is **pending** only when its `Cargo.toml` version equals its `.release-please-manifest.json`
    version and has no tag. A merged release pull request updates both together; before the first one, the
    manifest's `0.0.0` never equals the crate's version, so ordinary pushes build nothing.

### Workflows

- **`release.yml`**, on push to `main` and `workflow_dispatch` (input: an existing tag). `concurrency: release`, not
  cancelled.
  1. **pending**: compares each package's `Cargo.toml` version with the existing tags and outputs which components
     have a version with no tag yet (the merged release pull request), as `speedyspec-cli`'s `release-pending.sh`
     does.
  2. **build-server** / **build-runtime**: for each pending component (or the dispatched tag's component), the
     reusable workflow at that commit. They run in parallel.
  3. **release-please**: only when every pending component built. It creates the tags and GitHub releases with their
     changelogs (and keeps the release pull requests open on other pushes).
  4. **publish**: downloads the build artifacts, signs each binary, writes `SHA256SUMS`, and uploads binaries,
     `.minisig` files and `SHA256SUMS` to the release.
     **A re-run never replaces a published binary** (my call): builds are not reproducible, so a new binary under
     an old URL would fail every server's checksum, and a server records a version that fails verification as
     failed for good. A dispatch for a release whose assets are complete (every target's binary, signature and
     `SHA256SUMS`) changes nothing and only redeploys the indexes. One with assets missing deletes that release's
     binaries, signatures and `SHA256SUMS` and uploads a full new set; that is safe because an incomplete release is
     never in the index (below), so no server can have downloaded it. A bad published release is fixed by the next
     version, not by replacing it.
  5. **pages**: rebuilds both indexes and deploys them (below). It also runs on its own by dispatch, to rebuild the
     indexes without a release.
- **`release-build.yml`** (`workflow_call`; inputs `ref`, `package`, `binary`, `php: bool`). Matrix of the three
  targets: `ubuntu-latest` through `scripts/linux-php.sh` (`linux/amd64`), `ubuntu-24.04-arm` (`linux/arm64`),
  `macos-15`. Per target:
  - checks out `ref`; with `php`, builds the static PHP with the same cache key as CI's php job;
  - runs the tests CI runs for that component on that target (the server: its build without PHP,
    `check-no-php-in-server.sh` and the PHP-backed suite; the runtime: the extensions test and the PHP-backed suite);
  - `cargo build --release --locked -p <package> --target <triple>`;
  - **smoke**: `<binary> manifest` parses, and its `app` and `version` match the component's id and the version
    being released; on Linux, `file` reports a static executable;
  - uploads `<binary>-<triple>` as an artifact (`if-no-files-found: error`, retention 7 days).
- **Release profile** (my call): `[profile.release] strip = "symbols"`, `lto = "thin"`, `codegen-units = 1`.

### Assets

For each release, on its GitHub release:

- `<binary>-<triple>` for each target, for example `speedyspec-server-x86_64-unknown-linux-musl`;
- `<binary>-<triple>.minisig`;
- `SHA256SUMS`, sorted, in `sha256sum` format.

### Signing

- `minisign -S` with the secret key from the repository secret `MINISIGN_SECRET_KEY` and its password from
  `MINISIGN_PASSWORD`, written to a file only for the signing step (mode 0600) and deleted after it. The trusted
  comment names the binary and version.
- The public key is committed as `minisign.pub` and printed in the README. It goes in `server.toml`'s `keys` for
  `server` and `php-runtime`.
- Jacob generates the key pair and adds both secrets. The pipeline fails closed without them: nothing is published.
- After signing, the publish job checks every signature against `minisign.pub` before uploading.

### Indexes on Pages

- `scripts/release-index.sh <component> <app-id>` lists the repository's GitHub releases whose tag starts with
  `<component>-v` (with `gh api`, paginated), skips drafts and pre-releases, and writes:
  ```json
  {
    "app": "server",
    "releases": [
      {
        "version": "0.2.0",
        "published": "<the release's published_at>",
        "artifacts": [
          {
            "target": "x86_64-unknown-linux-musl",
            "url": "https://github.com/SpeedySpec/speedyspec-server/releases/download/speedyspec-server-v0.2.0/speedyspec-server-x86_64-unknown-linux-musl",
            "sha256": "<from the release's SHA256SUMS>",
            "minisig": "<the .minisig asset's URL>"
          }
        ]
      }
    ]
  }
  ```
  A release missing any target's binary, signature or checksum is left out of the index, with a warning in the job
  log (it is incomplete: a failed upload, or one in progress).
- The pages job writes `site/server/index.json` and `site/php-runtime/index.json`, plus a short `site/index.html`
  linking both and the public key, and deploys with `actions/upload-pages-artifact` and `actions/deploy-pages`
  (`permissions: pages: write, id-token: write`).
- URLs: `https://releases.speedyspec.com/server/index.json` and
  `https://releases.speedyspec.com/php-runtime/index.json`. Pages must be set to deploy from GitHub
  Actions (a one-time repository setting, Jacob's to change).

### Documentation

- README: the two `source` URLs and the public key in the example config, a "Releasing" section (merge the release
  pull request; binaries and index follow; re-run with the tag), and how to verify a download by hand.

## Testing

- `scripts/release-index.sh` against a recorded `gh api` response (a fixture directory): complete releases are
  listed with the right URLs, checksums and signature URLs; drafts, pre-releases, other components' tags and
  incomplete releases are left out; the output parses with the server's own `ReleaseIndex::parse` (a Rust test that
  runs the script).
- `scripts/release-pending.sh` against fixture tags and manifests: a component is pending only when its crate
  version equals its manifest version and has no tag; the `0.0.0` sentinel is never pending.
- The publish job's re-run rule against fixture releases: a complete release is left alone; an incomplete one has
  its assets replaced as a set.
- The smoke step's checks run locally against a release build on macOS and in the Linux container.
- actionlint on every workflow.
- End to end, once: after the first release, a managed server configured with the published `source` URLs and
  `minisign.pub` installs, verifies and switches to both, and `release list` shows them.

## Dependencies

- A minisign key pair (Jacob), the two secrets, and Pages set to deploy from Actions.
- `minisign` on the publish runner (installed in the job).

## Left to the builder

- Exact step names, artifact names inside the workflow, and the fixture format for `gh api` responses.
- Whether `release-pending.sh` is shared with or copied from `speedyspec-cli`.
- The wording of the Pages `index.html`.
