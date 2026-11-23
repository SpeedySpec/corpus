# WP-Cron

`speedyspec-wp-cron-contracts`, `speedyspec-wp-cron`, `speedyspec-wp-cron-options`, `speedyspec-wp-cron-php`,
`speedyspec-php-runtime` (the database areas and the wake slot), `speedyspec-runtime-contract`, `speedyspec-server`
(`php/cron.rs`, `php/handler.rs`, `config.rs`), `compat/cron`. The port of WordPress 7.1.5's `cron.php` and
`wp-cron.php` to Rust, with a scheduler in the server that calls the app's cron path instead of WordPress's loopback
request. It was built from four plans, merged as #78 and #80 (options in the worker), #86 (the scheduling API, the
runner and the server's scheduler) and #88 (release registration).

These are the Minor findings from each plan's final whole-branch review. None of them makes an event run twice or
lose a write on ordinary input: they concern crafted input, hook counts, test coverage, configuration checks and
documentation.

## Options in the worker

### No test that an open transaction is gone by the next request

- **Where:** `speedyspec-php-runtime`, `worker.rs` (`registry.reset_all()` after each request).
- **What happens:** a request that leaves a transaction open relies on `MySqlStore::reset` (`COM_RESET_CONNECTION`) to
  drop it. Nothing tests that the next request does not see it.
- **Suggested fix:** a MySQL test that opens a transaction in one request and checks `@@in_transaction`, or an
  uncommitted row, in the next.

### `php_worker_open_transaction` does not mean what its name says

- **Where:** `speedyspec-php-runtime`, `worker.rs`; `docs/php.md` ("Native WordPress areas").
- **What happens:** `MySqlStore::reset` ignores rollback errors, so the event is logged only when the connection reset
  itself fails, usually a lost connection. It never marks "a transaction was left open", though the docs read as if
  it logs every rollback.
- **Suggested fix:** rename the event or log the rollback itself; match the docs to it.

### The connection reset delays every response

- **Where:** `speedyspec-php-runtime`, `worker.rs`.
- **What happens:** `reset_all` runs before `end_request`, so the client waits a round trip (and `SET NAMES`) on every
  request.
- **Suggested fix:** run it after `end_request`; the worker still finishes it before accepting the next request.

### Test helpers inherit the caller's database variables

- **Where:** `speedyspec-php-runtime`, `tests/support/mod.rs` (`spawn`).
- **What happens:** a shell or container with `DB_NAME` exported makes `no_db_name_boots_as_before` and
  `worker_serves_the_port` boot with a database.
- **Suggested fix:** remove the `DB_*` variables in `spawn` unless the test passes them.

### An empty `TABLE_PREFIX` is accepted

- **Where:** `speedyspec-php-runtime`, `database.rs` (`database_config`).
- **What happens:** `TABLE_PREFIX=` becomes the prefix `""`, while empty `DB_CHARSET` and `DB_COLLATE` count as unset.
  A `.env` line `TABLE_PREFIX=` most likely means "the default".
- **Suggested fix:** treat empty as unset (`wp_`), or refuse it with a message.

### Nothing shows a prepared release loads beside the native areas

- **Where:** `speedyspec-php-runtime`, `tests/fixtures`.
- **What happens:** the fixtures load stub WordPress functions, not a release prepared with the ports'
  `prepare_release`. Nothing shows a real release boots beside the native options, wpdb and cron functions without
  redeclaring one.
- **Suggested fix:** a test that loads a prepared 7.1.5 release in a worker.

### The MySQL CI job's settings leak into its PHP build

- **Where:** `.github/workflows/ci.yml` (`mysql` job), `scripts/linux-php.sh`.
- **What happens:** the job's `SPEEDYSPEC_TEST_MYSQL_URL` also puts the "Build static PHP" step on `--network host`
  (harmless). It shares the `php` job's cache key, so one of the two cache saves may log "unable to reserve cache".
