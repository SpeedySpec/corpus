# SpeedySpec CLI Kubernetes Driver Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `speedyspec app install <app> --driver kubernetes --namespace <ns>` applies the resources a plugin returns
to a cluster, waits for them, and records the install so `speedyspec app uninstall <name>` can remove it, with no
Helm or kubectl on the machine.

**Architecture:** A new crate `speedyspec-driver-kubernetes` implements the existing `Driver` trait. It talks to
the cluster through the `kube` client on a current-thread async runtime kept inside the crate, so the trait and the
rest of the CLI stay synchronous. Pure logic (manifest identity, the install record, readiness rules) is separate
from the code that calls the cluster, and is tested without one; the calling code is tested against an in-memory
fake cluster served over loopback HTTP, and once end to end against a throwaway k3s in Docker.

**Tech Stack:** Rust 1.99; `kube` 5 (default features: `client`, `rustls-tls`, `ring`), `k8s-openapi`, `tokio`
(current-thread runtime only), `serde_json`; the existing workspace crates.

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-cli-design.md` (section "Kubernetes driver" and
"Kubernetes contract")

**This is plan 3 of 4.** Plan 2 (install flow, zip driver) is on the branch `feat/install-zip`, open as
`SpeedySpec/speedyspec-cli#1`. Plan 4 is self-update and release builds.

## Global Constraints

- Work in `packages/speedyspec-cli` on a branch `feat/kubernetes`, cut from `main` if pull request 1 has been
  merged, otherwise from `feat/install-zip`. Do not push or open a pull request without Jacob's go-ahead.
- **Never touch a real cluster.** No test, script or manual check may use the user's kubeconfig, context or
  credentials. Every test that runs the binary or builds a client sets `KUBECONFIG` to a path that does not exist
  and passes an explicit `--server`. The only cluster ever contacted is the fake one on loopback, or the throwaway
  k3s container this plan's script starts and removes.
- `packages/speedyspec-cli-contracts` is not changed: `KubernetesResources` and `WaitFor` 1.0.0 are used as they
  are. If a task seems to need a contracts change, stop and report.
- The binary stays self-contained: no kubectl, Helm or system certificate store at run time.
- The token is never written to stdout, stderr, the install record, the install request, a progress line or an
  error message. Resource bodies (which may hold Secret data) are never printed; messages name `kind/name` only.
- Only command output goes to stdout. Every diagnostic the CLI composes goes to stderr and starts `speedyspec: `.
- Exit codes: 0 success, 1 failure, 2 usage error.
- The CLI contains no knowledge of any application. Example plugins live under `examples/` and test fixtures.
- `speedyspec-driver` and `speedyspec-driver-kubernetes` must not depend on `speedyspec-php` or ext-php-rs. The
  `Driver` trait stays synchronous; no async type appears in any public signature outside the new crate.
- Nothing may be installed on the host. Docker images may be pulled; `kind`, `k3d`, `kubectl` and `helm` may not
  be installed. Never set `SPEEDYSPEC_ALLOW_HOST_INSTALL` on the host.
- Field manager for every apply: `speedyspec`. Labels on every applied resource:
  `app.kubernetes.io/managed-by=speedyspec`, `speedyspec.com/app=<app id>`,
  `speedyspec.com/install=<install name>`. Install record: a Secret named `speedyspec-install-<install name>`,
  type `speedyspec.com/install`, in `--namespace`.
- PHP files (fixtures, examples) follow `.claude/rules/coding-style.md`.
- Conventional Commits. One implementer at a time in the checkout; stage explicit paths, never `git add -A`.

## Review Focus

Each line has a test in the task named in brackets.

1. Installing again under the same name with a smaller or different set of resources. What was applied before and
   is no longer returned stays running, is reported, and stays in the record so uninstall still removes it.
   [Task 4, Task 5]
2. An install that fails part-way: the API rejects the third resource, or a workload never becomes ready. The
   record already exists, says so, and lists everything attempted, so `app uninstall` cleans up. [Task 5, Task 8]
3. A field owned by another manager (someone ran `kubectl edit`, or a controller writes it). The install fails
   naming the resource, the field and the other manager, and changes nothing by force unless `--force-conflicts`
   was given. [Task 5]
4. Uninstalling something that has drifted: a recorded resource is already gone; a recorded resource no longer
   carries this install's label (it was replaced by something else); the record is missing; a namespace the
   install did not create. Gone is fine, relabelled is skipped and reported, missing is a clear error, and a
   namespace the install did not create is never deleted. [Task 8]
5. Credentials and secrets in output: a wrong or expired token, an empty `--token-file -`, an API error on a
   Secret. The message says what failed without the token or any resource body. [Task 3, Task 7]

## File Structure

```
crates/speedyspec-driver/src/
  driver.rs            PluginRef; install() gains `plugins`; uninstall support (plan + run), default "cannot"
  contracts.rs         KubernetesResources and WaitFor mirrors, decode_kubernetes_resources, value_to_json
crates/speedyspec-driver-zip/src/driver.rs      follows the trait change
crates/speedyspec-driver-kubernetes/            new crate
  src/lib.rs           re-exports
  src/connect.rs       flags + environment -> a cluster connection (server/token, or kubeconfig)
  src/manifest.rs      resource identity, defaulting the namespace, labels, checks beyond the contract
  src/record.rs        the install record: schema, encode/decode, merging with the previous record
  src/readiness.rs     is this object ready? pure rules over the object's JSON
  src/cluster.rs       the calls: discovery, namespaces, apply, get, delete, waiting
  src/install.rs       the install sequence over cluster.rs
  src/uninstall.rs     the uninstall sequence
  src/driver.rs        KubernetesDriver: impl Driver, owns the async runtime
  tests/               install.rs, uninstall.rs (fake cluster); e2e.rs (#[ignore], real k3s)
crates/speedyspec-testkit/src/
  server.rs            a programmable handler (method, path, query, headers, body -> response)
  kube.rs              FakeCluster: an in-memory API server for the kinds the tests use
crates/speedyspec-cli/src/
  attribution.rs       remembers which plugins answered a validated filter
  app.rs               passes the answering plugins to the driver; `app uninstall`
  drivers.rs           registers KubernetesDriver
  commands.rs          clap tree for `app uninstall`
examples/install-demo-kubernetes/    plugin.php, speedyspec-plugin.json, README.md
scripts/k3s.sh         up | down | kubeconfig: a throwaway k3s in Docker for the end-to-end test
```

---

### Task 1: Driver interface: answering plugins and uninstall

**Files:**
- Modify: `crates/speedyspec-driver/src/driver.rs`, `src/lib.rs`; `crates/speedyspec-driver-zip/src/driver.rs`;
  `crates/speedyspec-cli/src/attribution.rs`, `src/app.rs`
