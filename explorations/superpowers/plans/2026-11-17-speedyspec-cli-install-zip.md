# SpeedySpec CLI Install Flow and Zip Driver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `speedyspec app install <app> --driver zip` downloads and unpacks what a plugin names, and
`speedyspec plugin install` / `plugin remove` manage the CLI's own plugins, so a bare binary can fetch a plugin and
then install an application with it.

**Architecture:** A `Driver` trait in `speedyspec-driver` describes what every driver gives the CLI (its flags, the
settings it shares with plugins, the check on a plugin's answer, the install itself). `speedyspec-driver-zip`
implements it on top of a small fetch-verify-extract engine that `plugin install` reuses directly. The install flow
in `speedyspec-cli` resolves the user's choices, asks plugins through hooks checked by the existing per-plugin
attribution, runs the driver, and tells plugins the result.

**Tech Stack:** Rust 1.99; a blocking HTTPS client with TLS roots compiled in; a zip reader; SHA-256; the existing
workspace crates. No async runtime.

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-cli-design.md`

**This is plan 2 of 4.** Plan 1 (foundation) is built and on GitHub. Plan 3 is the Kubernetes driver and
`app uninstall`; plan 4 is self-update and release builds.

## Global Constraints

- Work in `packages/speedyspec-cli` (repo `SpeedySpec/speedyspec-cli`) on a branch `feat/install-zip` cut from
  `main`. Do not push or open a pull request without Jacob's go-ahead.
- `packages/speedyspec-cli-contracts` is **not changed** by this plan: `InstallOption`, `InstallRequest` and
  `ZipSource` 1.0.0 are used as they are. If a task seems to need a contracts change, stop and report.
- The binary stays self-contained: no curl, unzip, PHP, Composer or system certificate store may be needed at run
  time. TLS root certificates are compiled in.
- The CLI contains no knowledge of SpeedySpec or WordPress. Example plugins live under `examples/` and test
  fixtures; no crate's `src/` names an application.
- Only command output goes to stdout. Every diagnostic the CLI composes goes to stderr and starts `speedyspec: `.
- Exit codes: 0 success, 1 failure, 2 usage error.
- A driver must leave nothing half-written at a destination when a download or extraction fails.
- Nothing is ever loaded from, or written relative to, the working directory unless the user named that path.
- Nothing may be installed on the host by scripts or tests. Never set `SPEEDYSPEC_ALLOW_HOST_INSTALL` on the host.
- Tests never use the public network. HTTP is served from a loopback listener started by the test.
- `speedyspec-hooks`, `speedyspec-driver` and `speedyspec-driver-zip` must not depend on `speedyspec-php` or
  ext-php-rs. Only `speedyspec-cli` names `MemoryHooks`.
- PHP files (fixtures, examples) follow `.claude/rules/coding-style.md`.
- Conventional Commits. One implementer at a time in the checkout; stage explicit paths, never `git add -A`.

## Review Focus

Each line has a test in the task named in brackets.

1. A download that is cut off mid-body, returns an HTTP error, or redirects from `https` to `http`. Nothing appears
   at the destination and the error names the URL and what happened. [Task 2]
2. An archive with hostile or odd entry names: `../x`, an absolute path, a backslash path, a drive letter, a NUL
   byte, the same path twice, a file where an earlier entry made a directory. Never a write outside the
   destination; a clear error or a defined result for each. [Task 2]
3. `--force` into a destination that already contains a symlink pointing outside the base. The install must not
   write through it. [Task 3]
4. `app install` run with stdin not a terminal and without `--no-interaction` (a script, CI). It must behave as
   non-interactive and fail with the list of what is missing, never wait for input. [Task 4]
5. A plugin declares an install option named like a built-in flag (`driver`, `part`, `base`, `force`, `help`), or
   two plugins declare the same option for one install. Neither may silently change what a flag means: the clash
   is reported against the plugin that declared the later one, and the install is refused. [Task 5]

## File Structure

```
crates/speedyspec-driver/src/
  contracts.rs        existing; add InstallOption and ZipSource mirrors and decoders
  driver.rs           new: Driver trait, DriverFlag, DriverInput, Prepared, DriverError, Progress
  lib.rs              re-exports
crates/speedyspec-driver-zip/        new crate
  Cargo.toml
  src/lib.rs          re-exports
  src/fetch.rs        Fetch trait, HttpFetch (HTTPS, redirects, GITHUB_TOKEN for api.github.com)
  src/extract.rs      verify + safe extraction of one archive into one directory
  src/install.rs      multi-source staging, placement, --force merge, clean-up
  src/driver.rs       ZipDriver: impl Driver
  tests/              loopback server helper, archive builders
crates/speedyspec-php/src/runtime.rs   add Php::construct
crates/speedyspec-cli/src/
  boot.rs             Runtime gains `hooks`
  prompt.rs           new: Prompter trait, TerminalPrompter, interactivity rule
  install.rs          new: resolving choices (app, parts, driver, options) — no I/O of its own
  app.rs              `app install` orchestration; `app list` uses the driver registry
  drivers.rs          new: the registry of drivers compiled into this binary
  plugin_cmd.rs       new: `plugin install`, `plugin remove`, source resolution
  commands.rs         clap tree additions and dispatch
crates/speedyspec-plugin/src/
  store.rs            new: directory name for a plugin, find by name, remove
examples/install-wordpress/           plugin.php, speedyspec-plugin.json, README.md
```

---

### Task 1: Driver interface and the install contracts in Rust

**Files:**
- Create: `crates/speedyspec-driver/src/driver.rs`
- Modify: `crates/speedyspec-driver/src/contracts.rs`, `src/lib.rs`
- Test: unit tests in both files

**Interfaces:**
- Consumes: `Value`, `Key` (`speedyspec-hooks`); `ObjectReader`, `ContractError`, the `Decoder` helpers already
  in `contracts.rs`.
- Produces, in `contracts.rs`:

```rust
pub struct InstallOption { pub name: String, pub label: String, pub default: Option<String>,
                           pub required: bool, pub secret: bool }
pub struct ZipSource { pub url: String, pub destination: String, pub sha256: Option<String>,
                       pub strip_components: u32 }
pub fn decode_install_option(value: &Value, reader: &dyn ObjectReader) -> Result<InstallOption, ContractError>;
pub fn decode_zip_source(value: &Value, reader: &dyn ObjectReader) -> Result<ZipSource, ContractError>;
```

- Rules. `InstallOption`: class exactly `SpeedySpec\Cli\Contracts\InstallOption`; `name` matches
  `^[a-z][a-z0-9-]*$`; `label` non-empty. `ZipSource`: class exactly `...\ZipSource`; `url` starts `https://`, or
  `http://` only when the host is `localhost`, `127.0.0.1` or `[::1]`; `destination` is relative, non-empty, has
  no `..` segment, no leading `/` or `\`, no drive letter and no NUL (`.` means the base itself); `sha256`, when
  present, is exactly 64 hex characters (stored lower-cased); `stripComponents` is an integer 0 to 32.
  `ContractError::contract` is `"InstallOption"` or `"ZipSource"`.
- Produces, in `driver.rs`:

```rust
pub struct DriverFlag { pub name: &'static str, pub help: &'static str,
                        pub takes_value: bool, pub default: Option<&'static str> }
pub struct DriverInput<'a> { pub flags: &'a BTreeMap<String, String>,   // given flags; a switch is "1"
                             pub interactive: bool }
pub struct Prepared { pub settings: Vec<(String, String)>,              // shared with plugins, never secrets
                      pub state: Box<dyn Any> }                         // the driver's own, for install()
pub struct DriverError(pub String);                                     // Display + Error
pub trait Progress { fn step(&mut self, message: &str); }               // one line of progress, to stderr

pub trait Driver {
    fn id(&self) -> &'static str;                                       // also names hook speedyspec_driver_<id>
    fn flags(&self) -> Vec<DriverFlag>;
    fn prepare(&self, input: &DriverInput) -> Result<Prepared, DriverError>;
    /// Checks what one plugin callback returned on the driver's filter.
    fn check_answer(&self, answer: &Value, reader: &dyn ObjectReader) -> Result<(), ContractError>;
    /// Runs the install with the final, already checked answer. Returns what the after-hook receives.
    fn install(&self, prepared: &Prepared, answer: &Value, reader: &dyn ObjectReader,
               progress: &mut dyn Progress) -> Result<Value, DriverError>;
}
```

- [ ] **Step 1: Write the failing tests** (fake `ObjectReader` as in the existing tests):

```rust
#[test] fn decodes_install_option_with_defaults()      // default None, required false, secret false
#[test] fn rejects_install_option_bad_name()           // "Admin Email" -> field "name"
#[test] fn rejects_install_option_wrong_class()        // stdClass -> field "", names the expected class
#[test] fn decodes_zip_source()                        // sha256 upper-case in -> lower-case out; strip 1
#[test] fn zip_url_rules() {                           // table: (url, ok?)
    // https://example.com/a.zip ok; http://127.0.0.1:8080/a.zip ok; http://localhost/a.zip ok;
    // http://[::1]:9/a.zip ok; http://example.com/a.zip err; ftp://x/a.zip err; "a.zip" err;
    // http://127.0.0.1.evil.com/a.zip err; http://localhost@evil.com/a.zip err     -> field "url"
}
#[test] fn zip_destination_rules() {                   // table: (destination, ok?)
    // "." ok; "site" ok; "a/b" ok; "" err; "/abs" err; "../x" err; "a/../../x" err; "a\\b" err;
    // "C:x" err; "a\0b" err                                                          -> field "destination"
}
#[test] fn zip_sha256_rules()                          // 63 chars err; non-hex err; None ok -> field "sha256"
#[test] fn zip_strip_components_rules()                // -1 err; 33 err; "1" err (not an int); 0 and 32 ok
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-driver`. Expected: does not compile.
- [ ] **Step 3: Implement the two decoders** with the existing `Decoder` helpers, and `driver.rs`.
- [ ] **Step 4: Run.** Expected: all pass (the 15 existing plus the 8 new).
- [ ] **Step 5: Commit.** `feat: add driver trait and install contract decoders`

---

### Task 2: Fetch, verify and extract one archive (`speedyspec-driver-zip`)

The engine both the zip driver and `plugin install` use. It puts one archive's contents into one directory that
the caller owns (a staging directory), and nothing anywhere else.

**Files:**
- Create: `crates/speedyspec-driver-zip/Cargo.toml`, `src/lib.rs`, `src/fetch.rs`, `src/extract.rs`
- Test: `tests/extract.rs`, `tests/fetch.rs`, `tests/support/mod.rs` (loopback HTTP server, zip builders)

**Interfaces:**
- Produces:

```rust
pub enum FetchError { Status { url: String, status: u16 }, Insecure { url: String, to: String },
                      TooManyRedirects { url: String }, Io { url: String, message: String } }
