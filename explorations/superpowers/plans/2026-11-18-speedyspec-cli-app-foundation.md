# SpeedySpec CLI — App Maintenance Foundation Implementation Plan (plan 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The mechanisms every app maintenance command rests on, proven on zip installs with fixture apps:
installs findable by name (`app installs`), the new contracts and hooks, a Rust MySQL client behind
`app db export | import | query | tables | size | check --install <name>`, the PHP extensions WordPress needs, and the
runner that executes an app's own PHP in a child process.

**Architecture:** A zip install now writes a record to `$SPEEDYSPEC_HOME/installs/<name>.json`. Commands that act on
an install load that record, boot the CLI's plugins as usual, and ask the app's plugin through two new filters how
to run the app (`speedyspec_app_runtime`) and how to reach its database (`speedyspec_app_database`). Database work is
done in Rust by a new crate, `speedyspec-db`, on the `mysql` crate. App code runs in a child process: the binary
starts itself as `speedyspec self run-task` (hidden), which boots PHP with the contracts and no CLI plugins. It
loads the app's entry file if asked, runs the task's PHP file, and writes a JSON result with every PHP diagnostic
to a file the parent names. A `DatabaseConnection` can be returned directly or produced by such a task, which is how a
WordPress plugin will read `wp-config.php` in plan 6.

**Tech Stack:** Rust 1.99 (edition 2024); `mysql` 28 (`default-features = false`, features `minimal-rust`,
`rustls-tls-ring`); `serde_json`; `tempfile`; static-php-cli as before, plus database and XML extensions; Docker
images `mysql:8.4` and `mariadb:11.4` for the database tests; Pest for the contracts package.

**Spec:** `docs/superpowers/specs/2026-11-18-speedyspec-cli-app-maintenance-design.md` (approved 2026-11-18). It builds
on `docs/superpowers/specs/2026-11-17-speedyspec-cli-design.md`, which still binds where the new one is silent. This
plan is item 1 of the design's "Order of work".

## Global Constraints

- Repositories: `packages/speedyspec-cli` (`SpeedySpec/speedyspec-cli`, branch `feat/app-foundation` from `main`) and
  `packages/speedyspec-cli-contracts` (`SpeedySpec/speedyspec-cli-contracts`, branch `feat/app-contracts` from `main`).
- **Nothing outward-facing without Jacob's go-ahead:** no push, no pull request, no tag, no release, no workflow run.
  Read-only `gh` calls are fine.
- **Never touch a real database.** Tests connect only to the throwaway containers that `scripts/mysql.sh` starts
  (loopback ports, container names `speedyspec-mysql` and `speedyspec-mariadb`, label `speedyspec.mysql=1`). The
  database tests are `#[ignore]` and read their connection only from `SPEEDYSPEC_E2E_MYSQL` / `SPEEDYSPEC_E2E_MARIADB`.
  They fail, never fall back to anything else, when those are unset.
- **Secrets:** a database password is never printed, logged, recorded in an install record, put in a process argument
  or environment variable of the runner, or included in an error message. It travels to the runner only inside the
  request file (mode 0600, deleted after use) when a task needs it.
- Nothing installed on the host; Docker images may be pulled. Never set `SPEEDYSPEC_ALLOW_HOST_INSTALL` on the host.
- The CLI knows nothing about WordPress. Fixtures in this plan are tiny fake apps; the WordPress app plugin is plan 6.
- Contracts package changes are additive (a minor release, `1.1.0`): existing classes do not change.
- Stage explicit paths, never `git add -A`. Conventional Commits with no breaking-change markers. Commit messages end
  with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; PR bodies end with
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
- `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --all --check` clean after every task; the
  Linux suite (`scripts/linux.sh cargo test --workspace`) at the end of each task that touches Rust.

## Review Focus

Each line has a test in the task named in brackets.

1. **Data that does not survive a dump.** Binary columns, NUL bytes, `NULL` versus the string `'NULL'`, 4-byte
   UTF-8 (emoji), backslashes and quotes, zero dates, very long rows, a table with no rows, a table name that needs
   quoting. Export then import must give identical tables, by `CHECKSUM TABLE`, on MySQL and MariaDB. Export streams,
   so a large table is never held in memory. [Task 4]
