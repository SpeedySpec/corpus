# SpeedySpec CLI — WordPress Files: Setup, Plugins and Themes Implementation Plan (plan 6)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `speedyspec app install wordpress` ends with a `wp-config.php` that php-fpm can serve. `speedyspec plugin` and
`speedyspec theme` list, install, update and delete an install's plugin and theme files from wordpress.org. CLI plugins
can depend on composer packages (Guzzle and the like), installed by the CLI itself from each plugin's `composer.lock`.

**Architecture:** The CLI stays generic. After the driver places the files, it asks `speedyspec_app_setup` for a
`SetupPlan` and runs its `AppTask`s through the plan 5 runner, recording progress so a failed setup can be finished
by running the same command again. For plugins and themes it asks `speedyspec_app_extensions` where they live
(an `ExtensionLayout` with an `inspect` closure that reads headers) and `speedyspec_extension_sources` where to get
them (`ExtensionSource`s whose `resolve` closure returns an `ExtensionArchive`). It then downloads and unpacks with
the zip driver's checked extraction and swaps directories into place. Everything WordPress-specific lives in two new
plugin repositories: the WordPress app plugin (options, `wp-config.php` writing and reading, header parsing, layout)
and the wordpress.org source plugin (the plugin and theme APIs, over Guzzle). A new Rust crate, `speedyspec-composer`,
installs a plugin's locked composer packages and writes its autoloader, so the CLI needs no composer on the machine.

**Tech Stack:** Rust 1.99 (edition 2024); the plan 5 runner and `speedyspec-db`; `speedyspec-driver-zip`'s
`extract` and `speedyspec-fetch`; `speedyspec-testkit` (`Server`, `build_zip`); PHP 8.5 embedded; Pest for the PHP
repositories; Docker `mysql:8.4` and `mariadb:11.4` through `scripts/mysql.sh`.

**Spec:** `docs/superpowers/specs/2026-11-18-speedyspec-cli-app-maintenance-design.md`, read with its **Amendment
(2026-11-18, before plan 6)**, which overrides the rest: the CLI never executes WordPress, setup and plugins and
themes are file work, activation and `app sites` are dropped, and the CLI does not include WordPress's legacy
functions. This plan is item 2 of the design's "Order of work", cut down by the amendment.

## Global Constraints

- Repositories and branches:
  - `packages/speedyspec-cli-contracts`: branch `feat/extension-contracts` from `main`.
  - `packages/speedyspec-cli`: branch `feat/wordpress-files` from `feat/wp-hooks`.
  - New local repositories `packages/speedyspec-cli-wordpress` and `packages/speedyspec-cli-wordpress-org`
    (`git init -b main`). They are not created on GitHub without Jacob's go-ahead.
- **Nothing outward-facing without Jacob's go-ahead:** no push, no pull request, no tag, no release, no workflow run,
  no GitHub repository created. Read-only `gh` calls are fine.
- **The CLI never executes WordPress** (Amendment). No task loads `wp-load.php` or `wp-settings.php`, and nothing
  modifies WordPress core files. Every WordPress `AppTask` has `loadsApp: false`.
- **Stay in this session's lane** (Jacob, 2026-11-18). Never edit, branch off or rewire another session's code:
  speedyspec-server, the crates copied from it (`copied-from.json`), and the PHP area wiring in `boot.rs` and
  `runner_child.rs` stay as `feat/wp-hooks` set them. This session's CLI work is Rust. `scripts/check-copies.sh`
  passes after every task.
- **Never touch a real database.** Database tests use only the `scripts/mysql.sh` containers through
  `SPEEDYSPEC_E2E_MYSQL` / `SPEEDYSPEC_E2E_MARIADB`, are `#[ignore]`, and fail when those are unset.
- **Secrets:** the database password is never printed, logged, recorded in an install record, put in a process
  argument or environment variable, or included in an error message. It reaches a setup task only inside the runner's
  request file (mode 0600, deleted after use).
- **Tests never contact the internet.** wordpress.org is replaced by `speedyspec-testkit`'s loopback `Server`
  through `SPEEDYSPEC_WORDPRESS_ORG_API` (the source plugin) and `SPEEDYSPEC_WORDPRESS_DOWNLOADS` (the app plugin);
  composer dists in test locks point at the loopback server.
  The one exception is Task 8's manual end-to-end script, which downloads a pinned WordPress release and runs only
  when Jacob runs it.
- Nothing installed on the host; Docker images may be pulled. Never set `SPEEDYSPEC_ALLOW_HOST_INSTALL`.
- Contracts changes are additive (minor release `1.2.0`); existing classes do not change.
- PHP under `packages/` follows `.claude/rules/coding-style.md` (braces on new lines, `declare(strict_types=1)`,
  4 spaces, docblocks that say why).
