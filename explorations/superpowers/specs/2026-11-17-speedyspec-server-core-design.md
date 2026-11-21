# SpeedySpec Server core design

Date: 2026-11-17
Status: approved by Jacob, 2026-11-17

This is the first of five projects that together make the SpeedySpec web server. In order:

1. **Server core** (this document): HTTP/1, HTTP/2 and HTTP/3, routing to apps, static files from memory, independent
   releases per app with rollback, and socket handoff for upgrading the server itself.
2. Sign-in and the admin gate: sessions, users from `wp_users`, the admin locked behind sign-in.
3. PHP workers: a static PHP with the standard extensions, in supervised processes, serving the front-end API and
   the blog.
4. Live connections: WebSocket and WebTransport in Rust, with a bridge so PHP plugins can use them.
5. Moving wp-admin to Rust, screen by screen, behind the admin API. Ongoing.

Each later project gets its own design. This one defines the seams they plug into.

## Intent

Most of the WordPress clone is moving from PHP to Rust; in the end only plugins will be PHP. The server is the piece
that makes that move possible step by step: one Rust process that serves the SpeedySpec admin and the Rust admin API
directly, and runs PHP applications (the front-end API, the blog, other PHP front-ends) in supervised worker
processes it can kill and replace when one goes bad, the way php-fpm does. It sits behind a reverse proxy.

It also owns its own upgrades. Each application ships and updates independently. The server keeps the last *n*
releases of each one on disk, downloads a new one completely before using it, switches in one step, can roll back
instantly to any release it still has, and deletes the ones beyond *n*. The server binary is upgraded the same way,
by handing its listening sockets to the new version.

Everything static is held in memory and served from there.

## Goals

- One Rust server, built on Salvo, answering HTTP/1.1, HTTP/2 and HTTP/3 from one set of routes.
- Apps mounted by host and path. Static apps served entirely from memory, pre-compressed, with correct caching
  headers.
- Four independently released apps: **server**, **admin**, **front-end API**, **blog**. Further apps of the same
  kinds can be added by configuration.
- Per app: keep the last *n* releases, download and verify fully before use, switch in one step, roll back to any
  kept release, prune the rest.
- Each release declares which versions of the other apps it works with. A switch or rollback that would break a
  declared requirement is refused with the reason.
- Upgrading the server binary by socket handoff, with no dropped HTTP requests and automatic fall-back to the previous
  binary if the new one fails.
- Seams for projects 2 to 4: an access check per mount, a worker-pool contract for PHP apps, and a place for live
  connection handlers.

## Non-goals

- Sign-in, sessions and users (project 2). Project 1 ships only the access-check contract and placeholder
  implementations (see "Access check").
- Running PHP (project 3). Project 1 manages front-end API and blog releases fully (download, keep, switch, roll
  back, prune) but serves their routes through a placeholder pool that answers 503.
- WebSocket and WebTransport application traffic and the PHP bridge (project 4). Project 1 accepts HTTP/3 and makes
  sure the handoff treats long-lived connections correctly, so project 4 does not have to revisit it.
- The admin API's endpoints (project 5 and project 2's sign-in endpoints). Project 1 only reserves its mount.
- Converting `speedyspec-admin-astro` to a static build and replacing its Laravel Echo / Reverb client. That is a
  change to that package, needed before the admin can be served; it is listed under "Dependencies".
- ACME / automatic certificates. The reverse proxy, or certificate files, provide TLS.
- Keeping WebTransport sessions alive across a server upgrade (needs eBPF routing by QUIC connection ID).
- Windows.
- Automatic image updates and rollbacks for immutable mode on Kubernetes (a release manager or operator watching
  for new images). Jacob deferred this; for now the `speedyspec` CLI handles Kubernetes upgrades.
- Plugins as releases. PHP plugins are installed into the site, WordPress-style, and handled by project 3.

## Anti-goals

- A request served with files from two different releases of the same app.
- A release becoming live before it is fully downloaded, verified, unpacked and loaded.
- An upgrade or rollback that leaves the server with nothing serving.
- App-specific knowledge in the core. The core knows app *kinds* (static, php, server), never "the blog".
- Contracts in the same crate as their implementations.