- Test: unit tests in `attribution.rs`; `crates/speedyspec-driver-zip/tests/install.rs`;
  `crates/speedyspec-cli/tests/install.rs`

**Interfaces:**
- Consumes: the `Driver` trait and `AttributedHooks` as they are on `feat/install-zip`.
- Produces, in `driver.rs`:

```rust
pub struct PluginRef { pub name: String, pub version: String }

pub struct UninstallInput<'a> { pub name: &'a str,                       // the install name
                                pub flags: &'a BTreeMap<String, String>,
                                pub interactive: bool }
pub struct UninstallPlan { pub summary: Vec<String>,                     // shown before confirming
                           pub state: Box<dyn Any> }

pub trait Driver {
    // unchanged: id, flags, prepare, seed, check_answer, report
    fn install(&self, prepared: &Prepared, answer: &Value, reader: &dyn ObjectReader,
               plugins: &[PluginRef], progress: &mut dyn Progress) -> Result<Value, DriverError>;

    /// Flags `app uninstall` accepts for this driver. Empty means the driver cannot uninstall.
    fn uninstall_flags(&self) -> Vec<DriverFlag> { Vec::new() }
    /// Finds the install and says what removing it will delete. Changes nothing.
    fn plan_uninstall(&self, input: &UninstallInput) -> Result<UninstallPlan, DriverError> {
        Err(DriverError(format!("the {} driver cannot uninstall", self.id())))
    }
    /// Removes what `plan_uninstall` found. Returns lines of command output.
    fn uninstall(&self, plan: &UninstallPlan, progress: &mut dyn Progress)
        -> Result<Vec<String>, DriverError> {
        Err(DriverError(format!("the {} driver cannot uninstall", self.id())))
    }
}
```

- Produces, in `attribution.rs`: `AttributedHooks::take_contributors(&self) -> Vec<String>`: the owners, in call
  order and without repeats, of every wrapped callback that returned on a filter that had a validator since the
  last call. A callback with no owner contributes nothing. Cleared by `take_contributors` and by
  `clear_validators`.
- `app.rs`: after the driver filter, `take_contributors()`, map each name to its version through
  `runtime.loaded`, and pass the list to `driver.install`.

- [ ] **Step 1: Write the failing tests.**

```rust
// attribution.rs
#[test] fn contributors_are_the_owners_that_answered_a_validated_filter()
    // owners "a/a" and "b/b" each add a callback on "h"; validate("h"); filter -> ["a/a", "b/b"];
    // a second take_contributors() is empty
#[test] fn contributors_skip_unvalidated_hooks_and_unowned_callbacks()
#[test] fn contributors_have_no_repeats()               // one owner, two callbacks -> one entry

// speedyspec-driver-zip tests/install.rs
#[test] fn zip_driver_cannot_uninstall()                // uninstall_flags() empty; plan_uninstall is Err with
                                                        // "the zip driver cannot uninstall"

// speedyspec-cli tests/install.rs: existing tests must still pass unchanged (the zip driver ignores `plugins`)
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-cli --bin speedyspec attribution::` and
  `cargo test -p speedyspec-driver-zip --test install`. Expected: does not compile.
- [ ] **Step 3: Implement** the trait changes, the zip driver's new signature, contributor tracking and the
  `app.rs` call.
- [ ] **Step 4: Run.** `cargo test --workspace`. Expected: all pass (258 existing plus the 4 new).
- [ ] **Step 5: Commit.** `feat: tell drivers which plugins answered and let them uninstall`

---

### Task 2: The Kubernetes contract in Rust

**Files:**
- Modify: `crates/speedyspec-driver/src/contracts.rs`, `src/lib.rs`, `Cargo.toml` (add `serde_json`)
- Test: unit tests in `contracts.rs`

**Interfaces:**
- Consumes: `Value`, `Key`, `ObjectReader`, `ContractError`, the `Decoder` helpers.
- Produces:

```rust
pub struct WaitFor { pub kind: String, pub name: String, pub namespace: Option<String>,
                     pub condition: Option<String> }
pub struct KubernetesResources { pub resources: Vec<serde_json::Value>,   // each a JSON object
                                 pub wait_for: Option<Vec<WaitFor>> }     // None = the default set
pub fn decode_kubernetes_resources(value: &Value, reader: &dyn ObjectReader)
    -> Result<KubernetesResources, ContractError>;
pub fn value_to_json(value: &Value, reader: &dyn ObjectReader) -> Result<serde_json::Value, String>;
```

- `value_to_json`, the rule a PHP developer already knows from `json_encode`: a PHP list (keys 0..n in order)
  is a JSON array, **including the empty array `[]`**; any other array is a JSON object with its keys as
  strings; an object of class `stdClass` is a JSON object of its properties, so `new stdClass` is `{}`; null,
  bool, int, float and string map directly. Refused, with the reason: an object of any other class, a string
  that is not UTF-8, a float that is NaN or infinite.
- `decode_kubernetes_resources`. The value must be an object of class exactly
  `SpeedySpec\Cli\Contracts\KubernetesResources` (`ContractError::contract` is `"KubernetesResources"`).
  - `resources`: a list. Each item converts with `value_to_json` to a JSON object having string `apiVersion`
    and `kind` (both non-empty) and an object `metadata` with a non-empty string `name`; `metadata.namespace`,
    when present, is a non-empty string. Errors use paths such as `resources[2].metadata.name`. Two items with
    the same `apiVersion` group, `kind`, namespace and name are an error on the later one.
  - `waitFor`: null, or a list of objects of class exactly `...\WaitFor` with non-empty `kind` and `name`.
    Each must name a resource in `resources` by `kind` and `name` (and `namespace` when it gives one); otherwise
    an error at `waitFor[i]` saying no such resource is being installed.
- An empty `resources` list is valid here; the driver refuses to install nothing.

- [ ] **Step 1: Write the failing tests** (fake `ObjectReader`, extended to answer for `stdClass`):

```rust
#[test] fn json_conversion_follows_json_encode() {
    // [] -> [];  [1, "a"] -> [1,"a"];  ["k" => 1] -> {"k":1};  [1 => "a"] -> {"1":"a"};
    // stdClass{} -> {};  stdClass{a: [stdClass{}]} -> {"a":[{}]};  nested mixes
}
#[test] fn json_conversion_refuses_other_objects_and_bad_scalars()   // a Closure-like Opaque, Bytes, NaN
#[test] fn decodes_resources_and_default_wait()        // waitFor null -> wait_for None
#[test] fn decodes_an_explicit_empty_wait()            // waitFor [] -> Some(vec![])
#[test] fn rejects_wrong_class()                       // field "", names the expected class
#[test] fn rejects_resource_without_identity() {       // table -> field
    // no apiVersion -> resources[0].apiVersion; kind "" -> resources[0].kind;
    // no metadata -> resources[0].metadata; metadata.name missing -> resources[0].metadata.name;
    // metadata.namespace "" -> resources[0].metadata.namespace; a list item that is a string -> resources[0]
}
#[test] fn rejects_duplicate_resources()               // same kind/ns/name twice -> resources[1]
#[test] fn same_name_in_two_namespaces_is_not_a_duplicate()
#[test] fn wait_for_must_name_an_installed_resource()  // -> waitFor[0]
#[test] fn wait_for_namespace_narrows_the_match()
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-driver`. Expected: does not compile.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: all pass.
- [ ] **Step 5: Commit.** `feat: decode the Kubernetes contract`