- Stage explicit paths, never `git add -A`. Conventional Commits with no breaking-change markers. Commit messages end
  with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --all --check` clean after every Rust task;
  the Linux suite (`scripts/linux.sh cargo test --workspace`) at the end of each task that touches Rust.

## Review Focus

Each line has a test in the task named in brackets.

1. **Slugs that are paths.** `../x`, `a/b`, `.`, `..`, the empty string, a slug with a NUL byte or a leading dash,
   and `plugin delete ../../etc`. Each is refused as a usage error (exit 2) before anything is downloaded, created or
   removed. [Task 5]
2. **Archives that do not match.** A plugin zip with two top-level entries, files at the top level, an entry that
   escapes, a symlink pointing outside, or a top-level directory named differently from the slug. One top-level
   directory is installed as `<dir>/<slug>` whatever its name. Anything else is refused, and nothing is left behind
   (no temp directory, no half-written plugin). [Task 5]
3. **An update that fails halfway.** A download error, a bad checksum, an unsafe archive or a failed rename during
   `update` leaves the old version in place and working, and `update --all` carries on with the next one and exits 1.
   [Task 5]
4. **Re-running a setup.** A first run that fails at task 2 records `status: "files"` and `setup_done: 1`. Running the
   same command again downloads nothing, skips task 1 and runs task 2. Running it on a finished install leaves an
   existing `wp-config.php` alone. [Task 4, Task 6]
5. **Composer locks that do not fit.** A dist whose sha1 does not match, an archive entry that escapes, a package name
   or autoload path that leaves `vendor/`, a lock with only `path` dists, a package needing an extension the embedded
   PHP lacks, and two plugins locking the same package at different versions. Each is refused with the package named
   (the last one is a warning), and a failed install leaves the previous `vendor/` as it was. [Task 3]
6. **Values that break PHP strings.** A database password or name containing `'`, `\`, `$`, `"`, a newline or
   `?>` survives writing `wp-config.php` and reading it back byte for byte, and the file still parses. No error message
   from a failing setup task contains the password. [Task 6]

## File Structure

```
packages/speedyspec-cli-contracts/
  src/SetupPlan.php, src/ExtensionLayout.php, src/Extension.php, src/ExtensionSource.php, src/ExtensionArchive.php
  tests/ContractsTest.php                                    + cases for the five
packages/speedyspec-cli/
  crates/speedyspec-composer/                                new: composer.lock, dists, vendor/, autoloader
  crates/speedyspec-plugin/src/                              + register the Rust autoloader before the entry
  crates/speedyspec-cli/src/plugin_cmd.rs                    + dependencies on install
  crates/speedyspec-cli/src/composer_cmd.rs                  new: `speedyspec composer install | dump-autoload | show | validate`
  crates/speedyspec-driver/src/extensions.rs                 new: mirrors and decoders for the five contracts
  crates/speedyspec-cli/src/setup.rs                         new: the setup phase of app install
  crates/speedyspec-cli/src/installs.rs                      + status and setup_done
  crates/speedyspec-cli/src/extensions.rs                    new: list, install, update, delete on disk
  crates/speedyspec-cli/src/extension_cmd.rs                 new: `plugin` and `theme` commands
  crates/speedyspec-cli/tests/{setup,extensions,composer}.rs new
  crates/speedyspec-cli/tests/fixtures/plugins/              setup-app, ext-app, ext-source
packages/speedyspec-cli-wordpress/                           new repository: the WordPress app plugin
  speedyspec-plugin.json, plugin.php, composer.json
  src/{Options,ConfigFile,Headers,Layout}.php
  tasks/write-config.php, tasks/read-config.php, tasks/stub/wp-settings.php
  tests/*.php (Pest), scripts/e2e.sh, README.md
packages/speedyspec-cli-wordpress-org/                       new repository: the wordpress.org source plugin
  speedyspec-plugin.json, plugin.php, composer.json
  src/{Api,Source}.php, tests/*.php (Pest), tests/fixtures/*.json, README.md
```

---

### Task 1: (dropped)

Leaving WordPress's legacy hook functions out of the CLI meant editing speedyspec-server's crates, which belong to
another session (Jacob, 2026-11-18). It was reverted (70de001). The CLI never loads WordPress, so nothing here needs
it.

### Task 2: Contracts 1.2.0 and their Rust mirrors

**Files:** contracts `src/{SetupPlan,ExtensionLayout,Extension,ExtensionSource,ExtensionArchive}.php`,
`tests/ContractsTest.php`, `README.md`; CLI `crates/speedyspec-driver/src/extensions.rs` (new),
`crates/speedyspec-driver/src/lib.rs`.