pub trait Fetch { fn fetch(&self, url: &str, to: &mut dyn Write) -> Result<u64, FetchError>; } // bytes written
pub struct HttpFetch;  impl HttpFetch { pub fn new() -> HttpFetch }  impl Fetch for HttpFetch

pub struct Archive<'a> { pub url: &'a str, pub sha256: Option<&'a str>, pub strip_components: u32 }
pub enum ExtractError { Fetch(FetchError), Checksum { url: String, expected: String, actual: String },
                        NotAZip { url: String, message: String }, Unsafe { entry: String, problem: String },
                        Empty { url: String }, Io { path: PathBuf, message: String } }
/// Downloads `archive` to a temporary file inside `into`'s parent, verifies it, and extracts it into the
/// empty directory `into`. On any error `into` is left empty and the temporary file is removed.
pub fn fetch_and_extract(archive: &Archive, into: &Path, fetch: &dyn Fetch) -> Result<u64, ExtractError>; // files
```

- `HttpFetch`: follows up to 10 redirects; refuses a redirect from `https` to `http` (`Insecure`); accepts
  `http` only to loopback hosts (as Task 1's URL rule); any status other than 2xx is `Status`; streams the body to
  `to` without holding it in memory; a body shorter than its `Content-Length`, or a connection reset, is `Io`.
  Sends `Authorization: Bearer $GITHUB_TOKEN` only to host `api.github.com`, and only when the variable is set.
  User agent `speedyspec/<crate version>`. TLS roots are compiled in.
- Extraction rules, applied to each entry's name after dropping `strip_components` leading segments (an entry
  with too few segments is skipped):
  - reject, as `Unsafe` naming the entry: an absolute path, a `..` segment, a backslash, a drive letter, a NUL, an
    empty segment in the middle;
  - a symlink entry is created only if its target, resolved from the entry's own directory, stays inside `into`;
    otherwise `Unsafe`;
  - before writing an entry, its parent directory must resolve (following symlinks already extracted) to a path
    inside `into`; otherwise `Unsafe`;
  - the same path twice: the later entry replaces the earlier; a file where a directory exists, or the reverse, is
    `Unsafe`;
  - file mode bits are kept, masked to `0o777`;
  - an archive that yields no files is `Empty`.
- The download and the extraction both stream to disk. Dependencies are the implementer's choice within the
  Global Constraints; say which in the report.

- [ ] **Step 1: Write the test support.** A loopback HTTP server on an ephemeral port that serves a table of
  `path -> (status, headers, body, behaviour)` where behaviour can be `Normal`, `TruncateAfter(n)` (declares the
  full `Content-Length`, sends `n` bytes, closes) or `RedirectTo(url)`; and a zip builder taking
  `&[(name, Entry)]` with `Entry::File(bytes, mode)`, `Entry::Dir`, `Entry::Symlink(target)`, able to write raw
  entry names the zip crate's safe API would refuse.

- [ ] **Step 2: Write the failing tests:**

```rust
// tests/fetch.rs
#[test] fn fetches_a_body()                       // 200 "hello" -> to == b"hello", returns 5
#[test] fn follows_redirects()                    // 302 -> 302 -> 200
#[test] fn stops_after_ten_redirects()            // loop -> TooManyRedirects
#[test] fn reports_http_status()                  // 404 -> Status { status: 404 }, nothing written
#[test] fn truncated_body_is_an_error()           // TruncateAfter(3) of 10 -> Io
#[test] fn refuses_http_to_a_non_loopback_host()  // "http://example.com/x" -> Insecure, no connection made
#[test] fn token_goes_only_to_the_github_api()    // unit test on the header-building function:
                                                  // api.github.com with GITHUB_TOKEN -> header present;
                                                  // github.com, objects.githubusercontent.com, 127.0.0.1 -> absent

