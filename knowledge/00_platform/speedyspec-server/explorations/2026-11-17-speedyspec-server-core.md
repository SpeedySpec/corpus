# Exploration: The SpeedySpec server core

**Status:** superseded in part by the PHP runtime and admin gate design
**Created:** 2026-11-17
**Author:** Claude, for the project owner
**Context:** One Rust server that serves the apps, releases each one on its own, and upgrades itself without downtime.
**Approved by:** the project owner on 2026-11-17
**Socialized with:** not socialized

---

## Problem Statement

The WordPress clone is moving from PHP to Rust, and that move needs a host that can run both sides at once. Nothing
served the static admin from memory, ran PHP apps under supervision, or let each app (admin site, front-end API,
blog) ship and roll back on its own. Operators had no safe way to upgrade the server itself: a restart drops
requests, and a bad binary has no automatic way back.

## Current State

- No server existed. `SpeedySpec/speedyspec-server` started from an empty repository (its first commit is an
  initial commit, then a Salvo proof).
- `speedyspec-cli` was the only Rust program. Its workspace layout, Rust 1.99 toolchain and Kubernetes driver set the
  patterns this design copied.
- The admin site (`speedyspec-admin-astro`) was built for a Node adapter and used a Laravel Echo / Reverb client, so
  it could not yet be served as static files.

## Proposed Solution

### Architecture

```
           reverse proxy (TLS optional)
                     |
   +-----------------v------------------+
   | keeper (speedyspec-server run)      |  owns every TCP/UDP socket, never exits on upgrade
   |   - restarts crashed generations    |
   |   - forwards keeper.sock commands   |
   +-----------------+------------------+
                     | sockets via SCM_RIGHTS, JSON-lines protocol
   +-----------------v------------------+
   | generation N (serve role, Salvo)    |  HTTP/1.1, HTTP/2, HTTP/3
   |   mounts: host + path prefix -> app |
   |   static apps: whole file set in RAM|
   |   php apps: worker pool (project 3) |
   |   ReleaseManager: check/install/    |
   |   switch/rollback/prune             |
   +-----------------+------------------+
                     |
   data_dir: state.json, releases/<app>/<version>/, cluster/members/<pod>.json
```

Crates follow the DDD layer names: `-domain` (contracts and values, no I/O), `-app` (use cases), `-infra-fs`,
`-infra-http`, `-infra-minisign`, `-infra-memory`, `-infra-kubernetes`, `-static`, `-handoff`, `-presentation`
(the only crate besides the binary that names Salvo), and the `speedyspec-server` binary.

### Data Model

- **App kinds:** `server` (exactly one), `static`, `php`. The core knows kinds, never particular apps.
- **Release manifest** (`speedyspec-release.json`): `app`, SemVer `version`, `kind`, `requires` (app id to SemVer
  range), and a kind section (`static` with `root`, `not_found` and cache rules; `php` stored unread).
- **Release index:** JSON per app listing versions and per-target artifacts with `sha256` and a minisign signature.
- **`state.json`:** per app the active, previous, kept, failed and pinned versions; later plans added server trial
  fields, a pending group, the leader's `epoch`, a `staged` set and a `rollout`. Every field defaults, so older
  state files keep loading.

### Interfaces

Contracts in `-domain`, each bound separately: `ReleaseSource`, `ReleaseStore`, `StateStore`, `SignatureVerifier`,
`AppRuntime` (`prepare`, `activate`, `retire`), `AccessCheck`, `Leadership`, `ClusterMembers`, and later
`BinaryInspector`. The control commands are `release list|check|install|switch|rollback|pin|unpin`, `status`,
`reload` and `manifest`.

## Design Decisions

1. **Salvo for HTTP.** It serves HTTP/1, 2 and 3 from one router and supports WebTransport out of the box.
   *Trade-off:* HTTP/3 depends on Salvo's own `salvo-http3` keeping up, and its community is smaller than axum's.
   Salvo is confined to `-presentation` and the binary to limit that risk.
2. **Rust app code compiled into the server, upgraded by socket handoff.** An admin API release and a server
   self-upgrade become one mechanism. *Trade-off:* every Rust change is a server release. Reversed for the admin API
   by the PHP runtime and admin gate design, which made the admin API a PHP app.
3. **One release per app, guarded by declared requirements.** A switch or rollback that breaks a `requires` range
   is refused with both sides named. *Trade-off:* multi-app changes need a group switch.
4. **Two modes.** Managed (writable `data_dir`, self-managing) and immutable (baked-in images, never writes, upgrades
   by the CLI). A config that mixes `source` and `path` is refused. *Trade-off:* immutable mode leaves rollback to
   the orchestrator.
5. **A keeper process that owns the sockets.** It never exits during a handoff, so systemd or the container runtime
   never sees a restart. *Trade-off:* a second process role and a versioned keeper–generation protocol to keep
   backward compatible forever.
