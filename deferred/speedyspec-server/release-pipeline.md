# Release pipeline

`.github/workflows/release.yml`, `release-build.yml`, `scripts/release-*.sh`, `release-please-config.json`,
`README.md`. From the release pipeline plan (#11), and findings recorded 2026-11-21.

## One failed publish leg skips the Pages redeploy for the leg that succeeded

- **Where:** `release.yml`, the `pages` job's `if:`.
- **What happens:** the publish matrix runs with `fail-fast: false`, but `pages` runs only when
  `needs.publish.result != 'failure'`. If the runtime's publish fails and the server's succeeds, the server's new
  release is not in the index until someone runs the workflow by hand.
- **Suggested fix:** run `pages` unless publish was cancelled. The index already leaves out incomplete releases.

## A failed library-tag push fails release-please and skips publishing

- **Where:** `release.yml`, the `release-please` job's "Tag the library crates" step, and `publish`'s `if:`.
- **What happens:** the library tags are pushed in the same job as release-please. If that push fails, the job fails,
  and `publish` (which needs release-please to succeed on a push) does not run, though the GitHub release was already
  created. The release is left without binaries until a manual dispatch.
- **Suggested fix:** move the library tagging to its own job, or let `publish` run when the release itself was
  created.

## Release assets are counted by name only

- **Where:** `scripts/release-publish-plan.sh`, `scripts/release-index.sh`.
- **What happens:** both scripts decide an asset exists by its name. GitHub lists an upload that did not finish with
  a state other than `uploaded`, so a broken asset can make a release look complete, and the index can link to it.
- **Suggested fix:** also require `.state == "uploaded"`.

## Third-party actions are on mutable refs

- **Where:** `ci.yml`, `release.yml`, `release-build.yml`.
- **What happens:** actions are referenced by tag or branch, including `dtolnay/rust-toolchain@master` in the build
  jobs whose output is signed. A change upstream runs in the release build without review.
- **Suggested fix:** pin every third-party action to a commit SHA.

## The release build skips the aws-lc-sys check

- **Where:** `release-build.yml`, compared with `ci.yml`'s `test` job.
- **What happens:** CI checks that `aws-lc-sys` is not in the dependency tree, and builds the server where no static
  PHP has been built. The release build does neither: it builds PHP first, then builds and tests with it present. It
  does run `check-no-php-in-server.sh`.
- **Suggested fix:** add the `cargo tree -i aws-lc-sys` check, and a server build before PHP is built, to the server's
  release build.

## Changes to the release-script tests release the server

- **Where:** `crates/speedyspec-server-domain/tests/release_scripts.rs`; `release-please-config.json`
  (`cargo-workspace` plugin).
- **What happens:** the tests for the release scripts live in `speedyspec-server-domain`, which the server depends
  on. A commit that only changes those tests bumps `speedyspec-server-domain`, and through the workspace plugin, the
  server, so a CI-only change makes a server release.
- **Suggested fix:** move the release-script tests to a crate the server does not depend on (or one that is not a
  release-please package).

## gh api --paginate without per_page=100

- **Where:** `scripts/release-index.sh`.
- **What happens:** the release list is fetched 30 per page, so it takes about three times as many API calls as it
  needs. It works.
- **Suggested fix:** request `repos/$GITHUB_REPOSITORY/releases?per_page=100`.

## A dispatch for a complete release still runs the full build

- **Where:** `release.yml`, the `pending` job (dispatch branch) and `build-*`.
- **What happens:** running the workflow by hand with the tag of a release that is already complete builds and tests
  all three targets before the publish step finds there is nothing to do.
- **Suggested fix:** check the release's assets in `pending`, and skip the build when it is complete.

## release-please makes a GitHub release for any pending pull request, even when nothing was built

- **Where:** `release.yml`, the `release-please` job (`googleapis/release-please-action@v4` runs on every push to
  main).
- **What happens:** release-please creates a GitHub release for any merged pull request that still has the
  `autorelease: pending` label, on any push, whether or not this run built anything. On 2026-11-21 it created an empty
  release `speedyspec-server-v0.1.1`, tagged at 31bf856 (#28, which had been reverted in #30). Release pull request
  #31 then merged with the same version: release-please logged "Duplicate release tag", failed on `already_exists`,
  and still relabelled #31 `autorelease: tagged`. The 0.1.1 release has no assets while that tag stays.
- **Suggested fix:** pass `skip-github-release: true` to the action when `pending` found nothing to release. Also
  change the README's recovery steps (see below).

## The README's recovery steps and token notes are out of date

- **Where:** `README.md`, the release section.
- **What happens:**
  - the recovery steps say to revert the release pull request's commit, then remove its `autorelease: pending`
    label. If the revert is merged first, its push lets release-please create the release for the reverted pull
    request (as happened with #28). The label must be cleared before the revert is merged.
  - it says release pull requests are opened with the workflow's own token, so CI does not run on them. Since #27,
    release-please uses `RELEASE_TOKEN` when it is set, and CI ran on release pull request #31.
- **Suggested fix:** reorder the recovery steps (clear the label, then merge the revert), and describe the token as
  it is now.
