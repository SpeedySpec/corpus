# SpeedySpec Server Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A `speedyspec-server` binary that, in immutable mode, serves baked-in static apps from memory over
HTTP/1.1, HTTP/2 and HTTP/3, routes by host and path to mounts, gates `signed-in` mounts through an access-check
contract, answers placeholders for PHP apps and the admin API, and refuses to start on an inconsistent set of
releases.

**Architecture:** A Cargo workspace of small crates following the DDD layer names. `-domain` holds value types and
contracts with no I/O; `-app` holds the compatibility check; `-static` builds and serves an in-memory file set using
only the `http` and `bytes` types; `-infra-access-dev` holds the two placeholder access checks; `-presentation` is the
only crate that knows Salvo; the `speedyspec-server` binary loads config, wires everything and binds sockets itself
(so the later keeper can own them). Task 1 is a proof that Salvo serves all three protocols from sockets bound
outside Salvo.

**Tech Stack:** Rust 1.99 (edition 2024), Salvo 1.2.x (`quinn`, `rustls`, `ring`, `http1`, `http2`, `http2-cleartext`;
default features off so `aws-lc-rs` is not pulled in, my call: `ring` builds on musl without cmake),
tokio, http, bytes, serde / serde_json / toml, semver, globset, mime_guess, blake3, brotli, flate2, ipnet,
percent-encoding, tracing + tracing-subscriber (JSON), clap. Dev: tempfile, rcgen, reqwest (and a quinn/h3 client
helper from Task 1).

**Spec:** `docs/superpowers/specs/2026-11-17-speedyspec-server-core-design.md`

**This is plan 1 of 4.** The spec covers four pieces that each ship working software. The later plans are written
once this one is done, because Task 1's outcome (how Salvo takes pre-bound sockets) shapes plans 2 and 3:

1. **Foundation and immutable mode (this plan).**
2. Managed releases on one host: release index, download, minisign verification, unpacking, `state.json`, the
   lifecycle, rollback, group switch, pin, prune, auto-update, static fall-back grace, the control socket and the
   `release` / `status` commands.
   It also wires `release list` and `status` for immutable mode (reporting baked-in versions) and the refusals of
   `release install|switch|rollback|pin` and `reload` there, since all of them need the control socket.
3. Keeper and socket handoff: generations, server releases, trial and fall-back, resume, `reload`, keeper re-exec,
   crash restart with back-off, systemd notify, certificate reload on `SIGHUP`.
4. Cluster: `Leadership` and `ClusterMembers`, the two-step switch, rolling server upgrades, the Kubernetes Lease and
   the kind/k3d end-to-end test.

## Global Constraints

- New git repository at `packages/speedyspec-server/` (`SpeedySpec/speedyspec-server`), **not** tracked by
  `wordpress-develop`. Create it locally with `git init -b main`. **Do not create a GitHub repository or push**; that
  needs Jacob's go-ahead.
- Rust 1.99, edition 2024, `rust-version = "1.99"` in the workspace, matching `packages/speedyspec-cli`.
- Salvo 1.2.x with its `quinn` feature for HTTP/3. Salvo may be named only in `speedyspec-server-presentation` and
  `speedyspec-server` (the binary). Every other crate is framework-neutral.
- Unix only: CI on Linux x86_64 (musl) and macOS arm64.
- Contracts in `speedyspec-server-domain`; implementations in separate crates, each replaceable on its own. Crate
  names use `-domain`, `-app`, `-infra-*`, `-presentation`. `-domain` does no I/O.
- The core knows app *kinds* (`server`, `static`, `php`), never particular apps such as "the blog".
- Exit codes: 0 success, 1 failure, 2 usage error.
- Logs are JSON lines on stderr. Every refusal names what was refused and why.
- Commit messages use Conventional Commits (`feat:`, `fix:`, `build:`, `ci:`, `test:`, `docs:`).

## Review Focus

Each line has a test in the task named in brackets.

1. **Paths that try to slip past a mount.** `/admin/../x`, `/%61dmin/x`, `//admin/x`, `/admin%2Fx`, a NUL byte. A
   request must reach exactly the mount its normalized path names; encoded slashes, backslashes and NUL are 400, and
   `..` above the root is 400. A `signed-in` mount can never be reached through a `public` one. [Task 7]
2. **Host header spelling.** `Example.COM:443`, `example.com.`, `[::1]:8080`. These must match the mount for host
   `example.com` (or `::1`), and an `X-Forwarded-Host` from an untrusted peer must be ignored. [Task 7]
3. **Accept-Encoding with weights.** `br;q=0, gzip`, `*;q=0.5`, `identity;q=0`, no header at all. The best acceptable
   encoding wins, `q=0` excludes, and with nothing acceptable the identity body is still sent. [Task 5]
4. **Range edge cases.** `bytes=-500`, `bytes=9999999-`, `bytes=0-1,4-5`, `Range` together with
   `Accept-Encoding: br`. Suffix ranges work, unsatisfiable is 416 with `Content-Range: bytes */<len>`, multiple
   ranges get a 200 with the full body, and a range is always taken from the uncompressed body. [Task 5]
5. **File names that need encoding.** `über menü.html` and `a+b.css` in a release, requested percent-encoded. They
   are served; `+` is not treated as a space. [Task 5]