---

### Task 3: Prove the client links, and connect to a cluster

This is the risk gate, as the PHP proof was in plan 1: the Kubernetes client must build into the static musl
binary and talk to an API server, before anything is built on it.

**Files:**
- Create: `crates/speedyspec-driver-kubernetes/Cargo.toml`, `src/lib.rs`, `src/connect.rs`
- Modify: `crates/speedyspec-testkit/src/server.rs`, `src/lib.rs`; create `crates/speedyspec-testkit/src/kube.rs`
- Test: `crates/speedyspec-driver-kubernetes/tests/connect.rs`

**Interfaces:**
- Produces, in testkit:

```rust
// server.rs: a server whose every request goes through a closure.
pub struct Received { pub method: String, pub path: String, pub query: String,      // query without '?'
                      pub headers: HashMap<String, String>, pub body: Vec<u8> }
pub struct Reply { pub status: u16, pub content_type: String, pub body: Vec<u8> }
impl Server { pub fn handling(handler: impl Fn(&Received) -> Reply + Send + Sync + 'static) -> Server; }

// kube.rs: an in-memory API server. Knows core/v1 (Namespace, ConfigMap, Secret, Service, Pod,
// PersistentVolumeClaim) and apps/v1 (Deployment, StatefulSet, DaemonSet), batch/v1 (Job), and any custom
// resource definition registered with `define`.
pub struct FakeCluster { /* shared state */ }
impl FakeCluster {
    pub fn start() -> FakeCluster;                       // has namespace "default"
    pub fn url(&self) -> String;                         // http://127.0.0.1:<port>
    pub fn token(&self) -> &'static str;                 // requests without `Bearer <token>` get 401
    pub fn object(&self, api_version: &str, kind: &str, namespace: Option<&str>, name: &str)
        -> Option<serde_json::Value>;
    pub fn put(&self, object: serde_json::Value);        // seed or overwrite, as another actor would
    pub fn set_status(&self, api_version: &str, kind: &str, namespace: Option<&str>, name: &str,
                      status: serde_json::Value);
    pub fn define(&self, group: &str, version: &str, kind: &str, plural: &str, namespaced: bool);
    pub fn fail_next(&self, method: &str, kind: &str, status: u16, message: &str);
    pub fn own_field(&self, api_version: &str, kind: &str, namespace: Option<&str>, name: &str,
                     manager: &str, field: &str);        // a later apply touching it is a 409 Conflict
    pub fn requests(&self) -> Vec<Received>;
}
```

  The fake answers discovery (`/version`, `/api`, `/api/v1`, `/apis`, `/apis/<group>/<version>`), GET, DELETE
  and server-side apply (`PATCH` with `Content-Type: application/apply-patch+yaml`, requiring a `fieldManager`
  query parameter; it stores the object, bumps `metadata.generation` when `spec` changes, records the manager,
  and answers 409 with a Kubernetes `Status` body listing the conflicting field and manager unless
  `force=true`). Unknown paths are 404 with a `Status` body.

- Produces, in `connect.rs`:

```rust
pub struct ConnectFlags<'a> { pub server: Option<&'a str>, pub token_file: Option<&'a str>,
                              pub ca_cert: Option<&'a str>, pub insecure_skip_tls_verify: bool,
                              pub context: Option<&'a str> }
pub enum ConnectError { Usage(String), Failure(String) }
pub struct Connection { /* a kube::Client and the runtime handle it runs on */ }
pub fn connect(flags: &ConnectFlags, env_token: Option<&str>, stdin: &mut dyn Read)
    -> Result<Connection, ConnectError>;
impl Connection { pub fn server_version(&self) -> Result<String, String>; }   // e.g. "v1.34.1"
```

- Rules for `connect`:
  - Token: `--token-file <path>` (`-` reads `stdin` to its end) wins over the environment value
    (`SPEEDYSPEC_KUBE_TOKEN`, passed in as `env_token`). Surrounding whitespace is trimmed. An empty token is
    `Usage`. There is no flag that takes the token itself.
  - `--server` given: it must be `https://`, or `http://` to a loopback host (the `is_fetchable_url` rule); a
    token is then required (`Usage` naming both ways to give one). `--ca-cert` is a PEM file; with
    `--insecure-skip-tls-verify` the certificate is not checked. Giving both is `Usage`.
  - `--server` not given and no token from either source: use the user's kubeconfig (`KUBECONFIG`, then the
    default path), with `--context` when given. No kubeconfig is `Usage` saying to give `--server` and a token.
  - A token without `--server` is `Usage`.
  - No error message contains the token.
- `Connection` owns a current-thread `tokio` runtime; every method blocks on it. Nothing async is public.
- `Cargo.toml`: `kube = "5"` with its default features, `k8s-openapi` with the newest `v1_*` feature it offers,
  `tokio` with `rt` and `time` only, `serde_json`, `speedyspec-driver`, `speedyspec-hooks`. Say in the report
  exactly which versions and features resolved, and the binary size change.

- [ ] **Step 1: Write the testkit additions** and a test of `FakeCluster` itself in `speedyspec-testkit`:
  apply a ConfigMap twice with a plain HTTP client, read it back, see the 401 without the token, and see the 409
  after `own_field`.
- [ ] **Step 2: Write the failing tests in `tests/connect.rs`** (`KUBECONFIG` set to a missing path in each):

```rust
#[test] fn reads_the_server_version()                 // server = fake.url(), env_token = fake.token()
#[test] fn token_file_beats_the_environment()         // file has the right token, env has a wrong one -> ok
#[test] fn token_from_stdin()                         // --token-file - with "  <token>\n" on stdin -> ok
#[test] fn empty_token_is_usage_error()               // file with "\n" -> Usage; and --token-file - with
                                                      // nothing on stdin -> Usage
#[test] fn server_without_token_is_usage_error()      // message names SPEEDYSPEC_KUBE_TOKEN and --token-file
#[test] fn token_without_server_is_usage_error()
#[test] fn plain_http_only_to_loopback()              // http://example.com:6443 -> Usage; https://... accepted
                                                      // by the rule (no connection attempted in this test)
#[test] fn ca_cert_and_insecure_together_is_usage_error()
#[test] fn no_kubeconfig_and_no_server_is_usage_error()
#[test] fn wrong_token_is_reported_without_the_token() // env_token "s3cr3t-wrong": server_version() is Err,
                                                       // message says unauthorized, never contains the token
#[test] fn uses_a_kubeconfig_when_nothing_else_is_given()  // KUBECONFIG -> a temp file pointing at the fake
                                                           // with its token; no flags -> ok
```