## Constraints

- Rust 1.99, edition 2024, matching `packages/speedyspec-cli`.
- Salvo 1.2.x (released 2026-11-17), with its `quinn` feature for HTTP/3. Jacob chose Salvo.
- Unix only: Linux x86_64 and aarch64 (musl, static), macOS arm64 for development. Socket handoff uses Unix file
  descriptor passing.
- Must run under systemd (managed mode) and as a container's main process, including on Kubernetes (immutable
  mode). See "Deployment modes".
- Contracts live in their own crate; implementations in separate crates, each replaceable on its own. Crate names
  use the DDD layer names (`-domain`, `-app`, `-infra-*`, `-presentation`).
- Rust app code (the admin API, later wp-admin) is compiled into the server binary. Jacob chose this, so that an API
  release and a server self-upgrade are the same mechanism.
- The admin API is Rust only. Requests for admin screens owned by PHP plugins are forwarded to the front-end API with
  the signed-in user attached (wired in projects 2 and 3).

## Approaches

### HTTP stack

- A. tokio + hyper + axum, with quinn + h3 for HTTP/3. Most widely used; one tower router for all protocols. But
  WebTransport on h3 is experimental and would have to be wired in by hand.
- B. Pingora. Excellent socket handoff and proxying, but built to be a proxy, with no HTTP/3 or WebTransport.
- C. actix-web. Mature, no native HTTP/3.
- **D. Salvo (chosen).** HTTP/1, 2 and 3 in one server, WebTransport supported out of the box (`webtransport`
  examples), WebSocket, compression and a `tower-compat` layer. Costs: HTTP/3 goes through Salvo's own
  `salvo-http3`, so WebTransport depends on Salvo keeping up; smaller community than axum. Mitigation (my call): the
  core's logic lives in framework-neutral crates, and Salvo appears only in the presentation crate and the binary.

Jacob asked about Salvo after A was proposed; after checking, Salvo replaced A as the recommendation and Jacob chose it.

### How Rust app code ships

- **A. Compiled into the server, upgraded by socket handoff (chosen).** One binary; an admin API release is a server
  release. Self-upgrade comes for free from the same mechanism.
- B. Separate Rust processes behind the server, supervised like PHP workers. Independent of the server binary and
  crash-isolated, but adds a local hop and a second deployment unit per Rust app.
- C. WebAssembly modules loaded at run time. Hot-swappable and sandboxed, but database access and async I/O under
  WASM would slow down moving wp-admin to Rust.

Jacob chose A.

### Releases

- One release for everything: simpler, always consistent. Rejected.
- **One release per app (chosen):** admin, front-end API and blog update independently of each other and of the
  server. Mismatches are guarded by declared requirements between apps.

Jacob chose one release per app.

## Design

### Deployment modes

The server runs in one of two modes, set by `mode` in the config.

**Managed** (`mode = "managed"`, the default): a host or VM, run by systemd, with a writable `data_dir`. Everything
in "Release lifecycle" and "Socket handoff" applies: apps download, switch and roll back on their own, and the server
upgrades itself. The first server binary is the one systemd starts; on first start it is registered as the first
confirmed server release (copied into `releases/server/<version>/`), and from then on the keeper runs the confirmed
binary from the data directory.

Managed mode also runs on Kubernetes when the pod is given a writable volume for `data_dir` (Jacob's correction). The
keeper is then the container's main process, which is why it must never exit during a handoff. See "Managed mode on
Kubernetes" below for replicas.

**Immutable** (`mode = "immutable"`): container images with read-only storage, on plain containers or on Kubernetes
without a volume. Self-upgrade is disabled here (Jacob's decision). Upgrades are done by building a new image and
rolling it out with the `speedyspec` CLI (its Kubernetes driver). In this mode:

- Every app's files are baked into the image. Config gives each app a fixed `path` (and the release manifest found
  there) instead of a `source`. Nothing is downloaded, kept, switched, rolled back or pruned.
- The server never writes to disk. There is no `state.json`; `data_dir` is not required.
- `release install|switch|rollback|pin` and `reload` are refused with a message pointing at the CLI. `release list`
  and `status` still work, reporting the baked-in versions.
- The compatibility check still runs at start over the baked-in manifests, and the server refuses to start if the
  set is inconsistent. A bad image then fails fast instead of serving a mismatched set.
- The keeper still runs as the container's main process, but only to restart a crashed generation (my call: the
  serving process stays restartable without the container restarting). No handoff happens.