// tests/extract.rs  (each: into is an empty temp dir; "leaves into empty" asserted on every error case)
#[test] fn extracts_files_and_directories()       // a.txt, d/b.txt -> contents equal; returns 2
#[test] fn strips_leading_components()            // wordpress/index.php, strip 1 -> index.php; top-level file skipped
#[test] fn keeps_mode_bits_masked()               // 0o4755 in -> 0o755 on disk  (#[cfg(unix)])
#[test] fn verifies_sha256()                      // right sum ok; wrong sum -> Checksum, into empty
#[test] fn rejects_non_zip()                      // body "not a zip" -> NotAZip
#[test] fn rejects_empty_result()                 // only directories, or everything stripped -> Empty
#[test] fn rejects_parent_traversal()             // "../x" and "a/../../x" -> Unsafe; nothing outside into
#[test] fn rejects_absolute_and_odd_names()       // "/etc/x", "a\\b", "C:x", "a\0b", "a//b" -> Unsafe each
#[test] fn symlink_inside_is_created()            // link -> ./a.txt                         (#[cfg(unix)])
#[test] fn symlink_outside_is_rejected()          // link -> ../../outside, and link -> /etc -> Unsafe
#[test] fn writes_through_an_inside_symlink()     // "real/" dir, "d" -> "./real", then "d/x" -> real/x exists
#[test] fn never_writes_through_an_outside_symlink()  // `into` already holds "d" -> an outside dir (placed by
                                                  // the test, not the archive); entry "d/x" -> Unsafe; the
                                                  // outside dir is unchanged
#[test] fn later_duplicate_replaces_earlier()     // a.txt "1" then a.txt "2" -> "2"
#[test] fn file_versus_directory_conflict()       // "a" file then "a/b" file -> Unsafe
#[test] fn failed_download_leaves_nothing()       // 500 -> Fetch(Status); into empty; no temp file beside it
```

- [ ] **Step 3: Run.** `cargo test -p speedyspec-driver-zip`. Expected: does not compile.
- [ ] **Step 4: Implement `fetch.rs` and `extract.rs`.**
- [ ] **Step 5: Run.** Expected: all pass. Then `scripts/linux.sh cargo test -p speedyspec-driver-zip` (the TLS
  and zip dependencies must build for static musl). Expected: all pass.
- [ ] **Step 6: Commit.** `feat: add fetch, verify and safe extraction of zip archives`

---

### Task 3: Installing several sources, and the zip driver

**Files:**
- Create: `crates/speedyspec-driver-zip/src/install.rs`, `src/driver.rs`
- Modify: `src/lib.rs`
- Test: `tests/install.rs`

**Interfaces:**
- Consumes: Task 1's `Driver`, `DriverFlag`, `DriverInput`, `Prepared`, `DriverError`, `Progress`, `ZipSource`,
  `decode_zip_source`; Task 2's `fetch_and_extract`, `Fetch`, `HttpFetch`.
- Produces:

```rust
pub struct Placement { pub url: String, pub destination: String, pub path: PathBuf }   // path is absolute
pub enum InstallError { Extract(ExtractError), Occupied { path: PathBuf },
                        OutsideBase { destination: String }, Io { path: PathBuf, message: String },
                        Partial { placed: Vec<Placement>, failed: String, message: String } }
pub fn install_sources(sources: &[ZipSource], base: &Path, force: bool, fetch: &dyn Fetch,
                       progress: &mut dyn Progress) -> Result<Vec<Placement>, InstallError>;