- **Suggested fix:** set the variable on the test step only; give the job its own cache key or restore-only access.

## The scheduling API

### `Unchanged` gives up without checking for a stale cache

- **Where:** `speedyspec-wp-cron`, `write.rs` (`commit`).
- **What happens:** a write the options port reports as unchanged returns `could_not_set` at once, even when a stale
  cache caused it (unscheduling an event the cache lacks but the database holds). WordPress does the same.
- **Suggested fix:** when the stored bytes differ from the version read, reload and retry instead.

### `wp_schedule_single_event` reads its parameters, not the event a filter changed

- **Where:** `speedyspec-wp-cron`, `write.rs` (`schedule_single_event`).
- **What happens:** the key and the duplicate window come from the parameters. WordPress reads `$event->timestamp`,
  `$event->hook` and `$event->args` after `pre_schedule_event`, which a filter can change through the object handle.
- **Suggested fix:** read them back from the event object after the filter.

### `wp_reschedule_event`'s arithmetic differs on unusual input

- **Where:** `speedyspec-wp-cron`, `write.rs` (`plan_reschedule`).
- **What happens:** a fractional timestamp is truncated before subtracting (PHP truncates the difference), so the
  next time can be a second off. A numeric-string interval gives a float timestamp where PHP gives an integer. A zero
  interval throws `Error`, not `DivisionByZeroError`.
- **Suggested fix:** follow PHP's order of conversions and its exception class.

### Writes into junk entries differ from PHP

- **Where:** `speedyspec-wp-cron`, `php.rs` (`set3`), `write.rs` (`unschedule_hook`).
- **What happens:** writing an event under a timestamp whose entry is a string replaces the string with an array,
  where PHP fatals. `wp_unschedule_hook` keeps `null` and `false` timestamp entries that WordPress's `empty()` removes.
- **Suggested fix:** follow WordPress for both, or record the difference in `compat/cron/breaks.json`.

### A `cron_schedules` filter that returns a non-array

- **Where:** `speedyspec-wp-cron`, `schedules.rs`.
- **What happens:** the port treats it as empty; WordPress throws a `TypeError` from `array_merge`.
- **Suggested fix:** throw the same error.

### A losing insert is logged as a database error

- **Where:** `speedyspec-wp-options-sql`, `repository.rs` (`add_if_absent`).
- **What happens:** when two processes add the `cron` option at once, the losing `INSERT` is not quiet, so wpdb
  records (and under `WP_DEBUG` prints) a "Duplicate entry" error.
- **Suggested fix:** send it quietly, as the reads are.

### Gaps in the recorded scenarios

- **Where:** `compat/cron/scenarios.py`.
- **What happens:** the watched hooks leave out `pre_option_cron` and `option_cron`, so a change in how often the port
  reads the option would not show. No scenario passes PHP objects as arguments; only `speedyspec-wp-cron-php`'s
  `tests/php.rs` has one, with a single `ArrayObject`.
- **Suggested fix:** watch both hooks; add scenarios with repeated and nested objects.

### The write-storm behaviour is not in the user-facing docs

- **Where:** `docs/php.md`.
- **What happens:** under a sustained burst of concurrent writes, a schedule call can return `could_not_set` (or
  `false`) after five conflicts, where WordPress would silently lose one write. The design and the pull request say
  so; the docs do not.
- **Suggested fix:** say it in the cron section of `docs/php.md`.

## The runner

### A filter that rewrites `cron` on read defeats the claim

- **Where:** `speedyspec-wp-cron`, `run.rs` with `write.rs` (`commit`).
- **What happens:** when a filter rewrites the option as it is read, `commit` falls back to an unconditional write.
  For a claim, both runners then find the event present, and both run it.
- **Suggested fix:** treat that case like a store a plugin keeps elsewhere (WordPress's own sequence, once per
  runner, as WordPress has it), or rule on it.

### `NextWake::take`'s documentation is wrong