6. **New server binaries start on trial** (default 2 minutes) and fall back to the confirmed binary on exit.
   *Trade-off:* the old generation keeps its memory until the trial ends.
7. **Static apps held in memory**, pre-compressed, swapped as one value, with a one-hour grace for the previous
   release's immutable assets. *Trade-off:* memory use, capped at 512 MiB per release set by default.
8. **On Kubernetes, one elected pod leads** through a Lease; only it writes shared state, every write carries its
   epoch, and switches happen in two steps (staged, then active). *Trade-off:* coordination over a shared
   ReadWriteMany volume. Later narrowed: the PHP runtime and admin gate design says replicas must also work
   independently with no `[cluster]`, and cluster mode is used only when configured.
9. **Unix only, TLS on `ring`.** `ring` builds on musl without cmake. *Trade-off:* no Windows.

## Implementation Phases

1. **Foundation and immutable mode.** Salvo proof, domain types, compatibility check, in-memory static serving,
   config, path and host normalization, access-check placeholders. *Delivers:* a server that serves baked-in apps
   over HTTP/1.1, h2c, HTTPS and HTTP/3 and refuses inconsistent sets. Shipped as direct commits on `main` (no PR).
2. **Managed releases on one host.** Index, resumable downloads, minisign, hardened unpacking, `state.json`, the
   `ReleaseManager`, swappable handles, auto-update, control socket. *Delivers:* self-managed app releases.
   Shipped in SpeedySpec/speedyspec-server#1 (210 tests).
3. **Keeper and socket handoff.** Generations, server releases, pending groups, trial and fall-back, `reload`,
   keeper re-exec, systemd notify, `SIGHUP` certificate reload, `Alt-Svc`. *Delivers:* zero-downtime self-upgrade.
   Shipped in SpeedySpec/speedyspec-server#2 (284 tests).
4. **Cluster.** `Leadership`, member files, epoch-stamped writes, two-step switch, pod-by-pod rollout, `/readyz` and
   `/livez`, Kubernetes manifests and a k3s end-to-end test. *Delivers:* managed mode across replicas. Shipped in
   SpeedySpec/speedyspec-server#3 (350 tests).

## Fleet Impact

- **Server:** the whole `speedyspec-server` repository comes from this design (confirmed by the merged PRs).
- **Admin site:** must build as static files with every page generated at build time, and ship the release archive
  and manifest. Whether that conversion is done is unverified here.
- **Front-end API and blog:** their releases are managed from day one, but served by a 503 placeholder until the PHP
  foundation added worker pools.
- **CLI:** handles Kubernetes upgrades in immutable mode; its Kubernetes driver provisions the Lease Role. The
  server's `scripts/k3s.sh` was copied from the CLI.
- **Admin API:** first a reserved Rust mount (`/admin/api/`, answering 404). Removed by the PHP runtime and admin gate
  design.

## Open Questions

1. **Automatic image updates in immutable mode on Kubernetes** — the project owner deferred it; an operator or release
   manager watching for images would settle it.
2. **Keeping WebTransport sessions alive across an upgrade** — needs eBPF routing by QUIC connection ID; out of scope
   until live connections are designed.
3. **Where a stray `H3_NO_ERROR` close comes from under load** — seen in 1 of 4 and 2 of 5 Linux runs of the switch
   and handoff load tests; the handoff test now tolerates it if a retry 100 ms later succeeds. A root-cause trace
   would settle whether it hides a UDP-release bug.
4. **ReadWriteMany behaviour** — the k3s test uses a single-node ReadWriteOnce volume, and serves releases from a
   loopback sidecar, so RWX semantics and HTTPS downloads in-cluster are untested. A multi-node test would settle it.
5. **Deferred minors from plan 2** — prune deletes before saving, `check_interval = "0s"` is accepted, no artifact
   size cap, control-socket permissions set just after `bind` with no request size cap, unpacked files not fsynced.
6. **Deferred minors from plan 3** — re-exec edge cases (request ids restart, a saved-state variable leaks to
   children), `reload` ignores changed certificate paths, `ConfirmServer` is reachable from `keeper.sock`, immutable
   mode creates `server.lock`.
7. **Forwarding release commands from a follower to the leader** — left until the admin API can carry it; followers
   refuse and name the leader for now.

## Spike Findings

**Salvo on pre-bound sockets**
- **Settles:** whether Salvo can serve from sockets bound outside it, which the keeper design depends on.
- **Built:** a proof test serving HTTP/1.1, h2c, HTTPS (ALPN h2) and HTTP/3 from `std` sockets in one `serve` call.
- **Established:** Salvo 1.2.1's acceptor constructors are public, so no patch or custom acceptor is needed; quinn
  0.11 builds an endpoint over a given `UdpSocket`; `stop_graceful` stops accepting while in-flight requests finish.