2. **Importing what people actually have.** A dump from `mysqldump` (conditional comments `/*!40101 … */`, `LOCK
   TABLES`, multi-row inserts, `--` and `#` comments), a dump with a semicolon inside a string, and a broken dump. A
   broken dump stops at the failing statement and names its line; the command says that statements before it have
   been applied (MySQL commits DDL at once). [Task 4]
3. **The password escaping.** Through errors from a refused connection, the `query` output, `--help`, the install
   record, the runner's arguments and environment, and a panic. [Task 4, Task 6]
4. **App code that misbehaves in the runner**: `exit()` or `die()`, a fatal error, an uncaught exception, output on
   stdout, its own error handler, changing the working directory, or never finishing. The parent always gets either a
   result or a clear error, with the diagnostics so far. A task that runs past its timeout is killed, and no child is
   left behind. [Task 5]
5. **Install records that disagree with the disk.** Two installs with the same name, a record for a base directory
   that is gone, a failed install, and a relative or unset `SPEEDYSPEC_HOME`, which follows the same rules as the
   plugins directory. [Task 1]

## File Structure

```
packages/speedyspec-cli-contracts/
  src/Install.php, src/AppRuntime.php, src/DatabaseConnection.php, src/AppTask.php
  tests/ContractsTest.php                          + cases for the four
packages/speedyspec-cli/
  scripts/mysql.sh                                 up | down | env: throwaway MySQL 8.4 and MariaDB 11.4 in Docker
  scripts/build-php.sh                             + database and XML extensions
  crates/speedyspec-plugin/src/discover.rs         home_dir() shared by plugins and installs
  crates/speedyspec-driver/src/app.rs              Install, AppRuntime, DatabaseConnection, AppTask mirrors + decoders
  crates/speedyspec-db/                            new: connect, export, import, query, tables, size, check
  crates/speedyspec-cli/src/installs.rs            install records: write, read, list
  crates/speedyspec-cli/src/runner.rs              parent side: run a task in a child process
  crates/speedyspec-cli/src/runner_child.rs        child side: `self run-task`
  crates/speedyspec-cli/src/app_target.rs          an install's runtime and database, through the hooks
  crates/speedyspec-cli/src/db_cmd.rs              `app db …`
  crates/speedyspec-cli/tests/{installs,runner,app_db}.rs
  crates/speedyspec-cli/tests/fixtures/apps/       tiny fake apps; fixtures/plugins/db-app, db-app-task
  .github/workflows/ci.yml                         + a database job
```

---

### Task 1: Install records and `app installs`

**Files:** `crates/speedyspec-plugin/src/discover.rs`, `crates/speedyspec-cli/src/installs.rs` (new),
`crates/speedyspec-cli/src/{app.rs,commands.rs}`, `crates/speedyspec-driver-zip/src/driver.rs`,
`crates/speedyspec-cli/tests/installs.rs` (new), `README.md`.

**Interfaces:**
- `speedyspec_plugin::home_dir() -> HomeDir { dir: Option<PathBuf>, warnings: Vec<String> }`. The rule that
  `user_plugin_dir` uses today (absolute `SPEEDYSPEC_HOME`, else the platform data directory, else none, with a
  warning for a relative value) moves here; `user_plugin_dir` becomes `home_dir().dir.join("plugins")`.
- The zip driver gains a `--name <install>` flag (default: the app id; same validation as the Kubernetes driver's
  install name, reusing `speedyspec_driver_kubernetes::install_name_problem`'s rule by moving it to
  `speedyspec_driver`).
- `installs::Record { schema: 1, name, app, parts: Vec<String>, driver: String, location: String, cli_version,
  installed_at: String (RFC 3339 UTC), status: "installed" }` (serde JSON; `location` is the absolute base path for
  zip). `installs::write(&Record) -> Result<(), String>` writes `<home>/installs/<name>.json` atomically (temp file in
  the same directory, then rename). `installs::read(name) -> Result<Record, String>`. `installs::list() ->
  Vec<Record>` skips and warns about unreadable files.
- `app install --driver zip` writes the record after the files are in place and before `speedyspec_install_after`.
  A failed install writes nothing. A record that already exists for the same name with a different location or app
  is refused before anything is downloaded: `an install named <n> already exists (app <a>, at <loc>); choose another
  --name, or remove <path>`. The same name, app and location is replaced (a re-install).