## File Structure

```
packages/speedyspec-server/
  Cargo.toml                          workspace: members crates/*, edition 2024, rust-version 1.99
  .gitignore                          target/
  .github/workflows/ci.yml            cargo test --workspace on ubuntu (musl target) and macos-14
  docs/salvo-listeners.md             Task 1's findings: how pre-bound sockets reach Salvo, per protocol
  crates/speedyspec-server-domain/    src/lib.rs app.rs manifest.rs mount.rs access.rs
  crates/speedyspec-server-app/       src/lib.rs compat.rs
  crates/speedyspec-server-static/    src/lib.rs build.rs respond.rs encoding.rs range.rs
  crates/speedyspec-server-infra-access-dev/  src/lib.rs
  crates/speedyspec-server-presentation/      src/lib.rs path.rs forwarded.rs handlers.rs service.rs
  crates/speedyspec-server/           src/lib.rs main.rs config.rs boot.rs listeners.rs release.json
                                      tests/serve.rs tests/support/{mod.rs,h3.rs,certs.rs}
                                      tests/fixtures/{admin-a,blog}/
```

Each `lib.rs` only declares modules and re-exports. `speedyspec-server` is a library plus a thin `main.rs`, so its
integration tests can use `speedyspec_server::listeners` and friends. `speedyspec-server-infra-access-dev` is an extra crate beyond the
spec's table (my call): the placeholder access checks are implementations, so they cannot live in `-domain`.

---

### Task 1: Prove Salvo serves HTTP/1.1, HTTP/2 and HTTP/3 from sockets it did not bind

This is a proof. The keeper (plan 3) binds every socket and hands it to a generation, so the server must be able to
serve from a `std::net::TcpListener` and a `std::net::UdpSocket` created outside Salvo. The spec leaves the mechanism
to the builder; this task decides it and writes it down.

**Files:**
- Create: `Cargo.toml`, `.gitignore`, `.github/workflows/ci.yml`, `docs/salvo-listeners.md`
- Create: `crates/speedyspec-server/Cargo.toml`, `src/lib.rs`, `src/main.rs` (stub `fn main() {}` for now),
  `src/listeners.rs`
- Test: `crates/speedyspec-server/tests/support/{mod.rs,certs.rs,h3.rs}`, `tests/listeners_proof.rs`

**Interfaces:**
- Produces: `listeners::BoundTcp { listener: std::net::TcpListener, tls: Option<TlsFiles> }`,
  `listeners::BoundUdp { socket: std::net::UdpSocket, tls: TlsFiles }`,
  `listeners::TlsFiles { cert: PathBuf, key: PathBuf }`, and
  `listeners::serve(tcp: Vec<BoundTcp>, udp: Vec<BoundUdp>, service: salvo::Service, shutdown: impl Future<Output = ()>) -> anyhow::Result<()>`.
  Task 8 calls `serve`; plan 3 reuses it in a generation.
- Produces (tests): `support::certs::self_signed(dir: &Path) -> TlsFiles` (rcgen, for `localhost` and `127.0.0.1`),
  `support::h3::get(addr: SocketAddr, ca_pem: &Path, path: &str) -> (u16, http::HeaderMap, Vec<u8>)`.

- [ ] **Step 1: `git init -b main` in `packages/speedyspec-server/` and create the workspace** with the members,
  edition and `rust-version` from the Global Constraints, plus `.gitignore` and the CI workflow (two jobs:
  `ubuntu-latest` with target `x86_64-unknown-linux-musl`, and `macos-14`; each runs `cargo test --workspace`).
- [ ] **Step 2: Write the failing proof test** `tests/listeners_proof.rs::serves_all_protocols_from_prebound_sockets`:
  bind `TcpListener` on `127.0.0.1:0` (cleartext), another `TcpListener` and a `UdpSocket` on the *same* port with
  TLS from `self_signed`, build a Salvo `Service` whose only route returns `200 "ok"`, call `serve`, then assert:
  - HTTP/1.1 GET on the cleartext socket → 200, body `ok`;
  - HTTP/2 prior-knowledge (h2c) GET on the cleartext socket → 200, `response.version() == HTTP_2`;
  - HTTPS GET with ALPN on the TLS socket → 200, `version() == HTTP_2`;
  - `support::h3::get` on the UDP socket → 200, body `ok`;
  - after the shutdown future resolves, `serve` returns `Ok(())` within 5 s.
- [ ] **Step 3: Run it to see it fail.** `cargo test -p speedyspec-server --test listeners_proof` → fails to compile
  (`serve` missing).
- [ ] **Step 4: Implement `listeners::serve`.** Find the Salvo 1.2 way to accept an existing `std` socket for each
  protocol (a `TryFrom` on its TCP acceptor, `quinn::Endpoint::new` with the given `UdpSocket` behind its QUIC
  listener, or a small custom `Acceptor`), and join the acceptors so one `Service` answers all of them. Set every
  socket non-blocking before handing it to tokio. If Salvo cannot take a pre-bound UDP socket without patching, stop
  and report to Jacob before continuing: the handoff design depends on it.
