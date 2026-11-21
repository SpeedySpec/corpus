# SpeedySpec CLI — Backup and Restore Implementation Plan (plan 7)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `speedyspec app backup --install <name>` writes one `tar.gz` with the install's files, a dump of its database
and a manifest. `speedyspec app restore <file> --install <name>` puts it back onto the same site, or onto another one
with `--url` and `--table-prefix`, moving the site in Rust.

**Architecture:** The app's plugin says what a backup holds by answering a new filter, `speedyspec_app_backup`, with
a `BackupSpec`. That is plain data: the paths to include, exclude and preserve; where the site URL is in the
database; which columns hold prefixed keys; the app's version. Everything else is Rust, in a new crate,
`speedyspec-backup`:
- the archive (manifest, `files/`, `database.sql`);
- a search and replace that understands PHP-serialized values;
- a rewrite of our own dump format that renames the table prefix and replaces the site URL while the dump streams
  into the import.

The database work goes through a small trait, implemented by `speedyspec_db::Connection` and faked in tests, so the
backup and restore flows are tested without MySQL. Zip installs only, as for setup.

**Tech Stack:** Rust 1.99 (edition 2024); `tar` and `flate2` (gzip, already in the lock through `zip`); `speedyspec-db`;
`serde_json`; `tempfile`; Pest for the WordPress plugin; Docker `mysql:8.4` and `mariadb:11.4` for the database tests.

**Spec:** `docs/superpowers/specs/2026-11-18-speedyspec-cli-app-maintenance-design.md`, section "Backup and restore",
read with its Amendment. The Amendment overrides the rest: the CLI never runs WordPress, and backup and restore,
including the move to a new URL and prefix, are in Rust (Jacob, 2026-11-19). The `speedyspec_app_move` hook of the
original design is dropped. This plan is item 3 of the design's "Order of work".

## Global Constraints

- Repositories and branches:
  - `packages/speedyspec-cli`: branch `feat/backup-restore` from `origin/main`.
  - `packages/speedyspec-cli-contracts`: branch `feat/backup-contracts` from `origin/main`.
  - `packages/speedyspec-cli-wordpress`: branch `feat/backup` from `origin/main`.
- **Nothing outward-facing without Jacob's go-ahead:** no push, no pull request, no tag, no release, no workflow run.
  Read-only `gh` calls are fine.
- **Stay in this session's lane.** Never edit, branch off or rewire another session's code: speedyspec-server, the
  crates copied from it (`copied-from.json`), and the PHP area wiring in `boot.rs` and `runner_child.rs`.
  `scripts/check-copies.sh` passes after every task. This plan's CLI work is Rust.
- **The CLI never executes WordPress.** No task loads `wp-load.php` or `wp-settings.php`. WordPress knowledge
  (`siteurl` in `{prefix}options`, `{prefix}user_roles`, `{prefix}capabilities`) lives only in the WordPress plugin's
  `BackupSpec`. The CLI applies it as data.
- **Never touch a real database.** Database tests use only the `scripts/mysql.sh` containers through
  `SPEEDYSPEC_E2E_MYSQL` / `SPEEDYSPEC_E2E_MARIADB`, are `#[ignore]`, and fail when those are unset.
- **Secrets:** a backup holds the database and `wp-config.php`. It is created with mode 0600, and the command says so
  on stderr. The database password is never printed, logged, recorded in the manifest or an install record, put in
  an argument or environment variable, or included in an error message.
- **A restore never silently overwrites a site.** Over a target whose database already has tables with the target's
  prefix, it asks (`--yes` skips the question; without a terminal and without `--yes` it refuses, exit 2). It first
  takes a backup of what is there and names its path.