- `speedyspec app installs` prints `name <TAB> app <TAB> driver <TAB> location` for each record, sorted by name, then
  on stderr `speedyspec: Kubernetes installs are kept in their cluster; app uninstall finds them there`.

- [ ] **Step 1: Failing tests** in `tests/installs.rs` (using `common::Speedyspec` and the zip fixtures that exist):
  `zip_install_writes_a_record`; `install_name_defaults_to_the_app`; `same_name_elsewhere_is_refused_before_downloading`
  (the server saw no request); `reinstall_replaces_the_record`; `failed_install_writes_no_record`;
  `app_installs_lists_records_sorted`; `unreadable_record_is_skipped_with_a_warning`;
  `relative_home_warns_like_the_plugins_dir`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-cli --test installs` Expected: fail (no `--name`, no `installs`).
- [ ] **Step 3: Implement.** Also move the install-name rule to `speedyspec-driver` and point both drivers at it.
- [ ] **Step 4: Run** the crate tests, then `cargo test --workspace`. Expected: all pass.
- [ ] **Step 5: README**: `--name` for zip, `app installs`, where records live.
- [ ] **Step 6: Commit** `feat: record zip installs and list them with app installs`.

---

### Task 2: Contracts 1.1.0 and their Rust mirrors

**Files:** in `speedyspec-cli-contracts`: `src/{Install,AppRuntime,DatabaseConnection,AppTask}.php`,
`tests/ContractsTest.php`, `README.md`. In the CLI: `crates/speedyspec-driver/src/app.rs` (new) and `lib.rs`.

**Interfaces (PHP, `final readonly`, in the existing style):**
- `Install(string $name, string $app, array $parts, string $driver, ?string $root)`: `$root` is the local directory
  for zip and null for drivers whose files are elsewhere.
- `AppRuntime(?string $entry = null, array $ini = [], array $server = [])`: `$entry` relative to the install root;
  `$ini` and `$server` are string maps (php.ini settings, `$_SERVER` entries set before the entry file loads).
- `DatabaseConnection(string $host, string $database, string $user, string $password, int $port = 3306,
  ?string $socket = null, string $tablePrefix = '', string $charset = 'utf8mb4', string $engine = 'mysql')`.
- `AppTask(string $file, array $input = [], bool $loadsApp = true, int $timeoutSeconds = 600)`: `$file` is an absolute
  path to a PHP file that returns a `callable(array $input): mixed`; the result must be JSON-encodable.

**Interfaces (Rust, `speedyspec_driver::app`):** `Install`, `AppRuntime`, `DatabaseConnection` (with a `Debug` that
prints `password: "<redacted>"`), `AppTask { file: PathBuf, input: serde_json::Value, loads_app: bool,
timeout: Duration }`. Decoders: `decode_app_runtime`, `decode_database_connection`, `decode_app_task`, and
`decode_database_or_task -> Result<DatabaseOrTask, ContractError>` (`enum DatabaseOrTask { Connection(..), Task(..) }`).
All use the existing `Decoder` and `ContractError` paths. Rules: `entry` relative and inside the root (reuse the zip
driver's destination rule); `engine == "mysql"` (MariaDB too); `port` 1–65535; `charset` matches `[A-Za-z0-9_]+`;
`tablePrefix` matches `[A-Za-z0-9_]*`; the task file is absolute and exists; `timeoutSeconds` is 1–86 400.
(`speedyspec-driver` does not depend on PHP; the CLI builds the PHP `Install` object for the hooks with
`Php::construct`.)

- [ ] **Step 1: Failing Pest tests** for the four classes (construction, defaults, readonly) and **failing Rust unit
  tests** in `app.rs`: each rule above accepted and rejected with its field path; and `debug_never_shows_the_password`.
- [ ] **Step 2: Run** `vendor/bin/pest` in the contracts repo and `cargo test -p speedyspec-driver app::`. Expected:
  both fail.
- [ ] **Step 3: Implement** the PHP classes, with docblocks in the coding style, and the Rust module.
- [ ] **Step 4: Run** both again, and the CLI workspace with `SPEEDYSPEC_CONTRACTS_DIR=../speedyspec-cli-contracts`.
  Expected: pass, with the build warning that the contracts are not the pinned version (expected until Task 8).
- [ ] **Step 5: Commit** in the contracts repo: `feat: add Install, AppRuntime, DatabaseConnection and AppTask`. In
  the CLI: `feat(driver): mirror the app contracts`.

---

### Task 3: The PHP extensions WordPress needs

**Files:** `scripts/build-php.sh`, `crates/speedyspec-php/tests/{php.rs,fixtures/extensions.php}`, `docs/php-embed.md`.

**Interfaces:** the extension list becomes
`ctype,curl,dom,exif,fileinfo,filter,iconv,mbstring,mysqli,mysqlnd,openssl,pdo,pdo_mysql,phar,simplexml,sodium,tokenizer,xml,xmlreader,xmlwriter,zip,zlib`
(the craft stamp changes, so PHP is rebuilt everywhere and CI's cache key changes with the script's hash).

- [ ] **Step 1: Failing harness case** `extensions_wordpress_needs` with `extensions.php` printing, for each of
  `mysqli`, `mysqlnd`, `pdo_mysql`, `dom`, `simplexml`, `xmlreader`, `xmlwriter`, `sodium`, `iconv`, `fileinfo`,
  `exif`: `extension_loaded(...) ? 'y' : 'n'`, and `in_array('mysql', PDO::getAvailableDrivers()) ? 'y' : 'n'`.
  Expected output: twelve `y`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-php --test php` Expected: the case fails (`nnnnnnnnnnnn` or a missing
  class).