- **Where:** `speedyspec-php-runtime`, `wake.rs`; the comment in `tests/worker.rs` (`earliest_wake_wins`).
- **What happens:** both say a time in the past becomes 0; only negative times do.
- **Suggested fix:** correct the wording.

### `doing_cron` is deleted whoever set it

- **Where:** `speedyspec-wp-cron`, `run.rs`.
- **What happens:** WordPress deletes the transient only if it still holds its own value. The runner deletes it
  unconditionally, so one runner can clear another's value and let `spawn_cron()` ask for a call early. The only cost
  is one extra, harmless call.
- **Suggested fix:** delete it only when it still holds the value this run set.

### Weak tests in the runner

- **Where:** `speedyspec-php-runtime`, `tests/database.rs`, `tests/cron_run_once.rs`; `speedyspec-wp-cron-php`,
  `tests/php.rs`.
- **What happens:**
  - `scheduling_reports_next_cron` checks only an upper bound, so a stale or zero value passes.
  - `throwing_callback_stays_claimed` uses one event, so it never shows that the next call goes on with the rest, and
    no test has a callback that reschedules its own hook.
  - `twenty_due_events_run_once_across_two_workers` counts double runs well but never shows the two workers actually
    contended.
- **Suggested fix:** add a lower bound; add a second event and a self-rescheduling callback; record
  `claimed_by_others` and report it.

### `WorkerReport` gained a public field

- **Where:** `speedyspec-runtime-contract`, `report.rs`.
- **What happens:** `next_cron` was added to a struct with public fields and no `#[non_exhaustive]`. The wire format is
  compatible, but code outside the workspace that builds the struct with a literal stops compiling.
- **Suggested fix:** mark it `#[non_exhaustive]` with a constructor, before the crate is published.

### The option's read hooks fire fewer times than in WordPress

- **Where:** `speedyspec-wp-cron`, `run.rs`; `compat/cron/breaks.json`.
- **What happens:** the runner reads the `cron` option once per event; `wp-cron.php` reads it in both
  `wp_schedule_event()` and `wp_unschedule_event()`. `pre_option_cron` and `option_cron` fire fewer times, and
  `breaks.json` records only the update hooks.
- **Suggested fix:** record it with the update hooks' difference.

## The server's scheduler

### Cron settings that cannot work are accepted

- **Where:** `speedyspec-server`, `config.rs`.
- **What happens:** `budget` at or above `timeout` is accepted, so every call times out while the runner is still
  working and its worker is killed. `max_interval = "0s"` is accepted too.
- **Suggested fix:** refuse both at parse, naming `app.<id>.php.cron.budget` or `max_interval`.

### A cron path with a query string

- **Where:** `speedyspec-server`, `php/calls.rs` (the `CronCalls` implementation).
- **What happens:** `?budget=` is appended as is, so a declared path that already has a query becomes `/x?a=1?budget=`.
- **Suggested fix:** refuse a `?` in `php.cron.path` in the manifest, or append with `&`.

### The first call after a managed boot can be a minute late

- **Where:** `speedyspec-server`, `php/cron.rs` (`IDLE`).
- **What happens:** while an app has no live cron path the scheduler looks again every 60 s, whatever `max_interval`
  says. In managed mode the schedulers start before the first release is live, so the first call can come about a
  minute after boot even with `max_interval = "10s"`.
- **Suggested fix:** wait `min(60 s, max_interval)`.

### No test of a call in flight across a switch

- **Where:** `speedyspec-server`, `php/cron.rs` tests.
- **What happens:** `switch_keeps_one_scheduler` only changes the path the target returns. No test keeps a call in
  flight while the live handler or pool is replaced.
- **Suggested fix:** an integration test that switches the release during a slow cron call.

### `PhpRequestHandler`'s `Debug` leaves out the wake sink

- **Where:** `speedyspec-server`, `php/handler.rs`.
- **What happens:** the `Debug` output does not show whether a wake sink is set. Cosmetic.
- **Suggested fix:** include it.
