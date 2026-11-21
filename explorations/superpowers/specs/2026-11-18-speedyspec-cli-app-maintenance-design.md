# SpeedySpec CLI: finishing setup and maintaining installed apps

Date: 2026-11-18
Status: draft, awaiting approval
Builds on: `2026-11-17-speedyspec-cli-design.md` (the CLI design; still binding where this document does not change it)

## Amendment (Jacob, 2026-11-18, before plan 6): the CLI never runs WordPress

This amendment overrides the rest of the document where they disagree.

- **Neither the CLI nor speedyspec-server ever executes stock WordPress.** The CLI downloads WordPress and leaves it
  ready for the machine's own php-fpm to serve. speedyspec-server runs only SpeedySpec's own code. The installed
  WordPress is not modified.
- **What the runner is for.** The runner still runs PHP that the app plugin ships (`AppTask`). For WordPress, that is
  only PHP that does not boot WordPress (`loadsApp: false`): writing `wp-config.php`, and reading it with
  `ABSPATH` pointing at a stub.
- **The CLI does not include WordPress's legacy functions** (Jacob). The Rust WordPress Hook API (`add_filter` and
  the rest, `WP_Hook`, the hook globals), added on `feat/wp-hooks`, is not registered in the CLI's process or in the
  runner. The CLI's own plugin API stays `SpeedySpec\Hooks\Hook`.
- **Finishing setup is file work.** For WordPress, setup writes `wp-config.php` (database settings, fresh salts, table
  prefix) from the WordPress sample file. Running `wp_install()` is dropped. The site title, admin user and the
  first options are set in the browser, through WordPress's own installer on the php-fpm site.
- **Plugins and themes are file work.** `plugin` and `theme` keep `list`, `install`, `update` and `delete`.
  `activate`, `deactivate`, `enable` and `disable`, `--site` and `--network`, and `app sites` are dropped: activating
  runs WordPress's code, and the anti-goal against editing the database directly still holds. People activate in
  wp-admin. `delete` removes the files, and WordPress deactivates a plugin whose files are gone.
- **Backup and restore (Jacob, 2026-11-19):** everything is in Rust, including the move to a new URL and table
  prefix. That means a search and replace over the dump that understands PHP-serialized values (string lengths
  recomputed), and the prefix rename of tables and of prefixed option and meta keys.
- **Kubernetes:** as designed, minus what ran the app. There is no activation through an app task, and maintenance
  mode is file work: the app's plugin names the file, which is `.maintenance` for WordPress.
- **Activation and the premise test (Jacob, 2026-11-19):** activating a plugin tests it, then records it as active
  in the database, in Rust. The plugin is run together with the install's other active plugins. If it raises an
  error there, it is not activated and the CLI shows a warning naming the plugin and the error. This replaces the
  anti-goal against activating by editing the database. The test runs in SpeedySpec's own runtime (Jacob): the
  plugin is loaded with the other active plugins and its activation hooks (`register_activation_hook` callbacks)
  run. An error while loading or in an activation hook means it is not activated. The test is therefore the premise
  test itself, does this plugin run on SpeedySpec, and it waits until SpeedySpec can boot.

## Intent

Today `speedyspec app install` puts an app's files in place (zip) or applies its resources (Kubernetes) and stops.
For WordPress, someone then has to open the web installer; nothing helps after that either. Jacob wants an install to
end with a site that works, and the same tool to keep it working: back it up, restore it, reach its database, and
manage its plugins and themes.

The second reason is SpeedySpec itself. SpeedySpec keeps WordPress's plugin and theme model (the `wp-plugin-legacy`,
`wp-theme-legacy` and `wp-app-upgrader-legacy` packages are WordPress's code moved into packages), and Jacob wants to
test the premise that WordPress plugins and themes work in SpeedySpec. The CLI should be able to install and activate
the same plugin in a WordPress site and a SpeedySpec site and show whether it works in both.

## Goals

1. **Finish setup.** After the files or resources are in place, the install completes the app's own setup. For
   WordPress: write `wp-config.php` (database settings, salts, site URL), create the tables and run WordPress's own
   installer (site title, admin user and email), then set options and activate the plugins and theme the app plugin
   asks for. For SpeedySpec: whatever its app plugin defines (see Constraints).
2. **Backup and restore**, both uses (Jacob): put a backup back onto the same site (undo a bad update), and restore
   it onto a different site (production to staging, a zip install onto Kubernetes), which rewrites the site URL and
   can change the table prefix.