- [ ] **Step 3: Rebuild** `scripts/build-php.sh` (macOS), `scripts/linux.sh scripts/build-php.sh` (x86_64), and
  aarch64 as in plan 4 (Docker volume if the shared checkout runs out of file handles). Expected: `libphp.a` built.
- [ ] **Step 4: Run** the harness, the workspace, the Linux suite, then release builds and `scripts/smoke.sh` for the
  three local targets. Record the new sizes beside the old in `docs/php-embed.md`.
- [ ] **Step 5: Commit** `build(php): add the database and XML extensions WordPress needs`.

---

### Task 4: `speedyspec-db`, the MySQL client

**Files:** `crates/speedyspec-db/{Cargo.toml,src/lib.rs,src/dump.rs,src/split.rs}` (new),
`crates/speedyspec-db/tests/{split.rs,mysql.rs}`, `scripts/mysql.sh` (new), `.gitignore`.

**Interfaces:**
- `scripts/mysql.sh up | down | env`: starts `mysql:8.4` on `127.0.0.1:13306` and `mariadb:11.4` on
  `127.0.0.1:13307` (labels and names per the Global Constraints, random root password written to `.mysql/env` mode
  0600), waits until both accept connections, and `env` prints `SPEEDYSPEC_E2E_MYSQL=mysql://root:…@127.0.0.1:13306`
  and the MariaDB line. `down` removes only containers carrying the label.
- `Connection::open(&DatabaseConnection) -> Result<Connection, DbError>` (TLS preferred when the server offers it).
  `DbError`'s `Display` names the host, port, database and what failed, never the password. A refused login maps to
  `access denied for user <user>@<host> to database <db>`.
- `Connection::tables() -> Vec<TableInfo { name, rows_estimate, data_bytes, index_bytes }>`, limited to the
  connection's table prefix when it is not empty.
- `Connection::size() -> (u64 data, u64 index)`.
- `Connection::check() -> Vec<(table, status, message)>`, from `CHECK TABLE`.
- `Connection::query(sql, &mut dyn Write) -> Result<QueryOutcome, DbError>`: tab-separated rows with a header line,
  `NULL` for null, binary values as `0x…`; or `QueryOutcome::Affected(n)`.
- `Connection::export(&mut dyn Write, &ExportOptions { tables: Option<Vec<String>> })`. It writes a header comment
  (CLI version, server version, time) and `SET NAMES utf8mb4; SET FOREIGN_KEY_CHECKS=0;`. Then, per table, `DROP
  TABLE IF EXISTS`, `SHOW CREATE TABLE`, and multi-row `INSERT`s of at most 1 MiB each. Binary and BLOB columns go
  out as `0x…` hex, strings with MySQL escaping, `NULL` for null. Rows are streamed from an unbuffered query.
