# SpeedySpec CLI design

Date: 2026-11-17
Status: draft, awaiting approval

## Intent

Give people one downloadable binary, `speedyspec`, that installs an application onto whatever environment they have.
The binary knows how to *deliver* (download and unpack a zip, apply resources to a Kubernetes cluster) and nothing
about *what* is delivered. What is delivered comes from PHP plugins, which run inside the binary on an embedded PHP
interpreter. One plugin makes the CLI install SpeedySpec (admin, blog, REST/WebSocket/WebTransport backend); another
makes it install WordPress.

PHP is embedded, rather than required on the machine, so that a fresh server with nothing on it can run the installer,
and so that plugins can extend the CLI itself: add subcommands, add install options, and answer the driver hooks.

The direction of travel for the project is to modernise old PHP and replace it with Rust. That is why the hook system
is written in Rust and only exposed to PHP, and why the embedded PHP is always the latest stable release.

## Goals

- A single self-contained binary per platform. No PHP, Composer, Helm or kubectl needed on the machine.
- `speedyspec app install` installs an application through a chosen driver.
- Drivers are mechanisms owned by the CLI. Each driver has a hook where a plugin supplies the specifics.
- Each driver's hook has a small, versioned PHP contract describing what the plugin must return.
- Plugins can add subcommands and install options, and can run code before and after an install.
- The CLI can install its own plugins, so a bare binary can fetch the functionality it needs.
- The CLI can update itself to the latest release.
- First build ships two drivers end to end: **zip** and **Kubernetes**.

## Non-goals

- The SpeedySpec and WordPress installer plugins themselves. Only small fixture/example plugins are built here.
- The other drivers: (S)FTP, Docker, Composer, Functions, AWS container platforms. Each is a later piece of work with
  its own contract. The driver interface must make adding them a matter of adding a crate and a contract.
- Helm. The Kubernetes driver talks to the cluster directly and does Helm's job for these installs. Charts are not
  rendered or reused.
- Windows binaries.
- Serving or running the installed application with the embedded PHP. When that is designed, PHP for web
  requests runs out of process, in workers a Rust supervisor can kill and bring back, as php-fpm does (Jacob's
  direction). The CLI's single long-lived in-process PHP is not the model for it: one fatal error leaves that engine
  degraded for the rest of the run, and a non-thread-safe PHP serves one request at a time.
- Sandboxing, signing or a registry for plugins.
- NATS, RabbitMQ or any other broker backend for hooks. Only the contract that allows them, and the memory backend.

## Anti-goals

- The CLI containing any knowledge of SpeedySpec or WordPress. If installing a new application needs a CLI release,
  the design has failed.
- A contract that makes the plugin do the driver's mechanics (downloading, extracting, calling the cluster API).
- A plugin needing a system PHP, Composer or any other tool at run time.
- Contract definitions living in the same package as a driver implementation or a plugin.

## Constraints

- Rust, in this repository, under `packages/`. Local toolchain is Rust 1.99.
- PHP: the latest stable release at build time. Today that is 8.5.x; 8.6.0 is scheduled for 2026-11-19. The exact
  version is pinned in one file and bumped as soon as the binding and the static build both support the new release.
- PHP files under `packages/` follow `.claude/rules/coding-style.md` (4 spaces, `declare(strict_types=1);`, opening
  braces on a new line, docblock rules).
- Contracts live in their own package; implementations live in separate packages and are replaceable.
- Plugin code is arbitrary PHP running with the user's privileges. The CLI must say so when installing a plugin.

## Approaches

### How PHP gets inside the binary

- **A. Statically linked PHP (chosen).** PHP's embed library is built with static-php-cli (`--build-embed` produces
  `libphp.a` and headers) and linked into the binary; ext-php-rs (its `embed` feature) provides the binding. Rust
  functions are PHP functions directly. One file to download and real in-process hooks. Costs a heavier build per
  platform, and Windows is hard.