- [ ] **Step 3: Run.** `cargo test -p speedyspec-driver-kubernetes`. Expected: does not compile.
- [ ] **Step 4: Implement `connect.rs`.**
- [ ] **Step 5: Run on macOS.** Expected: all pass.
- [ ] **Step 6: Run on Linux.** `scripts/linux.sh cargo test -p speedyspec-driver-kubernetes -p speedyspec-testkit`.
  Expected: all pass, built for static musl. If the client will not build or link for musl, try in order:
  `kube` with `default-features = false` and `client`, `rustls-tls`, `ring` named explicitly; a different
  `k8s-openapi` version feature; then stop and report with the exact linker error. Do not switch to OpenSSL or a
  system TLS library, and do not replace the client with hand-written HTTP without asking.
- [ ] **Step 7: Commit.** `feat: connect to a Kubernetes cluster`

**Stop here and report to Jacob if the client cannot be made to build statically on either platform.**

---

### Task 4: Manifests and the install record (pure)

**Files:**
- Create: `crates/speedyspec-driver-kubernetes/src/manifest.rs`, `src/record.rs`
- Test: unit tests in both

**Interfaces:**
- Consumes: `serde_json::Value`; `PluginRef` (Task 1).
- Produces:

```rust
// manifest.rs
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct ResourceId { pub api_version: String, pub kind: String,
                        pub namespace: Option<String>, pub name: String }
impl ResourceId { pub fn group(&self) -> &str;                      // "" for core
                  pub fn display(&self) -> String; }                // "Deployment/web" or "ns/Deployment/web"
pub struct InstallIdentity<'a> { pub app: &'a str, pub name: &'a str, pub namespace: &'a str }
pub fn identify(manifest: &serde_json::Value) -> ResourceId;        // as written, before defaulting
/// Sets the three labels (replacing a plugin's own values for those keys) and, when `namespaced` and the
/// manifest names no namespace, `metadata.namespace`. Returns the final identity.
pub fn prepare_manifest(manifest: &mut serde_json::Value, namespaced: bool, identity: &InstallIdentity)
    -> Result<ResourceId, String>;
pub fn install_name_problem(name: &str) -> Option<String>;          // DNS label, at most 40 characters
pub fn record_name(install_name: &str) -> String;                   // "speedyspec-install-<name>"

// record.rs
pub const SCHEMA: u32 = 1;
#[derive(Clone, Debug, PartialEq, Eq)] pub enum Status { Installing, Installed, Failed }
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Recorded { pub id: ResourceId, pub retained: bool }      // retained: from an earlier install only
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Record { pub schema: u32, pub name: String, pub app: String, pub parts: Vec<String>,
                    pub status: Status, pub message: Option<String>, pub cli_version: String,
                    pub plugins: Vec<PluginRef>, pub created_namespaces: Vec<String>,
                    pub resources: Vec<Recorded> }
impl Record {
    pub fn to_secret(&self, namespace: &str) -> serde_json::Value;  // the Secret manifest holding it
    pub fn from_secret(secret: &serde_json::Value) -> Result<Record, String>;
    /// The record for a new install over `previous`: resources no longer returned are kept, marked
    /// retained, after the new ones; namespaces the earlier install created stay recorded.
    pub fn superseding(previous: Option<&Record>, fresh: Record) -> Record;
    pub fn dropped(&self) -> Vec<&ResourceId>;                      // the retained ones, for the report
}
```

- `prepare_manifest` has two errors: a cluster-wide resource that names a namespace (`"<kind> is cluster-wide
  and cannot have a namespace"`), and a Secret named `record_name(identity.name)` in `identity.namespace`
  (`"that name is reserved for the install record"`). A namespaced resource may name any namespace.
- The record Secret: `type: speedyspec.com/install`, the three labels, and one `stringData`/`data` key `record`
  holding the JSON. `from_secret` refuses a Secret of another type, a missing key, JSON that does not parse, and
  a `schema` greater than `SCHEMA` (`"written by a newer speedyspec"`).
- `superseding` refuses nothing; an `app` that differs from the previous record's is the caller's concern
  (Task 5 makes it an error).

- [ ] **Step 1: Write the failing tests:**

```rust
// manifest.rs
#[test] fn identifies_core_and_grouped_resources()     // "v1" -> group ""; "apps/v1" -> "apps"
#[test] fn sets_labels_over_the_plugins_own()          // plugin set managed-by=helm -> speedyspec
#[test] fn defaults_the_namespace_only_when_namespaced_and_absent()
#[test] fn cluster_wide_resource_with_a_namespace_is_refused()
#[test] fn the_record_name_is_reserved()               // only for kind Secret, that name, that namespace
#[test] fn install_name_rules()                        // "site" ok; "Site", "a_b", "", "-a", 41 chars -> Some
// record.rs
#[test] fn round_trips_through_a_secret()
#[test] fn secret_has_the_type_and_labels()
#[test] fn refuses_a_foreign_secret_and_a_newer_schema()
#[test] fn first_install_has_nothing_retained()
#[test] fn dropped_resources_are_retained_after_the_new_ones()   // previous [a,b,c], fresh [b,d] ->
                                                                 // [b,d, a(retained), c(retained)]
#[test] fn a_returning_resource_is_no_longer_retained()          // previous a(retained), fresh [a] -> [a]
#[test] fn created_namespaces_accumulate()
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-driver-kubernetes --lib`. Expected: does not compile.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: 13 pass.
- [ ] **Step 5: Commit.** `feat: add Kubernetes manifest identity and the install record`

---

### Task 5: Applying an install

**Files:**
- Create: `crates/speedyspec-driver-kubernetes/src/cluster.rs`, `src/install.rs`
- Test: `crates/speedyspec-driver-kubernetes/tests/install.rs` (against `FakeCluster`)

**Interfaces:**
- Consumes: `Connection` (Task 3); `ResourceId`, `prepare_manifest`, `InstallIdentity`, `Record`, `Recorded`,
  `Status` (Task 4); `KubernetesResources` (Task 2); `PluginRef`, `Progress` (Task 1).
- Produces:

