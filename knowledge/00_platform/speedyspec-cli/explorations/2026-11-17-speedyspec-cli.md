# Exploration: The speedyspec CLI, an installer with embedded PHP plugins

**Status:** built
**Created:** 2026-11-17
**Author:** Claude, for the project owner
**Context:** One self-contained `speedyspec` binary that delivers an application through drivers, with the "what" supplied by PHP plugins.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

People needed one download that could install SpeedySpec or WordPress onto whatever they had: a bare server, a
directory, or a Kubernetes cluster. A fresh machine may have no PHP, Composer, Helm or kubectl. The installer also had
to stay generic. If shipping a new application meant a new CLI release, the tool would not scale. Plugin authors
needed a way to extend the CLI (commands, install options, steps before and after an install) without a system PHP.

## Current State

Nothing existed. The work started two new public repositories, `SpeedySpec/speedyspec-cli` (a Cargo workspace) and
`SpeedySpec/speedyspec-cli-contracts` (the Composer package `speedyspec/cli-contracts`). They follow the release-please
pattern of the existing `speedyspec-*` repositories. The foundation (plan 1) was committed straight to `main`, starting
at commit `848a2fe` ("prove static PHP embed on macOS and Linux"). Later plans landed as pull requests.

## Proposed Solution

### Architecture

```
  user ──> speedyspec (Rust binary, PHP 8.5 linked in statically)
              │
              ├─ boot: pre-scan flags ─> load plugins (manifest + contract range) ─> build command tree
              │
              ├─ hook registry (Rust trait; memory backend) <── SpeedySpec\Hooks\Hook (PHP class)
              │        ▲ plugins answer filters with contract objects
              │
              ├─ Driver trait ──> zip driver  (fetch, verify, stage, place)
              │               └─> kubernetes driver (server-side apply, record Secret, wait)
              │
              └─ self update / self plugin  (GitHub releases, SHA256SUMS)
```

- The CLI owns the mechanics (download, unpack, apply to a cluster). Plugins only say what to deliver.
- Contracts are final readonly PHP value classes in their own package. The binary bundles one pinned release and
  checks every plugin answer against it in Rust.
- Hooks fired: `speedyspec_init`, `speedyspec_apps`, `speedyspec_commands`, `speedyspec_install_options`,
  `speedyspec_install_before`, `speedyspec_driver_zip`, `speedyspec_driver_kubernetes`, `speedyspec_install_after`.

### Data Model

- `App` (id, label, parts, drivers), `AppPart`, `Command` and its arguments and options, `InstallOption`,
  `InstallRequest`.
- `ZipSource`: `url`, `destination`, optional `sha256`, optional `stripComponents`.
- `KubernetesResources`: an ordered list of complete manifests, plus an optional `waitFor` list.
- Plugin manifest `speedyspec-plugin.json`: name, version, contract range, entry file.
- Kubernetes install record: a Secret `speedyspec-install-<name>` holding status, app, parts, plugin, created
  namespaces and the full resource list.

### Interfaces

- `speedyspec app list | install | uninstall`, `speedyspec self plugin list | install | remove`,
  `speedyspec self update [--check]`, `speedyspec php <file>`, `speedyspec --version`.
- Rust: a `Hooks` trait whose every method is fallible (so a broker backend can fail), a `Value` type with an
  `Opaque` variant that never leaves the process, and a `Driver` trait that stays synchronous.
- PHP: `Hook::add`, `remove`, `has`, `run`, `filter`, `ran`, with WordPress argument order.

## Design Decisions

1. **Link PHP statically into the binary** (static-php-cli plus ext-php-rs). Gives one file and real in-process hooks.
   *Trade-off:* a heavy build per platform, and Windows is out.
2. **Write the hook registry in Rust, expose it to PHP.** The project owner wanted compact storage and Rust code that
   can fire hooks without PHP. A hook with no PHP callback never enters PHP. *Trade-off:* the PHP bridge must keep
   object identity, fibers and errors right.
3. **Hooks are a contract with replaceable backends.** Only the memory backend ships. A broker backend (NATS, RabbitMQ)
   would carry the firing, never the callbacks. *Trade-off:* every call is fallible even when the memory backend can't fail.
