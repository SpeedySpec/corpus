# Exploration: The PHP runtime as its own release, and the admin gate

**Status:** built
**Created:** 2026-11-19
**Author:** Claude, for the project owner
**Context:** Moving embedded PHP out of the server into a separately released runtime, and gating the admin behind sign-in.
**Approved by:** approved by building, 2026-11-19 (the spec reads "draft for review"; it carries the project owner's rulings dated 2026-11-18 to 2026-11-20)
**Socialized with:** not socialized

---

## Problem Statement

The server held the embedded PHP, so every runtime change, and every step of porting WordPress to Rust, needed a
server release. The server should change rarely. Separately, the admin had no real gate: signed-in mounts were
either denied outright or open in development, so nothing kept admin files and admin API responses from visitors who
had not signed in. Maintenance during upgrades, and an outage, had no defined answer either.

## Current State

- After the PHP foundation (SpeedySpec/speedyspec-server#4), the server binary had a `php` Cargo feature and a hidden
  `php-worker` role; `EmbeddedPhpPool` spawned the server's own executable as workers.
- The server core design said the admin API was Rust compiled into the server, behind a built-in `/admin/api/` mount
  that answered 404.
- `AccessCheck` had only two implementations: `deny` and `dev-allow-all`.
- The admin site relied on its own proxy files (`public-files.json`, `shared-files.json`) for public paths.

## Proposed Solution

### Architecture

```
 visitor --> speedyspec-server (no PHP)
              | 1. normalise path; internal path? -> 404
              | 2. app unhealthy? -> plain 503      app in maintenance? -> page from memory, 503
              | 3. gated mount: match release access rules
              |      public / not-installed / maintenance / page / sign-in
              |      session answer cached by SHA-256(Cookie), 60 s
              v
           FastCGI on a private Unix socket
              |
   speedyspec-php-runtime worker (one pool per PHP app, own site_dir and env)
     admin-api (gate app) | frontend-api | blog
              ^
   server's own calls on reserved workers: health probes, site, session,
   changes (polled every 1 s), migrations
```

### Data Model

- **App kind `runtime`:** one binary per target triple, manifest printed by `speedyspec-php-runtime manifest`, with
  the contract majors it speaks.
- **`php` manifest additions:** `runtime` (version range), `access`, `gate` (version and endpoint paths), `health`
  (`live`, `ready`), `switch` (`maintenance` scope and `migrate` path).
- **`static` manifest additions:** `access`, `maintenance_page`.
- **State additions:** per-app maintenance sources, runtime trials, and a `migrated` mark on a staged cluster set.

### Interfaces

- `speedyspec-runtime-contract`: `CONTRACT_MAJOR = 1`, `WorkerSpec`, `WorkerReport` (NUL-prefixed `FCGI_STDERR`
  record with memory, request count and poisoned flag), `RuntimeManifest`.
- `speedyspec-server-gate`: rule matching, remembered session answers, revocation, maintenance sources, health
  counters, and the `GateCalls` port (`probe`, `site`, `session`, `changes`, `migrate`).
- `AccessDecision` gains `Forbidden`, `Disabled` and `Busy`.
- Commands: `maintenance on|off|clear [<app>...]`, `release retry-migration <app>`.

## Design Decisions

1. **The PHP runtime is a separate binary, released like any app.** A runtime release never needs a server release.
   *Trade-off:* a versioned contract between the two binaries; both sides refuse an unknown major.
   [ADR 0025](../../adr/0025-supervised-php-worker-pools.md) records this as superseding where its pools ran.
2. **The admin API is PHP, its own app, apart from the front-end API.** Admin areas depend on plugins, and it acts
   with elevated privileges. The project owner reversed the server core's "admin API is Rust only".
   *Trade-off:* the server now depends on a PHP app to answer its gate questions.
3. **The gate's questions, health and revocation are ordinary app endpoints over FastCGI** (chosen by the project
   owner). *Trade-off:* revocation is polled once a second rather than pushed.
4. **Releases declare their access rules,** matched before static resolution, first match wins. Not signed in is 401,
   signed in without the page 403, a disabled conditional entry 404, alike for existing and missing paths; protected
   responses are `private, no-store`. *Trade-off:* the admin site must list visitor paths, not resolved files.
5. **Only positive session answers are remembered,** keyed by a hash of the whole `Cookie` header. Sign-out forgets at
   once; other changes within about a second; a failed poll forgets everything. *Trade-off:* extra PHP calls for
   signed-out visitors, capped at 16 pending.
6. **Per-app environments,** so the admin API can hold stronger credentials than public apps. *Trade-off:* the
   operator manages several credential sets.
7. **A new runtime is on trial and rolls back on its own** if any app on it turns unhealthy (the project owner's
   choice). *Trade-off:* a two-minute window of extra state.
8. **An unhealthy app answers a plain 503, not the maintenance page** (the project owner, 2026-11-20). An outage is
   not planned maintenance. *Trade-off:* visitors see a bare error during an outage.
9. **Replicas work independently; cluster mode is used only when configured** (the project owner, 2026-11-19).
   Without `[cluster]`, each replica switches, migrates and rolls back on its own, so migrations must be safe to
   repeat. *Trade-off:* the release pipeline must document that rule.
10. **Site maintenance from the admin API never covers the admin itself,** so admins can sign in to end it.

## Implementation Phases

1. **Contract crate and runtime binary.** *Delivers:* `speedyspec-php-runtime` with `worker` and `manifest`.
2. **The `runtime` kind and runtime switches.** *Delivers:* every PHP app on a runtime moves to new pools in one step.
3. **Server without PHP.** *Delivers:* `scripts/check-no-php-in-server.sh` in CI.
4. **Per-app environment and reserved workers.**
5. **Health probes and per-app maintenance.**
6. **Access rules, gate decisions, revocation.**
7. **Maintenance from the admin API and operators; migrations on switch.**
8. **Runtime trial and automatic rollback; docs.**

All shipped in SpeedySpec/speedyspec-server#6 (merged 2026-11-19). SpeedySpec/speedyspec-server#10 carried the
review's remaining decisions and fixes, and SpeedySpec/speedyspec-server#11 added release pipelines for the server
and the runtime.

Building changed the design in these ways:

- **The worker spec travels on stdin** (`worker --spec-stdin`), never in argv, and workers keep only an allowlist of
  the server's environment, so per-app credentials don't leak through the process table.
- **Each pool reserves two workers, not one:** one for health probes, one for gate calls and migrations.
- **Health first fed maintenance.** PR #6 put an unhealthy app into maintenance; PR #10 changed it to the plain 503
  the spec now describes.
- **Workers that fail to start back off** (100 ms doubling to 10 s). Without it, tests saw tens of thousands of
  starts in one second.
- **`keep` defaults to 2,** and pruning never removes a runtime trial's rollback target (with `keep = 1` it had been
  pruned).
