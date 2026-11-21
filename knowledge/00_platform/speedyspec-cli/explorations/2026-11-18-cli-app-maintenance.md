# Exploration: Finishing setup and maintaining installed apps with the CLI

**Status:** built in part
**Created:** 2026-11-18
**Author:** Claude, for the project owner
**Context:** How `speedyspec` finishes an install and then maintains it: database, backup and restore, plugin and theme files, on zip and Kubernetes installs.
**Approved by:** the project owner on 2026-11-18
**Socialized with:** not socialized

---

## Problem Statement

`speedyspec app install` stopped once files or resources were in place. A WordPress site still needed its web
installer, and nothing helped afterwards: no backup, no restore, no database access, no plugin or theme management.
The project owner wanted an install to end with a working site, and one tool to keep it working. A second need was
SpeedySpec itself. It keeps WordPress's plugin and theme model, and the project owner wants to test whether WordPress
plugins really run on SpeedySpec. That means installing and activating the same plugin on both and comparing results.

## Current State

The CLI from the 2026-11-17 design, in `SpeedySpec/speedyspec-cli`: zip and Kubernetes drivers (PRs #1 and #2),
self-update and four-target releases (PR #3), contracts 1.0.x. Zip installs left no record, so nothing could find
them by name later. The embedded PHP had no database or XML extensions. Plugin management then sat at top-level
`plugin`; PR #9 moved it to `self plugin` so `plugin` and `theme` could mean an installed app's.

## Proposed Solution

### Architecture

```
 speedyspec (parent)                                   app's plugin (PHP, via hooks)
   │  app db / backup / restore / plugin / theme   <──  AppRuntime, DatabaseConnection,
   │                                                    SetupPlan, BackupSpec, ExtensionLayout
   ├─ speedyspec-db (Rust MySQL client, rustls) ───> MySQL / MariaDB
   ├─ speedyspec-backup (tar.gz, serialized-aware replace, prefix rename)
   └─ runner: re-executes itself as hidden `self run-task`
          zip:        child process in the install root
          kubernetes: runner pod (Job on the app pod's node, app volumes mounted,
                      busybox `sleep`; the CLI streams its own binary in through exec)
```

### Data Model

- Zip install record: `$SPEEDYSPEC_HOME/installs/<name>.json` with app, parts, driver, location, CLI version, time,
  `status` (`files` or `installed`) and `setup_done`.
- Contracts 1.1.0 to 1.5.0, all additive: `Install`, `AppRuntime`, `DatabaseConnection`, `AppTask`; `SetupPlan` and
  the extension classes; `BackupSpec`, `SiteUrl`, `PrefixedColumn`; later `TableColumn` and `keepUrls`; `Workload`,
  `MaintenanceFile`, and a TLS mode on `DatabaseConnection`.
- Backup: one `tar.gz` with `manifest.json` first, then `files/`, then `database.sql`. Mode 0600; it holds secrets.

### Interfaces

- `speedyspec app installs`; `app db export | import | query | tables | size | check --install <name>`.
- `speedyspec app backup --install <name> [--to <file>]`;
  `app restore <file> --install <name> [--url <new>] [--table-prefix <p>] [--yes]`.
- `speedyspec plugin | theme  list | install <slug>[@version] | update <slug>|--all | delete <slug>`.
- New hooks: `speedyspec_app_runtime`, `speedyspec_app_database`, `speedyspec_app_setup`, `speedyspec_app_backup`,
  `speedyspec_app_extensions`, `speedyspec_extension_sources`.

## Design Decisions

1. **The CLI binary does the work itself** (the project owner's choice). A Rust MySQL client, a runner that is the
   same binary, and the same runner on Kubernetes. *Trade-off:* a larger binary and the most work.
2. **The CLI never executes stock WordPress** (amendment by the project owner, 2026-11-18). Setup only writes
   `wp-config.php` with fresh salts; `wp_install()` was dropped and the site is finished in the browser. *Trade-off:*
   the install does not end fully configured.
3. **Plugins and themes are file work.** Activate, deactivate, theme enable, `--site`, `--network` and `app sites`
   were dropped; people activate in wp-admin. *Trade-off:* the multisite design and the premise test wait.
4. **Backup, restore and the move are Rust** (the project owner, 2026-11-19). The search and replace understands
   PHP-serialized values and recomputes byte lengths; the prefix rename covers tables and prefixed keys.
   `speedyspec_app_move` was dropped. *Trade-off:* multisite domains and JSON-escaped URLs are not moved.
5. **WordPress knowledge stays in plugins as data.** `siteurl`, prefixed columns and the `.maintenance` file come
   from the WordPress plugin's `BackupSpec` and `AppRuntime`. *Trade-off:* each app needs its own plugin.
6. **A restore never silently overwrites.** Over existing tables it asks (`--yes` skips; no terminal and no `--yes`
   exits 2), then backs up what is there first. The target's own `wp-config.php` is preserved. *Trade-off:* the
   pre-restore backup costs time and disk.
7. **Secrets reach the runner only in a 0600 request file**, never an argument, environment variable, record, log
   or pod spec. *Trade-off:* extra file handling on every task.
8. **Composer packages for CLI plugins are installed in Rust** from the plugin's `composer.lock`, with no generated
   `vendor/autoload.php` (the project owner: never `composer.phar`). It replaced an earlier `http_get` helper.
   *Trade-off:* the CLI keeps its own copy of part of Composer's behaviour.

## Implementation Phases

1. **Foundation (plan 5).** Install records, contracts 1.1.0, database and XML extensions, `speedyspec-db`, the
   runner, `app db`. *Delivers:* `app installs` and `app db`. Shipped in PR #10.
2. **WordPress files (plan 6).** Setup phase, `plugin` and `theme`, Rust Composer, the two WordPress plugin
   repositories, contracts 1.2.0. *Delivers:* a served-ready WordPress install. Shipped in PR #12, review fixes in PR #14.
3. **Backup and restore (plan 7).** `speedyspec-backup`, contracts 1.3.0, the WordPress `BackupSpec`. *Delivers:*
   same-site and moved restores for zip installs. Shipped in PR #13; contracts 1.4.0 (`keepUrls`) in PR #15.
4. **Kubernetes maintenance (plan 8).** Contracts 1.5.0, database TLS, exec, runner pod, every command on Kubernetes,
   the maintenance file. *Delivers:* parity between drivers, and backups that cross drivers. Shipped in PR #16;
   contracts 1.5.0 pinned in PR #17.
5. **SpeedySpec app plugin and the premise test (plan 9).** Not built; it waits until SpeedySpec can boot.

## Fleet Impact

- `SpeedySpec/speedyspec-cli`: new crates `speedyspec-db`, `speedyspec-backup`, `speedyspec-composer`; the binary grew
  with the new PHP extensions (confirmed by merged PRs #10 to #17).
- `SpeedySpec/speedyspec-cli-contracts`: releases v1.1.0 to v1.5.0 (confirmed).
- `SpeedySpec/speedyspec-cli-wordpress` (confirmed: PRs #1 to #9 merged, including Kubernetes support) and
  `SpeedySpec/speedyspec-cli-wordpress-org` (confirmed to exist).
- Installed WordPress sites: core files are never modified; only `wp-config.php`, `wp-content` plugins and themes,
  and the database on restore.
- SpeedySpec sites: none yet; unverified until plan 9.

## Open Questions

1. **Plan 9: the SpeedySpec app plugin and plugin activation** — a 2026-11-19 amendment restores activation as a test
   run in SpeedySpec's own runtime, then a database write in Rust. It waits for the WordPress Rust port and a
   bootable SpeedySpec; the CLI is complete without it.
2. **Parked Kubernetes edge cases from plan 8** — PR #16 lists: `tls: verify` also trusts public authorities; the
   runner writes as the app container's user (root on the official image); workload pods are found by `matchLabels`
   only. Settled when a real need raises them.
3. **Moving multisite networks and JSON-escaped URLs** — would need more than a URL replace over the dump.
4. **Remote or encrypted backups, other databases, the rest of wp-cli** — non-goals for now; each would be a later
   plugin or design.

## Success Criteria

- [x] Export then import gives identical tables on MySQL 8.4 and MariaDB 11.4 — `CHECKSUM TABLE` tests in throwaway containers.
- [x] App code that exits, dies, throws or hangs still gives the parent a result or a clear error — runner tests.
- [x] The database password never appears in arguments, environment, output or records — Linux runner tests (first run in PR #13).
- [x] A failed setup resumes from the failed task without downloading again — `tests/setup.rs`.
- [x] Moved serialized values still unserialize in real PHP — `speedyspec-backup` tests against embedded PHP.
- [x] A 50 000-row dump and a 20 MB file back up in flat memory — measured peak 347 KiB in PR #13.
- [x] Zip backups restore onto Kubernetes and back — k3s tests in PR #16.
- [x] Runner pods are removed on error and Ctrl-C — k3s runner tests in PR #16.
- [ ] The same plugin activates on WordPress and SpeedySpec — the premise test, plan 9.

---

## Non-Goals

- **A SpeedySpec plugin registry** — later, as another source plugin.
- **Providing web server, PHP or MySQL** for zip installs — they are assumed present.
- **Encrypted or remote backups** — local owner-only files for now.
- **Databases other than MySQL and MariaDB** — WordPress needs only these.
- **Content, users, cron and the rest of wp-cli** — later plugin commands.

## Alternatives Considered

### Option: Hand off to each app's tools (wp-cli, mysqldump)
- **Pros:** fastest route to many WordPress commands.
- **Cons:** SpeedySpec has no such tool; depends on host PHP; output must be parsed.
- **Verdict:** rejected.

### Option: Drive the running app over its API
- **Pros:** no direct database or file access.
- **Cons:** needs a served site and a logged-in user; cannot finish setup or back up files.
- **Verdict:** rejected as a foundation; maybe useful later.

### Option: Run WordPress's own code for setup and activation
- **Pros:** real activation hooks run; full setup.
- **Cons:** the CLI would execute stock WordPress.
- **Verdict:** dropped by the 2026-11-18 amendment.

### Option: A runner Job whose image carries the binary (the spec's sketch)
- **Pros:** no streaming of the binary.
- **Cons:** needs an image per release.
- **Verdict:** plan 8 used a pinned busybox pod and streamed the checked binary through exec instead.

## Dependencies

- The CLI from the 2026-11-17 design, released with `SHA256SUMS` (the runner pod reuses self-update's checks).
- Each contracts minor release published before the CLI could pin it.
- Docker for the throwaway MySQL, MariaDB and k3s used in tests.

## References

- Source spec: `2026-11-18-speedyspec-cli-app-maintenance-design.md`.
- Source plans: `2026-11-18-speedyspec-cli-app-foundation.md`, `2026-11-19-speedyspec-cli-backup-restore.md`,
  `2026-11-19-speedyspec-cli-kubernetes-maintenance.md`.
- [SpeedySpec/speedyspec-cli-wordpress](https://github.com/SpeedySpec/speedyspec-cli-wordpress),
  [SpeedySpec/speedyspec-cli-wordpress-org](https://github.com/SpeedySpec/speedyspec-cli-wordpress-org)
- PRs: [#9](https://github.com/SpeedySpec/speedyspec-cli/pull/9), [#10](https://github.com/SpeedySpec/speedyspec-cli/pull/10),
  [#12](https://github.com/SpeedySpec/speedyspec-cli/pull/12), [#13](https://github.com/SpeedySpec/speedyspec-cli/pull/13),
  [#14](https://github.com/SpeedySpec/speedyspec-cli/pull/14), [#15](https://github.com/SpeedySpec/speedyspec-cli/pull/15),
  [#16](https://github.com/SpeedySpec/speedyspec-cli/pull/16), [#17](https://github.com/SpeedySpec/speedyspec-cli/pull/17)
- Related: [2026-11-17-speedyspec-cli.md](2026-11-17-speedyspec-cli.md)