- Rollback is the orchestrator's job (for example, rolling back the Kubernetes deployment).

**Managed mode on Kubernetes: one pod leads.** Jacob chose this. With several replicas, all pods share one
ReadWriteMany volume as `data_dir`, and a single elected **leader** runs every release operation for the whole
deployment. The others are **followers** that serve whatever the leader has made live.

- **Election.** A `Leadership` contract in `-domain`, with two implementations: `single` (always leader; hosts under
  systemd, and a single replica) and `kubernetes` (a `coordination.k8s.io/v1` Lease, using the pod's service account).
  Config: `[cluster] leadership = "single" | "kubernetes"`, plus the Lease name and namespace. Defaults (my call):
  lease 15 s, renew deadline 10 s, retry 2 s. A leader that cannot renew before its renew deadline stops writing at
  once, before anyone else can take the Lease. The pod needs a Role allowing `get`, `create` and `update` on that one
  Lease, which the CLI's Kubernetes driver provisions.
- **Only the leader writes shared state.** It alone checks sources, downloads, verifies, unpacks, prunes and writes
  `state.json`. Every write carries an increasing `revision` and the leader's Lease epoch (its `leaseTransitions`
  count); followers ignore a state whose epoch is older than one they have already seen. `flock` is not used on the
  shared volume, because it is unreliable on network file systems; within one pod, handoff still uses it on the
  pod-local run directory.
- **Each pod writes only its own member file**, `<data_dir>/cluster/members/<pod>.json`: its heartbeat (every 2 s),
  the revision it has prepared, the revision it serves, its generation's version and whether that is on trial. A pod
  whose heartbeat is older than 10 s is not counted as live.
- **Switching together (two steps).** The leader writes the new set as *staged*. Every pod sees it (it polls
  `state.json` every 2 s, my call: file-change notifications are unreliable on network volumes), prepares it in memory
  and reports *prepared*. When every live pod has reported, or `cluster.prepare_timeout` (default 60 s) passes, the
  leader marks the set *active*, and all pods switch within one poll interval. A pod that failed to prepare keeps
  serving the old set and reports not-ready to its Kubernetes readiness probe, so Kubernetes stops sending it traffic
  until it catches up. If no pod prepared successfully, the leader abandons the switch and marks the release failed.
- **Server upgrades roll pod by pod.** The leader sets the desired server version. Pods upgrade one at a time by
  socket handoff, each finishing its trial before the next starts. A failed trial on any pod stops the rollout,
  marks the version failed, and reverts the pods already upgraded, including any pending group.
- **Pruning** never removes a release that any live pod reports as serving or prepared.
- **Changing leader.** A new leader deletes stale `.partial` directories, reads `state.json` and the member files,
  and carries on any staged switch or rollout from where the state shows it, or abandons it if the release can no
  longer be verified.
- **Control commands** work on any pod for reads (`status`, `release list`). On a follower, commands that change
  releases are refused with the leader's pod name, so you can run them there (my call; forwarding between pods is
  left for when the admin API provides it).
- A single host with `leadership = "single"` runs the same code: one member, always leader, with no waiting on others.

A config with `mode = "immutable"` and any app `source` set, or `mode = "managed"` with an app `path` set, is an
error at start (my call: a mode mixes nothing).

### Packages

A new package, `packages/speedyspec-server/`, a Cargo workspace in its own public repository
(`SpeedySpec/speedyspec-server`), nested under `packages/` the way `speedyspec-cli` is (my call, following the CLI's
pattern).