- [ ] **Step 5: Run the test until it passes**, then run it on Linux too
  (`docker run --rm -v "$PWD":/w -w /w rust:1.99-alpine sh -c 'apk add musl-dev && cargo test --workspace'`).
  Expected: PASS on both.
- [ ] **Step 6: Write `docs/salvo-listeners.md`**: the Salvo version, which acceptor API is used per protocol, any
  custom acceptor and why, how shutdown is wired, and anything that will matter for handoff (whether Salvo can stop
  accepting on TCP while in-flight requests continue, whether the QUIC endpoint can be told to stop reading).
- [ ] **Step 7: Commit.** `build: workspace and proof that Salvo serves h1, h2 and h3 from pre-bound sockets`

---

### Task 2: Domain types and contracts

**Files:**
- Create: `crates/speedyspec-server-domain/Cargo.toml` (deps: serde, serde_json, semver with serde, http, thiserror),
  `src/lib.rs`, `src/app.rs`, `src/manifest.rs`, `src/mount.rs`, `src/access.rs`
- Test: unit tests in each module (`#[cfg(test)]`)

**Interfaces:**
- Produces, `app.rs`:
  `pub struct AppId(String)` with `AppId::parse(&str) -> Result<AppId, DomainError>` accepting `[a-z][a-z0-9-]*`,
  `as_str()`, `Display`, `Ord`, serde as a string;
  `pub enum AppKind { Server, Static, Php }` (serde lowercase).
- Produces, `manifest.rs`:
  `pub struct ReleaseManifest { pub app: AppId, pub version: semver::Version, pub kind: AppKind, pub requires: BTreeMap<AppId, semver::VersionReq>, pub static_: Option<StaticSection>, pub php: Option<serde_json::Value> }`
  (JSON key `static` for `static_`);
  `pub struct StaticSection { pub root: String, pub not_found: Option<String>, pub cache: Vec<CacheRule> }`;
  `pub struct CacheRule { pub pattern: String /* JSON key "match" */, pub cache_control: String }`;
  `ReleaseManifest::parse(bytes: &[u8]) -> Result<ReleaseManifest, DomainError>`;
  `pub const MANIFEST_FILE: &str = "speedyspec-release.json";`
- Produces, `mount.rs`:
  `pub enum HostMatch { Any, Exact(String) }`, `pub enum Access { Public, SignedIn }` (serde `public` /
  `signed-in`), `pub struct Mount { pub host: HostMatch, pub path: String, pub app: AppId, pub access: Access }`,
  `pub struct MountTable`, `MountTable::new(Vec<Mount>) -> Result<MountTable, DomainError>`,
  `MountTable::resolve(&self, host: &str, path: &str) -> Option<&Mount>`.
- Produces, `access.rs`:
  `pub struct Identity { pub user_id: Option<u64> }` with `Identity::anonymous()`;
  `pub enum AccessDecision { Allowed(Identity), Denied }`;
  `pub struct AccessRequest<'a> { pub access: Access, pub headers: &'a http::HeaderMap, pub client: std::net::IpAddr }`;
  `pub trait AccessCheck: Send + Sync { fn check<'a>(&'a self, request: AccessRequest<'a>) -> Pin<Box<dyn Future<Output = AccessDecision> + Send + 'a>>; }`
  (boxed future so project 2 can query a session store; `dyn AccessCheck` must be object-safe).
- Produces: `pub enum DomainError` (thiserror) with one variant per rule below, each message naming the bad value.

- [ ] **Step 1: Write the failing tests.**
  - `app_id_accepts_valid` (`admin`, `frontend-api`, `a1`) and `app_id_rejects_invalid` (`""`, `Admin`, `1a`,
    `-a`, `a_b`, `a b`).
  - `manifest_parses_spec_example`: parse the admin JSON from the spec's "Release artifacts" section; assert
    `version == 5.2.0`, `kind == Static`, `requires["server"] == "^3.1"`, `static_.root == "dist"`,
    `cache[0].pattern == "_astro/**"`.
  - `manifest_rejects`: missing `version`; `version: "five"`; `requires: {"Bad": "^1"}`; `kind: "static"` without a
    `static` section; `kind: "php"` *with* a `static` section; `static.root` that is absolute or contains `..`.
  - `mount_table_rejects`: path not starting and ending with `/`; duplicate host+path; exact host not lowercase.
  - `mount_resolve_longest_prefix`: mounts `* /`, `* /admin/`, `* /admin/api/` → `/admin/api/x` picks
    `/admin/api/`, `/admin/x` picks `/admin/`, `/blog` picks `/`.
  - `mount_resolve_exact_host_beats_any`: `example.com /` and `* /admin/` → host `example.com`, path `/admin/x`
    picks `example.com /` (an exact host with any matching prefix beats every `*` mount); host `other.org` picks
    `* /admin/`.
  - `mount_resolve_none`: only `* /admin/` → `/x` is `None`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-domain` → FAIL (types missing).
- [ ] **Step 3: Implement the types above.** `resolve` expects an already-normalized host and path (Task 7 does the
  normalizing); `path` matches a mount when it starts with the mount's `path`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-domain` → PASS.
- [ ] **Step 5: Commit.** `feat(domain): app ids, release manifests, mounts and the access-check contract`

---

### Task 3: Compatibility check