pub struct ZipDriver;   impl ZipDriver { pub fn new() -> ZipDriver }   impl Driver for ZipDriver
```

- `install_sources`, in this order:
  1. Create `base` if missing and canonicalise it. Resolve each destination against it; a destination that
     resolves outside `base` (through an existing symlink) is `OutsideBase`.
  2. Judge "occupied" **once, now**, against what is on disk: without `force`, a destination that exists and is
     not an empty directory is `Occupied`, unless it is `base` itself or an ancestor of another destination and
     contains nothing but the staging directory. With `force`, nothing is occupied.
  3. Create one staging directory `<base>/.speedyspec-staging-<random>/`, with one subdirectory per source.
  4. `fetch_and_extract` every source into its staging subdirectory. Any failure here removes the staging
     directory and returns; nothing outside staging has changed.
  5. Move into place in the order given. Without `force`: create the destination and move the staged entries in;
     a destination filled by an earlier source in this run is fine. With `force`: merge, replacing files of the
     same path and leaving everything else; never write through a symlink whose target is outside `base`
     (`Io` naming the path).
  6. If a move fails without `force`: remove what this run placed, remove staging, return the error. With
     `force`: remove staging and return `Partial` listing what was placed and what was not.
  7. Remove the staging directory and return one `Placement` per source.
- `ZipDriver`: `id()` is `"zip"`. Flags: `base` (takes a value, default `.`, help "Directory to install into")
  and `force` (switch, help "Merge into non-empty destinations, replacing files of the same path").
  `prepare` makes `base` absolute against the working directory (it need not exist) and returns settings
  `[("base", <absolute path>)]`. `check_answer` requires a list and decodes each item with `decode_zip_source`
  (errors carry the item index in `field`, e.g. `[1].url`); two items with the same `destination` are an error on
  the second. An empty list is valid at the check; `install` with an empty list is
  `DriverError("no plugin supplied anything to install")`. `install` returns a list of arrays, one per source,
  with string keys `url`, `destination`, `path`.

- [ ] **Step 1: Write the failing tests** (a fake `Fetch` backed by a map from URL to zip bytes, with an option to
  fail a given URL):

```rust
#[test] fn installs_one_source_into_an_empty_base()        // files present; one Placement with absolute path
#[test] fn creates_a_missing_base()
#[test] fn installs_three_parts_to_three_destinations()    // admin/, blog/, api/ each with its files
#[test] fn parent_then_child_destination_works()           // "." then "wp-content/plugins/x"
#[test] fn child_then_parent_destination_works()           // "wp-content/plugins/x" then "." (occupied judged once)
#[test] fn refuses_a_non_empty_destination()               // pre-existing file -> Occupied; file untouched;
                                                           // no staging directory left
#[test] fn empty_existing_directory_is_not_occupied()
#[test] fn force_merges_and_keeps_other_files()            // existing keep.txt stays; same-path file replaced
#[test] fn force_never_deletes()                           // existing directory not in the archive remains
#[test] fn failed_download_changes_nothing()               // 2nd of 3 fails -> Extract(..); base has no new
                                                           // files and no staging directory
#[test] fn failed_checksum_changes_nothing()
#[test] fn destination_through_outside_symlink_is_refused()  // base/out -> /tmp/elsewhere; destination "out/x"
                                                             // -> OutsideBase                    (#[cfg(unix)])
#[test] fn force_does_not_write_through_outside_symlink()  // base/site/link -> outside dir; archive has
                                                           // link/evil.txt -> error; outside dir unchanged
#[test] fn staging_directory_is_removed_on_success()
#[test] fn driver_prepare_makes_base_absolute()            // flags {} -> settings base == cwd; {"base":"x"} -> cwd/x
#[test] fn driver_check_answer_reports_item_index()        // 2nd item bad url -> field "[1].url"
#[test] fn driver_check_answer_rejects_duplicate_destination()  // -> field "[1].destination"
#[test] fn driver_install_returns_placements_as_arrays()   // keys url, destination, path
#[test] fn driver_install_with_no_sources_is_an_error()
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-driver-zip --test install`. Expected: does not compile.
- [ ] **Step 3: Implement `install.rs`, then `driver.rs`.**
- [ ] **Step 4: Run.** `cargo test -p speedyspec-driver-zip`. Expected: all pass.
- [ ] **Step 5: Commit.** `feat: add multi-source zip install and the zip driver`

---

### Task 4: Resolving the user's choices

Pure decision logic for `app install`, with prompting behind a trait so it can be tested without a terminal.

**Files:**
- Create: `crates/speedyspec-cli/src/prompt.rs`, `src/install.rs`
- Test: unit tests in both

**Interfaces:**
- Consumes: `App`, `AppPart`, `InstallOption` (`speedyspec-driver`).
- Produces:

```rust
// prompt.rs
pub trait Prompter {
    fn choose(&mut self, label: &str, choices: &[(String, String)]) -> Result<String, PromptError>; // (id, label)
    fn text(&mut self, label: &str, default: Option<&str>, secret: bool) -> Result<String, PromptError>;
}
pub struct PromptError(pub String);
pub struct TerminalPrompter;                       // prompts on stderr, reads stdin; secret input not echoed
/// Interactive only when the flag is absent AND stdin is a terminal.
pub fn interactive(no_interaction: bool, stdin_is_terminal: bool) -> bool;

// install.rs
pub struct Given<'a> { pub app: Option<&'a str>, pub parts: &'a [String], pub driver: Option<&'a str>,
                       pub options: &'a BTreeMap<String, String> }
pub struct Choices { pub app: String, pub parts: Vec<String>, pub driver: String }
pub enum ResolveError { Usage(String), Failure(String) }          // exit 2 / exit 1
pub fn resolve_choices(given: &Given, apps: &[App], usable_drivers: &[&str],
                       prompter: Option<&mut dyn Prompter>) -> Result<Choices, ResolveError>;
pub fn resolve_options(declared: &[InstallOption], given: &BTreeMap<String, String>,
                       prompter: Option<&mut dyn Prompter>) -> Result<Vec<(String, String)>, ResolveError>;
```

- `resolve_choices` (`prompter` is `None` when not interactive):
  - App: the given id must be a known app (`Usage` listing known ids); not given → prompt when interactive, else
    `Usage("give an app: ...")`; no apps at all → `Failure("No apps. Install a plugin that provides one.")`.
  - Parts: each given id must be a part of the app (`Usage` listing them); none given → every part whose
    `default` is true; if that is empty and the app has parts → `Usage("give --part: ...")`. An app with no
    parts installs with an empty list. Parts are never prompted for.
  - Driver: the app's drivers that are in `usable_drivers`. None → `Failure` naming the app's drivers and saying
    this binary cannot run them. Given → must be one of them (`Usage`). Not given: exactly one → use it; several
    → prompt when interactive, else `Usage("give --driver: ...")`.
- `resolve_options`: for each declared option in order: the given value if present; else, when interactive,
  prompt (`secret` options without echo, default offered); else the default; else, if `required`,
  collect it as missing. A given key that is not declared is `Usage`. After the loop, any missing required
  options are one `Usage` error listing all of them as `--<name>`. A non-required option with no value and no
  default is left out of the result. An empty answer to a prompt means "use the default"; for a required option
  with no default the prompt repeats.

- [ ] **Step 1: Write the failing tests** (a scripted `Prompter` that returns queued answers and records the
  prompts it was asked):

```rust
#[test] fn interactive_needs_a_terminal_and_no_flag()   // (false,true)->true; (true,true)->false;
                                                        // (false,false)->false; (true,false)->false