| Crate | Job |
|---|---|
| `speedyspec-server-domain` | Contracts and value types: app ids and kinds, versions, release manifest, requirement ranges, the release state, and the traits below. No I/O. |
| `speedyspec-server-app` | Use cases: check, install, switch, rollback, prune, the compatibility check, boot-time recovery. Depends only on `-domain`. |
| `speedyspec-server-infra-fs` | Release store and state file on disk; unpacking with path checks. |
| `speedyspec-server-infra-http` | Release source over HTTPS (the index format below). |
| `speedyspec-server-infra-minisign` | Signature verification. |
| `speedyspec-server-infra-memory` | In-memory store, source, verifier and leadership for tests. |
| `speedyspec-server-infra-kubernetes` | `Leadership` through a Kubernetes Lease. |
| `speedyspec-server-static` | Building and serving an in-memory static file set. |
| `speedyspec-server-handoff` | The keeper and the generation protocol (socket passing, ready/drain messages). |
| `speedyspec-server-presentation` | Salvo wiring: listeners, routing by mount, static handler, placeholder handlers, control socket. |
| `speedyspec-server` | The `speedyspec-server` binary: config, wiring, subcommands. |

Traits in `-domain` (each bound separately, so one can be replaced without the rest):

- `ReleaseSource`: list the versions available for an app; fetch a release artifact and its signature.
- `ReleaseStore`: put a verified artifact, unpack it, list kept releases, remove one, read a release's manifest.
- `StateStore`: read and atomically write the release state.
- `SignatureVerifier`: verify an artifact against the keys configured for its app.
- `AppRuntime`: what a mount's kind does when a release becomes live or stops being live (`prepare`, `activate`,
  `retire`). Implementations: static (this project), php (placeholder here, real in project 3), server (handoff).
- `AccessCheck`: decide whether a request may reach a mount (see "Access check").
- `Leadership`: whether this pod is the leader now, its Lease epoch, and a signal when that changes.
- `ClusterMembers`: write this pod's member file; read every pod's.

### Apps and mounts

An **app** has an id (`[a-z][a-z0-9-]*`), a **kind**, and a release channel. The kinds:

| Kind | Example | What a release contains | How it goes live |
|---|---|---|---|
| `server` | `server` | The binary for each target platform | Socket handoff |
| `static` | `admin` | A directory of files | Loaded into memory, swapped in one step |
| `php` | `frontend-api`, `blog` | A PHP application directory | Handed to the worker pool (project 3): new pool, old pool drains |

There is exactly one `server` app. Any number of `static` and `php` apps may be configured.

**Mounts belong to configuration, not to releases** (my call: where an app is reachable is a deployment decision).
A mount is a host (exact, or `*` for any) plus a path prefix, pointing at an app, with an access policy. The longest
matching prefix on the most specific host wins. The admin API is a built-in mount of the `server` app (default
`/admin/api/`), not a release of its own.

### Configuration

One TOML file (my call), path given by `--config`, default `/etc/speedyspec/server.toml`. It holds: listeners, data
directory, trusted proxies, per-app settings (kind, release source URL, signing public keys, `keep` count, update
policy), mounts, and limits. Example:

```toml
data_dir = "/var/lib/speedyspec"

[[listener]]
address = "127.0.0.1:8080"         # what the reverse proxy talks to: HTTP/1.1 and h2c

[[listener]]
address = "0.0.0.0:443"
tls = { cert = "/etc/speedyspec/tls/fullchain.pem", key = "/etc/speedyspec/tls/key.pem" }
http3 = true                       # also binds UDP 443 for HTTP/3 and WebTransport

[proxy]
trusted = ["127.0.0.1/32", "10.0.0.0/8"]

[app.server]
kind = "server"
source = "https://releases.example.com/server/index.json"
keys = ["RWQ..."]
keep = 3
update = "manual"

[app.admin]
kind = "static"
source = "https://releases.example.com/admin/index.json"
keys = ["RWQ..."]
keep = 3
update = "auto"

[[mount]]
host = "*"
path = "/admin/"
app = "admin"
access = "signed-in"
```

`Forwarded` / `X-Forwarded-*` headers are honoured only from `proxy.trusted` addresses. TLS certificates are read from
files and reloaded on `SIGHUP`. A listener without `tls` serves cleartext HTTP/1.1 and h2c only; HTTP/3 needs TLS.