- B. A bundled PHP executable run as a child process, talking over a pipe. Easier build and crash isolation, but every
  hook call becomes a message round trip and it is not an embedded extension.
- C. Linking against a system PHP. Rejected: breaks "download one binary".

Jacob chose A.

### How Kubernetes installs work

- **A. Talk to the cluster directly (chosen).** The user passes the server and a token. The plugin returns the
  resources it wants; the CLI applies them, waits for readiness and labels them so it can find and remove them.
- B. Render Helm charts in the CLI. Reuses existing charts, but there is no solid Rust implementation of Helm's
  templating, so it means bundling or shelling out to Helm.
- C. A now, plus a later separate Helm driver.

Jacob chose A, without the later Helm driver.

## Design

### Packages

Two new packages, each its own public git repository checked out under `packages/`, the way
`speedyspec-wp-hook-domain` and `speedyspec-support` are (nested repositories, untracked by `wordpress-develop`, each
with its own release-please config and release workflow). Jacob confirmed this.

- `packages/speedyspec-cli/` (`SpeedySpec/speedyspec-cli`) is a Cargo workspace.
- `packages/speedyspec-cli-contracts/` (`SpeedySpec/speedyspec-cli-contracts`) is a Composer package
  (`speedyspec/cli-contracts`) holding the PHP contracts.

The two version independently. The CLI pins one contracts release and fetches it at build time; during local
development it may be pointed at the sibling checkout instead.

Crates in the workspace, each with one job:

| Crate | Job |
|---|---|
| `speedyspec-cli` | The `speedyspec` binary: command tree, prompts, wiring. |
| `speedyspec-hooks` | The hook contract: the trait every backend implements, and the value type. No PHP. |
| `speedyspec-hooks-memory` | The in-memory hook backend the CLI uses. |
| `speedyspec-php` | Boots embedded PHP, registers the built-in `speedyspec` extension, converts values. |
| `speedyspec-driver` | The driver trait and the Rust mirror of each contract. No implementations. |
| `speedyspec-driver-zip` | Zip driver. |
| `speedyspec-driver-kubernetes` | Kubernetes driver. |
| `speedyspec-plugin` | Plugin manifest, discovery, loading order, install/remove. |
| `speedyspec-selfupdate` | Finding, verifying and swapping in a newer binary. |

`speedyspec-driver` is to drivers what the contracts package is to plugins: implementations depend on it, never on
each other.

### Commands

```
speedyspec app list
speedyspec app install <app> [--part <id>]... --driver <driver> [driver flags] [plugin options]
speedyspec app uninstall <install-name> --driver kubernetes [driver flags]
speedyspec self plugin install <zip URL | owner/repo[@tag]>
speedyspec self plugin list
speedyspec self plugin remove <name>
speedyspec self update [--check]
speedyspec php <file> [args...]
speedyspec --version
```

Global flags: `--no-plugins` (skip loading plugins), `--no-interaction` (never prompt; fail on anything missing).

When a required choice (app, driver, an install option) is missing and the terminal is interactive, the CLI prompts
for it. Options marked secret are read without echo.

Plugins are loaded before arguments are parsed, so the command tree is built at run time and plugin subcommands and
options appear in `--help`.

`speedyspec php` runs a PHP file on the embedded interpreter with the hook API available. It exists so plugin
authors can exercise their code without a system PHP. (My call.)

### Hooks

The registry is Rust. It holds actions and filters by name with integer priorities (default 10, lower runs first,
insertion order within a priority), the same semantics as WordPress. Callbacks can be Rust closures or PHP callables.

PHP sees it as a class provided by the built-in extension, `SpeedySpec\Hooks\Hook`, with static methods. That class
is the API. Actions and filters share one registry, so one method serves both: `Hook::add()`, `Hook::remove()`,
`Hook::has()`, `Hook::run()`, `Hook::filter()`, `Hook::ran()`, with the WordPress argument order.