#[test] fn uses_given_app_parts_and_driver()
#[test] fn unknown_app_is_usage_error_listing_apps()
#[test] fn no_apps_is_a_failure()
#[test] fn missing_app_prompts_when_interactive()       // prompter asked once with every app
#[test] fn missing_app_without_prompter_is_usage_error()  // prompter None -> Usage; never blocks
#[test] fn defaults_to_default_parts()                  // parts a(default) b(not) -> ["a"]
#[test] fn unknown_part_is_usage_error()
#[test] fn no_default_parts_requires_part_flag()
#[test] fn app_without_parts_installs_with_none()
#[test] fn single_usable_driver_is_chosen()
#[test] fn several_usable_drivers_prompt_or_require_flag()
#[test] fn driver_not_usable_is_reported()              // app lists only "kubernetes" -> Failure naming it
#[test] fn given_driver_must_be_usable_for_the_app()
#[test] fn options_given_beat_defaults()
#[test] fn options_fall_back_to_defaults_without_prompter()
#[test] fn missing_required_options_are_listed_together()  // two missing -> one Usage naming both
#[test] fn undeclared_given_option_is_usage_error()
#[test] fn prompts_for_options_and_hides_secrets()      // recorded prompt for the secret option has secret == true
#[test] fn empty_answer_uses_default_or_asks_again()
#[test] fn optional_option_without_value_is_omitted()
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-cli install:: prompt::`. Expected: does not compile.
- [ ] **Step 3: Implement `prompt.rs` and `install.rs`.** `TerminalPrompter` reads secret input without echo.
- [ ] **Step 4: Run.** Expected: 21 pass.
- [ ] **Step 5: Commit.** `feat: resolve app install choices and options`

---

### Task 5: `speedyspec app install`

**Files:**
- Create: `crates/speedyspec-cli/src/drivers.rs`
- Modify: `crates/speedyspec-cli/src/boot.rs`, `src/app.rs`, `src/commands.rs`, `Cargo.toml`;
  `crates/speedyspec-php/src/runtime.rs` (add `construct`), its harness
- Test: `crates/speedyspec-cli/tests/install.rs`, `tests/fixtures/plugins/zip-*`, a case in
  `crates/speedyspec-php/tests/php.rs`

**Interfaces:**
- Consumes: Tasks 1, 3, 4; `AttributedHooks::validate` / `clear_validators` / `take_rejections`, `Checked`
  (`attribution.rs`); `PhpReader`, `Runtime`, `check_list` (`boot.rs`); `Php`.
- Produces:
  - `Php::construct(&self, class: &str, args: &[Value]) -> Result<Value, PhpError>`: a new PHP object. Implement it
    with a PHP closure evaluated once at boot (`static fn (string $c, array $a) => new $c(...$a)`) and
    `Php::call`, not with new unsafe code.
  - `Runtime` gains `pub hooks: Rc<AttributedHooks>`, declared before `php` so it is dropped first.
  - `drivers::all() -> Vec<Box<dyn Driver>>` (only `ZipDriver`) and `drivers::ids() -> Vec<&'static str>`.
    `app list` marks a driver with `?` only when it is not in `ids()`.
  - Hooks fired, with the PHP values they carry:

| Hook | Kind | Value / arguments | Returns |
|---|---|---|---|
| `speedyspec_install_options` | filter | value: empty list; args: app id, driver id | list of `InstallOption` |
| `speedyspec_install_before` | action | args: `InstallRequest` | |
| `speedyspec_driver_zip` | filter | value: empty list; args: `InstallRequest` | list of `ZipSource` |
| `speedyspec_install_after` | action | args: `InstallRequest`, result | |

- Command: `speedyspec app install [<app>] [--part <id>]... [--driver <id>] [driver flags] [--<option> <value>]...`
  - Driver flags from every driver in `drivers::all()` are declared on the subcommand. A flag belonging to a
    driver other than the chosen one is a usage error naming the driver it belongs to.
  - Option flags need two passes, because options depend on the app and driver. Before building the clap tree,
    scan the arguments: if the command is `app install`, an app id is present, and the driver is determined
    (given with `--driver`, or the app has exactly one usable driver), fire `speedyspec_install_options` and add
    one `--<name> <value>` flag per accepted option. Otherwise no option flags exist, and an unknown `--x` gets
    clap's usage error plus one line: `speedyspec: install options can be given as flags once --driver is known`.
  - Option acceptance, checked per plugin callback with a validator so the plugin is named: decodes as
    `InstallOption`; name not `part`, `driver`, `help`, `version`, `no-plugins`, `no-interaction`, `plugin-dir`,
    nor any driver flag name (`[i].name: "<name>" is a built-in flag`); not already declared
    (`[i].name: duplicate name "<name>"`).
  - **Any rejection on an install hook refuses the install.** The spec: a plugin returning something that does not
    match a contract is a failure naming the plugin, the hook and the field, and nothing is installed. So after
    the options filter and after the driver filter, if `take_rejections()` is non-empty: print each as
    `speedyspec: <owner>: <hook>: <problem>`, then `speedyspec: nothing was installed`, exit 1. (This differs
    from start-up, where a bad app or command is dropped and the rest carry on.)