Reverse proxy note: most reverse proxies cannot forward WebTransport. Where WebTransport is needed, clients must reach
a TLS + `http3` listener on this server directly. Which proxy is used is a deployment detail outside this project.

### Release artifacts

A release of a `static` or `php` app is a `.tar.zst` archive (my call) with `speedyspec-release.json` at its root:

```json
{
    "app": "admin",
    "version": "5.2.0",
    "kind": "static",
    "requires": { "server": "^3.1" },
    "static": {
        "root": "dist",
        "not_found": "404.html",
        "cache": [
            { "match": "_astro/**", "cache_control": "public, max-age=31536000, immutable" },
            { "match": "**/*.html", "cache_control": "no-cache" }
        ]
    }
}
```

- `version` is SemVer. `requires` maps other app ids to SemVer ranges. A requirement on an app that is not
  configured is ignored and logged once (my call: otherwise a server that names `frontend-api` could never be
  installed where no front-end API runs).
- `php` manifests carry a `php` section whose contents project 3 defines; project 1 stores it unread.
- A `server` release is the binary itself, one artifact per target triple; its manifest (with its own `requires`, for
  example on `frontend-api` for plugin forwarding) is embedded in the binary and printed by
  `speedyspec-server manifest`.

**Release index.** Each app's `source` URL serves a JSON index (my call; generic, so GitHub Releases or any static
host can serve it):

```json
{
    "app": "admin",
    "releases": [
        {
            "version": "5.2.0",
            "published": "2026-11-17T12:00:00Z",
            "artifacts": [
                { "target": "any", "url": "https://.../admin-5.2.0.tar.zst", "sha256": "…", "minisig": "https://.../admin-5.2.0.tar.zst.minisig" }
            ]
        }
    ]
}
```

`target` is `any` for static and php apps and a Rust target triple for the server. Pre-release versions are ignored
unless the app's config sets `prerelease = true`.

**Verification.** An artifact must match its `sha256` and carry a valid minisign (Ed25519) signature from one of the
app's configured `keys` (my call: minisign is small, well understood and has a Rust implementation). An app with no
keys configured refuses to install anything unless `allow_unsigned = true` is set, which the server logs as a warning
on every start.

**First start with no releases** (my call). A `static` or `php` app with nothing installed has its mounts answer 503
with `Retry-After`. `auto` apps run the lifecycle straight away at start; `manual` apps wait for
`release install`.

### On disk

```
<data_dir>/
  state.json
  releases/<app>/<version>/          unpacked release (php, static)
  releases/<app>/<version>.partial/  download/unpack in progress; deleted on start
  releases/server/<version>/speedyspec-server
  cluster/members/<pod>.json         one per pod: heartbeat and what it has prepared and serves

<run_dir>/                           pod- or host-local, never shared; default /run/speedyspec
  keeper.sock                        keeper control socket
  control.sock                       control socket of the serving generation
  state.lock                         lock between generations of this pod
```

`state.json` records, per app: the active version, the kept versions in install order, and for the server the
confirmed version and any version on trial. It is written to a temporary file, fsynced, renamed over the old one, and
the directory fsynced. Unpacking rejects absolute paths, `..` components, symlinks pointing outside the release, and
device files.

### Release lifecycle

For one app:

1. **Check.** Fetch the index; the newest version newer than the active one (respecting `prerelease`) is the
   candidate.
2. **Download** into `<version>.partial/`, resumable on retry.
3. **Verify** checksum and signature.
4. **Unpack** and read the manifest. Its `app` and `kind` must match the configuration; its `version` must match the
   index entry.
5. **Compatibility check** (below). Failure stops here; the downloaded release is kept so it can be switched to later
   when its requirements are met.
6. **Rename** `<version>.partial/` to `<version>/`. The release is now *kept* but not live.
7. **Prepare** through the kind's `AppRuntime`: static loads and compresses every file into memory; php asks the pool
   to start workers on the new directory; server starts a new generation (see "Socket handoff"). Prepare failing
   leaves the current release live and marks the new one failed in the state, with the reason.