```rust
// cluster.rs — every method blocks; errors carry kind/name and the API's message, never a body.
pub enum ClusterError { NotFound, Unauthorized(String),
                        Conflict { resource: String, details: Vec<(String, String)> },  // (field, manager)
                        UnknownKind(String), Other(String) }
impl Connection {
    pub fn scope(&self, api_version: &str, kind: &str) -> Result<bool, ClusterError>;   // true = namespaced
    pub fn get(&self, id: &ResourceId) -> Result<serde_json::Value, ClusterError>;
    pub fn apply(&self, manifest: &serde_json::Value, force: bool) -> Result<(), ClusterError>;
    pub fn delete(&self, id: &ResourceId) -> Result<(), ClusterError>;                  // NotFound if absent
    pub fn namespace_exists(&self, name: &str) -> Result<bool, ClusterError>;
    pub fn create_namespace(&self, name: &str, identity: &InstallIdentity) -> Result<(), ClusterError>;
}

// install.rs
pub struct InstallOptions<'a> { pub identity: InstallIdentity<'a>, pub parts: &'a [String],
                                pub force_conflicts: bool, pub plugins: &'a [PluginRef],
                                pub cli_version: &'a str }
pub struct Applied { pub resources: Vec<ResourceId>, pub retained: Vec<ResourceId> }
pub fn apply_install(connection: &Connection, answer: &KubernetesResources, options: &InstallOptions,
                     progress: &mut dyn Progress) -> Result<Applied, String>;
```

- `scope` answers from discovery and caches it. For a kind discovery does not know, it runs discovery again,
  up to 30 seconds, before `UnknownKind`: a plugin may install a CustomResourceDefinition and then a resource of
  that kind in one answer.
- `apply_install`, in order:
  1. Refuse an empty `resources` (`"no plugin supplied anything to install"`).
  2. Read the existing record, if any. An existing record whose `app` differs is an error naming both apps and
     saying to choose another `--name`.
  3. Resolve every resource: `scope`, then `prepare_manifest`. Any error here stops before the cluster is
     changed.
  4. Create `identity.namespace`, and every other namespace a resource names, if it does not exist. Remember
     which were created.
  5. Write the record with status `Installing`, built with `Record::superseding`, listing **every** resource
     about to be applied. This happens before the first apply.
  6. Apply each resource in the order given (`progress`: `applying <display>`).
  7. On any failure in 4 to 6: write the record again with status `Failed` and a message naming the resource
     (no body), then return the error. What was applied stays. A failure to write that record is added to the
     error, not swallowed.
  8. Return the applied ids and the retained ones. Marking the record `Installed` is Task 6's last step.
- A `Conflict` becomes: `"<display>: field <field> is owned by <manager>; use --force-conflicts to take it
  over"`, one line per conflicting field.

- [ ] **Step 1: Write the failing tests** (each builds a `Connection` to a `FakeCluster`; `web()` is a
  Deployment, `settings()` a ConfigMap, `secret()` a Secret with a value `s3cr3t-value`):

```rust
#[test] fn applies_resources_in_order_with_labels_and_field_manager()
    // request log: PATCH ...configmaps/settings then PATCH ...deployments/web; each has
    // fieldManager=speedyspec; stored objects carry the three labels and namespace "shop"
#[test] fn creates_missing_namespaces_and_records_them()   // --namespace shop (absent) + a resource in "data"
#[test] fn existing_namespace_is_not_recorded_as_created()
#[test] fn record_is_written_before_the_first_apply()      // in the request log the record Secret's PATCH
                                                           // comes before any resource PATCH; status installing
#[test] fn record_lists_everything_about_to_be_applied()
#[test] fn failure_marks_the_record_failed_and_keeps_what_was_applied()
    // fail_next("PATCH", "Deployment", 422, "spec.replicas: Invalid value") -> Err names Deployment/web and
    // the API message; record status Failed; resources still lists all three; ConfigMap is in the cluster
#[test] fn error_messages_never_contain_resource_bodies()  // fail the Secret; error lacks "s3cr3t-value"
#[test] fn conflict_names_field_and_manager()              // own_field(.., "kubectl-edit", ".spec.replicas")
#[test] fn force_conflicts_takes_ownership()               // same, force -> ok; request had force=true
#[test] fn reinstall_with_fewer_resources_retains_the_dropped_ones()
    // install [settings, web]; install [web] -> Applied.retained == [ConfigMap/settings]; ConfigMap still in
    // the cluster; record resources == [web, settings(retained)]
#[test] fn reinstall_under_another_app_is_refused()        // record app "shop", now "blog" -> Err; nothing applied
#[test] fn empty_answer_is_refused_before_touching_the_cluster()   // no PATCH in the request log
#[test] fn invalid_resource_stops_before_anything_is_applied()     // a cluster-wide kind with a namespace
#[test] fn custom_resource_after_its_definition()          // define("example.test","v1","Widget",..) a moment
                                                           // after the CRD is applied; the Widget then applies
#[test] fn unknown_kind_is_reported()                      // kind "Nope" -> Err naming it, after the retry
                                                           // window (shortened by the test)
```

- [ ] **Step 2: Run.** `cargo test -p speedyspec-driver-kubernetes --test install`. Expected: does not compile.
- [ ] **Step 3: Implement `cluster.rs`, then `install.rs`.** Server-side apply through the client's dynamic API
  with discovery; the retry window for unknown kinds is a parameter so tests can shorten it.
- [ ] **Step 4: Run.** Expected: 15 pass.
- [ ] **Step 5: Commit.** `feat: apply an install to a cluster and record it`

---

### Task 6: Waiting for readiness

**Files:**
- Create: `crates/speedyspec-driver-kubernetes/src/readiness.rs`
- Modify: `src/install.rs`
- Test: unit tests in `readiness.rs`; more cases in `tests/install.rs`

**Interfaces:**
- Consumes: `Connection::get`, `ResourceId`, `WaitFor`, `Applied`, `Record`.
- Produces:

```rust
// readiness.rs
pub enum Readiness { Ready, NotYet(String), Failed(String) }     // NotYet/Failed carry a short reason
pub fn readiness(kind: &str, object: &serde_json::Value, condition: Option<&str>) -> Readiness;
pub fn default_wait(resources: &[ResourceId]) -> Vec<ResourceId>;  // Deployment, StatefulSet, DaemonSet, Job
pub fn parse_timeout(text: &str) -> Result<Duration, String>;      // "90s", "10m", "1h"; default "10m"

// install.rs
pub fn wait_and_finish(connection: &Connection, applied: &Applied, wait_for: Option<&[WaitFor]>,
                       identity: &InstallIdentity, timeout: Duration, interval: Duration,
                       progress: &mut dyn Progress) -> Result<(), String>;   // identity finds the record
```