- **Tests never contact the internet.**
- Contracts changes are additive (minor release `1.3.0`). Existing classes do not change.
- PHP follows `.claude/rules/coding-style.md`. Stage explicit paths, never `git add -A`. Conventional Commits, no
  breaking markers, commit messages ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Before every task's commit:
  - `cargo clippy --workspace --all-targets -- -D warnings`;
  - `rustfmt --edition 2024 --check` on the files the task touched (never `cargo fmt --all`, which reformats other
    sessions' files).
- The Linux suite (`scripts/linux.sh cargo test --workspace`) when Docker is available. Afterwards, remove the
  `target-linux*/` folders or the Docker target volume it built into.

## Review Focus

Each line has a test in the task named in brackets.

1. **Serialized values that change length.** A URL inside `s:N:"…";` becomes longer or shorter. Several URLs sit in
   one string. A serialized string holds another serialized value. Arrays and objects (`O:`, `C:`) hold them, along
   with `r:`/`R:` references, `E:` enums and multibyte characters, with lengths counted in bytes. A string that only
   looks serialized (`s:5:"abc";`) or is truncated is replaced as plain text. After the move, PHP's own
   `unserialize()` reads every rewritten value, and the old URL is gone. [Task 3]
2. **A prefix that is a prefix of something else.** With old prefix `wp_`, a table named `wp_blog_wp_x` is renamed
   once at the start, not at the inner `wp_`. A table without the old prefix is left alone and warned about. In the
   prefixed columns (`option_name`, `meta_key`), only values that start with the old prefix change. `wp_` inside
   other text is not touched. [Task 4]
3. **Restore that fails halfway.** If the import fails after some statements, the files are not swapped yet. The
   message names the pre-restore backup that undoes it and the statement that failed. A bad archive (not gzip, a
   `files/` entry that escapes, a symlink pointing out, no manifest, a newer format) is refused before anything is
   touched. [Task 6]
4. **The target's own configuration.** `preserve` paths (`wp-config.php`) are kept from the target, never taken from
   the backup. The target's database settings are read before anything is replaced. A backup of another app is
   refused, and different parts are warned about. [Task 6]
5. **Big data.** A table of 50 000 rows and a 20 MB file back up and restore without being held in memory: the dump
   goes through a 0600 temp file, the import streams from the archive, and peak memory stays flat. [Task 5, Task 8]

## File Structure

```
packages/speedyspec-cli-contracts/
  src/BackupSpec.php, src/SiteUrl.php, src/PrefixedColumn.php   new (1.3.0)
  tests/ContractsTest.php                                        + cases
packages/speedyspec-cli/
  contracts-version                                              1.2.0 (Task 1), then 1.3.0 (Task 9)
  crates/speedyspec-driver/src/backup.rs                         new: mirrors and decoders
  crates/speedyspec-backup/                                      new crate
    src/serialized.rs     PHP-serialized search and replace
    src/rewrite.rs        our dump format: prefix rename, URL replace, prefixed columns, streaming
    src/archive.rs        tar.gz: manifest.json, files/, database.sql
    src/manifest.rs       the manifest
    src/flow.rs           backup and restore over the SiteDatabase trait
  crates/speedyspec-db/src/lib.rs                                + import_rewriting, + site_value
  crates/speedyspec-cli/src/backup_cmd.rs                        new: app backup, app restore
  crates/speedyspec-cli/tests/backup.rs                          new: CLI tests (database ones #[ignore])
packages/speedyspec-cli-wordpress/
  plugin.php, src/Version.php                                    + speedyspec_app_backup
  tests/VersionTest.php, scripts/e2e.sh                          + backup and restore
```

---

### Task 1: Pin contracts 1.2.0 (plan 6's Task 9)

**Files:** `contracts-version`, the fixture manifests that ask for `^1.1` (`tests/fixtures/plugins/{setup-app,ext-app,
ext-source}/speedyspec-plugin.json`), the tests that name the bundled version (`tests/cli.rs`,
`tests/plugin_install.rs`). In both plugin repositories: `speedyspec-plugin.json` (`^1.2`), and `composer.json` (the
contracts from the release instead of the path repository).

- [ ] **Step 1:** Update `contracts-version`: `version=1.2.0`, the `v1.2.0` tag's tarball URL, and the `sha256` of that
  tarball as `scripts/fetch-contracts.sh` checks it. Run `scripts/fetch-contracts.sh`. Raise the fixtures to
  `^1.2`, and update the version the tests expect.
- [ ] **Step 2: Run** `cargo test --workspace` without `SPEEDYSPEC_CONTRACTS_DIR`. Expected: pass with no build
  warning, and `speedyspec --version` shows `contracts 1.2.0`. Commit: `build: embed contracts 1.2.0`.
- [ ] **Step 3:** In the plugin repositories, `^1.2` in the manifests. Run Pest and `tests/cli.sh` against the new
  binary. Expected: pass. Commit each: `build: require contracts 1.2`.

### Task 2: Contracts 1.3.0 and their Rust mirrors

**Files:** contracts `src/{BackupSpec,SiteUrl,PrefixedColumn}.php`, `tests/ContractsTest.php`, `README.md`; CLI
`crates/speedyspec-driver/src/backup.rs` (new), `crates/speedyspec-driver/src/lib.rs`.

**Interfaces (PHP, `SpeedySpec\Cli\Contracts\`, `final readonly`):**
- `BackupSpec(array $include = ['.'], array $exclude = [], array $preserve = [], ?SiteUrl $siteUrl = null, array
  $prefixedColumns = [], ?string $appVersion = null)`. Paths are relative to the install root.
  - `exclude` is never archived.
  - `preserve` is archived, but a restore always keeps the target's own copy.
  - `prefixedColumns` is a list of `PrefixedColumn`.
- `SiteUrl(string $table, string $keyColumn, string $key, string $valueColumn)`: the row that holds the site's URL.
  `table` is without the prefix (`options`).
- `PrefixedColumn(string $table, string $column)`: a column whose values may start with the table prefix and must be
  renamed with it (`options`.`option_name`, `usermeta`.`meta_key`).

**Interfaces (Rust, `speedyspec_driver::backup`):**
- `BackupSpec`, `SiteUrl` and `PrefixedColumn` mirrors, and `decode_backup_spec(&Value, &dyn ObjectReader) ->
  Result<BackupSpec, ContractError>`.
- Rules:
  - every path passes `destination_problem`;
  - `include` is not empty;
  - a `preserve` path must lie inside an included path;
  - table and column names are `[A-Za-z0-9_]+` (the table without a prefix);
  - `appVersion` is 1 to 64 characters with no whitespace.

- [ ] **Step 1: Failing Pest tests:** construction, defaults, readonly, final.
- [ ] **Step 2: Run** `vendor/bin/pest`. Expected: fail. **Step 3:** Implement with docblocks. Expected: pass.
  Commit in contracts: `feat: add the backup contracts`.
- [ ] **Step 4: Failing Rust tests** in `backup.rs`: one decode test per class, and a rules table naming the field for
  each refusal.
- [ ] **Step 5: Run** `cargo test -p speedyspec-driver backup::` (with `SPEEDYSPEC_CONTRACTS_DIR` at the contracts
  checkout). Expected: fail to compile. **Step 6:** Implement. Expected: pass. Commit:
  `feat(driver): mirror the backup contracts`.

### Task 3: Search and replace that understands PHP-serialized values

**Files:** `crates/speedyspec-backup/` (new: `Cargo.toml`, `src/lib.rs`, `src/serialized.rs`),
`crates/speedyspec-backup/tests/php.rs` (new).

**Interfaces:**
- `pub fn replace(value: &[u8], from: &[u8], to: &[u8]) -> Cow<'_, [u8]>`.
  - A value that parses as one complete PHP-serialized value is rewritten structurally: every string inside is
    replaced, its byte length recomputed, and a string whose contents are themselves serialized is recursed into.
  - Anything else is replaced as plain bytes.
  - An unchanged value comes back borrowed.
- The parser covers `N;`, `b:`, `i:`, `d:`, `s:`, `S:` (escaped strings), `a:`, `O:`, `C:` (whose payload is opaque
  and replaced as plain bytes, with its length recomputed), `r:`, `R:` and `E:`. Anything malformed or truncated, or
  with trailing bytes, is not serialized.
- Recursion is capped at depth 64; past it, the value is treated as plain.

- [ ] **Step 1: Failing unit tests**, one per Review Focus 1 case:
  - a longer URL;
  - a shorter one;
  - two URLs in one string;
  - nested serialization;
  - an array of objects;
  - references kept as they are;
  - an enum (`E:` holds no URL, so it is untouched);
  - a multibyte string;
  - `s:5:"abc";` (wrong length, so plain);
  - truncated input;
  - trailing bytes;
  - a 1000-level nesting that hits the cap without overflowing the stack;
  - an unchanged value that stays borrowed.
- [ ] **Step 2: Failing test with PHP** in `tests/php.rs` (`harness = false`, booting the embedded PHP as the
  `speedyspec-php` tests do): it serializes a fixed set of PHP values with real `serialize()`, replaces with this
  module, and checks that `unserialize()` reads each one and gives the expected structure with the URL changed.
- [ ] **Step 3: Run** `cargo test -p speedyspec-backup`. Expected: fail. **Step 4:** Implement. Expected: pass.
  Commit: `feat(backup): search and replace that keeps PHP-serialized values valid`.

### Task 4: Rewriting a dump as it streams

**Files:** `crates/speedyspec-backup/src/rewrite.rs`, `crates/speedyspec-db/src/lib.rs` (`import_rewriting`).

**Interfaces:**
- `pub struct Move { pub from_prefix: String, pub to_prefix: String, pub urls: Option<(String, String)>, pub
  prefixed_columns: Vec<(String, String)> }`. Tables are without the prefix. A `Move` that changes nothing is a
  no-op.
- `pub fn rewrite(statement: &[u8], plan: &Move, warnings: &mut Vec<String>) -> Cow<'_, [u8]>` reads statements in
  the form `Connection::export` writes them:
  - `DROP TABLE IF EXISTS`, `CREATE TABLE` and `INSERT INTO <table> (<columns>) VALUES (…),(…)`: a table name that
    starts with `from_prefix` gets `to_prefix` instead, once and at the start. A table without it is left alone, and
    a warning names it.
  - In `INSERT` values, each string literal is unescaped. Then:
    - in a prefixed column (found by position in the column list), a value that starts with `from_prefix` has that
      start replaced;
    - every string literal gets the URL replacement through `serialized::replace`;
    - the result is escaped again as `quote_bytes` does.
  - `NULL`, numbers and `0x…` literals pass through untouched.
  - Any other statement passes through untouched.
- `Connection::import_rewriting(&mut self, input: impl BufRead, rewrite: &mut dyn FnMut(&[u8]) -> Vec<u8>) ->
  Result<u64, ImportError>` is `import` with each statement passed through `rewrite` before it runs. It uses the same
  splitter, the same error naming and the same database-switch refusal, applied after rewriting.

- [ ] **Step 1: Failing unit tests:**
  - a table rename, including `wp_blog_wp_x`;
  - a table without the prefix, which is warned about;
  - `option_name` values `wp_user_roles` → `site_user_roles`, while `wpx_thing` is untouched;
  - a serialized `option_value` holding the URL;
  - backslashes, quotes and NUL bytes surviving unescaping and escaping;
  - a `0x` binary literal untouched;
  - a no-op `Move` that returns the statement borrowed;
  - a statement that is neither DDL nor `INSERT`, passed through.
- [ ] **Step 2: Run** `cargo test -p speedyspec-backup rewrite::`. Expected: fail. **Step 3:** Implement.
  Expected: pass.
- [ ] **Step 4: Failing database test** in `crates/speedyspec-db/tests/mysql.rs` (`#[ignore]`):
  `import_rewriting_renames_and_replaces`. It exports a fixture with prefix `wp_` and a serialized option, then
  imports it with a `Move` to `site_` and a new URL. On MySQL and MariaDB, `site_options` exists, `wp_options`
  does not, and `unserialize` of the option in the embedded PHP gives the new URL.
- [ ] **Step 5: Run** it with `scripts/mysql.sh up` (only when Docker runs). Expected: fail, then pass. Commit:
  `feat(backup): rename the prefix and move the URL while a dump streams in`.

### Task 5: The archive, and `app backup`

**Files:** `crates/speedyspec-backup/src/{archive.rs,manifest.rs,flow.rs}`, `crates/speedyspec-cli/src/backup_cmd.rs`
(new), `crates/speedyspec-cli/src/{commands.rs,main.rs}`, `crates/speedyspec-cli/tests/backup.rs` (new), fixture
`tests/fixtures/plugins/backup-app/`, `README.md`.

**Interfaces:**
- `trait SiteDatabase { fn export(&mut self, out: &mut dyn Write) -> Result<(), String>; fn import(&mut self, input:
  &mut dyn BufRead, plan: &Move) -> Result<u64, String>; fn has_prefixed_tables(&mut self) -> Result<bool, String>;
  fn site_url(&mut self, at: &SiteUrl) -> Result<Option<String>, String>; fn prefix(&self) -> &str; }`. It is
  implemented for `speedyspec_db::Connection` in `backup_cmd.rs` and faked in the crate's tests.
- `Connection::site_value(&mut self, table: &str, key_column: &str, key: &str, value_column: &str) ->
  Result<Option<String>, DbError>` (in `speedyspec-db`). It reads one value from the prefixed table with a prepared
  statement. The identifiers are checked as `[A-Za-z0-9_]+` and quoted, so nothing from a plugin reaches the SQL
  text unescaped. `site_url` uses it, and `has_prefixed_tables` uses the existing `tables()`.
- `Manifest { format: 1, app, parts, cli_version, app_version: Option<String>, extensions: Vec<(kind, slug,
  version)>, engine: "mysql", table_prefix, site_url: Option<String>, source_install, source_driver, created_at }`,
  as `manifest.json`. A newer `format` is refused with "made by a newer speedyspec".
- The archive:
  - `manifest.json` first, then `files/…` (the included paths, minus the excluded ones, relative to the root), then
    `database.sql`;
  - gzip;
  - created 0600 through a temp file in the destination's directory, renamed into place;
  - symlinks archived as links, never followed;
  - the dump written to a 0600 temp file first (the tar header needs its size), then appended and deleted.
- `flow::backup(root, spec, db, manifest, to: &Path) -> Result<(), String>`.
- `speedyspec app backup --install <name> [--to <file>]`:
  - the default file is `<home>/backups/<name>-<UTC timestamp>.tar.gz`;
  - an existing `--to` is refused unless `--force`;
  - it prints the path on stdout, and on stderr `speedyspec: the backup holds the site's database and secrets (such
    as wp-config.php); keep it private`;
  - `extensions` come from `speedyspec_app_extensions`, when a plugin answers it.

- [ ] **Step 1: Failing tests:**
  - `flow` tests with the fake database:
    - the entries come in the documented order;
    - excluded paths are absent;
    - a symlink stays a link;
    - the file mode is 0600;
    - a 20 MB file and a 50 000-row fake dump go through with peak memory under 64 MB (measured with a counting
      global allocator in the test binary).
  - CLI tests in `tests/backup.rs` that need no database (fixture `backup-app`, whose database answer points at a
    closed port):
    - an unreachable database fails before any file is left behind;
    - `--to` refuses an existing file.
  - `#[ignore]` CLI tests with the MySQL container:
    - `backup_writes_the_documented_archive`;
    - `backup_says_it_holds_secrets`;
    - `the_password_is_nowhere_in_the_manifest_or_output`.
- [ ] **Step 2: Run.** Expected: fail. **Step 3:** Implement. Expected: pass (database tests when Docker runs).
  Commit: `feat: app backup writes the install's files, database and manifest to one archive`.

### Task 6: `app restore`

**Files:** `crates/speedyspec-backup/src/flow.rs`, `crates/speedyspec-cli/src/backup_cmd.rs`,
`crates/speedyspec-cli/tests/backup.rs`, `README.md`.

**Interfaces:**
- `speedyspec app restore <file> --install <name> [--url <new>] [--table-prefix <prefix>] [--yes]`. Zip installs only,
  through `Target::root`.
- The order, each step before anything is changed except where it says so:
  1. **Read the archive's manifest** (the first entry). A bad archive is refused before anything is touched: not
     gzip, no manifest, a newer format, or a `files/` entry that escapes or links out (checked as each entry is
     unpacked).
  2. **Check the target.** Its app must equal the manifest's; parts that differ get a warning. Read its `BackupSpec`
     and its database settings (`speedyspec_app_database`), then open the connection.
  3. **Work out the move.** `from_prefix` is the manifest's prefix and `to_prefix` is `--table-prefix`, or the
     target's own prefix. The URLs come from the manifest's site URL and `--url`. The prefixed columns come from the
     target's `BackupSpec`. `--url` without a site URL in the manifest is refused.
  4. **Ask.** If the target's database has prefixed tables, ask `Restore over the site in <install>? Its files and
     database are replaced. [y/N]`. `--yes` skips the question; without a terminal and without `--yes`, refuse with
     exit 2.
  5. **Back up what is there**, to `<home>/backups/<name>-before-restore-<timestamp>.tar.gz`, and name the path on
     stderr.
  6. **Unpack** `files/` into `<root>/.speedyspec-restore-<random>/`.
  7. **Import** `database.sql` straight from the archive through `import_rewriting` with the move. Print the
     rewrite's warnings.
  8. **Swap the files.** Each top-level entry under the included paths is renamed aside and the new one renamed in,
     except `preserve` paths, which stay the target's. The old entries are then removed.
- A failure from step 6 on says what was changed, and names the pre-restore backup that undoes it. If the import
  fails, it also names the failing statement and line, and says the files were not replaced.
- Restoring onto the same site is the same command without `--url` and `--table-prefix`.

- [ ] **Step 1: Failing tests:**
  - `flow` tests with the fake database:
    - each bad-archive case refused with nothing changed;
    - a different app refused;
    - different parts warned about;
    - `preserve` kept from the target;
    - an import that fails leaves the files unswapped and the message naming the backup;
    - the swap putting back an entry when a rename fails;
    - `--url` without a site URL in the manifest refused.
  - CLI tests:
    - `restore_over_content_needs_yes_when_not_interactive` (exit 2, nothing changed);
    - `#[ignore]` database round trips: `same_site_restore_undoes_a_change` (back up, change a row and a file,
      restore, both back) and `restore_to_another_url_and_prefix` (the target's tables have the new prefix,
      `siteurl` is the new URL, a serialized option unserializes with the new URL, `wp-config.php` is the
      target's);
    - `a_restore_takes_a_backup_first`.
- [ ] **Step 2: Run.** Expected: fail. **Step 3:** Implement. Expected: pass (database tests when Docker runs).
  Commit: `feat: app restore puts a backup back, onto the same site or another URL and prefix`.

### Task 7: The WordPress plugin answers `speedyspec_app_backup`

**Files:** `speedyspec-cli-wordpress`: `plugin.php`, `src/Version.php` (new), `tests/VersionTest.php` (new),
`README.md`.

**Interfaces:**
- `speedyspec_app_backup` (for `app === 'wordpress'`) returns:

  ```php
  BackupSpec(
      ['.'],
      ['wp-content/cache', 'wp-content/upgrade', '.maintenance'],
      ['wp-config.php'],
      new SiteUrl('options', 'option_name', 'siteurl', 'option_value'),
      [new PrefixedColumn('options', 'option_name'), new PrefixedColumn('usermeta', 'meta_key')],
      Version::of($root),
  )
  ```
- `Version::of(string $root): ?string` reads `$wp_version = '…';` from `wp-includes/version.php` with a regex, without
  including the file (it is WordPress code).

- [ ] **Step 1: Failing Pest tests:**
  - `Version::of` reads a real `version.php` (committed as a fixture);
  - a missing file gives null;
  - a file without the line gives null.
- [ ] **Step 2: Run** `vendor/bin/pest`. Expected: fail. **Step 3:** Implement the class and the hook. Expected: pass.
  Commit: `feat: tell speedyspec what a WordPress backup holds`.

### Task 8: End to end, and docs

**Files:** `speedyspec-cli-wordpress/scripts/e2e.sh`, `crates/speedyspec-cli/tests/backup.rs`, CLI `README.md` (the
commands, the archive format, the move and its limits), `speedyspec-cli-wordpress/README.md`.

- `e2e.sh` gains:
  - a backup of the e2e site;
  - a same-site restore after deleting a row;
  - a restore onto a second install with another URL and prefix, after which `app db query` shows the new
    `siteurl` and `site_options`.
- The README states the move's limits:
  - multisite networks are not moved (their `blogs` and `site` domains need more than a URL replace);
  - JSON-escaped URLs (`https:\/\/…`) are not replaced;
  - tables in the target that the backup does not have are left in place.
- A big-data `#[ignore]` test against the MySQL container: `a_large_table_round_trips_in_flat_memory`, 50 000 rows,
  `CHECKSUM TABLE` equal after the round trip.

- [ ] **Step 1:** Write the e2e additions, the big-data test and the docs. **Step 2: Run** the workspace, clippy, the
  plugin's Pest suite and `tests/cli.sh`, and the database tests and e2e only when Docker runs and Jacob says so.
  Commit each repository: `docs: …` / `test: …`.

### Task 9: Pin contracts 1.3.0

Blocked until Jacob merges the contracts PR and its release.

- [ ] **Step 1:** Update `contracts-version` to 1.3.0 and the fixtures and the WordPress plugin to `^1.3`. **Step 2:**
  Run everything again without `SPEEDYSPEC_CONTRACTS_DIR`. Expected: pass. Commit: `build: embed contracts 1.3.0`.