- `split::Statements::new(impl BufRead)`: an iterator of `(line_number, statement)`. It understands `'…'`, `"…"` and
  `` `…` `` quoting with backslash escapes, `--`, `#` and `/* … */` comments, and keeps `/*!… */` conditional comments
  in the statement. `DELIMITER` is an error naming the line ("stored routines are not supported").
- `Connection::import(impl BufRead) -> Result<u64 statements, ImportError { line, statement_number, applied, error }>`.

- [ ] **Step 1: Failing unit tests** in `tests/split.rs` (no database): a semicolon inside each quote style; escaped
  quotes; each comment form; a conditional comment kept; multi-line statements with their first line number; a
  trailing statement without a semicolon; `DELIMITER` refused with its line.
- [ ] **Step 2: Failing integration tests** in `tests/mysql.rs`, `#[ignore]`, run against both servers:
  - `round_trip_keeps_every_value`: tables holding binary data with NULs, NULL versus `'NULL'`, emoji, backslashes
    and quotes, `0000-00-00` with the server's `sql_mode` allowing it, a 4 MiB text row, an empty table, and a
    table named `` `we;ird` ``. Export, import into a fresh database, and compare `CHECKSUM TABLE` for every table.
  - `imports_a_mysqldump_file`: a checked-in fixture written by `mysqldump` 8.4 (`--opt`).
  - `broken_dump_names_the_line`.
  - `wrong_password_is_reported_without_it`.
  - `query_prints_rows_and_affected_counts`.
  - `tables_respects_the_prefix`.
  - `export_streams`: 200 000 rows, with the export's peak memory staying under 64 MiB (measured with a counting
    writer and the process's RSS before and after).
- [ ] **Step 3: Run** `cargo test -p speedyspec-db` Expected: compile fails. Then `scripts/mysql.sh up` and the
  ignored tests with `eval "$(scripts/mysql.sh env)"` Expected: fail.
- [ ] **Step 4: Implement**; add `.mysql/` to `.gitignore`.
- [ ] **Step 5: Run** the unit tests, the ignored tests against both servers (`--test-threads=1`), clippy, fmt and
  the Linux build. Then `scripts/mysql.sh down`. Expected: all pass, containers gone.
- [ ] **Step 6: Commit** `feat(db): a MySQL client for export, import, query, tables, size and check`.

---

### Task 5: The runner

**Files:** `crates/speedyspec-cli/src/{runner.rs,runner_child.rs,self_cmd.rs,commands.rs}`,
`crates/speedyspec-cli/tests/runner.rs`, `crates/speedyspec-cli/tests/fixtures/apps/{tiny,noisy,dies,fatal,slow}/`.

**Interfaces:**
- Parent: `runner::run(root: &Path, runtime: &AppRuntime, task: &AppTask) -> Result<TaskOutcome, RunnerError>`.
  `TaskOutcome { result: serde_json::Value, diagnostics: Vec<Diagnostic>, output: String }`, where
  `Diagnostic { level: "warning" | "notice" | "deprecated" | "error" | "exception", message, file, line }`.