8. **Switch**: the prepared release becomes live in one step and `state.json` is updated.
9. **Retire** the previous release through its runtime (free memory, drain the old pool, drain the old generation).
10. **Prune**: delete kept releases beyond `keep`, oldest first. The active release and, for the server, the
    confirmed binary are never pruned.

**Rollback** is steps 5, 7, 8 and 9 with an already-kept version. It needs no network.

**Update policy** per app: `manual` (check and install only when asked) or `auto` (check every
`check_interval`, default 15 minutes, and run the whole lifecycle). The default is `manual` for the server and `auto`
for other apps (my call: replacing the binary unattended is the riskier change). An app can be pinned
(`speedyspec-server release pin <app> <version>`), which stops auto-updates for it until unpinned.

**Compatibility check.** When app X is about to become version v, given the other apps' active versions:

- for every requirement in v's manifest on app Y, Y's active version must satisfy it; and
- for every other app Y whose active manifest requires X, v must satisfy that range.

A failure names both sides, for example: `admin 6.0.0 requires server ^4, active server is 3.2.1`.

**Switching several apps together.** When a change spans apps (a new admin needs a new server and the server needs
the new admin), neither can switch alone. `speedyspec-server release switch admin@6.0.0 server@4.0.0` prepares every
named release, checks compatibility against the resulting set as a whole, and switches them as one step: if any
prepare fails, none switches (my call). When the set includes the server, the other apps switch first inside the new
generation as part of its start-up, before it reports ready.

**Who runs release operations** (my call). Exactly one process at a time runs the lifecycle: the *active*
generation (the one the keeper last told to serve) of the leader pod (see "Managed mode on Kubernetes"; on a single
host that is always this pod). Within a pod, writes take an exclusive `flock` on `<run_dir>/state.lock`. A generation told to drain stops its auto-update timer and refuses
release commands at once. The keeper writes only the server's trial fields (on trial, confirmed, failed) and the
pending group (below), under the same lock. Operations are serialised: one lifecycle at a time across all apps.