- Rules (`condition` given means: ready when `status.conditions` has that `type` with `status: "True"`,
  whatever the kind):
  - Deployment: `status.observedGeneration >= metadata.generation`, and `updatedReplicas`, `availableReplicas`
    both equal `spec.replicas` (default 1).
  - StatefulSet: `observedGeneration` current, `readyReplicas == spec.replicas`, and
    `currentRevision == updateRevision`.
  - DaemonSet: `observedGeneration` current, `numberReady` and `updatedNumberScheduled` both equal
    `desiredNumberScheduled`.
  - Job: condition `Complete` is `True` → Ready; condition `Failed` is `True` → `Failed` with its message.
  - Pod: condition `Ready` is `True`, or `status.phase` is `Succeeded`; phase `Failed` → `Failed`.
  - Any other kind without `condition`: `Failed("no readiness rule for <kind>; give a condition")`. The contract
    check in the driver (Task 7) refuses this before installing.
- `wait_and_finish`: the wait set is `wait_for` resolved to applied ids (an explicit empty list waits for
  nothing), or `default_wait`. It polls each every `interval`, printing a progress line when a resource's reason
  changes. All ready → record status `Installed`. A `Failed`, or the timeout → record status `Failed` with a
  message naming what was not ready and why, and that message is the error. Nothing is rolled back.

- [ ] **Step 1: Write the failing tests:**

```rust
// readiness.rs
#[test] fn deployment_rules() {       // table of status JSON -> Ready / NotYet(reason)
    // all equal -> Ready; availableReplicas 0 of 2 -> NotYet("0 of 2 available");
    // observedGeneration behind -> NotYet; missing status -> NotYet
}
#[test] fn statefulset_rules()
#[test] fn daemonset_rules()
#[test] fn job_complete_failed_and_running()
#[test] fn pod_ready_succeeded_and_failed()
#[test] fn custom_condition_on_any_kind()            // Widget with conditions [{type: Available, status: True}]
#[test] fn unknown_kind_without_condition_fails()
#[test] fn default_wait_picks_workloads_only()       // ConfigMap, Service, Deployment, Job -> the last two
#[test] fn timeout_parsing()                         // "90s","10m","1h" ok; "10", "abc", "0s", "" -> Err
// tests/install.rs
#[test] fn waits_until_a_deployment_is_available()   // a thread sets the status ready after 150 ms;
                                                     // interval 50 ms -> Ok; record status Installed
#[test] fn timeout_marks_the_record_failed()         // never ready; timeout 300 ms -> Err naming
                                                     // Deployment/web and "0 of 1 available"; status Failed
#[test] fn failed_job_stops_the_wait_at_once()       // Job with Failed=True -> Err before the timeout
#[test] fn explicit_empty_wait_returns_immediately() // wait_for Some(&[]) -> Ok without any GET of workloads
#[test] fn waits_on_a_custom_condition()
```

- [ ] **Step 2: Run.** Expected: does not compile.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** `cargo test -p speedyspec-driver-kubernetes`. Expected: all pass.
- [ ] **Step 5: Commit.** `feat: wait for installed workloads to become ready`

---

### Task 7: The Kubernetes driver in the CLI

**Files:**
- Create: `crates/speedyspec-driver-kubernetes/src/driver.rs`
- Modify: `crates/speedyspec-driver-kubernetes/src/lib.rs`; `crates/speedyspec-cli/src/drivers.rs`, `Cargo.toml`
- Test: unit tests in `driver.rs`; `crates/speedyspec-cli/tests/kubernetes.rs`,
  `tests/fixtures/plugins/k8s-*`

**Interfaces:**
- Consumes: Tasks 1 to 6.
- Produces: `pub struct KubernetesDriver; impl KubernetesDriver { pub fn new() -> Self }  impl Driver`.
  - `id()`: `"kubernetes"`. `seed()`: `Value::Null`.
  - Flags: `server`, `token-file`, `ca-cert`, `context`, `namespace`, `name`, `timeout` (take values);
    `insecure-skip-tls-verify`, `force-conflicts` (switches).
  - `prepare`: `--namespace` is required (`DriverError` naming it) and must be a DNS label; `--name` defaults to
    the app id and must pass `install_name_problem`; `--timeout` parses (default `10m`); then `connect` (the
    token from `SPEEDYSPEC_KUBE_TOKEN`, stdin for `--token-file -`). Settings shared with plugins are exactly
    `namespace` and `name`. Never the server, never the token.
  - `check_answer`: `Value::Null` is valid (a plugin that has nothing to add passes it through);
    otherwise `decode_kubernetes_resources`, then every `waitFor` entry and every default-wait resource must
    have a readiness rule or a `condition` (`waitFor[i]: no readiness rule for <kind>; give a condition`).
  - `install`: a null answer is `DriverError("no plugin supplied anything to install")`; else `apply_install`
    then `wait_and_finish`. Returns a list of arrays with string keys `apiVersion`, `kind`, `namespace` (empty
    for cluster-wide), `name`, for applied resources only.
  - `report`: one line per applied resource, `<namespace or ->\t<kind>/<name>`.
- A plugin adding to another plugin's answer builds a new `KubernetesResources` from the one it was given; the
  class is readonly. Document this in the README (Task 9).