- **Does not establish:** behaviour across processes or under load.
- **Consequence:** `serve` takes a router rather than a `Service` (Salvo's `Service` is not `Clone`).

**Socket handoff mechanics**
- **Settles:** pause and resume on one listener, sockets shared across processes, and handing a QUIC endpoint over.
- **Built:** four proof tests, plus the handoff load test.
- **Established:** a stopped Salvo `Server` cannot restart, so resume means a fresh server on a fresh `try_clone`;
  a closed quinn endpoint keeps reading the shared UDP socket for about 3 × PTO and refuses new handshakes (4,733
  refused HTTP/3 requests in one upgrade). The fix is an order: drop the server config, close with code `0x5353`,
  wait 50 ms for close packets to leave, rebind onto a throwaway socket, then stop.
- **Does not establish:** WebTransport session handling.
- **Consequence:** while draining, HTTP/1.x answers carry `Connection: close` and TCP accepts stop after a 1 s grace.

## Success Criteria

- [x] Static apps served over HTTP/1.1, HTTP/2 and HTTP/3 from memory — `serves_static_over_h1_h2_h3`.
- [x] Path tricks (`..`, encoded slashes, NUL) never reach the wrong mount — presentation tests, plan 1.
- [x] Immutable mode writes nothing and refuses inconsistent or mixed configs — `writes_nothing` and refusal tests.
- [x] Hostile archives are refused and leave nothing behind — `-infra-fs` tests, plan 2.
- [x] A static switch under load has zero failed or mixed responses — `switch_under_load` (with the flake above).
- [x] Server upgrade under load drops no HTTP requests — `handoff_under_load`, with the retry tolerance noted above.
- [x] A binary that dies on trial falls back to the confirmed one — plan 3 upgrade tests.
- [x] Leader, followers, deposed-leader writes and rollout revert behave as specified — plan 4 cluster tests.
- [x] Three replicas on k3s switch the admin and roll a server upgrade — `kubernetes_e2e`, run in its own CI job.
- [ ] Multi-node ReadWriteMany behaviour — not tested.

---

## Non-Goals

- **Sign-in and sessions** — project 2; later replaced by the PHP runtime and admin gate design.
- **Running PHP** — project 3, the PHP foundation.
- **WebSocket and WebTransport traffic** — project 4; only the handoff's treatment of long-lived connections is here.
- **ACME certificates** — the reverse proxy or certificate files provide TLS.
- **Windows** — Unix descriptor passing is required.
- **Plugins as releases** — PHP plugins are installed into the site.

## Alternatives Considered

### Option: tokio + hyper + axum, with quinn and h3
- **Pros:** most widely used; one tower router.
- **Cons:** WebTransport on h3 is experimental and hand-wired.
- **Verdict:** rejected after the project owner asked about Salvo; Salvo replaced it as the recommendation.

### Option: Pingora
- **Pros:** strong socket handoff and proxying.
- **Cons:** built as a proxy; no HTTP/3 or WebTransport.
- **Verdict:** rejected.

### Option: actix-web
- **Pros:** mature.
- **Cons:** no native HTTP/3.
- **Verdict:** rejected.

### Option: Rust apps as separate supervised processes, or as WebAssembly modules
- **Pros:** independent of the server binary; WASM is hot-swappable.
- **Cons:** an extra hop and deployment unit; WASM slows database and async I/O work.
- **Verdict:** rejected by the project owner in favour of compiling into the server.

### Option: one release for everything
- **Pros:** always consistent.
- **Cons:** no independent updates.
- **Verdict:** rejected by the project owner.

## Dependencies

- Salvo 1.2.x with `quinn`, Rust 1.99, and the CLI's workspace conventions.
- The admin site built as static files with a release pipeline producing the archive, index and signature.
- Each app's own repository producing its release artifacts.

## References

- [SpeedySpec/speedyspec-server](https://github.com/SpeedySpec/speedyspec-server)
- [SpeedySpec/speedyspec-server#1](https://github.com/SpeedySpec/speedyspec-server/pull/1) — managed releases
- [SpeedySpec/speedyspec-server#2](https://github.com/SpeedySpec/speedyspec-server/pull/2) — keeper and socket handoff
- [SpeedySpec/speedyspec-server#3](https://github.com/SpeedySpec/speedyspec-server/pull/3) — cluster
- [The PHP foundation](2026-11-18-php-foundation.md)
- [The PHP runtime and admin gate](2026-11-19-php-runtime-and-admin-gate.md)
- Source spec: `2026-11-17-speedyspec-server-core-design.md`
- Source plans: `2026-11-17-speedyspec-server-foundation.md`, `2026-11-17-speedyspec-server-managed-releases.md`,
  `2026-11-18-speedyspec-server-keeper-handoff.md`, `2026-11-18-speedyspec-server-cluster.md`