**A group switch that includes the server** is recorded in `state.json` before the new generation starts, as a
pending group with every member's previous version. The trial covers the whole group. If the new generation fails its
trial, every other member of the group goes back to its previous version as part of the fall-back (see "Trial and
fall-back"), so the generation that takes over never boots into a set its own requirements forbid. The same applies
to a server rollback: a rollback to a binary whose requirements the current apps do not meet is refused, unless it is
named together with matching app versions in one group switch.

**Static apps across a switch** (my call). A browser that loaded release A may still ask for A's hashed assets after
the switch to B. For `release.static.fallback_grace` (default 1 hour) after a switch, a path that is missing from the
live release but present in the previous one *and* matched by an `immutable` cache rule is served from the previous
release. Only those files of the previous set stay in memory for the grace period; the rest is freed at retire.

### Static files from memory

When a static release is prepared, every file under its `static.root` is read into memory. For each file:

- content type from the extension (with `charset=utf-8` for text types);
- a strong ETag from the content hash;
- brotli and gzip variants for compressible types over 1 KiB, kept only when smaller (my call);
- `Cache-Control` from the first matching `cache` rule, else `no-cache`.

Serving: `GET` and `HEAD` only; `Accept-Encoding` negotiation with `Vary: Accept-Encoding`; `If-None-Match` → 304;
single-range `Range` requests on the uncompressed body. Path resolution: exact file, then `<path>/index.html`, then
`<path>.html`; a directory without a trailing slash redirects to the slash form. Nothing found serves `not_found`
with status 404 if set, else a plain 404.

The whole file set of a release is one value swapped atomically, so every request sees exactly one release. During a
switch the old and new sets are both in memory; `limits.static_memory` (default 512 MiB per app) caps a release's set
and prepare fails if it is exceeded.

### Access check

Every mount has `access = "public"` or `"signed-in"`. The presentation layer asks `AccessCheck` before any handler
runs, static files included, so nothing behind `signed-in` is ever sent to an anonymous visitor. Project 2 provides
the real implementation (session cookie, `wp_users`). Project 1 ships two:

- `deny`: every `signed-in` request gets 401. The default.
- `dev-allow-all`: lets everything through. The server refuses to start with it unless every listener is on a
  loopback address.

The check returns either "denied" or an identity value that later projects attach to forwarded requests; in
project 1 the identity is empty.

### Placeholder handlers

- Mounts of `php` apps answer 503 with `Retry-After` until project 3 supplies the worker pool.
- The admin API mount answers 404 for every path until project 2 and project 5 add endpoints.

### Socket handoff

**Keeper.** The binary runs in two roles. `speedyspec-server run` starts the **keeper**: a small long-lived process
that binds every listener (TCP and UDP), then starts a **generation**, a child process running the configured server
binary in `serve` role, passing it the sockets over a Unix socket (`SCM_RIGHTS`) along with the config path. The
keeper serves no traffic. It exists so that the process systemd or the container runtime watches never exits during an upgrade, and so a
crashed generation can be restarted without restarting the service (my call). Socket handoff happens in managed mode
only; in immutable mode the keeper only restarts crashed generations.

**Upgrading the server (generation N → N+1):**

1. The new binary is downloaded, verified and kept like any release.
2. The keeper starts generation N+1 from the new binary with the same sockets. N+1 loads config and state, prepares
   every live app (loads static sets, asks for PHP pools), and accepts TCP connections on the shared listeners.
   It does **not** read the UDP socket yet.
3. N+1 tells the keeper it is ready. If it does not do so within `handoff.ready_timeout` (default 30 s), or exits,
   the keeper kills it and N carries on; the release is marked failed.
4. The keeper tells N to drain. N stops accepting on TCP listeners, sends HTTP/2 `GOAWAY` and HTTP/3 `GOAWAY`,
   closes WebTransport sessions with an application close code meaning "server upgrading, reconnect", stops reading
   UDP and tells the keeper it has released it.
5. The keeper tells N+1 to start reading UDP. From here new QUIC connections and new TCP connections all reach N+1.
6. N finishes in-flight requests and keeps open WebSocket connections until they close or `handoff.drain_timeout`
   (default 5 minutes) passes, then closes what remains and exits.

The gap in step 4–5 affects only QUIC packets in flight, which QUIC retransmits.

**Trial and fall-back.** N+1 starts *on trial*. Draining is reversible while N+1 is on trial (my call): N keeps its
in-memory state until the trial ends. If N+1 exits within `handoff.trial_period` (default 2 minutes):

- if N is still running, the keeper tells N to **resume**: it takes the UDP socket back, accepts TCP again and
  re-enables release operations, reverting any pending group members first;
- if N has already exited, the keeper starts a fresh generation from the last confirmed binary, which reverts the
  pending group before reporting ready.

Either way the new version is recorded as failed. After the trial period N+1 becomes the confirmed version, the
pending group is cleared, and N is allowed to finish draining and exit. Rolling back the server is an upgrade to an
older kept binary, with the same steps.

**Reloading configuration.** `speedyspec-server reload` hands off to a new generation of the *same* binary, which
reads the config again (apps, mounts, keep counts, limits). It follows the upgrade steps, trial included. Listener
changes (addresses, TLS on/off, HTTP/3 on/off) need the keeper to rebind, which means a full restart; `reload` refuses
them and says so (my call).

**If a generation crashes outside an upgrade,** the keeper starts a new one from the confirmed binary, with
back-off (1 s doubling to 1 min) if it keeps crashing.

**Upgrading the keeper itself.** The keeper re-executes itself (`execve` in place, same process id) from the
confirmed binary when its own code version is older than the confirmed one, at the end of a successful trial. Its
listening sockets and the control socket to the running generation survive the `execve` (not close-on-exec), and the
running generation is unaffected. The keeper's protocol with generations is versioned and changes only by adding
messages (my call). Every keeper speaks every protocol version that has ever shipped, so rolling back to any kept
binary keeps working. A binary that needs a protocol newer than the keeper speaks is still refused with a message,
which only happens when the keeper has not yet re-executed after an upgrade.

Signals to the keeper: `SIGTERM`/`SIGINT` drain the current generation and exit; `SIGHUP` reloads certificates
(forwarded to the generation). Under systemd: `Type=notify`; the keeper sends `READY=1` once the first generation is
ready.

### Control

The binary doubles as the control tool. It talks to `<run_dir>/keeper.sock` (owner-only permissions), which the keeper
forwards to the active generation, so the path stays the same across handoffs (my call). `status` is answered from
both: the keeper's view of generations and trial, and the generation's view of apps.

```
speedyspec-server run [--config <path>]
speedyspec-server release list [<app>]
speedyspec-server release check [<app>]
speedyspec-server release install <app> [<version>]   # download, verify, keep; no switch
speedyspec-server release switch <app>@<version>...   # one or several apps, as one step
speedyspec-server release rollback <app> [<version>]  # default: the previously active version
speedyspec-server release pin <app> <version> | unpin <app>
speedyspec-server status
speedyspec-server reload
speedyspec-server manifest
```

`release install` followed by `release switch` is the manual path; `auto` apps run both. The admin API will expose
the same operations once project 2 provides sign-in; it calls the same `-app` use cases.

### Errors and logging

Every refusal names what was refused and why (failed check, which requirement, which file). Logs are structured
(JSON lines to stderr, my call), with each release operation, switch, handoff step and generation exit recorded.
`status` shows each app's active, kept and failed versions, the generation's version and whether it is on trial.

### Testing

- `-app`: unit tests against `-infra-memory` for the lifecycle, compatibility checks, multi-app switch, prune rules,
  and boot recovery (stale `.partial` directories, state pointing at a missing release falls back to the newest kept
  one and logs it).
- `-infra-fs`: atomic state writes, and unpacking hostile archives (absolute paths, `..`, escaping symlinks).
- `-infra-http` + `-infra-minisign`: against a local HTTPS server, including bad checksums, bad or missing signatures,
  interrupted and resumed downloads.
- `-static`: content types, ETags, 304s, compression negotiation, ranges, index and 404 resolution, memory limit.
- Switch under load: a load generator over HTTP/1.1, HTTP/2 and HTTP/3 while a static app switches repeatedly; zero
  failed requests and no response mixing releases (each fixture release stamps its version into every file).
- Handoff under load: the same load while the server upgrades; zero failed HTTP requests; an open WebSocket stays
  served by the old generation until closed; a WebTransport session receives the upgrading close code.
- Trial: a fixture binary that exits two seconds after becoming ready is rolled back to the confirmed binary.
- Leader and followers: three server processes on one shared directory with the in-memory leadership: a two-step
  switch lands on all three within one poll interval; a member that fails to prepare goes not-ready and the others
  switch; killing the leader mid-switch hands over and the new leader finishes or abandons it; a deposed leader's
  late write (older epoch) is ignored; a rolling server upgrade stops and reverts on one pod's failed trial.
- Kubernetes: an end-to-end test in CI on a throwaway cluster (kind or k3d) with three replicas, a ReadWriteMany
  volume and the Lease, covering an admin switch and a server upgrade.
- Immutable mode: serves baked-in apps from a read-only file system with no writes; refuses release commands;
  refuses to start on an inconsistent set or on a config mixing `source` and `path`.
- Access check: `deny` returns 401 for `signed-in` static files; `dev-allow-all` refuses to start on a non-loopback
  listener.
- CI on Linux x86_64 (musl) and macOS arm64.

## Dependencies

- `speedyspec-admin-astro` must build with `output: 'static'` (no Node adapter), its `[...slug].astro` route must
  produce every page at build time through `getStaticPaths`, and its Laravel Echo / Reverb client must be replaced in
  project 4. Its release pipeline must produce the archive and manifest above.
- Release pipelines for each app (CI producing the archive, `index.json` and minisign signature) are part of each
  app's own repository.

## Left to the builder

- Exact config keys and their validation messages, beyond the shape above.
- Internal layout of each crate, the keeper–generation message format (versioned), and the control-socket protocol.
- Whether Salvo accepts already-bound TCP and UDP sockets directly, or needs a small custom acceptor; confirm this
  first with a short spike, since the handoff depends on it.
- Crate choices for compression, tar/zstd, minisign and the load generator.
- Log field names.