**Interfaces (PHP, `SpeedySpec\Cli\Contracts\`, `final readonly`, promoted public properties):**
- `SetupPlan(array $tasks)`: a list of `AppTask`, run in order.
- `ExtensionLayout(array $directories, \Closure $inspect)`: `$directories` maps a kind (`plugin`, `theme`) to a
  directory relative to the install root. `$inspect(string $kind, string $path): ?Extension` is called with the
  absolute path of each entry in that directory (directory or file) and returns null for anything that is not an
  extension.
- `Extension(string $kind, string $slug, string $name, ?string $version = null)`.
- `ExtensionSource(string $name, \Closure $resolve)`: `$resolve(string $kind, string $slug, ?string $version):
  ?ExtensionArchive`. Null means "not here"; a version of null means the latest.
- `ExtensionArchive(string $kind, string $slug, string $version, string $url, ?string $sha256 = null)`.

**Interfaces (Rust, `speedyspec_driver::extensions`):**
- `SetupPlan { tasks: Vec<AppTask> }`, `ExtensionLayout { directories: BTreeMap<Kind, String>, inspect: Value }`,
  `Extension { kind, slug, name, version: Option<String> }`, `ExtensionSource { name: String, resolve: Value }`,
  `ExtensionArchive { kind, slug, version, url, sha256: Option<String> }`, and `enum Kind { Plugin, Theme }`
  (`as_str`, `parse`).
- `decode_setup_plan`, `decode_extension_layout`, `decode_extension`, `decode_extension_sources` (a list),
  `decode_extension_archive`, each `(&Value, &dyn ObjectReader) -> Result<_, ContractError>`, following `app.rs`.
- `pub fn slug_problem(slug: &str) -> Option<&'static str>`. A slug is 1 to 200 bytes of `[A-Za-z0-9._-]`, does not
  start with `.` or `-`, and is not `.` or `..`.
- Rules: a kind is `plugin` or `theme`; directories pass `destination_problem` and are not `.`; a version is 1 to 64
  bytes with no whitespace or `/`; the url passes `is_fetchable_url`; sha256 is 64 lowercase hex characters; the two
  closures are closure objects (as `Command::handler`).

- [ ] **Step 1: Failing Pest tests** in the contracts repo: each class constructs with the documented defaults, is
  `final` and `readonly`, and exposes its properties.
- [ ] **Step 2: Run** `vendor/bin/pest`. Expected: fail (classes missing).
- [ ] **Step 3: Implement** the five classes with docblocks. Expected: Pest passes. Commit in the contracts repo:
  `feat: add the setup plan and extension contracts`.
- [ ] **Step 4: Failing Rust tests** in `extensions.rs` with `FakeReader`: one decode test per contract, a rules
  table (each rule above refused, with the field named), `slug_problem` for every input in Review Focus 1, and
  `sources_keep_their_order`.
- [ ] **Step 5: Run** `cargo test -p speedyspec-driver extensions::`. Expected: fail to compile.
- [ ] **Step 6: Implement**, then run the CLI workspace with `SPEEDYSPEC_CONTRACTS_DIR=../speedyspec-cli-contracts`.
  Expected: pass, with the warning that the contracts are not the pinned version (expected until Task 9). Commit:
  `feat(driver): mirror the setup plan and extension contracts`.

### Task 3: Composer dependencies for CLI plugins, in Rust

Replaces the `http_get` of the first draft (reverted, 43dd4d2): plugins take HTTP, and anything else, from composer
packages, which the CLI installs itself (Jacob: Rust-native, nothing on the machine, never `composer.phar`). Done
after Task 5 and before Task 7, which is the first to need it.

**Files:** `crates/speedyspec-composer/` (new: `lock.rs`, `install.rs`, `autoload.rs`, `classmap.rs`, `platform.rs`),
`crates/speedyspec-composer-php/` (new: the `composer` native area and the plugin host), `crates/speedyspec-cli/src/composer_child.rs` (new),
`crates/speedyspec-plugin/src/` (the loader), `crates/speedyspec-cli/src/plugin_cmd.rs`, `crates/speedyspec-cli/src/boot.rs`
(CA roots for PHP), `crates/speedyspec-cli/tests/composer.rs` (new), fixtures `tests/fixtures/composer/`, `README.md`.

**Interfaces:**
- No resolving: the plugin ships `composer.lock` and the CLI installs exactly its `packages` (never `packages-dev`).
  A plugin whose `composer.json` requires anything other than `php` and `ext-*` without a `composer.lock` is refused:
  `<plugin> needs composer packages but has no composer.lock; run composer update in the plugin`.
- `speedyspec_composer::Lock::read(dir: &Path) -> Result<Option<Lock>, ComposerError>`. `Lock { root: Autoload,
  packages: Vec<Package> }`, `Package { name, version, dist: Dist { url, shasum: Option<String> }, autoload: Autoload,
  require: Vec<(String, String)> }`, `Autoload { psr4: Vec<(String, Vec<String>)>, psr0: Vec<(String, Vec<String>)>,
  classmap: Vec<String>, files: Vec<String> }`. `root` is the plugin's own `composer.json` autoload. Package names
  match composer's rule (`vendor/name`, lowercase), every autoload path is relative and inside its package, and only
  `zip` dists are installed (`path`, `git` and the rest are refused, naming the package).
- `speedyspec_composer::install(dir: &Path, lock: &Lock, fetch: &dyn Fetch, platform: &Platform) ->
  Result<Installed, ComposerError>`:
  - Builds `vendor.speedyspec-new/` beside `vendor/`, then swaps it in, so a failure leaves the old `vendor/` alone.
  - Each dist is downloaded with `speedyspec-fetch` (same URL rules as all downloads), its sha1 checked when the lock
    has one, and unpacked with the zip driver's checked extraction into `vendor/<name>`. A single top-level directory
    is stripped, as composer does.
  - `Platform { php_version: String, extensions: Vec<String> }` comes from the embedded PHP. Each package's `ext-*`
    requirement must be loaded; its `php` constraint is checked for the forms `^`, `~`, `>=`, `>`, `<=`, `<`, `=`,
    `*`, `x.y.*`, spaces (and) and `||` / `|` (or). Any other form is a warning, not a refusal.
- **All composer code is Rust** (Jacob); the only PHP that runs is the composer plugins' own code. Nothing of
  composer is written as PHP: no generated `vendor/autoload.php`. `autoload.rs` keeps each plugin's PSR-4 and PSR-0
  maps and classmap in Rust and writes them to `vendor/speedyspec-autoload.json`. A native function of this
  session's own `composer` area, registered with `spl_autoload_register` (appended, never prepended), looks a class
  up in those maps and requires its file. The `files` entries are required by Rust in lock order, root last.
  `classmap.rs` builds the classmap by scanning PHP files in Rust: `namespace` and `class`, `interface`, `trait` and
  `enum` declarations, skipping comments, strings and heredocs.
- The plugin loader registers the plugin's Rust autoloader from `vendor/speedyspec-autoload.json`, when it exists,
  and requires its `files`, before the entry file. The `composer` area is appended to the CLI's areas in `boot.rs`;
  the existing areas are left as they are. Plugins whose
  locks share a package at different versions load anyway, with a warning naming both plugins, the package and the
  version that wins (the first loaded).
- **`speedyspec composer <subcommand>`** (Jacob): the composer subcommands this replica supports, each acting on
  the directory given by `--working-dir <dir>` (default: the current directory), as composer's own flag does.
  `composer` joins the reserved command names. Supported in this plan:
  - `composer install [--no-plugins] [--allow-plugins]`: installs exactly the `composer.lock` packages (never dev
    ones), writes the autoload maps and runs the composer plugins (only with `--allow-plugins`). `--no-dev` is
    accepted and has no effect, because dev packages are never installed.
  - `composer dump-autoload`: rebuilds the autoload maps from what is in `vendor/` and the root `composer.json`.
  - `composer show`: lists the installed packages as `name <TAB> version <TAB> type`.
  - `composer validate`: checks `composer.json` and `composer.lock` the way the replica reads them, naming each problem.

  Any other composer subcommand (`update`, `require`, `remove` and the rest, which need dependency resolving) is
  refused as a usage error: `speedyspec composer supports install, dump-autoload, show and validate; run composer
  itself for <name>`.
- `self plugin install` runs the same install after unpacking (composer plugins only with
  `--allow-composer-plugins`), and a failure removes the half-installed plugin. For a development directory used with
  `--plugin-dir`, run `speedyspec composer install --working-dir <dir>`.
- **Composer plugins** (Jacob: replicate in Rust the code that runs them; never run `composer.phar`). After the files
  and the autoload maps are in place, packages of type `composer-plugin` run in a separate embedded PHP process of the
  CLI (`self composer-plugins`, hidden, the runner's pattern: own process group, request and result in files, one
  process per install). It boots with a `composer` native area of this session's own, in Rust, that declares the
  plugin API plugins use: `Composer\Plugin\PluginInterface`, `Composer\Plugin\Capable`,
  `Composer\EventDispatcher\EventSubscriberInterface`, `Composer\IO\IOInterface` (backed by Rust, writing to
  stderr, never asking), `Composer\Composer` with `getPackage()`, `getConfig()`, `getRepositoryManager()
  ->getLocalRepository()->getPackages()`, `getInstallationManager()->getInstallPath()` and `getEventDispatcher()`,
  the package classes (`getName`, `getPrettyName`, `getVersion`, `getType`, `getExtra`, `getAutoload`),
  `Composer\Script\Event`, `Composer\Installer\PackageEvent` and the event-name constants. It registers
  the Rust autoloader, instantiates each plugin's `extra.class`, calls `activate()`, registers its subscribed
  events, and dispatches what composer dispatches after an install: `post-package-install` per package in lock order,
  then `post-install-cmd` and `post-autoload-dump`. Root `scripts` that are PHP callables (`Class::method`) run too; a
  shell-command script is not run and is reported. A plugin that calls an API this replica lacks (custom installers
  through `addInstaller`, for one) fails the install with `<package>: composer's <method> is not supported by
  speedyspec`. Plugins run only when `speedyspec composer install` is given `--allow-plugins` (or `self plugin install`
  `--allow-composer-plugins`), as composer's `allow-plugins`, since they run code at install time.
- TLS for PHP: dropped during execution (see the ledger). The static PHP's OpenSSL already reads `/etc/ssl/cert.pem`
  and `/etc/ssl/certs`, and `SSL_CERT_FILE` overrides them.

- [ ] **Step 1: Failing unit tests** in the crate: lock parsing (good lock, `packages-dev` ignored, bad name, escaping
  autoload path, `path` dist), the classmap scanner (namespaced and global classes, enums, a class name in a string or a
  comment is not one, heredocs, `::class` is not a declaration), the `php` constraint forms, and the generated
  autoloader for each map kind.
- [ ] **Step 2: Failing integration tests** in `tests/composer.rs`, with lock fixtures whose dists the test
  `Server` serves: `a_psr4_dependency_loads` (a plugin command uses a class from its vendor package),
  `files_autoload_runs`, `classmap_autoload_loads`, `a_bad_sha1_is_refused_and_keeps_the_old_vendor`,
  `an_unsafe_dist_is_refused`, `a_missing_extension_names_the_package`, `no_lock_with_requirements_is_refused`,
  `self_plugin_install_installs_dependencies`, `a_failed_dependency_install_removes_the_plugin`,
  `composer_install_for_a_development_directory`, `composer_dump_autoload_rebuilds_the_maps`, `composer_show_lists_packages`,
  `composer_validate_names_problems`, `an_unsupported_composer_subcommand_is_a_usage_error`, `two_versions_of_one_package_warn`,
  `a_composer_plugin_is_activated_and_hears_the_events` (a fixture `composer-plugin` subscribing to
  `post-package-install` and `post-autoload-dump` writes what it heard, in order), `composer_plugins_need_permission`,
  `a_php_callable_root_script_runs`, `a_shell_script_is_reported_not_run`, `an_unsupported_composer_api_names_it`,
  and `a_composer_plugin_fatal_error_fails_the_install`.
- [ ] **Step 3: Run** `cargo test -p speedyspec-composer` and `cargo test -p speedyspec-cli --test composer`.
  Expected: fail.
- [ ] **Step 4: Implement.** Expected: pass; workspace, clippy, fmt (rustfmt on this session's files only), Linux
  suite. Commit: `feat: install a plugin's composer dependencies from its lock file`.

### Task 4: The setup phase of `app install`

**Files:** `crates/speedyspec-cli/src/setup.rs` (new), `crates/speedyspec-cli/src/{app.rs,installs.rs}`,
`crates/speedyspec-cli/tests/setup.rs` (new), fixture `tests/fixtures/plugins/setup-app/` (a zip app served by the
test's `Server`, whose plugin returns a `SetupPlan` of three tasks in its own `tasks/` directory), `README.md`.

**Interfaces:**
- `installs::Record` gains `status: Status` (`Files` | `Installed`, JSON `"files"` | `"installed"`) and
  `setup_done: u64`. Records without them read as `Installed` and `0`, so schema 1 stands.
- `setup::plan(runtime, &Install, request: &Value) -> Result<Option<SetupPlan>, String>` filters
  `speedyspec_app_setup` from null with the `Install` and the `InstallRequest`, validated per answer and blamed on its
  plugin, like `app_target::ask`.
- `setup::run(runtime, &Install, &SetupPlan, from: usize, progress) -> Result<(), SetupFailure { done: usize,
  message }>` runs tasks `from..` through `runner::run` in the install root. It prints
  `setup: <task file name> (<n>/<total>)` on stderr before each task and the task's result on stderr when it is a
  string. It stops at the first failure.
- `app install` order for a local install: the record is written with `status: files, setup_done: 0` right after the
  driver's install. Then come the setup plan, `speedyspec_install_after`, and the record rewritten as
  `installed` (or `files, setup_done: <done>` on failure, exit 1 with `speedyspec: setup stopped at <task>: <message>;
  run the same command again to finish it`). A plan for an install with no local root is not run:
  `speedyspec: setup for <driver> installs is not supported yet; skipped`.
- Re-running: when a record with the same name, app and location has `status: files`, the driver's install is
  skipped (`speedyspec: files already in place; finishing setup`) and setup starts at `setup_done`. Options are
  resolved again as usual, since the record holds none.

- [ ] **Step 1: Failing tests** in `tests/setup.rs`:
  - `setup_tasks_run_in_order_after_the_files`: each task appends its index to `order.txt` in the root, giving
    `1 2 3`. The record says `installed`.
  - `a_failing_task_stops_setup_and_is_named` (task 2 throws): exit 1, stderr names `task2.php` and the exception
    message, `order.txt` is `1`, and the record has `files` and `setup_done: 1`.
  - `running_again_finishes_from_the_failed_task`: after the failure, the fixture's failure switch file is removed
    and the command is run again. The server sees no second download, `order.txt` becomes `1 2 3`, and the record
    says `installed`.
  - `the_secret_option_reaches_the_task_and_nowhere_else`: a `secret` option `db-password=s3cr3t-'"$` reaches task 1
    (which writes it to a file in the root). It is absent from stdout, stderr, the record, and (on Linux) the
    runner's `/proc/<pid>/cmdline` and `environ`.
  - `no_setup_plan_means_no_setup`: an app without one installs as before.
- [ ] **Step 2: Run** `cargo test -p speedyspec-cli --test setup`. Expected: fail.
- [ ] **Step 3: Implement.** Expected: pass, plus the existing `install`, `installs` and `kubernetes` tests;
  workspace, clippy, fmt, Linux suite. Commit: `feat: finish an install with its app's setup tasks`.

### Task 5: `plugin` and `theme`: list, install, update, delete

**Files:** `crates/speedyspec-cli/src/{extensions.rs,extension_cmd.rs}` (new), `crates/speedyspec-cli/src/commands.rs`
(`RESERVED` gains `theme`; the two commands in the tree and dispatch), `crates/speedyspec-cli/tests/extensions.rs`
(new), fixtures `tests/fixtures/plugins/ext-app/` (layout `plugin => ext/plugins`, `theme => ext/themes`; `inspect`
reads a one-line `Name: … Version: …` file) and `ext-source/` (a source whose `resolve` reads the server URL from
`SPEEDYSPEC_TEST_SOURCE`), `README.md`.

**Interfaces:**
- Commands (both kinds alike, `--install <name>` required, zip installs only through `Target::root`):
  ```
  speedyspec plugin list                     --install <n>   # slug <TAB> name <TAB> version, sorted by slug
  speedyspec plugin install <slug>[@<ver>]   --install <n>
  speedyspec plugin update <slug> | --all    --install <n>
  speedyspec plugin delete <slug> [--yes]    --install <n>
  ```
- `extensions::layout(runtime, &Target) -> Result<ExtensionLayout, String>` filters `speedyspec_app_extensions` from
  null with the `Install`. Null gives `no plugin says where <app>'s plugins and themes are; install the plugin for
  that app`. A layout without the asked kind gives `<app> has no themes` (or plugins).
- `extensions::resolve(runtime, &Install, kind, slug, version) -> Result<ExtensionArchive, String>` filters
  `speedyspec_extension_sources` from `[]` with the `Install`, calls each `resolve` in order (with
  `call_catching`, output to stderr), and takes the first non-null answer. Each answer is decoded and must match
  the kind and slug asked for. If none answers: `no source has the <kind> <slug>[ at version <v>]`.
- `extensions::list(runtime, &layout, kind, root) -> Result<Vec<Extension>, String>` calls `inspect` for each entry
  of the directory, skips nulls, and requires each returned slug to equal the entry's name (a file's name without
  `.php`).
- On-disk work, all inside `<root>/<dir>`:
  - Unpack into a fresh `.speedyspec-<random>` temp directory in `<root>/<dir>` with `fetch_and_extract` (checked
    paths and links, `strip_components: 0`). Require exactly one top-level entry, a directory, then rename it to
    `<dir>/<slug>`. The temp directory is removed in every outcome.
  - `install` refuses when `<dir>/<slug>` exists (`the <kind> <slug> is already installed; use update`).
  - `update` requires it to exist. It compares the installed version (from `inspect`) with the archive's and reports
    `<slug> is already at <v>` when they are equal. Otherwise it renames the old directory to
    `.speedyspec-old-<random>`, renames the new one in and removes the old one. If the second rename fails, the
    first is undone.
  - `update --all` updates every listed extension whose source answers. Ones no source has are reported as
    `skipped <slug>: no source has it`. A failure is reported and the rest go on; the exit code is 1 if any failed.
  - `delete` asks `Delete the <kind> <slug> from <install>? [y/N]` (refused without `--yes` when not interactive),
    then removes `<dir>/<slug>`, or `<dir>/<slug>.php` for a single-file plugin, and prints `deleted <kind> <slug>`.
    The CLI gives no app-specific advice; the WordPress plugin's README says that WordPress deactivates a plugin whose
    files are gone.
- Bad slugs (`slug_problem`) and a `@` with an empty version are usage errors (exit 2) before any hook is called.

- [ ] **Step 1: Failing tests** in `tests/extensions.rs` (`testkit::Server` serving zips built with `build_zip`):
  `install_then_list`, `install_a_given_version`, `install_refuses_an_existing_one`, `install_renames_the_top_directory`,
  `archives_that_do_not_match_are_refused_and_leave_nothing` (each Review Focus 2 case; afterwards the directory holds
  no `.speedyspec-*` entry), `slugs_that_are_paths_are_usage_errors` (each Review Focus 1 case; the server gets no
  request and the directory is unchanged), `update_replaces_the_files`, `update_to_the_same_version_says_so`,
  `a_failed_update_keeps_the_old_version` (404, bad checksum and unsafe archive), `update_all_goes_on_after_a_failure`,
  `delete_needs_yes_when_not_interactive`, `delete_removes_a_single_file_plugin`, `no_layout_is_named`,
  `no_source_is_named`, `a_source_answering_another_slug_is_refused`, and `theme_commands_use_the_theme_directory`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-cli --test extensions`. Expected: fail.
- [ ] **Step 3: Implement.** Expected: pass; workspace, clippy, fmt, Linux suite. Commit:
  `feat: plugin and theme commands that install, update, delete and list files`.

### Task 6: The WordPress app plugin

**Files:** new repository `packages/speedyspec-cli-wordpress`: `speedyspec-plugin.json` (`name`
`speedyspec/wordpress`, `contracts` `^1.2`, entry `plugin.php`), `composer.json` (Pest; contracts from
`../speedyspec-cli-contracts` as a path repository), `plugin.php`, `src/{Options,ConfigFile,Headers,Layout}.php`,
`tasks/write-config.php`, `tasks/read-config.php`, `tasks/stub/wp-settings.php` (empty), `tests/*.php`, `README.md`,
`LICENSE` (BSD-3-Clause, as the contracts). In the CLI: `examples/install-wordpress/README.md` points to the new
repository.

**Interfaces (hooks, each answering only for `app === 'wordpress'`):**
- `speedyspec_apps`: `App('wordpress', 'WordPress', [], ['zip'])`.
- `speedyspec_driver_zip`: `ZipSource(<downloads>/latest.zip, '.', stripComponents: 1)`, where `<downloads>` is
  `SPEEDYSPEC_WORDPRESS_DOWNLOADS` or `https://wordpress.org`.
- `speedyspec_install_options`: `db-name` (required), `db-user` (required), `db-password` (required, secret),
  `db-host` (default `localhost`), `table-prefix` (default `wp_`).
- `speedyspec_app_setup`: `SetupPlan([AppTask(tasks/write-config.php, <the five options>, loadsApp: false)])`.
- `speedyspec_app_database`: `AppTask(tasks/read-config.php, loadsApp: false)`.
- `speedyspec_app_extensions`: `ExtensionLayout(['plugin' => 'wp-content/plugins', 'theme' =>
  'wp-content/themes'], Layout::inspect(...))`.

**Interfaces (PHP classes, `SpeedySpec\CliWordPress\`):**
- `ConfigFile::render(string $sample, array $values, array $salts): string` makes `wp-config.php` from
  `wp-config-sample.php` the way WordPress's own `setup-config.php` does. It replaces `database_name_here`,
  `username_here`, `password_here`, `localhost` and the `$table_prefix` value, escaping with
  `addcslashes($value, "\\'")`, and replaces each `put your unique phrase here` with its own salt. It throws when an
  expected line is missing from the sample, so a changed sample fails loudly. `ConfigFile::salts(): array` gives 8
  salts of 64 characters from `random_bytes`.
- `tasks/write-config.php` returns `fn (array $input): string`. When `wp-config.php` exists it returns
  `wp-config.php already exists; left as it is`. Otherwise it writes the rendered file with the sample's
  permissions (php-fpm, often another user, must read it) and returns `wrote wp-config.php`. Its errors never include
  input values.
- `tasks/read-config.php` returns `fn (): array`. It defines `ABSPATH` as the absolute path of `tasks/stub/`, includes
  the root's `wp-config.php`, and returns `host`, `port`, `socket`, `database`, `user`, `password`, `tablePrefix` and
  `charset` (`DB_CHARSET`, or `utf8mb4` when empty). `DB_HOST` forms: `host`, `host:port`, `host:/path/to.sock`,
  `:/path/to.sock`, `[::1]:3307`.
- `Headers::plugin(string $file): ?array` and `Headers::theme(string $styleCss): ?array` read `Plugin Name` / `Theme
  Name` and `Version` from the first 8 KiB, as WordPress's `get_file_data` does. `Layout::inspect` uses them: a plugin
  directory is the first `.php` file at its top level with a `Plugin Name`; a single `.php` file in the plugins
  directory is a single-file plugin; a theme is a directory with a `style.css` that has a `Theme Name`.

- [ ] **Step 1: Failing Pest tests:** `ConfigFile` against the real `wp-config-sample.php` committed as a fixture
  (`tests/fixtures/wp-config-sample.php`, from the WordPress release used in Task 8). Every Review Focus 5 value round
  trips through `render`, then `PhpToken::tokenize($code, TOKEN_PARSE)` (a parse error throws), then `read-config.php` against the rendered file. Salts are distinct and the
  placeholder is gone. A sample missing `password_here` throws. Also each `DB_HOST` form, `Headers` on Hello Dolly-style
  and Akismet-style headers, CRLF files, a missing name gives null, and `Layout::inspect` for the three shapes.
- [ ] **Step 2: Run** `vendor/bin/pest`. Expected: fail.
- [ ] **Step 3: Implement** the classes, tasks and `plugin.php` (thin wiring of the hooks to the classes). Expected:
  Pest passes.
- [ ] **Step 4: Failing CLI-level test in this repository**, `tests/cli.sh`, run with `SPEEDYSPEC_BIN` set to a CLI
  build from `feat/wordpress-files`. It serves a minimal fake WordPress zip (with the real sample and an empty
  `wp-content/plugins`) from a loopback server, runs `app install wordpress --driver zip --base <tmp> --no-interaction`
  with the five options, and checks:
  - `wp-config.php` parses (`speedyspec php` running `PhpToken::tokenize($code, TOKEN_PARSE)` on it).
  - `self database --install wordpress` prints the host, database and prefix without the password.
  - Running the install again reports `left as it is` and the file is unchanged.
  - A failure in `write-config.php` (an unwritable root) does not print the password.

  Ruling: the loopback server is `speedyspec php` running a 20-line PHP socket server from `tests/serve.php`, because
  nothing may be installed on the host. If the embedded PHP cannot bind a socket, use `docker run` with the
  `php:8.5-cli` image instead and record a ruling.
- [ ] **Step 5: Run** it. Expected: pass. Commit in the new repository: `feat: the WordPress app plugin for speedyspec`.
  Commit in the CLI: `docs: point the WordPress example to speedyspec-cli-wordpress`.

### Task 7: The wordpress.org source plugin

**Files:** new repository `packages/speedyspec-cli-wordpress-org`: `speedyspec-plugin.json` (`name`
`speedyspec/wordpress-org`, `contracts` `^1.2`), `composer.json`, `plugin.php`, `src/{Api,Source}.php`,
`tests/*.php`, `tests/fixtures/{plugin-akismet,theme-twentytwentyfive,plugin-not-found,theme-not-found}.json`
(trimmed copies of real API answers), `README.md`, `LICENSE`.

**Interfaces:**
- `speedyspec_extension_sources`: appends `ExtensionSource('wordpress.org', Source::resolve(...))`.
- `composer.json` requires `guzzlehttp/guzzle` `^7`, and `composer.lock` is committed (the CLI installs from it).
- `Api` takes a `GuzzleHttp\ClientInterface` (production: a `Client` with a 30 s timeout; tests: Guzzle's
  `MockHandler`) and a base (`SPEEDYSPEC_WORDPRESS_ORG_API` or `https://api.wordpress.org`):
  - plugins: `<base>/plugins/info/1.2/?action=plugin_information&request[slug]=<slug>&request[fields][versions]=1`
  - themes: `<base>/themes/info/1.2/?action=theme_information&request[slug]=<slug>&request[fields][versions]=1`
- `Source::resolve(string $kind, string $slug, ?string $version): ?ExtensionArchive`. Latest is `version` with
  `download_link`; a given version is `versions[$version]`. A not-found answer (an `error` key) or an unknown version
  is null. Any other failure (network, bad JSON, missing fields) throws `\RuntimeException` naming wordpress.org and the
  slug, which the CLI reports. `sha256` is null: wordpress.org publishes none.
- `rawurlencode` for the slug; only `https://` download links (or loopback under the test base) are returned.

- [ ] **Step 1: Failing Pest tests** with fixture fetchers: latest plugin, a given plugin version, unknown version is
  null, not found is null for both kinds, latest theme, a broken JSON body throws naming the slug, a non-https
  download link throws, and the requested URL is exactly the documented one.
- [ ] **Step 2: Run** `vendor/bin/pest`. Expected: fail.
- [ ] **Step 3: Implement.** Expected: pass.
- [ ] **Step 4: Failing CLI-level test** `tests/cli.sh` (same server approach as Task 6) with both plugins loaded on
  a fake WordPress install: `plugin install akismet`, `plugin list` shows `akismet <TAB> Akismet Anti-spam <TAB>
  <ver>`, `plugin update akismet` says `already at`, an older pinned version installed then updated, and `theme install
  twentytwentyfive` then `theme delete twentytwentyfive --yes`. All of it is served from fixtures on loopback.
- [ ] **Step 5: Run** it. Expected: pass. Commit: `feat: the wordpress.org source plugin for speedyspec`.

### Task 8: End to end against a real WordPress release, and docs

**Files:** `packages/speedyspec-cli-wordpress/scripts/e2e.sh`, `README.md` of both new repositories; CLI `README.md`
(the setup phase, `plugin` and `theme`, composer dependencies, and that the CLI never runs WordPress).

**Interfaces:**
- `scripts/e2e.sh` (manual; downloads from wordpress.org; needs `SPEEDYSPEC_BIN` and `scripts/mysql.sh up` from the
  CLI). It installs the pinned release (`WORDPRESS_VERSION` and its `sha1` from `https://wordpress.org/wordpress-<v>.zip.sha1`,
  checked by the script, then served as `latest.zip` from the Task 6 loopback server through
  `SPEEDYSPEC_WORDPRESS_DOWNLOADS`) against the MySQL container, then runs `app db tables`
  (empty: setup does not run `wp_install()`) and `app db size`. Then it runs `plugin install hello-dolly`, `plugin
  update --all`, `theme install twentytwentyfive` and `plugin delete hello-dolly --yes`, all against the real
  api.wordpress.org. It prints the path for opening the site with php-fpm.

- [ ] **Step 1: Write** `e2e.sh` and the docs.
- [ ] **Step 2: Run** the CLI workspace, Linux suite, clippy, fmt, both plugin repositories' Pest suites and
  `tests/cli.sh`. Expected: all pass. Leave `e2e.sh` for Jacob to run, or run it only with his go-ahead (it downloads
  from wordpress.org). Commit each repository: `docs: …` / `test: add a manual end-to-end script`.

### Task 9: Pin contracts 1.2.0

Blocked until Jacob merges the contracts PR and its release. Ask when Tasks 1 to 8 are done.

**Files:** `contracts-version`, fixture manifests that need `^1.2`, tests that name the bundled version (`tests/cli.rs`,
`tests/plugin_install.rs`), and both plugin repositories' `composer.json` (contracts from the release instead of the
path repository).

- [ ] **Step 1:** Update `contracts-version` (`version=1.2.0`, the tag's tarball URL and its `sha256`) and the tests
  that name the version.
- [ ] **Step 2: Run** the workspace without `SPEEDYSPEC_CONTRACTS_DIR` (no build warning), the Linux suite and the
  database tests on both servers. Expected: all pass, and `speedyspec --version` shows `contracts 1.2.0`. Commit:
  `build: embed contracts 1.2.0`.