- **A sign-out wins over a session check in flight:** every forget bumps a counter, and an older check's answer is
  not remembered.

## Fleet Impact

- **Server:** loses all PHP crates, the `php-worker` role and the `/admin/api/` mount; gains the gate (confirmed,
  PR #6).
- **PHP runtime:** a new binary and release stream (confirmed, PRs #6 and #11).
- **Admin API:** must implement the gate endpoints (`site`, `session`, `changes`) and health probes. Whether it does
  yet is unverified.
- **Admin site:** must declare access rules and a maintenance page in its manifest, replacing its proxy files.
  Unverified.
- **Front-end API and blog:** releases gain `runtime`, `health` and optional `switch` fields. Unverified.
- **CLI:** its copied crates keep their paths; re-syncing them is the CLI's own work. No CLI change confirmed here.

## Open Questions

1. **Live connections** — WebSocket push of revocations, and replacing the `changes` poll, wait for the
   live-connections design; WebTransport after that.
2. **Routing in the server and `WP_Rewrite`** — still next for the runtime.
3. **The admin API's own endpoints** beyond the gate — separate work.
4. **Linux runs of the review fixes** — PR #10's fixes were not run on Linux locally; CI's PHP jobs were their first
   Linux run.
5. **Rulings to revisit if they hurt** — commands wait behind a running migration; incompatible runtime majors need
   an intermediate app release; a release that still ships its migration must keep declaring it.

## Success Criteria

- [x] The server builds and passes with no static PHP present — the check script and CI's non-PHP job.
- [x] A contract-major mismatch is refused by both sides, naming both — contract and worker tests.
- [x] A runtime switch moves every app with no failed requests; a failed prepare switches nothing — runtime switch
  tests.
- [x] A runtime unhealthy during its trial rolls back and is marked failed — `bad_runtime_rolls_back_on_its_own`.
- [x] 401, 403 and 404 alike for existing and missing paths — gate rule tests.
- [x] Internal paths get 404 for every spelling and never reach PHP — normalisation and variant tests.
- [x] Sign-out forgets at once; a sign-out wins over a session check in flight — revocation tests (the latter added
  in PR #10).
- [x] A failed migration keeps the old release live and maintenance on — migration tests.
- [x] Independent replicas each migrate safely — two servers sharing a fixture database.
- [ ] Linux CI green for the PR #10 fixes — not confirmed here.

---

## Non-Goals

- **Live connections** — the next design.
- **The admin API's endpoints beyond the gate** — their own work.
- **Converting admin screens and app features** — owned by other work.
- **Running stock WordPress** — neither binary ever does.
- **The server deciding permissions** — it only enforces the admin API's answers.

## Alternatives Considered

### Option: Internal FastCGI requests marked by a reserved parameter, plus an event socket
- **Pros:** push-based revocation.
- **Cons:** a hidden way into the runtime and a second channel.
- **Verdict:** rejected by the project owner.

### Option: A control socket per worker for everything
- **Pros:** one framed protocol for questions, health and events.
- **Cons:** PHP would need to run outside a request, and a passing health check would not prove the worker serves.
- **Verdict:** rejected by the designer.

### Option: Endpoints of the apps themselves
- **Pros:** one path, the same as visitors'; health checks prove real serving.
- **Cons:** polling instead of push.
- **Verdict:** chosen.

## Dependencies

- The PHP foundation's pool, FastCGI codec and site directory (SpeedySpec/speedyspec-server#4).
- The server core's release machinery, keeper and cluster mode.
- The admin API implementing the gate endpoints, and every PHP app's pipeline producing the new manifest fields.

## References

- [ADR 0022](../../adr/0022-fastcgi-over-a-unix-socket.md), [ADR 0025](../../adr/0025-supervised-php-worker-pools.md),
  [ADR 0026](../../adr/0026-a-writable-site-directory.md)
- [SpeedySpec/speedyspec-server#6](https://github.com/SpeedySpec/speedyspec-server/pull/6),
  [#10](https://github.com/SpeedySpec/speedyspec-server/pull/10),
  [#11](https://github.com/SpeedySpec/speedyspec-server/pull/11)
- [The server core](2026-11-17-speedyspec-server-core.md)
- [The PHP foundation](2026-11-18-php-foundation.md)
- Source spec: `2026-11-19-speedyspec-php-runtime-and-admin-gate-design.md`
- Source plan: `2026-11-19-speedyspec-php-runtime-and-admin-gate.md`