- Flow, after clap matches `app install`:
  1. If `php.poisoned()`: `speedyspec: a plugin failed while loading, so nothing will be installed; remove it or
     run with --no-plugins` and exit 1.
  2. `resolve_choices`; then options (reuse the first pass's list when it ran, else fire the filter now), then
     `resolve_options`. Interactive per `prompt::interactive`.
  3. `driver.prepare`. Build the request with `Php::construct("SpeedySpec\\Cli\\Contracts\\InstallRequest",
     [app, parts, driver, options as a string-keyed array, settings as a string-keyed array])`.
  4. `speedyspec_install_before`. A hook error aborts: `speedyspec: <owner>: speedyspec_install_before: <message>`,
     exit 1, nothing installed.
  5. Filter `speedyspec_driver_<id>` with a validator that calls `driver.check_answer` on each callback's return,
     so a bad return is blamed on exactly its plugin. Any rejection refuses the install (see above).
  6. `driver.install`. On failure: `speedyspec: <message>`, exit 1; the after-hook is not fired.
  7. `speedyspec_install_after` with the request and the result. A hook error is reported and the exit code is 1,
     but what was installed stays; say so: `speedyspec: installed, but <owner>: speedyspec_install_after: ...`.
  8. Print one line per placement to stdout: `<destination>\t<absolute path>`. Exit 0.
  - PHP output is on stderr during steps 2 and 5 (filters) and on stdout during steps 4 and 7 (the actions are
    where a plugin speaks to the user). Validators are cleared on every path. Progress lines go to stderr.

- [ ] **Step 1: Write the PHP harness case and see it fail.** In `crates/speedyspec-php/tests/php.rs`:

```rust
case_construct_builds_an_object      // preload a class P { public function __construct(public string $a,
                                     // public array $b) {} }; construct("P", ["x", ["k" => "v"]]);
                                     // class_of == "P"; properties_of == [("a","x"), ("b", {k: v})]
case_construct_unknown_class         // construct("Nope", []) -> Err(Exception) naming "Nope"; next eval works
case_construct_wrong_args            // construct("P", []) -> Err(Exception { class: "ArgumentCountError", .. })
```

- [ ] **Step 2: Implement `Php::construct`.** Run `cargo test -p speedyspec-php --test php`. Expected: all `ok`.

- [ ] **Step 3: Create the fixture plugins** (`speedyspec-fixture/<dir>`, contracts `^1.0`). Each reads the server
  address from the environment variable `ZIP_FIXTURE_URL` so tests can point it at their loopback server:

| Fixture | What it does |
|---|---|
| `zip-app` | App `site` ("Site", parts `web` default and `docs` not default, drivers `zip`). On `speedyspec_driver_zip` for app `site`: one `ZipSource` per selected part, `web` → `$url/web.zip` to `.`, `docs` → `$url/docs.zip` to `docs`, `stripComponents` 1 for `web`. Declares options `title` (required) and `token` (secret, optional) for app `site`. `speedyspec_install_before` echoes `before:<app>`; `speedyspec_install_after` echoes `after:<title> <number of placements> <strlen of token, 0 when absent>`. Ignores other apps. |
| `zip-two-drivers` | App `multi` with drivers `zip` and `kubernetes`. Answers zip with one source. |
| `zip-k8s-only` | App `cluster` with driver `kubernetes` only. |
| `zip-bad-answer` | For app `site`, appends a `ZipSource` with url `ftp://x/a.zip`. |
| `zip-bad-option` | For app `site`, declares options `driver` and `title`. |
| `zip-veto` | `speedyspec_install_before` throws `RuntimeException('not allowed')`. |
| `zip-after-fails` | `speedyspec_install_after` throws `RuntimeException('post step failed')`. |
| `zip-silent` | App `empty` with driver `zip`; answers nothing. |

- [ ] **Step 4: Write the failing tests in `tests/install.rs`.** Each starts the loopback server from Task 2's
  support code (move it to a small shared dev crate or include it by path; say which) serving `web.zip`
  (`site/index.html`, `site/app.js`) and `docs.zip` (`readme.md`), runs the binary with an empty
  `SPEEDYSPEC_HOME`, `--no-interaction` unless stated, and `--base <temp dir>`:

```rust
#[test] fn installs_default_parts()               // zip-app; `app install site --title Hi`: exit 0;
                                                  // base/index.html and base/app.js exist; no base/docs;
                                                  // stdout == "before:site\nafter:Hi 1 0\n.\t<base>\n"
#[test] fn installs_selected_parts()              // --part web --part docs -> base/docs/readme.md too;
                                                  // "after:Hi 2 0"; two placement lines
#[test] fn unknown_part_is_usage_error()          // --part nope -> exit 2; stderr lists web, docs
#[test] fn missing_required_option_is_usage_error()  // no --title -> exit 2; stderr contains "--title"; base empty
#[test] fn secret_option_is_passed_to_the_plugin()   // --token s3cret -> stdout has "after:Hi 1 6"; neither
                                                     // stdout nor stderr contains "s3cret"
#[test] fn refuses_non_empty_base_without_force() // base has keep.txt -> exit 1; stderr names the path;
                                                  // keep.txt untouched; no .speedyspec-staging-* left
#[test] fn force_merges()                         // --force -> exit 0; keep.txt still there; index.html present
#[test] fn failed_download_installs_nothing()     // server returns 500 for web.zip -> exit 1; stderr names the
                                                  // URL and 500; base has no files; after-hook not run
                                                  // (stdout has "before:site" and no "after:")
#[test] fn driver_is_chosen_when_only_one_is_usable()   // zip-app without --driver works (covered above);
                                                        // zip-two-drivers without --driver also works: only
                                                        // zip is usable in this binary
#[test] fn unusable_driver_is_reported()          // zip-k8s-only: `app install cluster` -> exit 1; stderr
                                                  // names "kubernetes"
#[test] fn given_unknown_driver_is_usage_error()  // --driver ftp -> exit 2
#[test] fn bad_answer_refuses_the_install()       // zip-app + zip-bad-answer: exit 1; base empty; stderr has
                                                  // "speedyspec-fixture/zip-bad-answer", "speedyspec_driver_zip",
                                                  // ".url" and "nothing was installed"; never names zip-app
#[test] fn clashing_option_refuses_the_install()  // zip-app + zip-bad-option, with --title Hi: exit 1; base
                                                  // empty; stderr reports `"driver" is a built-in flag` and
                                                  // the duplicate "title", both against zip-bad-option
#[test] fn before_hook_can_veto()                 // zip-app + zip-veto -> exit 1; stderr has "not allowed"
                                                  // and "speedyspec-fixture/zip-veto"; base empty
#[test] fn after_hook_failure_keeps_the_install() // zip-app + zip-after-fails -> exit 1; stderr has
                                                  // "installed, but" and "post step failed"; files present
#[test] fn nothing_to_install_is_an_error()       // zip-silent: `app install empty` -> exit 1; stderr has
                                                  // "no plugin supplied anything to install"
#[test] fn unknown_app_lists_apps()               // `app install nope` -> exit 2; stderr lists "site"
#[test] fn no_terminal_means_no_prompt()          // zip-app, stdin = /dev/null, WITHOUT --no-interaction and
                                                  // without --title -> exit 2 within the test timeout; never hangs
#[test] fn refuses_to_install_on_a_poisoned_engine()  // zip-app + the existing real-fatal-at-load fixture ->
                                                      // exit 1; stderr has "nothing will be installed"; base empty
#[test] fn option_flags_need_a_known_driver()     // zip-app; `app install --title x` (no app given, so no
                                                  // option flags exist) -> exit 2; stderr has clap's error
                                                  // and the "once --driver is known" line
#[test] fn app_list_marks_only_unimplemented_drivers()  // zip-two-drivers -> "multi\tMulti\tzip,kubernetes?\n"
#[test] fn help_shows_driver_and_option_flags()   // `app install site --help` (zip-app) lists --base, --force,
                                                  // --title, --token
```

- [ ] **Step 5: Run.** `cargo test -p speedyspec-cli --test install`. Expected: does not compile or fails.
- [ ] **Step 6: Implement** `drivers.rs`, the `Runtime` change, the two-pass tree in `commands.rs`, and the flow in
  `app.rs`.
- [ ] **Step 7: Run on macOS.** `cargo test --workspace`. Expected: all pass, including the existing suites.
- [ ] **Step 8: Run on Linux.** `scripts/linux.sh cargo test --workspace`. Expected: all pass.
- [ ] **Step 9: Commit.** `feat: add app install with the zip driver`

---

### Task 6: `speedyspec plugin install` and `plugin remove`

**Files:**
- Create: `crates/speedyspec-plugin/src/store.rs`, `crates/speedyspec-cli/src/plugin_cmd.rs`
- Modify: `crates/speedyspec-plugin/src/lib.rs`, `crates/speedyspec-cli/src/commands.rs`, `Cargo.toml`
- Test: unit tests in `store.rs` and `plugin_cmd.rs`; `crates/speedyspec-cli/tests/plugin_install.rs`

**Interfaces:**
- Consumes: `fetch_and_extract`, `Archive`, `Fetch`, `HttpFetch` (Task 2); `parse_manifest`, `Plugin`,
  `user_plugin_dir`, `discover` (`speedyspec-plugin`); `Prompter`, `interactive` (Task 4);
  `CONTRACTS_VERSION`.
- Produces:

```rust
// speedyspec-plugin store.rs
pub fn directory_name(plugin_name: &str) -> String;         // "vendor/name" -> "vendor.name"
pub fn find_installed(plugins_dir: &Path, plugin_name: &str) -> Option<PathBuf>;  // by manifest name
pub fn remove_installed(plugins_dir: &Path, plugin_name: &str) -> Result<PathBuf, PluginError>;

// speedyspec-cli plugin_cmd.rs
pub enum Source { Url(String), GitHub { owner: String, repo: String, tag: Option<String> } }
pub fn parse_source(text: &str) -> Result<Source, String>;
pub struct Resolved { pub url: String, pub strip_components: u32, pub describe: String }
pub fn resolve_github(owner: &str, repo: &str, tag: Option<&str>, api_base: &str, fetch: &dyn Fetch)
    -> Result<Resolved, String>;
```

- `parse_source`: a string starting `https://` or `http://` is `Url` (the URL rule of Task 1 applies);
  `owner/repo` or `owner/repo@tag` with `owner` and `repo` matching `^[A-Za-z0-9._-]+$` is `GitHub`; anything
  else is an error showing both accepted forms.
- `resolve_github`: GET `<api_base>/repos/<owner>/<repo>/releases/latest`, or `/releases/tags/<tag>`. If the
  release has an asset named exactly `speedyspec-plugin.zip`, use its `browser_download_url` with strip 0;
  otherwise use the release's `zipball_url` with strip 1. A 404 is `no release found for <owner>/<repo>[@tag]`.
  `api_base` is `https://api.github.com` unless `SPEEDYSPEC_GITHUB_API` is set (it must pass the Task 1 URL
  rule; it exists for tests and for GitHub Enterprise).
- `speedyspec plugin install <source> [--yes]`:
  1. No user plugin directory (`user_plugin_dir().dir` is `None`) → exit 1 with the warnings already printed and
     `speedyspec: there is no user plugin directory to install into; set SPEEDYSPEC_HOME to an absolute path`.
  2. Resolve the source. Print to stderr the URL it will download and
     `This plugin will run as you, with full access to your files and network.` Without `--yes`: ask to confirm
     when interactive; when not interactive, exit 2 with `speedyspec: confirm with --yes`.
  3. `fetch_and_extract` into a staging directory `<plugins>/.speedyspec-staging-<random>/new`.
  4. Find the plugin root: the staging directory if it holds a manifest, else its single subdirectory if that
     holds one; otherwise fail (`the archive has no speedyspec-plugin.json`).
  5. `parse_manifest` on it. Refuse a plugin whose `contracts` range the bundled version does not satisfy, with
     both versions in the message.
  6. Target is `<plugins>/<directory_name(name)>`. If a plugin with the same manifest name is installed (anywhere
     in the user plugin directory), move it aside inside staging, move the new one in, then delete the old; if
     moving the new one in fails, move the old one back.
  7. Remove staging. Print `<name>\t<version>\t<target dir>` to stdout. Exit 0.
  On any failure: staging is removed, an existing plugin of that name is untouched, exit 1.
- `speedyspec plugin remove <name> [--yes]`: only plugins in the user plugin directory can be removed
  (`speedyspec: <name> is not installed in <plugins dir>`, exit 1, also when it is loaded from `--plugin-dir`).
  Confirmation as for install. Prints `<name>\t<dir>` on stdout.
- Both commands work with `--no-plugins`, and are **not** refused on a poisoned engine: they are how a broken
  plugin gets fixed or removed. They never call PHP.

- [ ] **Step 1: Write the failing unit tests:**

```rust
// store.rs
#[test] fn directory_name_replaces_the_slash()      // "acme/install-wp" -> "acme.install-wp"
#[test] fn finds_an_installed_plugin_by_manifest_name()   // directory named anything
#[test] fn remove_deletes_only_that_plugin()
#[test] fn remove_unknown_is_an_error()

// plugin_cmd.rs
#[test] fn parses_sources() {                        // table
    // "https://x.test/p.zip" -> Url; "acme/plugin" -> GitHub{tag None}; "acme/plugin@v1.2.0" -> tag Some;
    // "http://example.com/p.zip" -> Err; "acme" -> Err; "acme/plugin/extra" -> Err; "../x/y" -> Err; "" -> Err
}
#[test] fn github_prefers_the_named_asset()          // fake Fetch returns release JSON with the asset ->
                                                     // url == browser_download_url; strip 0
#[test] fn github_falls_back_to_the_source_archive() // no such asset -> zipball_url; strip 1
#[test] fn github_tag_uses_the_tags_endpoint()       // fake records the requested URL: .../releases/tags/v1.2.0
#[test] fn github_missing_release_is_reported()      // 404 -> message names owner/repo
```

- [ ] **Step 2: Write the failing integration tests in `tests/plugin_install.rs`** (loopback server; `SPEEDYSPEC_HOME`
  is a temp directory; the plugin archive is built from the existing `good` fixture):

```rust
#[test] fn installs_from_a_url_then_the_plugin_loads()   // `plugin install <url> --yes` -> exit 0; stdout
                                                         // "speedyspec-fixture/good\t1.0.0\t<home>/plugins/
                                                         // speedyspec-fixture.good\n"; then `hello Ada` works
#[test] fn archive_with_one_wrapping_directory_works()   // good-1.0.0/plugin.php ...
#[test] fn shows_the_warning_and_needs_yes()             // without --yes, not interactive -> exit 2; stderr has
                                                         // "will run as you"; nothing installed
#[test] fn installs_from_owner_repo_through_the_api()    // SPEEDYSPEC_GITHUB_API=<loopback>; release JSON
                                                         // points at <loopback>/asset.zip
#[test] fn archive_without_manifest_is_refused()         // exit 1; plugins dir has no new entry, no staging
#[test] fn incompatible_contracts_are_refused()          // manifest "^2.0" -> exit 1; stderr has "^2.0", "1.0.0"
#[test] fn replaces_an_installed_plugin_of_the_same_name()   // install v1.0.0 then v1.1.0 -> plugin list shows
                                                             // 1.1.0 once
#[test] fn bad_new_archive_leaves_the_old_plugin()       // v1.0.0 installed; new archive has an invalid
                                                         // manifest -> exit 1; `plugin list` still 1.0.0
#[test] fn works_when_an_installed_plugin_is_broken()    // a plugin with a fatal at load is installed;
                                                         // `plugin install <url> --yes` still exits 0
#[test] fn remove_deletes_the_plugin()                   // `plugin remove speedyspec-fixture/good --yes` ->
                                                         // exit 0; `plugin list` empty
#[test] fn remove_refuses_a_plugin_dir_plugin()          // loaded via --plugin-dir only -> exit 1,
                                                         // "is not installed in"
#[test] fn no_user_plugin_directory_is_reported()        // SPEEDYSPEC_HOME unset, HOME relative -> exit 1
```

- [ ] **Step 3: Run.** `cargo test -p speedyspec-plugin store::` and `cargo test -p speedyspec-cli --test
  plugin_install`. Expected: does not compile.
- [ ] **Step 4: Implement `store.rs`, then `plugin_cmd.rs` and the clap wiring.** JSON parsing of the release
  response needs `serde_json` as a normal dependency of `speedyspec-cli`.
- [ ] **Step 5: Run.** `cargo test --workspace` on macOS, then `scripts/linux.sh cargo test --workspace`.
  Expected: all pass.
- [ ] **Step 6: Commit.** `feat: add plugin install and plugin remove`

---

### Task 7: Example plugin, documentation, and an end-to-end check

**Files:**
- Create: `examples/install-wordpress/speedyspec-plugin.json`, `plugin.php`, `README.md`
- Modify: `README.md`, `docs/php-embed.md` (only if a statement in it is now wrong)
- Test: `crates/speedyspec-cli/tests/examples.rs`

**Interfaces:**
- Consumes: the finished binary.
- Produces: a plugin named `speedyspec/install-wordpress` that registers app `wordpress` ("WordPress", no parts,
  driver `zip`) and answers `speedyspec_driver_zip` for that app with one
  `ZipSource('https://wordpress.org/latest.zip', '.', null, 1)`. It declares no options and does nothing else.

- [ ] **Step 1: Write the failing test in `tests/examples.rs`:**

```rust
#[test] fn wordpress_example_loads_and_is_listed()   // --plugin-dir examples/install-wordpress; `app list` ->
                                                     // stdout "wordpress\tWordPress\tzip\n"; stderr empty
#[test] fn wordpress_example_follows_the_php_style() // plugin.php starts "<?php\n\ndeclare(strict_types=1);\n",
                                                     // has no tabs, no line over 120 chars, no trailing spaces
```

- [ ] **Step 2: Run.** Expected: fails (no such directory).
- [ ] **Step 3: Write the example plugin and its README** (what it does, how to install the plugin from a
  checkout with `--plugin-dir`, the command to run).
- [ ] **Step 4: Run.** `cargo test -p speedyspec-cli --test examples`. Expected: 2 pass.
- [ ] **Step 5: End-to-end check against the real site (needs network; run once, by hand, not in the suite).**
  `cargo build --release -p speedyspec-cli`, then with `SPEEDYSPEC_HOME` set to an empty temp directory:
  `target/release/speedyspec --plugin-dir examples/install-wordpress app install wordpress --base <temp>/site`.
  Expected: exit 0; `<temp>/site/wp-load.php` and `<temp>/site/wp-includes/version.php` exist; there is no
  `<temp>/site/wordpress/` directory; stdout is `.\t<temp>/site`. Run it again: exit 1, "not empty" style message
  naming the path. Run it a third time with `--force`: exit 0. Record the output and the binary sizes
  (macOS release; Linux release via `scripts/linux.sh`) in the task report and the size in `docs/php-embed.md`.
- [ ] **Step 6: Update `README.md`:** the install commands, the four install hooks with a ten-line plugin example
  in the project PHP style, the zip contract (`url`, `destination`, `sha256`, `stripComponents`), what `--force`
  does and does not do, `plugin install` sources and its warning, and where plugins are stored.
- [ ] **Step 7: Run everything.** `cargo test --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`,
  `cargo fmt --all --check`, `scripts/linux.sh cargo test --workspace`. Expected: all clean.
- [ ] **Step 8: Commit.** `docs: add WordPress example plugin and document installing`

---

## What this plan leaves for later plans

- The Kubernetes driver, `app uninstall`, and the Kubernetes example plugin (plan 3).
- `self update`, release binaries for all four targets, fetching a pinned contracts release at build time, and a
  release workflow for the contracts repository (plan 4).
- (S)FTP, Docker, Composer, Functions and AWS drivers; hook backends other than memory. Not in any current plan.
- Carried over from the foundation review, still open: PHP warning lines do not name the plugin;
  `php://stdout` and `fwrite(STDOUT, ...)` bypass the start-up redirect; `has_filter($hook, $cb)` returns a bool.