There are no global functions (Jacob, 2026-11-18; the first build had WordPress-named aliases such as
`add_action` and `add_filter`, removed before anything depended on them). The CLI is new, so there is no code to
stay compatible with, and keeping the names free lets the CLI load code that declares its own, WordPress included,
for example to finish a WordPress install.

Priorities are numeric, as in WordPress (Jacob's requirement): `Hook::add('x', $cb, '20')` works. `$priority` and
`$acceptedArgs` accept any integer-valued numeric input, including numeric strings, even when the calling file
declares `strict_types=1`. `'20'` and `20` are the same priority. Anything non-numeric or fractional is a
`TypeError`.

The registry is in Rust for two reasons Jacob gave: to store actions and filters more efficiently than PHP arrays
do, and so that other Rust code can register and fire the same hooks without calling out to PHP. So:

- `speedyspec-hooks` is a standalone library with a Rust API that is the primary one. `Hook` in PHP is a thin layer
  over it.
- It is a contract, not an implementation. Jacob's requirement: the hook system must allow backends such as NATS and
  RabbitMQ alongside in-memory ones. So the contract is a trait in `speedyspec-hooks`, each backend is its own
  crate, and everything else (the PHP bridge, the CLI, drivers) holds the trait and never a concrete backend.
- How a broker implementation works (Jacob's correction): the *firing* goes through the broker, the callbacks do
  not. `do_action` and `apply_filters` (`Hook::run`, `Hook::filter`) call through NATS or RabbitMQ; every process
  that has callbacks on that hook receives the firing and calls its own callbacks from its own in-memory registry.
  Callbacks are never serialised or moved. Registering a callback stays a local operation on the in-memory
  registry, so a broker implementation composes with the memory backend rather than replacing it.
- The first build ships the in-memory backend, `speedyspec-hooks-memory`, only. Broker implementations are later
  work (my call), but the contract already allows them: the trait can be implemented by a layer that wraps another
  implementation, every operation can fail, and a shared conformance test suite in the contract crate is what any
  implementation has to pass.
- What crosses the broker is what a firing carries: the hook name, its arguments, and for a filter the value going
  out and the filtered value coming back. Never a callback. How priority order is kept across processes for a
  filter is for the design of that implementation. An argument or value that cannot leave the process (a PHP object or closure) is passed as-is locally; an
  implementation that has to publish it rejects it with a clear error rather than dropping it.
- It has its own native value type. Rust callers and Rust callbacks work with native values and never touch PHP.
- Firing a hook enters PHP only when a PHP callback is actually registered on it. A hook with only Rust callbacks,
  or none, costs no PHP call.
- The memory backend's storage is compact: hook names are interned and each hook's callbacks are kept in priority
  order at insert time, so firing is a straight walk with no sorting.

A value passed from one PHP callback to the next must arrive unchanged, objects included.

Callbacks may create and use PHP Fibers (Jacob's requirement). A callback running inside a fiber may suspend it; the
hook firing pauses with that fiber and continues with the remaining callbacks when it is resumed. Other code may
fire hooks in the meantime. Because callbacks always run in the process that registered them, this holds whichever
hook implementation is in use.

PHP runs on the main thread only, non-thread-safe build. Network and file work in Rust may be async, but hooks are
called from the main thread.

Hooks the CLI fires:

| Hook | Kind | Purpose |
|---|---|---|
| `speedyspec_init` | action | Fired once all plugins are loaded. |
| `speedyspec_apps` | filter | Plugins add `App` definitions: id, label, parts, and the drivers the app supports. |
| `speedyspec_commands` | filter | Plugins add `Command` definitions: name, description, arguments, options, handler. |
| `speedyspec_install_options` | filter | Given app and driver, plugins add `Option` definitions that become flags and prompts. |
| `speedyspec_install_before` | action | Given the install request, before the driver runs. |
| `speedyspec_driver_zip` | filter | Given the install request, returns a list of `ZipSource`. |
| `speedyspec_driver_kubernetes` | filter | Given the install request, returns a `KubernetesResources`. |
| `speedyspec_install_after` | action | Given the request and the driver's result, so a plugin can finish the job. |

The install request carries: app id, selected part ids, driver id, resolved option values, and the driver's own
settings that are safe to share (base directory for zip; namespace and install name for Kubernetes, never the token).

A driver is offered for an app only when a Rust driver with that id exists **and** the app's definition lists it. An
app with no usable driver is listed as such, not hidden.

Later drivers follow the same pattern: one filter `speedyspec_driver_<id>` and one contract.

### Contracts

`speedyspec/cli-contracts` holds final, readonly PHP value classes in `SpeedySpec\Cli\Contracts\`, plus nothing with
behaviour worth replacing. First version:

- `App` (id, label, parts, drivers), `AppPart` (id, label, default)
- `Command`, `CommandArgument`, `CommandOption`
- `InstallOption` (name, label, default, required, secret)
- `InstallRequest` (app, parts, driver, options, driver settings)
- `ZipSource`
- `KubernetesResources`

Driver support is declared per app, not per part. `AppPart::default` is a boolean, true unless stated. With no
`--part`, every default part is installed; there is no prompt for parts and no notion of a required part. (My call.)

Versioning (my call):

- The package follows semantic versioning. Adding an optional field is a minor release; anything else is major.
- The CLI binary bundles exactly one release of the contracts package, compiled in, and autoloads it before any
  plugin. There is one definition of each class in the process.
- Each plugin's manifest states the contract range it needs (`"contracts": "^1.2"`). A plugin whose range the bundled
  version does not satisfy is not loaded, with an error naming both versions.
- Plugin authors require the Composer package for development (editor, static analysis, tests) and must not ship it.
- Rust validates every value a plugin returns against the contract and reports the plugin, hook and field on failure.
- `speedyspec --version` prints the CLI, PHP and contracts versions.

#### Zip contract

`ZipSource`:

- `url` (required): where the archive is.
- `destination` (required): where its contents go, relative to the base directory the user chose.
- `sha256` (optional, my call): if present, the download is verified before extraction.
- `stripComponents` (optional, default 0, my call): leading path segments to drop. WordPress's archive wraps
  everything in `wordpress/`; without this a plugin could not put it at the base.

The filter returns a list, so an app with three parts can return three archives with three destinations.

#### Kubernetes contract

`KubernetesResources`:

- `resources` (required): an ordered list of complete Kubernetes resources, each an associative array with
  `apiVersion`, `kind`, `metadata` and the rest, exactly as they would appear in a manifest. Namespaces, ConfigMaps,
  Secrets, workloads, Services, Ingresses and custom resources are all allowed.
- `waitFor` (optional): which of those to wait on, as a list of `kind`, `name`, optional `namespace` and optional
  `condition`. Omitted means every Deployment, StatefulSet, DaemonSet and Job in `resources`. An empty list means
  wait for nothing. Readiness is built in for Deployment, StatefulSet, DaemonSet, Job and Pod. Any other kind must
  give `condition`, the name of a status condition that has to be `True`; without one it is a validation error.
  (My call.)

The plugin returns full resources rather than a simplified description (my call), so the CLI stays generic and the
contract does not chase Kubernetes features. The contracts package may add small PHP builder helpers for common
shapes later; they are not part of the first version.

### Zip driver

`--base <dir>` (default: current directory).

For each `ZipSource`: download over HTTPS following redirects, verify `sha256` if given, extract to a temporary
directory inside the base, then move into place. It must:

- reject a `destination` that is absolute or resolves outside the base;
- reject archive entries that would escape the destination (zip-slip), including through symlinks;
- refuse to write into a non-empty destination unless `--force` is given;
- leave nothing half-written at the destination if a download or extraction fails.

How those fit together when there are several sources (my call):

- Every source is downloaded, verified and extracted to staging before anything is moved into place. A failure up to
  that point changes nothing on disk outside staging.
- "Non-empty" is judged once, before the install starts, against what was already on disk. A destination filled by
  an earlier source in the same install does not count, so `.` followed by `wp-content/plugins/x` works.
- Sources are moved into place in the order the plugin returned them.
- `--force` merges: existing files with the same path are overwritten, everything else is left. It never deletes
  what is already there.
- If a move fails without `--force`, what this install moved is removed again. With `--force` that cannot be undone
  safely, so the CLI stops and reports exactly which sources were and were not put in place.

The result handed to `speedyspec_install_after` lists each source with the absolute path it was installed to.

### Kubernetes driver

Connection flags:

- `--server <url>` and a token. The token is read from `SPEEDYSPEC_KUBE_TOKEN` or `--token-file <path>` (`-` for
  stdin). There is no `--token <value>` flag, so the token does not land in shell history or process lists. (My
  call.)
- `--ca-cert <path>` or `--insecure-skip-tls-verify`.
- If neither server nor token is given, fall back to the user's kubeconfig. (My call.)
- `--namespace <name>` (always required: it is the default for resources that name none, and where the install
  record lives), `--name <install-name>` (default: app id), `--timeout <duration>` (default 10 minutes).
- `--force-conflicts` takes ownership of fields another manager owns. Without it a conflict is a failure that names
  the field and its owner. (My call.)

Install:

1. Ask the plugin for `KubernetesResources` and validate it.
2. Create `--namespace`, and any other namespace a resource refers to, if it does not exist.
3. Write the install record, a Secret named `speedyspec-install-<install-name>` in `--namespace`, **before**
   applying anything: status `installing`, app, parts, plugin name and version, the namespaces this install created,
   and the full list of resources it is about to apply. A failed install can then still be uninstalled.
4. Apply each resource in order with server-side apply, field manager `speedyspec`. Namespaced resources without a
   namespace get `--namespace`. Every resource is labelled `app.kubernetes.io/managed-by=speedyspec`,
   `speedyspec.com/app=<app id>` and `speedyspec.com/install=<install-name>`.
5. Wait for the `waitFor` resources to become ready, printing progress, until the timeout.
6. Mark the record `installed`.

Running install again with the same name re-applies (idempotent). Resources that were in the previous record and are
absent from the new set are left running, reported, and kept in the record so that uninstall still removes them;
pruning on re-install is not in the first build.

On failure, the CLI reports which resource failed and why and leaves what was applied in place for inspection. It
does not roll back. (My call.)

`app uninstall <install-name> --namespace <name>` reads the record, deletes the recorded resources in reverse order,
then the namespaces the record says this install created, then the record. It runs without a plugin.

Arbitrary kinds, including custom resources, must be supported, so the driver discovers resource types from the
cluster rather than compiling in a fixed set.

### Plugins

A plugin is a directory containing `speedyspec-plugin.json`:

```json
{
    "name": "speedyspec/install-wordpress",
    "version": "1.0.0",
    "contracts": "^1.0",
    "entry": "plugin.php"
}
```

At start-up the CLI loads every plugin from the user plugin directory (`$SPEEDYSPEC_HOME/plugins`, where
`SPEEDYSPEC_HOME` defaults to the platform's per-user data directory), then from any directory named explicitly with
`--plugin-dir <path>` (repeatable). Nothing is loaded from the working directory on its own: running the CLI inside
a cloned or unzipped folder must never execute that folder's PHP. (My call.) Loading means checking the contract range and requiring the entry
file, which registers its hooks. A plugin that needs Composer dependencies ships its own `vendor/` and requires its
own autoloader from the entry file.

A plugin that throws or dies while loading is reported by name and skipped; the CLI carries on. A PHP fatal error in
a plugin must never take the process down without a message naming the plugin.

`speedyspec plugin install <source>` uses the zip driver's machinery directly, with the user supplying the source:

- a URL to a zip, or
- `owner/repo` or `owner/repo@tag`, resolved against GitHub releases (latest release when no tag). It uses the
  release asset named `speedyspec-plugin.zip` if there is one, otherwise the release's source archive with its
  top-level directory stripped. (My call.)

Before installing it shows the source and a line saying the plugin will run as the user with full access, and asks
for confirmation (`--yes` skips it). It checks for a valid manifest before moving anything into the plugin directory,
and replaces an existing plugin of the same name only after the new one validates.

### Embedded PHP

- Built with static-php-cli as a static embed library and linked through ext-php-rs.
- Extensions compiled in (my call, the builder may extend): `ctype`, `curl`, `filter`, `mbstring`, `openssl`, `phar`,
  `tokenizer`, `zip`, `zlib`. JSON is always present.
- The built-in `speedyspec` extension provides `SpeedySpec\Hooks\Hook` and nothing else: no global functions.
- `php.ini` is not read from the machine. Errors and warnings from plugins go to stderr with the plugin name.

**This is the main technical risk.** Static linking of the embed library through ext-php-rs on the pinned PHP version
has not been proven here on macOS arm64 and Linux. The first step of the build must be a proof: a binary that boots
PHP, runs a PHP file, and has that file call a Rust function, on both platforms. If the newest PHP is not yet
supported by the binding, pin the newest that is and record why.

If the proof fails on a target, the allowed fallbacks, in order (my call): link libc dynamically on that target (glibc
instead of musl) while PHP stays statically linked in; patch the binding or the build recipe; drop that target from
the first release. Switching to a child-process PHP is not an allowed fallback. If none of these works on macOS arm64
or Linux x86_64, stop and come back to Jacob.

### Self-update and distribution

- CI builds release binaries for Linux x86_64 and aarch64 (statically linked, musl) and macOS x86_64 and arm64, and
  attaches them to a GitHub release together with a `SHA256SUMS` file.
- Releases are cut with release-please from `SpeedySpec/speedyspec-cli`, following the workflow pattern of the
  existing `speedyspec-*` repositories. The repository is public, so self-update, the download URLs and
  `plugin install owner/repo` need no credentials. Private plugin repositories are not supported in the first build.
- `speedyspec self update` finds the latest release, downloads the asset for the running platform, verifies it
  against `SHA256SUMS`, and replaces the running executable atomically. `--check` only reports whether a newer
  version exists. If the executable is not writable it says so and how to fix it.
- "Download the latest CLI" is the release page plus a stable URL per platform for the latest release.

### Errors

Every failure names what failed and what the user can do about it. Exit codes: 0 success, 1 failure, 2 usage error.
A plugin returning something that does not match a contract is a failure that names the plugin, the hook and the
field. Nothing is installed when validation fails.

### Testing

- `speedyspec-hooks`: a conformance suite for ordering, priorities, removal, re-entrancy and filters changing
  values, which `speedyspec-hooks-memory` (and any later backend) runs.
- Zip driver: tests against a local HTTP server, including checksum mismatch, zip-slip archives, destinations outside
  the base, non-empty destinations and interrupted downloads.
- Kubernetes driver: unit tests against a mocked API server, and an end-to-end test in CI against a throwaway cluster
  (kind or k3d) that installs, re-installs and uninstalls a fixture app.
- PHP boundary: integration tests that run fixture plugins through the real binary and check that hooks registered in
  PHP are seen by Rust and the reverse, that objects survive a PHP-to-PHP filter chain, and that a plugin with a
  fatal error is reported and skipped.
- Contracts package: PHP tests, using the same test tooling as the other `speedyspec-*` packages.
- Two example plugins live with the CLI as fixtures and documentation: one that installs WordPress by zip
  (`https://wordpress.org/latest.zip`, `stripComponents: 1`) and one that installs a small fixture app to Kubernetes.

## Left to the builder

- Crate choices (argument parsing, HTTP, zip, Kubernetes client, prompts), provided the binary stays self-contained.
- The exact shape of `Command`, `CommandArgument` and `CommandOption`, and how a plugin command handler receives its
  input and writes output.
- Internal layout of each crate and the CI workflow details.
- Progress output and wording.