3. **Database commands**: export, import, query, list tables, size, check; and, for WordPress, a search and replace
   that understands PHP-serialized values.
4. **Plugins and themes** for both apps: install, activate, deactivate, update, delete, list; plus theme switch.
   Activation runs the app's own activation code. Errors raised while a plugin or theme activates are reported
   against it.

## Non-goals

- A SpeedySpec plugin and theme registry. It will exist later (Jacob) as another source plugin; nothing here
  depends on it.
- Providing a web server, PHP or MySQL. A zip install assumes they are already on the machine (Jacob's choice A).
  Kubernetes installs bring their own, as their resources describe.
- Encrypting backups and storing them remotely (S3 and the like). Backups are local files, readable only by their
  owner, for now.
- Databases other than MySQL and MariaDB.
- The rest of wp-cli: content, users, cron, caches, multisite, scaffolding, i18n. The mechanism built here makes
  each of those a plugin command later; none is built now.
- Serving the apps with the embedded PHP. Unchanged from the CLI design.
- SpeedySpec's own setup steps (not decided yet; see Constraints).

## Anti-goals

- The CLI knowing anything about WordPress or SpeedySpec: where `wp-config.php` is, what `active_plugins` holds, how
  wordpress.org's API works. All of that is in plugins. The CLI owns generic mechanisms: running an app's code,
  talking to a MySQL database, reading and writing an install's files, archives.
- Activating a plugin or switching a theme by editing the database directly. It skips the activation hooks that
  create tables and settings, and leaves the plugin half set up.
- A restore that silently overwrites a site.
- A plugin doing a driver's mechanics (reaching into a pod, opening the database connection, streaming files).

## Constraints

- Everything in the CLI design still holds: one static binary per platform, nothing installed on the machine,
  plugins supply the what, drivers own the how, contracts in their own package, exit codes 0/1/2.
- No global hook functions in the CLI's PHP (removed 2026-11-18, Jacob): WordPress's own `add_action` and friends
  load beside the CLI's `SpeedySpec\Hooks\Hook`.
- SpeedySpec is not Laravel (Jacob). Its setup steps are still being built. The CLI must not assume any of its
  entry points or steps: the SpeedySpec app plugin supplies them when they exist, and everything here is proven on
  WordPress first.
- The "run the app's code" step cannot assume `wp-load.php`; each app plugin names its app's entry point.
- MySQL and MariaDB, as WordPress requires them. SpeedySpec's `wp-*` infrastructure packages reach the database
  through `$wpdb`, so they share it.

## Approaches

1. **The CLI binary runs the app itself (chosen, Jacob).** The embedded PHP gains MySQL support and the extensions
   WordPress expects; app code runs in a child process of the same binary that loads the app; database work is
   done by a MySQL client in Rust; on Kubernetes the same binary runs as a one-off Job beside the app's volumes.
   Nothing extra on the machine, the same behaviour for WordPress and SpeedySpec (which the premise test needs), and
   in the spec's direction of Rust plus out-of-process PHP. Costs a larger binary and the most work.
2. **Hand off to each app's tools** (wp-cli, `mysqldump`, the `wordpress:cli` image). Quickest to a long command
   list for WordPress, but SpeedySpec has no such tool, it depends on the host's PHP, and output must be parsed.
   Rejected.
3. **Talk to the running app over its API** (REST, Abilities API). Needs the site served and an authenticated user,
   so it cannot finish setup or back up files. Useful later, not as the foundation. Rejected.

## Design

### Commands

The command layout (Jacob, 2026-11-18). An app's plugins and themes are top-level `plugin` and `theme`. Whole-install
work (database, backup, restore) sits under `app`, beside `app install` and `app uninstall`. The CLI's own plugins
are `self plugin`, beside `self update`.

```
speedyspec plugin list | install <slug>[@version] | activate <slug> | deactivate <slug> | update [<slug>|--all] | delete <slug>
                  --install <name> [--site <url>] [--network]
speedyspec theme  list | install <slug>[@version] | enable <slug> | disable <slug> | activate <slug> | update [<slug>|--all] | delete <slug>
                  --install <name> [--site <url>] [--network]

speedyspec app installs                                  # the installs speedyspec knows about
speedyspec app sites --install <name>                    # the sites in an install: one, or every site in a network
speedyspec app db export [--to <file>] | import <file> | query <sql> | tables | size | check  --install <name>
speedyspec app backup  --install <name> [--to <file>]
speedyspec app restore <file> --install <name> [--url <new>] [--table-prefix <prefix>] [--yes]

speedyspec self plugin list | install <src> | remove <name>    # the CLI's own plugins
speedyspec self update [--check]
```

**Multisite is first-class** (Jacob). In a WordPress network, one install serves many sites.

- **Plugins:** installed once for the whole network, then activated either network-wide (`--network`) or on one
  site (`--site <url>`).
- **Themes:** installed once, enabled for the network (`theme enable --network`), then activated per site.
- **`--site`:** optional on a single-site install, which has one site.
- **Network-wide file work:** `install`, `update` and `delete` act on the whole install, because the files are
  shared. On a network they refuse `--site` rather than pretend to be per site.
- **`list`:** shows what is installed, and what is active where (network-wide, or on the given site).
- **Errors:** errors raised while a plugin or theme activates are reported against it and against the site.
- **The app's plugin describes the network.** It says whether an install is a network and what its sites are, so
  SpeedySpec can offer the same when it has networks.

Every command that acts on an install takes `--install <name>`, and the driver's connection flags where it needs
them (`--namespace` and the cluster flags for Kubernetes).

An install has to be findable by name afterwards. Kubernetes installs already keep a record in the cluster. Zip
installs get a record too (my call): `$SPEEDYSPEC_HOME/installs/<install>.json` with the app, parts, base path,
driver, CLI version and time; `app install --driver zip` writes it and accepts `--name` like the Kubernetes driver.
`app installs` lists the local records (and says Kubernetes installs are listed per cluster).

### Contracts (the contracts package, a minor release)

New readonly value classes in `SpeedySpec\Cli\Contracts\`, validated in Rust as today:

- `AppRuntime`: the app's entry file relative to the install root (`wp-load.php` for WordPress), `ini` settings
  (memory limit and the like), and, for Kubernetes, which workload serves the app (see "Details settled after
  review"). There is one embedded PHP, so no PHP version is chosen here.
- `DatabaseConnection`: engine (`mysql`), host, port or socket, database, user, password (secret), table prefix,
  charset.
- `AppTask`: a PHP file shipped in the app plugin's own directory plus a JSON-serializable input. The CLI runs it
  inside the app (see "Running the app's code") and gets back a JSON result.
- `SetupPlan`: the ordered tasks that finish an install (`AppTask`s), and the install options they need.
- `BackupSpec`: the paths under the install root to include and exclude.
- `ExtensionSource` answers: for a slug and an optional version, the archive URL (and `sha256` when the source has
  one), the version and the kind (`plugin` or `theme`); plus search and latest-version lookups.

### Hooks (new)

| Hook | Kind | The app or source plugin returns |
|---|---|---|
| `speedyspec_app_runtime` | filter | the `AppRuntime` for an install |
| `speedyspec_app_database` | filter | the `DatabaseConnection` for an install, or an `AppTask` whose result is one (see "Details settled after review") |
| `speedyspec_app_setup` | filter | the `SetupPlan` run at the end of `app install` |
| `speedyspec_app_backup` | filter | the `BackupSpec` |
| `speedyspec_app_move` | filter | the `AppTask`s that move a restored site to a new URL and table prefix |
| `speedyspec_app_extensions` | filter | the tasks for listing, activating and deactivating plugins and themes, and where they live under the install root |
| `speedyspec_extension_sources` | filter | the `ExtensionSource`s, in order (wordpress.org first) |

Each is called with the install's identity (app, parts, install name, driver) so one plugin can serve several apps.

### Running the app's code

The CLI starts itself again as a child process in a hidden runner mode. The runner:

- loads no CLI plugins (their functions and classes could clash with the app's) and no install hooks;
- loads the contracts, then the app's entry file from `AppRuntime`, then the task's PHP file, which receives its
  input and returns its result as JSON over a pipe;
- captures every PHP error, warning and uncaught exception with its file and line, and reports them attributed to
  the plugin or theme directory the file belongs to (for the premise test: "akismet: Warning: … in
  wp-content/plugins/akismet/…");
- dies with the task: one task per process, so one app's state never leaks into the next and a fatal error cannot
  leave the CLI's own PHP degraded.

The static PHP gains `mysqli`, `mysqlnd`, `pdo`, `pdo_mysql`, `dom`, `xml`, `simplexml`, `xmlreader`, `xmlwriter`,
`sodium`, `iconv`, `fileinfo` and `exif` (my call; the builder may add what WordPress's site health asks for). The
binary's growth is measured and recorded.

Where the runner runs is the driver's job:

- **zip**: on this machine, in the install's base directory.
- **Kubernetes**: a one-off Job in the install's namespace that mounts the app's volumes and reaches the database
  service from inside the cluster. The Job's container runs this release's own static binary (fetched from the
  GitHub release for its platform and checked against `SHA256SUMS`), so the runner is the same code everywhere.
  Its input and result travel through the Kubernetes API, not through logs.

### The database

A MySQL client in Rust (crate left to the builder; static, TLS through rustls) does `export`, `import`, `query`,
`tables`, `size` and `check`, and supplies the dump inside a backup. The connection comes from
`speedyspec_app_database`. Locally it connects directly; for Kubernetes it reaches the database through the API
(port-forward) or runs inside the runner Job, the builder's call. The password is a secret: never printed,
recorded or put in a URL.

`search-replace` is a WordPress plugin command, not a CLI one: it needs PHP's serialization rules and WordPress's
idea of which columns hold URLs. It runs as an `AppTask`.

### Finishing setup

`app install` gains a last phase: after the driver has placed the files or resources, the CLI asks
`speedyspec_app_setup` for a `SetupPlan`, asks for its install options (prompted, secrets without echo, as today),
and runs its tasks in order through the runner. A task that fails stops the install with the task's own message;
nothing is rolled back (as today), and the install record says how far it got, so it can be finished by running the
same command again. The WordPress app plugin writes `wp-config.php` with fresh salts, runs `wp_install()`, sets
options, and activates the plugins and theme it lists.

For Kubernetes the same plan runs in the runner Job after the app's workloads are ready. (A plugin can still add its
own Jobs to the resources, as today.)

### Backup and restore

A backup is one `tar.gz` file (my call; it streams, so large dumps never sit in memory):

- `manifest.json`: format version, app, parts, CLI version, app version, plugin and theme versions, database engine,
  table prefix, site URL, source install name and driver, creation time;
- `files/`: the paths `BackupSpec` names, relative to the install root;
- `database.sql`: a plain SQL dump.

Nothing in it depends on the driver that made it, so a backup of a zip install restores onto Kubernetes and the
other way round. The file is created readable only by its owner and the command says it contains secrets.

`restore` onto the same site puts the files and the database back. Onto a different site, given `--url` (and
`--table-prefix` when it differs), it then runs the app plugin's `speedyspec_app_move` tasks: for WordPress, the
serialization-aware search and replace of the old URL with the new one, and the table rename to the new prefix.
Restoring over a site that has content asks to confirm (`--yes` skips the question) and first takes a backup of
what is there, so a restore can itself be undone.

### Plugins and themes

- **Sources.** A shared "wordpress.org" source plugin answers `speedyspec_extension_sources` using wordpress.org's
  API. Both the WordPress and the SpeedySpec app plugins use it. The SpeedySpec registry becomes a second source
  plugin later, with no CLI change.
- **Install, update, delete, list** are file work and need no app code: the CLI downloads and unpacks the archive
  (the zip machinery, with its path checks) into the directory the app plugin names, and lists what is there from
  the file headers.
- **Activate, deactivate, theme switch** run the app's own code through an `AppTask` from
  `speedyspec_app_extensions`, so activation hooks really run. Errors raised during them are reported against the
  plugin or theme (see the runner).
- **The premise test** is then: the same `plugin install x --install <wordpress-install> && plugin activate x
  --install <wordpress-install>` and the same against a SpeedySpec install, compared. A command that runs a list of plugins
  through two installs and reports the differences can come later.

### Errors

As in the CLI design: every failure names what failed and what the user can do. Additionally, a failure inside the
app's code names the task and, where the file says so, the plugin or theme it came from.

### Testing

- Database and runner: integration tests against a throwaway MySQL in Docker (the `mysql` and `mariadb` images),
  never a real database.
- WordPress end to end: a throwaway MySQL plus a WordPress release served locally, exercising install with setup,
  plugin install and activate (with a fixture plugin that raises a warning on activation), backup, same-site
  restore, and restore to another URL and prefix.
- Kubernetes: the same against the k3s cluster from `scripts/k3s.sh`.
- The SpeedySpec side of the premise test is added when SpeedySpec can boot.

### Order of work

Each item is one implementation plan, in this order (my call):

1. **Foundation**: zip install records and `app installs`; the new contracts and hooks; the MySQL client and the `app db`
   commands; the extra PHP extensions; the runner mode, with the zip driver running tasks locally.
2. **WordPress setup and plugins and themes**: the WordPress app plugin and the wordpress.org source plugin (in their
   own repositories, like the contracts), finishing setup at install, and `plugin` / `theme`, single-site and multisite.
3. **Backup and restore**, both uses, for zip installs.
4. **Kubernetes**: the runner Job, database access through the API, setup, plugins and themes, backup and restore.
5. **SpeedySpec**: its app plugin, once SpeedySpec's own setup exists; then the premise test.

## Details settled after review (my calls)

A fresh reviewer read this document as its builder would. None of the questions it raised needed Jacob; these are
the answers, each my call, for Jacob to check.

- **Kubernetes runner volumes.** `AppRuntime` names the workload that serves the app (a Deployment or StatefulSet
  in the install). The runner Job copies that workload's volumes and mount paths, and the install root is the
  container path the app plugin gives. It is scheduled onto the same node as the app's pod, so ReadWriteOnce claims
  work. The app keeps running during setup and plugin and theme work, as with wp-cli. Restore and updates of active
  plugins put the site into the app's maintenance mode, through an app task, rather than scaling it down.
- **Restoring onto a different site.** The target must already exist: to move a zip install onto Kubernetes, first
  `app install` it there, then `app restore … --install <name>`. The target's own configuration wins: `BackupSpec` gains a
  `preserve` list (for WordPress, `wp-config.php`) whose files are kept from the target. The target's
  `DatabaseConnection` is read before anything is replaced. "A site that has content" means the target database
  already has tables with the target's prefix.
- **Plugin and theme headers, delete and update.** Reading `Plugin Name:` and `style.css` headers is WordPress
  knowledge, so the app plugin's own PHP does it (in the CLI process, from files the CLI hands it). `list` also
  shows which are active, through an app task. `delete` deactivates and runs the plugin's uninstall routine
  through an app task, then removes the files. `update` of an active plugin turns maintenance mode on, replaces the
  files, runs the app's upgrade routines through an app task, and turns maintenance mode off.
- **Re-running after a failed setup.** The re-run skips the file phase when the install record says files are in
  place, and resumes at the first task that has not finished. Install options are asked for again: the record never
  holds secrets. Setup tasks must be safe to run twice (`wp_install()` already checks whether the site is
  installed). During setup the database settings come from install options. Setup assumes the database and its user
  already exist on the machine (choice A).
- **The runner binary on Kubernetes.** By default the Job gets the release asset for its nodes' architecture,
  checked against that release's `SHA256SUMS`. `--runner-binary <path>` supplies a local Linux binary instead (for
  development builds, the k3s tests, and clusters without internet access); its checksum is computed locally and
  checked again in the Job.
- **Files crossing into the cluster.** The CLI gives app plugins a driver-neutral way to read and write files
  under the install root: a Rust-backed PHP class in the built-in extension. That is the driver's mechanism,
  offered to plugins, not plugins doing a driver's work. An app task's PHP file is copied into the runner with its
  plugin's directory. For WordPress the database settings come from running `wp-config.php` in the runner, with
  `ABSPATH` pointing at a stub whose `wp-settings.php` is empty. Constants defined from environment variables,
  common on Kubernetes, therefore resolve as the site sees them.
- **PHP version.** There is one embedded PHP (the latest stable). The premise test reports the PHP version with its
  results, so a plugin that fails only on a new PHP is not mistaken for a SpeedySpec problem.
- **Repositories.** The WordPress app plugin grows out of `examples/install-wordpress` into its own public
  repository, `SpeedySpec/speedyspec-cli-wordpress`, and the wordpress.org source plugin goes into
  `SpeedySpec/speedyspec-cli-wordpress-org`. Both are created when plan 2 starts. The contracts minor release comes
  first, in plan 1.

## Left to the builder

- The Rust MySQL crate, the tar and gzip crates, and how the runner passes input and output (stdin and stdout, or a
  file descriptor), provided the binary stays static and the format is versioned.
- The exact fields and method names of the new contracts, following the existing ones.
- How the Kubernetes runner Job gets the binary onto its volume (an init container fetching the release asset, or
  copying it through the API), provided it is verified against `SHA256SUMS`.
- `ini` defaults for the runner, and which extra PHP extensions beyond the list above WordPress needs.
- Progress output and wording.