4. **No global hook functions in PHP.** The first build had `add_action`-style aliases; the project owner had them
   removed on 2026-11-18 (PR #7) so loaded code, WordPress included, can declare its own. *Trade-off:* plugin authors
   cannot reuse WordPress habits by name.
5. **Talk to Kubernetes directly, no Helm.** The plugin returns full manifests and the CLI applies them with field
   manager `speedyspec`. *Trade-off:* existing charts cannot be reused.
6. **Never read the token from a flag.** It comes from an environment variable, a file, stdin or kubeconfig, so it
   stays out of shell history. *Trade-off:* one more step for the user.
7. **No rollback on a failed Kubernetes install.** The record is written first, so `app uninstall` can always clean
   up. *Trade-off:* a failed install leaves resources running until removed.
8. **Never load PHP from the working directory.** Running inside a cloned folder must not execute its code.
   *Trade-off:* plugin authors must name `--plugin-dir`.

## Implementation Phases

1. **Foundation (plan 1).** Static PHP proof, hook contract and memory backend, PHP bridge, contracts 1.0.0, plugin
   loading, the binary with plugin commands. *Delivers:* `--version`, `php`, `app list`, plugin commands. Shipped on
   `main` directly.
2. **Install flow and zip driver (plan 2).** Driver trait, safe fetch and extract, multi-source staging, prompts,
   plugin install and remove, WordPress example plugin. *Delivers:* `app install --driver zip`. Shipped in PR #1.
3. **Kubernetes driver (plan 3).** Connection, manifests, install record, readiness, uninstall, k3s end-to-end job.
   *Delivers:* `app install --driver kubernetes`, `app uninstall`. Shipped in PR #2.
4. **Self-update and releases (plan 4).** `speedyspec-fetch`, `speedyspec-selfupdate`, four-target builds,
   release-please, pinned contracts. *Delivers:* `self update` and published binaries. Shipped in PR #3, with fixes
   in PR #5; v1.0.0 released on 2026-11-18.
5. **WordPress files (plan 6).** Setup tasks after install, `plugin` and `theme` file commands, Composer packages
   for CLI plugins installed in Rust, two WordPress plugin repositories. *Delivers:* a WordPress install that ends
   with a `wp-config.php` php-fpm can serve. Shipped in PR #12. This plan follows the later app-maintenance design.

## Fleet Impact

- `SpeedySpec/speedyspec-cli`: the binary, released for Linux x86_64 and aarch64 and macOS x86_64 and arm64
  (confirmed: every release from v1.0.1 to v1.2.1 carries the four binaries and `SHA256SUMS`).
- `SpeedySpec/speedyspec-cli-contracts`: releases v1.0.0 through v1.5.0 (confirmed by its release list).
- `SpeedySpec/speedyspec-cli-wordpress` and `SpeedySpec/speedyspec-cli-wordpress-org`: created by plan 6 (confirmed
  to exist).
- The CLI never runs stock WordPress; it only downloads and prepares it for php-fpm.
- Effect on `speedyspec-server` and other apps: unverified. PR #11 copied server crates into the CLI and checks the
  copies for drift, so the two repositories now share code.

## Open Questions

1. **Broker hook backends** — not in any plan; would need a design for priority order across processes.
2. **Other drivers** ((S)FTP, Docker, Composer, Functions, AWS) — each needs its own contract and crate.
3. **Pruning on re-install, rollback, `app upgrade`, listing cluster installs** — deferred by plan 3; the install
   record is what they would build on.
4. **Signing and notarization** — plan 4 left Apple notarization, provenance and signature checks in `self update` for later.
5. **Carried-over minors** — PHP warnings do not name the plugin; writes to `php://stdout` bypass the redirect;
   in-process plugins can read `SPEEDYSPEC_KUBE_TOKEN` from the environment; uninstall does not warn about deleting
   CustomResourceDefinitions.
6. **Plan 9** (a SpeedySpec app plugin and activation) — waits for the WordPress Rust port. The CLI is complete
   without it.

## Spike Findings

**Settles:** whether PHP can be linked statically into a Rust binary and call Rust, on macOS arm64 and Linux x86_64.
**Built:** plan 1 Task 1, a test binary that boots PHP, runs a file and calls a Rust function.
**Established:** it works with PHP 8.5.12 (non-thread-safe), static-php-cli 2.8.6 and ext-php-rs 0.16.5. Linux builds
as a static-pie musl binary. The stock embed SAPI cannot add a module or ignore `php.ini`, so the CLI starts PHP with
its own minimal SAPI. Rust links with `-nodefaultlibs`, so the CPU-detection symbols PHP needs had to be linked by hand
(`libgcc` on Linux; `libclang_rt` on Intel macOS, found later in PR #5). No child-process fallback was needed.
**Does not establish:** Windows, or how PHP should serve web requests.
**Consequence:** the design stood. Later work moved PHP builds onto the server's shared recipe (PR #11). That recipe
needed a link flag fix before Linux x86_64 release builds worked again (PR #18).

A second, smaller gate in plan 3 confirmed that the `kube` client builds into the static binary; it added about 7 MB.

## Success Criteria

- [x] The binary runs with an empty environment and no system PHP — release smoke check runs `--version`.
- [x] Hook ordering, re-entrancy and filters pass a shared conformance suite — memory backend test.
- [x] A broken plugin is skipped and named; built-ins still work — CLI integration tests.
- [x] Zip installs refuse path escapes, symlink escapes and half-written results — driver tests; a reviewer's symlink-chain exploit was fixed before PR #1 merged.
- [x] Kubernetes install, re-install and uninstall work on a real cluster — k3s end-to-end tests in PR #2.
- [x] `self update` replaces a copy safely, including through a symlink and on bad downloads — loopback tests in PR #3.
- [x] Release binaries embed the pinned contracts — smoke check of the `--version` lines.
- [ ] Release binaries are built from the tagged commit — PR #20 fixed v1.1.0, which had been built from a newer commit; the next release run is still to confirm it.

---

## Non-Goals

- **Serving the installed app** — PHP for web requests belongs in killable worker processes, not the CLI's single engine.
- **The real installer plugins in the first build** — only fixtures and examples; the WordPress plugins came in plan 6.
- **Helm, Windows, plugin signing or registry** — out of scope for the first build.

## Alternatives Considered

### Option: PHP as a bundled child process
- **Pros:** easier build, crash isolation.
- **Cons:** every hook call becomes a round trip; not a real extension.
- **Verdict:** rejected by the project owner; explicitly barred as a fallback.

### Option: Link against a system PHP
- **Pros:** smallest build.
- **Cons:** breaks "download one binary".
- **Verdict:** rejected.

### Option: Render Helm charts, or add a Helm driver later
- **Pros:** reuses existing charts.
- **Cons:** no solid Rust Helm templating; would mean bundling or calling Helm.
- **Verdict:** the project owner chose direct apply with no later Helm driver.

## Dependencies

- static-php-cli and ext-php-rs supporting the newest PHP release.
- A published contracts tag before the binary could pin it (v1.0.0 gated plan 4's last task).
- GitHub releases being public, so updates and plugin installs need no credentials.

## References

- Source spec: `2026-11-17-speedyspec-cli-design.md`.
- Source plans: `2026-11-17-speedyspec-cli-foundation.md`, `2026-11-17-speedyspec-cli-install-zip.md`,
  `2026-11-17-speedyspec-cli-kubernetes.md`, `2026-11-18-speedyspec-cli-self-update-release.md`,
  `2026-11-18-speedyspec-cli-wordpress-files.md`.
- [SpeedySpec/speedyspec-cli](https://github.com/SpeedySpec/speedyspec-cli), [SpeedySpec/speedyspec-cli-contracts](https://github.com/SpeedySpec/speedyspec-cli-contracts)
- PRs: [#1](https://github.com/SpeedySpec/speedyspec-cli/pull/1), [#2](https://github.com/SpeedySpec/speedyspec-cli/pull/2),
  [#3](https://github.com/SpeedySpec/speedyspec-cli/pull/3), [#5](https://github.com/SpeedySpec/speedyspec-cli/pull/5),
  [#7](https://github.com/SpeedySpec/speedyspec-cli/pull/7), [#9](https://github.com/SpeedySpec/speedyspec-cli/pull/9),
  [#12](https://github.com/SpeedySpec/speedyspec-cli/pull/12), [#18](https://github.com/SpeedySpec/speedyspec-cli/pull/18),
  [#20](https://github.com/SpeedySpec/speedyspec-cli/pull/20)
- Related: [2026-11-18-cli-app-maintenance.md](2026-11-18-cli-app-maintenance.md)