**Files:**
- Create: `crates/speedyspec-server-app/Cargo.toml` (deps: `speedyspec-server-domain` only, plus semver, thiserror,
  tracing), `src/lib.rs`, `src/compat.rs`
- Test: unit tests in `compat.rs`

**Interfaces:**
- Consumes: `AppId`, `ReleaseManifest` (Task 2).
- Produces: `pub struct Incompatibility { pub app: AppId, pub version: Version, pub requires: AppId, pub range: VersionReq, pub actual: Version }`
  with `Display` exactly `"{app} {version} requires {requires} {range}, active {requires} is {actual}"`;
  `pub fn check_set(set: &BTreeMap<AppId, ReleaseManifest>) -> Result<(), Vec<Incompatibility>>`.
  Plan 2 adds the "app X becomes version v" form on top of `check_set`.

- [ ] **Step 1: Write the failing tests.**
  - `consistent_set_passes`: server 3.2.1 (requires `{}`), admin 5.2.0 requires server `^3.1` → `Ok(())`.
  - `reports_unmet_requirement`: admin 6.0.0 requires server `^4`, server 3.2.1 → one `Incompatibility` whose
    `to_string()` is `"admin 6.0.0 requires server ^4, active server is 3.2.1"`.
  - `reports_both_directions`: server 4.0.0 requires admin `^6`, admin 5.2.0 requires server `^3` → two entries,
    sorted by `app`.
  - `ignores_unconfigured_app`: admin requires `frontend-api ^2`, no `frontend-api` in the set → `Ok(())`, and a
    `tracing` event at `info` naming `admin` and `frontend-api` is emitted once per call (use
    `tracing-subscriber`'s test writer to capture it).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-app` → FAIL.
- [ ] **Step 3: Implement `check_set`.** For each manifest, for each requirement whose target is in the set, test
  `range.matches(&target.version)`; collect failures, sort by `(app, requires)`.
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-app` → PASS.
- [ ] **Step 5: Commit.** `feat(app): compatibility check across a set of releases`

---

### Task 4: Build an in-memory static file set

**Files:**
- Create: `crates/speedyspec-server-static/Cargo.toml` (deps: domain, http, bytes, globset, mime_guess, blake3,
  brotli, flate2, thiserror), `src/lib.rs`, `src/build.rs`
- Test: `crates/speedyspec-server-static/tests/build.rs`, fixtures written into a `tempfile::TempDir` by the tests

**Interfaces:**
- Consumes: `StaticSection`, `CacheRule` (Task 2).
- Produces: `pub struct StaticSet` (immutable once built; cheap to share behind `Arc`);
  `StaticSet::build(release_dir: &Path, section: &StaticSection, limit_bytes: u64) -> Result<StaticSet, StaticBuildError>`;
  `StaticSet::total_bytes(&self) -> u64` (identity plus every kept compressed variant);
  `StaticSet::file(&self, rel: &str) -> Option<&StaticFile>`;
  `pub struct StaticFile { pub content_type: String, pub etag: String, pub cache_control: String, pub identity: Bytes, pub br: Option<Bytes>, pub gzip: Option<Bytes> }`;
  `pub const DEFAULT_LIMIT_BYTES: u64 = 512 * 1024 * 1024;`
- `StaticBuildError` variants: `RootMissing(PathBuf)`, `NotFoundMissing(String)`, `LimitExceeded { limit: u64, needed: u64 }`,
  `BadPattern(String)`, `Io(PathBuf, io::Error)`, `UnsupportedFile(PathBuf)` (symlink or non-regular file).

Values the spec pins: content type from the extension, with `; charset=utf-8` appended for `text/*`,
`application/javascript`, `application/json` and `image/svg+xml`; strong ETag from the content hash (`"` + first 16
bytes of blake3 as hex + `"`); brotli and gzip variants only for compressible types and only for files over 1024
bytes, each kept only if smaller than the identity body; `Cache-Control` from the first matching rule (globs relative
to `root`, `/` separators), else `no-cache`. Compressible types (my call): `text/*`, `application/javascript`,
`application/json`, `application/manifest+json`, `application/xml`, `image/svg+xml`, `application/wasm`. Brotli
quality 11, gzip level 9 (build happens once per release, so the slow settings are fine).

- [ ] **Step 1: Write the failing tests** in `tests/build.rs`:
  - `builds_types_etags_and_cache_rules`: files `index.html`, `_astro/app.4f2a.js`, `logo.png`; rules from the spec
    example → `index.html` has `text/html; charset=utf-8` and `no-cache`; the JS has
    `public, max-age=31536000, immutable`; `logo.png` has `image/png` and `no-cache`; every ETag matches
    `^"[0-9a-f]{32}"$`; two files with the same bytes have the same ETag.
  - `compresses_only_large_compressible`: a 4 KiB repetitive `.css` gets `br` and `gzip`, both shorter than
    identity; a 512-byte `.css` gets neither; a 4 KiB `.png` gets neither.
  - `limit_exceeded`: `limit_bytes` of 100 with a 4 KiB file → `LimitExceeded { limit: 100, needed }` with
    `needed >= 4096`.
  - `errors`: missing `root` dir → `RootMissing`; `not_found: "404.html"` absent → `NotFoundMissing`; a symlink
    inside root → `UnsupportedFile`; a cache pattern `[` → `BadPattern`.
  - `keeps_unicode_names`: `über menü.html` and `a+b.css` → `file("über menü.html")` and `file("a+b.css")` are
    `Some`.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-static --test build` → FAIL.
- [ ] **Step 3: Implement `StaticSet::build`.** Walk `release_dir/root` without following symlinks; store files in
  a `HashMap<String, StaticFile>` keyed by the `/`-separated path relative to root (no leading slash). Check the
  limit as bytes accumulate and stop early when exceeded.
- [ ] **Step 4: Run** the test → PASS.
- [ ] **Step 5: Commit.** `feat(static): build an in-memory, pre-compressed file set from a release`

---

### Task 5: Serve from the static set

**Files:**
- Create: `crates/speedyspec-server-static/src/respond.rs`, `src/encoding.rs`, `src/range.rs`
- Test: `crates/speedyspec-server-static/tests/respond.rs`

**Interfaces:**
- Consumes: `StaticSet`, `StaticFile` (Task 4).
- Produces: `StaticSet::respond(&self, method: &http::Method, mount_prefix: &str, rel_path: &str, query: Option<&str>, headers: &http::HeaderMap) -> http::Response<bytes::Bytes>`.
  `rel_path` is the already-normalized, percent-decoded path after the mount prefix, without a leading slash
  (`""` is the mount root). Task 7 calls this.
- Internal: `encoding::choose(accept: Option<&HeaderValue>, has_br: bool, has_gzip: bool) -> Encoding` with
  `enum Encoding { Br, Gzip, Identity }`; `range::parse(value: &HeaderValue, len: u64) -> RangeOutcome` with
  `enum RangeOutcome { Full, Partial(u64, u64), Unsatisfiable }`.

Behaviour the spec pins, plus my calls marked:
- `GET` and `HEAD` only; anything else → 405 with `Allow: GET, HEAD`. `HEAD` sends the same headers as `GET`,
  including `Content-Length`, and an empty body.
- Resolution: exact file, then `<rel>/index.html` (for `""`, `index.html`), then `<rel>.html`. A request for a
  directory without a trailing slash (some file exists under `<rel>/`) → 308 to `<mount_prefix><rel>/` plus `?query`
  if any (my call: 308). Nothing found → the `not_found` file with status 404 and `Cache-Control: no-cache` if set,
  else 404 with body `not found` (text/plain).
- Headers on a file response: `Content-Type`, `Content-Length`, `ETag`, `Cache-Control`, `Accept-Ranges: bytes`,
  `Content-Encoding` when not identity, and `Vary: Accept-Encoding` whenever the file has any compressed variant.
- `If-None-Match` (a list, `W/` prefixes allowed, weak comparison, or `*`) matching → 304 with `ETag`,
  `Cache-Control` and `Vary` (if applicable), no body.
- Accept-Encoding: highest q wins, `br` before `gzip` on a tie, `q=0` excludes, `*` covers unlisted codings, a
  missing header means identity. If identity is excluded and nothing else is available, send identity anyway (my
  call: a body beats a 406 for static assets).
- `Range`: one `bytes=` range only, applied to the identity body, with `Content-Encoding` absent. Satisfiable → 206
  with `Content-Range: bytes <start>-<end>/<len>`; unsatisfiable → 416 with `Content-Range: bytes */<len>`; more
  than one range or a malformed header → 200 with the full response as if no `Range` was sent. `If-Range` with a
  strong ETag equal to the file's → honour `Range`; any other `If-Range` → full 200 (my call).

- [ ] **Step 1: Write the failing tests** in `tests/respond.rs`, building a set in a temp dir:
  - `resolves_exact_index_and_html`: `""` → `index.html`; `docs/` → `docs/index.html`; `about` → `about.html`.
  - `redirects_directory_without_slash`: `docs` with prefix `/admin/` and query `a=1` → 308,
    `Location: /admin/docs/?a=1`.
  - `not_found_uses_configured_page` and `not_found_plain` (status 404, bodies as above).
  - `method_not_allowed`: `POST` → 405, `Allow: GET, HEAD`.
  - `head_has_length_no_body`.
  - `conditional_304`: `If-None-Match: W/"x", <etag>` → 304; `If-None-Match: *` → 304; other → 200.
  - Review Focus 3, `accept_encoding_weights`: on a file with both variants: none → identity; `gzip, br` → br;
    `br;q=0, gzip` → gzip; `*;q=0.5` → br; `identity;q=0` on a file with no variants → identity body, 200; `Vary`
    present on the compressible file and absent on a `.png`.
  - Review Focus 4, `range_cases` on a 10 000-byte `.txt` with variants: `bytes=0-99` → 206, 100 bytes,
    `Content-Range: bytes 0-99/10000`, no `Content-Encoding` even with `Accept-Encoding: br`; `bytes=-500` → last
    500 bytes; `bytes=9999999-` → 416, `Content-Range: bytes */10000`; `bytes=0-1,4-5` → 200 full; `If-Range` with
    the file's ETag + `bytes=0-9` → 206; `If-Range: "other"` + `bytes=0-9` → 200 full.
  - Review Focus 5, `serves_encoded_names`: `rel_path` `über menü.html` and `a+b.css` (as Task 7 will pass them,
    already decoded) → 200 with the right bodies.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-static --test respond` → FAIL.
- [ ] **Step 3: Implement `respond`, `encoding::choose` and `range::parse`** per the behaviour list.
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-static` → PASS.
- [ ] **Step 5: Commit.** `feat(static): serve files with negotiation, conditionals and ranges`

---

### Task 6: Configuration and placeholder access checks

**Files:**
- Create: `crates/speedyspec-server-infra-access-dev/Cargo.toml`, `src/lib.rs`
- Create: `crates/speedyspec-server/src/config.rs` (and add deps to `crates/speedyspec-server/Cargo.toml`: domain,
  toml, serde, ipnet, thiserror)
- Test: `crates/speedyspec-server-infra-access-dev/src/lib.rs` (unit), `crates/speedyspec-server/src/config.rs` (unit)

**Interfaces:**
- Consumes: `AccessCheck`, `AccessDecision`, `Identity`, `Mount`, `MountTable`, `AppId`, `AppKind` (Task 2).
- Produces (`-infra-access-dev`): `pub struct DenySignedIn;` (every `SignedIn` request → `Denied`, `Public` →
  `Allowed(anonymous)`), `pub struct DevAllowAll;` (always `Allowed(anonymous)`).
- Produces (`config.rs`):
  `pub struct Config { pub mode: Mode, pub data_dir: Option<PathBuf>, pub run_dir: PathBuf, pub listeners: Vec<ListenerConfig>, pub trusted_proxies: Vec<ipnet::IpNet>, pub apps: BTreeMap<AppId, AppConfig>, pub mounts: MountTable, pub access: AccessImpl, pub static_memory: u64 }`;
  `pub enum Mode { Managed, Immutable }`; `pub enum AccessImpl { Deny, DevAllowAll }`;
  `pub struct ListenerConfig { pub address: SocketAddr, pub tls: Option<TlsFiles>, pub http3: bool }`;
  `pub struct AppConfig { pub kind: AppKind, pub path: Option<PathBuf>, pub source: Option<String> }`
  (plan 2 adds `keys`, `keep`, `update`, `check_interval`, `prerelease`, `allow_unsigned`; unknown keys are an error
  now, so plan 2 must add them, not silently accept them);
  `Config::parse(toml: &str) -> Result<Config, ConfigError>`; `Config::load(path: &Path) -> Result<Config, ConfigError>`;
  `pub const DEFAULT_CONFIG: &str = "/etc/speedyspec/server.toml";`
  `pub const ADMIN_API_PATH: &str = "/admin/api/";` (built-in mount of the server app, host `*`, `signed-in`).

TOML shape (from the spec's example, plus my calls): top-level `mode` (default `managed`), `data_dir`, `run_dir`
(default `/run/speedyspec`); `[[listener]]` `address`, `tls = { cert, key }`, `http3`; `[proxy] trusted`;
`[app.<id>]` `kind`, `path` | `source`; `[[mount]]` `host` (default `*`), `path`, `app`, `access` (default
`public`); `[access] implementation = "deny" | "dev-allow-all"` (default `deny`); `[limits] static_memory` in bytes
(default `StaticSet` `DEFAULT_LIMIT_BYTES`).

Validation, each a `ConfigError` whose message names the offending key and value:
1. Exactly one app has `kind = "server"`. It may not have `path`, `source` checks wait for plan 2, and no
   `[[mount]]` may point at it (its only mount is the built-in admin API).
2. Immutable mode: every `static` and `php` app has `path` and no `source`. Managed mode: no app has `path`, and
   `data_dir` is set. (Spec: "a mode mixes nothing".)
3. Every mount's `app` exists; mount paths start and end with `/`; no mount duplicates `* /admin/api/`.
4. `http3 = true` requires `tls`.
5. `access.implementation = "dev-allow-all"` requires every listener address to be loopback.
6. At least one listener.

- [ ] **Step 1: Write the failing tests.**
  - `infra-access-dev`: `deny_signed_in` (SignedIn → Denied, Public → Allowed) and `dev_allow_all`.
  - `config.rs`: `parses_spec_example` (the spec's TOML with `mode = "managed"`), `parses_immutable_example`
    (server + admin `static` with `path` + blog `php` with `path`); one `rejects_*` test per validation rule above,
    each asserting the error message contains the key name (`rejects_two_server_apps`,
    `rejects_mount_on_server_app`, `rejects_immutable_with_source`, `rejects_managed_with_path`,
    `rejects_managed_without_data_dir`, `rejects_unknown_mount_app`, `rejects_admin_api_duplicate`,
    `rejects_http3_without_tls`, `rejects_dev_allow_all_on_public_address` (message contains `0.0.0.0:443` and
    `loopback`), `rejects_no_listeners`, `rejects_unknown_key`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-infra-access-dev -p speedyspec-server config` → FAIL.
- [ ] **Step 3: Implement** the two access checks and `Config::parse` (deserialize into raw serde structs with
  `deny_unknown_fields`, then validate into `Config`; append the built-in admin API mount to the `MountTable`).
- [ ] **Step 4: Run** the same command → PASS.
- [ ] **Step 5: Commit.** `feat: configuration with mode rules and placeholder access checks`

---

### Task 7: Request handling: normalization, forwarding, access gate, dispatch

**Files:**
- Create: `crates/speedyspec-server-presentation/Cargo.toml` (deps: domain, static, salvo, http, bytes, ipnet,
  percent-encoding, tracing), `src/lib.rs`, `src/path.rs`, `src/forwarded.rs`, `src/handlers.rs`, `src/service.rs`
- Test: unit tests in `path.rs` and `forwarded.rs`; `crates/speedyspec-server-presentation/tests/service.rs` using
  Salvo's test client

**Interfaces:**
- Consumes: `MountTable`, `Mount`, `Access`, `AccessCheck`, `AccessRequest`, `AccessDecision` (Task 2); `StaticSet::respond`
  (Task 5).
- Produces, `path.rs`: `pub fn normalize(raw_path: &str) -> Result<String, PathRejection>`; result starts with `/`,
  is percent-decoded once, has `.` and empty segments removed and `..` resolved; a trailing `/` is kept.
  `PathRejection` → 400. Rejected: `%2F`/`%5C` (any case) in the raw path, `\` or NUL after decoding, invalid
  UTF-8 after decoding, `..` above the root.
- Produces, `forwarded.rs`: `pub struct RequestOrigin { pub client: IpAddr, pub host: String, pub https: bool }`;
  `pub fn origin(peer: IpAddr, headers: &HeaderMap, tls: bool, trusted: &[IpNet]) -> RequestOrigin`.
  Host from `Host` (or HTTP/2–3 `:authority`), normalized: lowercased, port removed, one trailing `.` removed,
  IPv6 brackets removed. Only when `peer` is trusted: `Forwarded` (`for=`, `host=`, `proto=`) if present, else
  `X-Forwarded-For` (the rightmost address that is not trusted), `X-Forwarded-Host`, `X-Forwarded-Proto`.
- Produces, `handlers.rs`: `pub enum AppHandle { Static(Arc<StaticSet>), NotInstalled, PhpPlaceholder, AdminApiPlaceholder }`.
  Responses: `NotInstalled` and `PhpPlaceholder` → 503 with `Retry-After: 30` (my call: 30 s); `AdminApiPlaceholder`
  → 404.
- Produces, `service.rs`: `pub struct ServeState { pub mounts: MountTable, pub apps: HashMap<AppId, AppHandle>, pub access: Arc<dyn AccessCheck>, pub trusted_proxies: Vec<IpNet> }`;
  `pub fn build_service(state: Arc<ServeState>) -> salvo::Service`. Plan 2 replaces the `HashMap` with live, swappable
  handles; keep the lookup behind one function (`ServeState::handle(&self, app: &AppId) -> Option<&AppHandle>`) so
  that change stays local.

Order per request: normalize path (400 on rejection) → origin → `MountTable::resolve(origin.host, path)`; no mount →
404. If the path equals the mount path without its trailing slash → 308 to the slash form (before the access check:
the redirect reveals nothing). Then if the mount is `SignedIn`, `AccessCheck::check`; `Denied` → 401 with
`Cache-Control: no-store` and body `sign-in required` (project 2 changes this response). Then dispatch to the app's
handle with `rel_path` = path minus the mount prefix.

- [ ] **Step 1: Write the failing tests.**
  - Review Focus 1, `path.rs`: `normalize("/admin/../x") == "/x"`; `"/%61dmin/x" == "/admin/x"`;
    `"//admin//x" == "/admin/x"`; `"/admin/./x/" == "/admin/x/"`; rejections for `/admin%2Fx`, `/admin%2fx`,
    `/a%5Cb`, `/a%00b`, `/../x`, `/%C3%28`.
  - Review Focus 2, `forwarded.rs`: `Host: Example.COM:443` → `example.com`; `example.com.` → `example.com`;
    `[::1]:8080` → `::1`; untrusted peer with `X-Forwarded-Host: admin.example.com` → host from `Host`; trusted peer
    with `X-Forwarded-For: 203.0.113.9, 10.0.0.5` (10.0.0.0/8 trusted) → client `203.0.113.9`; trusted peer with
    `Forwarded: for=198.51.100.7;host=a.example;proto=https` → those values.
  - `tests/service.rs` with mounts `* /` (static blog-like set, public), `* /admin/` (static set, signed-in), the
    built-in `/admin/api/` and `* /blog/` (php), and `DenySignedIn`:
    - `public_static_served`: `GET /` → 200, the public set's `index.html`.
    - `signed_in_denied`: `GET /admin/` and `GET /admin/_astro/app.js` → 401, `Cache-Control: no-store`, and the body
      never contains the admin file's bytes.
    - `traversal_cannot_reach_admin_through_public`: `GET /x/../admin/` → 401 (it resolves to the admin mount);
      `GET /admin%2Findex.html` → 400.
    - `mount_root_redirect`: `GET /admin` → 308 `Location: /admin/`.
    - `php_placeholder`: `GET /blog/` → 503, `Retry-After: 30`.
    - `admin_api_placeholder` with `DevAllowAll`: `GET /admin/api/anything` → 404.
    - `no_mount`: with only `* /admin/`, `GET /x` → 404.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server-presentation` → FAIL.
- [ ] **Step 3: Implement** `normalize`, `origin`, the handles and `build_service` (one catch-all Salvo handler that
  runs the order above and converts the `http::Response<Bytes>` from `respond` into Salvo's response).
- [ ] **Step 4: Run** `cargo test -p speedyspec-server-presentation` → PASS.
- [ ] **Step 5: Commit.** `feat(presentation): normalize, route by mount, gate signed-in mounts, dispatch`

---

### Task 8: The `speedyspec-server` binary in immutable mode

**Files:**
- Modify: `crates/speedyspec-server/src/main.rs`; Create: `src/boot.rs`, `src/release.json`
- Test: `crates/speedyspec-server/tests/serve.rs`, `tests/fixtures/admin-a/` (`speedyspec-release.json`,
  `dist/index.html` containing `admin-a`, `dist/_astro/app.js`), `tests/fixtures/blog/speedyspec-release.json`

**Interfaces:**
- Consumes: `Config::load` (Task 6), `check_set` (Task 3), `StaticSet::build` (Task 4), `build_service` /
  `ServeState` / `AppHandle` (Task 7), `listeners::serve` (Task 1), `DenySignedIn` / `DevAllowAll` (Task 6).
- Produces: `boot::server_manifest() -> ReleaseManifest` (app id = the configured server app's id, kind `Server`,
  version `env!("CARGO_PKG_VERSION")`, `requires` from `src/release.json` via `include_str!`);
  `boot::prepare(config: &Config) -> Result<ServeState, BootError>`; plan 3's generation calls `prepare` too.

Commands (clap): `speedyspec-server run [--config <path>]` (default `DEFAULT_CONFIG`), `speedyspec-server manifest`
(prints the server manifest as pretty JSON on stdout), `speedyspec-server --version` (prints
`speedyspec-server <version>`). No command → help on stderr, exit 2.

`run` in this plan:
1. Load and validate config; any error → one line `speedyspec-server: <message>` on stderr, exit 1.
2. `mode = "managed"` → `speedyspec-server: managed mode is not available in this build yet`, exit 1 (plans 2–3).
3. For each `static`/`php` app: read `<path>/speedyspec-release.json`; its `app` and `kind` must equal the config's,
   else exit 1 naming the file and the field.
4. `check_set` over those manifests plus `server_manifest()`; failures → exit 1 with one stderr line per
   `Incompatibility` (its `Display`).
5. Build every static set with the configured limit; `php` apps get `PhpPlaceholder`.
6. Bind each listener with `std::net` (and its UDP socket when `http3`), log one JSON line per socket
   `{"event":"listening","address":"<addr>","protocol":"tcp"|"udp"}`, then `listeners::serve`.
7. `SIGTERM`/`SIGINT` → stop accepting, wait for in-flight requests up to 30 s (my call), exit 0.
8. Never write to disk.

- [ ] **Step 1: Write the failing tests** in `tests/serve.rs` (spawn the built binary with `env!("CARGO_BIN_EXE_speedyspec-server")`,
  a temp config on `127.0.0.1:0`, read the bound ports from the `listening` log lines):
  - `serves_static_over_h1_h2_h3`: immutable config, admin mounted `* /` public with `dev-allow-all`, one cleartext
    and one TLS+`http3` listener → `GET /` returns a body containing `admin-a` over HTTP/1.1, h2c, HTTPS/h2 and h3
    (`support::h3::get`).
  - `signed_in_denied_by_default`: same, admin mounted `signed-in`, no `[access]` → 401 over HTTP/1.1.
  - `php_mount_placeholder`: blog `php` app at `* /blog/` → 503.
  - `refuses_inconsistent_set`: admin fixture's manifest requires `server ^99` → exit 1, stderr contains
    `requires server ^99, active server is`.
  - `refuses_mixed_mode`: immutable with `source` on admin → exit 1, stderr names `source`.
  - `refuses_dev_allow_all_on_public_address`: listener `0.0.0.0:0` with `dev-allow-all` → exit 1, stderr contains
    `loopback`.
  - `refuses_managed_for_now`: `mode = "managed"` → exit 1, stderr contains `managed mode is not available`.
  - `writes_nothing`: run with the fixture copied into a temp dir made read-only (`chmod -R a-w`) and the process's
    working directory set to another read-only temp dir; it serves `GET /` → 200 and after shutdown both dirs list
    exactly the files they started with.
  - `sigterm_exits_cleanly`: send `SIGTERM` → exit 0 within 5 s.
  - `manifest_and_version`: `manifest` prints JSON with `"kind": "server"` and the crate version; `--version` prints
    `speedyspec-server <version>`; no arguments → exit 2.
- [ ] **Step 2: Run** `cargo test -p speedyspec-server --test serve` → FAIL.
- [ ] **Step 3: Implement** `main.rs` (clap, JSON tracing subscriber on stderr) and `boot.rs` (`server_manifest`,
  `prepare`, then the `run` steps above). `src/release.json` starts as `{ "requires": {} }`.
- [ ] **Step 4: Run** `cargo test --workspace` → PASS on macOS; then on Linux with the Docker command from Task 1
  Step 5 → PASS.
- [ ] **Step 5: Commit.** `feat: speedyspec-server serves immutable releases from memory`