- Mechanics: it writes a request JSON (`root`, `entry`, `ini`, `server`, `task.file`, `task.input`,
  `loads_app`) to a 0600 temp file. It starts `current_exe()` as `self run-task --request <file> --result <file>`,
  with stdin closed, stdout and stderr captured into `output` (capped at 1 MiB, then truncated with a note), and the
  environment inherited unchanged (an app's config may read it, as `wp-config.php` often does with `getenv()`). The
  runner adds nothing to it, secrets least of all. It waits up to `task.timeout`, kills the child after that, reads
  the result file, and deletes both files on every path.
- A hidden parent-side command, `speedyspec self task --root <dir> --file <task.php> [--entry <file>] [--no-app]
  [--input <json>] [--timeout <s>]`, runs `runner::run` and prints the `TaskOutcome` as JSON (or the error, exit 1).
  It exists so integration tests can drive the runner through the real binary; it is not documented.
- `RunnerError`'s messages: `the task <file> did not finish within <n>s` (diagnostics so far included),
  `the task <file> ended without a result (exit status <code>)` with the last diagnostics and output, `the task
  <file> failed: <message>` for a task that threw.
- Child, `self run-task` (hidden from help, dispatched before boot like `self update`): boots PHP with the contracts
  preloaded and no CLI plugins. It installs an error handler (records, and returns false so PHP's own handling stays
  as it is), an exception handler, and a shutdown function that writes the result file even after `exit()`, `die()`
  or a fatal error (`error_get_last()`). It applies `ini`, sets `$_SERVER` entries and `chdir(root)`, requires the
  entry when `loads_app`, then requires the task file, which must return a callable. It calls it with the input and
  writes `{ "ok": true, "result": … }` or `{ "ok": false, "error": … }` with the diagnostics.

- [ ] **Step 1: Failing tests** in `tests/runner.rs`, each running the built binary's hidden `self task` against a
  fixture app:
  - `runs_a_task_inside_the_app`: the entry defines a function the task calls; the result comes back.
  - `task_without_the_app`: `loads_app: false`, and the entry is not loaded.
  - `warnings_are_reported_with_file_and_line`.
  - `exit_in_the_app_still_gives_a_result`.
  - `fatal_error_is_reported`.
  - `uncaught_exception_is_reported`.
  - `stdout_noise_does_not_corrupt_the_result`.
  - `own_error_handler_takes_over_after_it_is_set`: our handler is installed before the app loads. A warning raised
    before the app sets its own handler is recorded; one after is not (it is the app's to handle); a fatal error after
    it is still recorded, through the shutdown function. The test pins all three.
  - `slow_task_is_killed_at_its_timeout`, and no process is left: check `kill -0` on the child's pid.
  - `request_and_result_files_are_removed`.
  - `password_in_input_is_not_in_argv_or_env`: the task reads `/proc/self/cmdline` and `environ` on Linux, or
    `ps -o args=` plus `getenv` on macOS. The test asserts the password is absent from both, while an environment
    variable set for the parent is present.
- [ ] **Step 2: Run** `cargo test -p speedyspec-cli --test runner` Expected: fail.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the tests, the workspace and the Linux suite. Expected: all pass.
- [ ] **Step 5: Commit** `feat: run an app's own PHP in a child process`.

---

### Task 6: An install's runtime and database, through the hooks

**Files:** `crates/speedyspec-cli/src/app_target.rs` (new), `crates/speedyspec-cli/tests/fixtures/plugins/{db-app,db-app-task}/`,
`crates/speedyspec-cli/tests/fixtures/apps/db-app/`, `README.md` (hooks table).

**Interfaces:**
- `app_target::Target { record: Record, install: Install }`, from `installs::read(name)`.
- `app_target::runtime(&Runtime, &Target) -> Result<AppRuntime, String>`: filters `speedyspec_app_runtime`, seeded
  with null, and called with the `Install` object. A null answer means `no plugin says how to run <app>`.
- `app_target::database(&Runtime, &Target) -> Result<DatabaseConnection, String>`: filters
  `speedyspec_app_database`, seeded with null. An `AppTask` answer is run with `runner::run` and its result decoded
  as a `DatabaseConnection` array (`host`, `database`, `user`, `password`, optional rest). A null answer means `no
  plugin says where the database of <app> is`. Rejections are blamed on the answering plugin, as for the existing
  validated filters.
- Fixtures:
  - `db-app` plugin: offers app `db-app` (zip). Its `speedyspec_app_database` reads `<root>/db.json` with
    `json_decode` and returns a `DatabaseConnection`.
  - `db-app-task` plugin: returns an `AppTask` (`loadsApp: false`) whose file includes `<root>/config.php` with
    `ABSPATH` defined to a stub directory whose `settings.php` is empty. `config.php` defines constants from
    `getenv()`, then requires `ABSPATH . 'settings.php'`, the shape of `wp-config.php`. The task returns the
    connection from those constants. That proves the plan 6 mechanism without WordPress.

- [ ] **Step 1: Failing tests** (in `tests/app_db.rs`, the parts that need no database):
  - `database_from_a_direct_answer`.
  - `database_from_a_task`: the test sets `DB_HOST` and the other variables for the CLI process; `config.php` reads
    them with `getenv()` in the child, as `wp-config.php` does.
  - `no_plugin_answers` gives exit 1 and the message.
  - `bad_answer_is_blamed_on_the_plugin`.
  - `unknown_install` gives exit 1: `no install named <n>; app installs lists them`.
- [ ] **Step 2: Run** them. Expected: fail.
- [ ] **Step 3: Implement**, and add the two hooks to the README's hooks table with their seeds and arguments.
- [ ] **Step 4: Run** the tests and the workspace. Expected: pass.
- [ ] **Step 5: Commit** `feat: ask an app's plugin for its runtime and database`.

---

### Task 7: `app db` commands

**Files:** `crates/speedyspec-cli/src/{db_cmd.rs,commands.rs}`, `crates/speedyspec-cli/tests/app_db.rs`, `README.md`,
`.github/workflows/ci.yml`.

**Interfaces:** `speedyspec app db export [--to <file>] | import <file|-> | query <sql> | tables | size | check
--install <name>`.
- `export` writes to `--to` (created 0600; refused if it exists unless `--force`) or stdout, binary-safe.
- `import` reads a file or stdin, and asks to confirm unless `--yes` (`this replaces tables in database <db> on
  <host>`). On failure it says how many statements were applied.
- `query` prints the rows or `<n> rows affected`.
- `tables`, `size` and `check` print tab-separated lines.
- Exit codes: 0, 1 for failures, 2 for usage. CI gains a `database` job on `ubuntu-latest`: `scripts/mysql.sh up`,
  the ignored database tests of `speedyspec-db` and `speedyspec-cli` with `--test-threads=1`, and `scripts/mysql.sh
  down` in an `always()` step. `permissions: contents: read`.

- [ ] **Step 1: Failing tests** in `tests/app_db.rs`, `#[ignore]` (need `scripts/mysql.sh`). They zip-install the
  `db-app` fixture with a `db.json` pointing at the test server, then:
  - `export_then_import_round_trips`;
  - `export_refuses_an_existing_file`;
  - `import_needs_confirmation`;
  - `query_runs`;
  - `tables_size_and_check_print_lines`;
  - `the_password_never_appears`: every command's stdout and stderr, the install record, and `--help`, with a
    distinctive password.
- [ ] **Step 2: Run** them against both servers. Expected: fail.
- [ ] **Step 3: Implement**; add the CI job; README section "Working with an install's database".
- [ ] **Step 4: Run** everything: the workspace, the Linux suite, the ignored database tests against both servers,
  and actionlint (`rhysd/actionlint:1.7.12`). Expected: all pass. Then `scripts/mysql.sh down`.
- [ ] **Step 5: Commit** `feat: app db commands for an install's database`.

---

### Task 8: Pin contracts 1.1.0

**Gate.** It needs a published `v1.1.0` of `SpeedySpec/speedyspec-cli-contracts`.

- [ ] **Step 1: Check the gate.** `gh release view v1.1.0 --repo SpeedySpec/speedyspec-cli-contracts --json tagName`.
  **If it is missing, stop and ask Jacob.** The contracts branch must be pushed and merged, and release-please's PR
  merged, all by him or with his go-ahead.
- [ ] **Step 2:** Set `contracts-version` to 1.1.0 with the tag archive's URL and its SHA-256 (computed twice), and
  run `scripts/fetch-contracts.sh`.
- [ ] **Step 3: Run** the workspace without `SPEEDYSPEC_CONTRACTS_DIR` (no build warning), the Linux suite, and
  `scripts/smoke.sh` on a release build. Expected: `contracts 1.1.0`.
- [ ] **Step 4: Commit** `build: embed contracts 1.1.0`.

---

## What this plan leaves for later plans

- Plan 6: the WordPress app plugin and the wordpress.org source plugin in their own repositories; finishing setup at
  install; `plugin` and `theme` with multisite; `app sites`; `SetupPlan` and `ExtensionSource`.
- Plan 7: `app backup` and `app restore`, both uses; `BackupSpec`.
- Plan 8: Kubernetes for all of it: the runner Job, `--runner-binary`, database access through the API, the
  driver-neutral file access class.
- Plan 9: the SpeedySpec app plugin and the premise test.