- [ ] **Step 1: Create the fixture plugins** (each reads nothing from the environment; the cluster address comes
  from the test's flags):

| Fixture | What it does |
|---|---|
| `k8s-app` | App `shop` ("Shop", parts `web` default and `worker` not default, drivers `kubernetes`). Answers with a ConfigMap `settings` (data `title` from the option `title`, required), a Deployment `web`, and, when part `worker` is selected, a Deployment `worker`. `speedyspec_install_after` echoes `after:` plus the count of resources and the request's `settings['namespace']` and `settings['name']`. |
| `k8s-adds` | For app `shop`, returns a new `KubernetesResources` with the given resources plus a Service `web`. |
| `k8s-bad` | For app `shop`, returns a `KubernetesResources` whose second resource has no `metadata.name`. |
| `k8s-empty-map` | App `maps`: a ConfigMap with `data` set to `new stdClass` and a Pod whose volume uses `'emptyDir' => new stdClass` and `'args' => []`. |
| `k8s-no-rule` | App `odd`: one resource of kind `Widget` and `waitFor` naming it without a condition. |

- [ ] **Step 2: Write the failing tests.** `driver.rs` unit tests:

```rust
#[test] fn describes_itself()                        // id, seed null, the nine flags and which take values
#[test] fn namespace_is_required()
#[test] fn name_defaults_to_the_app()                // settings == [("namespace","shop"),("name","site")]
#[test] fn settings_never_include_server_or_token()
#[test] fn null_answer_passes_the_check_and_fails_the_install()
#[test] fn wait_without_a_rule_is_refused()          // field "waitFor[0]"
```

  `tests/kubernetes.rs`, driving the binary with `KUBECONFIG` set to a missing path, `SPEEDYSPEC_KUBE_TOKEN`
  set to the fake's token, `--no-interaction`, `--server <fake url>`, `--namespace shop`, `--timeout 3s`; a
  helper thread marks Deployments available as they appear unless the test says otherwise:

```rust
#[test] fn installs_default_parts()                  // exit 0; cluster has ConfigMap settings (title "Hi"),
                                                     // Deployment web, no worker; stdout ==
                                                     // "after:2 shop shop\nshop\tConfigMap/settings\nshop\tDeployment/web\n"
#[test] fn installs_selected_parts_under_a_name()    // --part web --part worker --name prod -> 3 resources;
                                                     // labels speedyspec.com/install=prod; record
                                                     // speedyspec-install-prod has parts ["web","worker"]
#[test] fn record_names_the_answering_plugins()      // k8s-app + k8s-adds -> record plugins are both, with
                                                     // versions; Service web applied
#[test] fn bad_answer_refuses_the_install()          // k8s-app + k8s-bad: exit 1; stderr has
                                                     // "speedyspec-fixture/k8s-bad", "speedyspec_driver_kubernetes",
                                                     // "resources[1].metadata.name", "nothing was installed";
                                                     // the cluster received no PATCH
#[test] fn empty_objects_and_empty_lists_survive()   // k8s-empty-map: stored ConfigMap data == {}; the Pod's
                                                     // volume emptyDir == {} and container args == []
#[test] fn wait_without_a_rule_is_refused()          // k8s-no-rule: exit 1; names "waitFor[0]"; no PATCH
#[test] fn not_ready_in_time_fails_and_says_what()   // no helper thread: exit 1; stderr names Deployment/web;
                                                     // record status failed; resources still in the cluster
#[test] fn missing_namespace_flag_is_reported()      // exit 1; stderr names --namespace; the cluster got
                                                     // no request
#[test] fn wrong_token_is_reported_without_it()      // SPEEDYSPEC_KUBE_TOKEN=s3cr3t-wrong: exit 1; stderr says
                                                     // unauthorized; neither stream contains "s3cr3t-wrong"
#[test] fn token_from_a_file_and_from_stdin()        // --token-file <path>; --token-file - with the token piped
#[test] fn token_never_reaches_the_plugin()          // a fixture variant echoes json_encode of the request in
                                                     // the after-hook: no token, no server URL
#[test] fn app_list_shows_kubernetes_as_usable()     // "shop\tShop\tkubernetes\n" (no "?")
#[test] fn zip_flags_are_refused_with_kubernetes()   // --base x -> exit 2: "--base belongs to the zip driver"
#[test] fn help_lists_both_drivers_flags()           // `app install shop --help` shows --namespace, --server,
                                                     // --token-file, --base
```

- [ ] **Step 3: Run.** Expected: does not compile.
- [ ] **Step 4: Implement `driver.rs`, register it in `drivers.rs`** (`ids()` becomes `["zip", "kubernetes"]`;
  update the existing tests that expected `kubernetes?` in `app list`, and the zip tests that relied on
  kubernetes being unusable: they now need `--driver zip`).
- [ ] **Step 5: Run on macOS.** `cargo test --workspace`. Expected: all pass.
- [ ] **Step 6: Run on Linux.** `scripts/linux.sh cargo test --workspace`. Expected: all pass.
- [ ] **Step 7: Commit.** `feat: add the Kubernetes driver`

---

### Task 8: `speedyspec app uninstall`

**Files:**
- Create: `crates/speedyspec-driver-kubernetes/src/uninstall.rs`
- Modify: `src/driver.rs`; `crates/speedyspec-cli/src/app.rs`, `src/commands.rs`
- Test: `crates/speedyspec-driver-kubernetes/tests/uninstall.rs`; more cases in
  `crates/speedyspec-cli/tests/kubernetes.rs`

**Interfaces:**
- Consumes: `Connection`, `Record`, `ResourceId`; `UninstallInput`, `UninstallPlan` (Task 1); `Prompter`,
  `interactive` (plan 2).
- Produces:

```rust
// uninstall.rs
pub struct Removal { pub record: Record, pub namespace: String }
pub fn find_install(connection: &Connection, name: &str, namespace: &str) -> Result<Removal, String>;
pub fn summarise(removal: &Removal) -> Vec<String>;      // what will be deleted, for the confirmation
pub struct Removed { pub deleted: Vec<ResourceId>, pub already_gone: Vec<ResourceId>,
                     pub skipped: Vec<(ResourceId, String)>, pub namespaces: Vec<String> }
pub fn remove_install(connection: &Connection, removal: &Removal, timeout: Duration, interval: Duration,
                      progress: &mut dyn Progress) -> Result<Removed, String>;
```

- `find_install`: reads the record Secret; missing → `"no install named <name> in namespace <namespace>"`.
- `summarise`: the app, the count of resources by kind, each namespace that will be deleted, and one explicit
  line when PersistentVolumeClaims are among them: `"this deletes <n> PersistentVolumeClaim(s) and the data in
  them"`.
- `remove_install`, in order:
  1. Each recorded resource, in reverse order (retained ones included): `get` it. Gone → `already_gone`. Present
     but without the label `speedyspec.com/install=<name>` → `skipped` with the reason `"no longer part of this
     install"`. Otherwise delete it.
  2. Wait, up to `timeout`, until every deleted resource is gone.
  3. Delete each namespace in `created_namespaces` except the record's own, and wait for it to go.
  4. Delete the record.
  5. If the record's own namespace is in `created_namespaces`, delete it last.
  A failure stops there and says what remains; the record is only deleted in step 4, so the command can be run
  again.
- `KubernetesDriver`: `uninstall_flags()` are the connection flags plus `namespace` and `timeout`;
  `plan_uninstall` connects, runs `find_install`, returns `summarise` as the summary; `uninstall` runs
  `remove_install` and returns lines `<namespace or ->\t<kind>/<name>\t<deleted|gone|skipped>` then one
  `-\tNamespace/<name>\tdeleted` per namespace.
- Command: `speedyspec app uninstall <install-name> [--driver <id>] [driver flags] [--yes]`.
  - `--driver` may be left out when exactly one compiled-in driver has `uninstall_flags()`.
  - It runs no plugin code and works with `--no-plugins`; it is not refused on a poisoned engine.
  - It prints the plan's summary to stderr, then asks to confirm unless `--yes`; not interactive without
    `--yes` is exit 2 with `speedyspec: confirm with --yes`.

- [ ] **Step 1: Write the failing tests.** `tests/uninstall.rs` (install through Task 5's functions first):

```rust
#[test] fn removes_everything_in_reverse_order()        // DELETE order in the request log is the reverse of
                                                        // the apply order; then the record
#[test] fn removes_retained_resources_too()
#[test] fn already_gone_is_not_an_error()               // delete one behind its back -> already_gone
#[test] fn a_relabelled_resource_is_skipped()           // put() the ConfigMap without the install label ->
                                                        // skipped; it is still in the cluster afterwards
#[test] fn deletes_only_namespaces_it_created()         // "shop" pre-existing, "data" created -> only "data"
#[test] fn the_records_own_namespace_goes_last()        // created "shop": record DELETE precedes namespace DELETE
#[test] fn missing_record_is_a_clear_error()
#[test] fn summary_warns_about_persistent_volume_claims()
#[test] fn a_failed_delete_keeps_the_record()           // fail_next("DELETE","Deployment",500,..) -> Err; the
                                                        // record is still there; a second run succeeds
#[test] fn uninstall_after_a_failed_install_cleans_up() // install with a failing third resource, then remove
```

  `tests/kubernetes.rs`:

```rust
#[test] fn uninstall_removes_the_install()              // install, then `app uninstall shop --namespace shop
                                                        // --yes` -> exit 0; cluster has none of it; stdout
                                                        // lists each resource as deleted
#[test] fn uninstall_needs_confirmation()               // without --yes -> exit 2; summary on stderr; nothing
                                                        // deleted
#[test] fn uninstall_works_without_plugins()            // --no-plugins, no --plugin-dir
#[test] fn uninstall_of_nothing_is_reported()           // exit 1; "no install named nope in namespace shop"
#[test] fn uninstall_driver_is_implied()                // no --driver given; kubernetes is the only one able
```

- [ ] **Step 2: Run.** Expected: does not compile.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** `cargo test --workspace` on macOS, then `scripts/linux.sh cargo test --workspace`.
  Expected: all pass.
- [ ] **Step 5: Commit.** `feat: add app uninstall`

---

### Task 9: Example plugin, a real cluster, and documentation

**Files:**
- Create: `examples/install-demo-kubernetes/speedyspec-plugin.json`, `plugin.php`, `README.md`;
  `scripts/k3s.sh`; `crates/speedyspec-driver-kubernetes/tests/e2e.rs`
- Modify: `README.md`, `docs/php-embed.md` (sizes), `.github/workflows/ci.yml`
- Test: `crates/speedyspec-cli/tests/examples.rs`

**Interfaces:**
- Consumes: the finished binary.
- Produces:
  - A plugin `speedyspec/install-demo-kubernetes`: app `demo` ("Demo", no parts, driver `kubernetes`), answering
    with a ConfigMap `demo`, a Deployment `demo` of one replica of `registry.k8s.io/pause:3.10`, and a Service
    `demo`. No options.
  - `scripts/k3s.sh up` starts `rancher/k3s` (a pinned tag) in Docker as container `speedyspec-k3s`, publishing
    the API on a loopback port, waits until it answers, and writes a kubeconfig to `.k3s/kubeconfig` inside the
    repository (git-ignored). `down` removes the container and `.k3s/`. It refuses to run if a container of that
    name exists and it did not create it, and it never reads or writes `~/.kube`.

- [ ] **Step 1: Write the failing tests in `tests/examples.rs`:**
  `kubernetes_example_loads_and_is_listed` (`demo\tDemo\tkubernetes\n`, stderr empty) and
  `kubernetes_example_follows_the_php_style` (same checks as the WordPress example).
- [ ] **Step 2: Write the example plugin and its README.** Run: the two tests pass.
- [ ] **Step 3: Write `scripts/k3s.sh`** and `tests/e2e.rs`, every test `#[ignore]` and reading the kubeconfig
  path from `SPEEDYSPEC_E2E_KUBECONFIG` (it fails with a clear message when that is unset, so it can never fall
  back to the user's cluster):

```rust
#[test] #[ignore] fn installs_reinstalls_and_uninstalls_on_a_real_cluster()
    // apply_install + wait_and_finish for a ConfigMap, a Deployment (pause image) and a Service into a new
    // namespace -> ready within 3 minutes; record Installed; install again without the Service -> retained;
    // remove_install -> the namespace and everything in it are gone
#[test] #[ignore] fn conflict_with_another_manager_on_a_real_cluster()
    // apply; patch spec.replicas with another field manager through the client; apply again -> the conflict
    // names the field and the manager; with force it succeeds
```

- [ ] **Step 4: Run the end-to-end test locally.** `scripts/k3s.sh up`, then
  `SPEEDYSPEC_E2E_KUBECONFIG=$PWD/.k3s/kubeconfig cargo test -p speedyspec-driver-kubernetes --test e2e -- --ignored`,
  then drive the release binary by hand with `KUBECONFIG=$PWD/.k3s/kubeconfig`:
  `speedyspec --plugin-dir examples/install-demo-kubernetes app install demo --namespace demo`, the same again
  (idempotent, exit 0), then `speedyspec --no-plugins app uninstall demo --namespace demo --yes`. Then
  `scripts/k3s.sh down`. Expected: both tests pass; install prints three lines; uninstall leaves no `demo`
  namespace. Record the output in the task report. If Docker cannot run k3s on this machine, say so and leave
  the end-to-end check to CI; do not install anything to work around it.
- [ ] **Step 5: Add a CI job** `kubernetes-e2e` on `ubuntu-latest`: check out both repositories, `scripts/k3s.sh
  up`, run the ignored tests with the host's stable Rust (this crate has no PHP dependency, so it needs no
  static PHP), `scripts/k3s.sh down` in an `always()` step. `permissions: contents: read`.
- [ ] **Step 6: Update `README.md`:** the Kubernetes driver's flags and how to give a token; what an install
  does in order; the record and its three labels; re-install and retained resources; that nothing is rolled
  back on failure; `--force-conflicts`; `app uninstall` and what it will and will not delete; the
  `KubernetesResources` contract with a short plugin in the project style, including `new stdClass` for `{}` and
  how one plugin adds to another's answer; the readiness rules.
- [ ] **Step 7: Run everything.** `cargo test --workspace`, `cargo clippy --workspace --all-targets -- -D
  warnings`, `cargo fmt --all --check`, `scripts/linux.sh cargo test --workspace`; build both release binaries
  and record their sizes in `docs/php-embed.md`. Expected: all clean.
- [ ] **Step 8: Commit.** `docs: add Kubernetes example plugin, end-to-end test and documentation`

---

## What this plan leaves for later plans

- `self update`, release binaries for all four targets, fetching a pinned contracts release at build time, a
  release workflow for the contracts repository (plan 4).
- Pruning retained resources on re-install; rolling back a failed install; `app upgrade` as its own command;
  listing installs (`app installs`). The record written here is what they would build on.
- Kubernetes exec credential plugins that need a terminal, and OIDC login flows, beyond what the client does on
  its own with an existing kubeconfig.
- (S)FTP, Docker, Composer, Functions and AWS drivers; hook backends other than memory. Not in any current plan.
- Carried over from the plan 2 review: the redirect and URL-parsing minors, `--title ""` satisfying a required
  option, and the untested plugin-install rollback.
