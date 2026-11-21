# SpeedySpec PHP runtime and admin gate design

Date: 2026-11-19
Status: draft for Jacob's review (builder check done)

This changes two things in the server core design (`2026-11-17-speedyspec-server-core-design.md`) and the PHP
foundation design (`2026-11-18-speedyspec-php-foundation-design.md`):

1. The embedded PHP over FastCGI becomes its own component, the **PHP runtime**, released and upgraded by the server
   like any app.
2. The server gains the real **admin gate**: sign-in and per-page access for the admin, the public pages' conditions,
   and per-app maintenance.

It replaces project 2 ("Sign-in and the admin gate") of the server core design, and reverses one of its constraints:
**the admin API is not Rust compiled into the server.** It is a PHP app (Jacob, 2026-11-18).

## Intent

**The server should change rarely, ideally not at all.** It upgrades everything else: the admin site, the admin API,
the front-end API, the blog, and now the PHP runtime they run on. So the server holds no PHP and no WordPress
knowledge. It serves static files from memory, routes requests, passes PHP requests to runtime processes over FastCGI,
supervises those processes, and gates the admin.

**The PHP runtime is where the work happens.** Over time its Rust WordPress API replaces WordPress's PHP API, area by
area, as the Hook API has. That needs frequent releases, and none of them should touch the server.

**The admin API is PHP** because much of the admin depends on plugins that add to and change admin areas. It is its
own app, apart from the front-end API, because it acts with elevated privileges. It is still publicly reachable: the
admin site's JavaScript calls it from the browser. It is a new internal REST API, not WordPress's `wp-json` REST API.

**The admin is unauthorized by default.** Nothing about the admin reaches a visitor who isn't signed in, except
the sign-in page, the maintenance page while maintenance is on, and the install page while the site isn't installed.
A plugin endpoint that forgets its own permission check must still be behind the gate.

## Goals

- The server binary builds and runs with no PHP in it. The runtime is a separate binary.
- The runtime is a release: downloaded, verified, kept, switched and rolled back by the same machinery as every app,
  with no server change for a new runtime version.
- A small, versioned contract between the server and the runtime, so each can refuse the other cleanly instead of
  misbehaving.
- Each PHP app (admin API, front-end API, blog) runs in its own pool of runtime processes, with its own site
  directory and environment, so each can have its own credentials.
- The admin gate: every admin site file and every admin API path requires sign-in, except the public entries the
  releases declare, each with its condition.
- Per-page access for the admin site's page files, decided by the admin API.
- Sign-out takes effect at once. Every other loss of access takes effect within about a second.
- Per-app maintenance, turned on by the admin API, by the server's own upgrades or by an operator. The maintenance
  page never needs PHP. An app whose health checks fail answers a plain server error instead (an outage is not
  maintenance; Jacob, 2026-11-20).

## Non-goals

- **Live connections.** WebSocket and WebTransport, server push and the PHP bridge are the next design. See
  "Transport roadmap".
- **The admin API's own endpoints**, beyond the few the gate calls (below). Building the admin API is its own work.
- **Converting admin screens**, the front-end API's features and the blog's features. Other sessions own them.
- **Routing in the server** (the PHP foundation's "What comes next"). Unchanged and still next for the runtime.
- **Running stock WordPress.** Neither the server nor the runtime ever does.
- **The server deciding permissions.** It enforces the admin API's answers; it never works out who may do what.

## Anti-goals

- A server release needed because a plugin added an endpoint, a runtime release shipped, or the admin API changed.
- Any admin file or admin API response reaching a visitor who isn't signed in, other than the declared public
  entries whose condition holds.
- A maintenance page that needs PHP to show.
- The server's internal calls reachable by visitors.
- A second channel between runtime and server, such as a separate event socket, beside FastCGI.
- Contracts in the same crate as their implementations.

## Constraints

- All of the server core design's constraints still hold, except "The admin API is Rust only", which this design
  reverses.
- Rust 1.99, edition 2024. Unix only: Linux x86_64 and aarch64 (musl, static), macOS arm64 for development.
- The PHP foundation's pool behaviour is kept as designed: one request per worker, poison and recycling, `prepare`,
  `activate`, `retire`, and the site directory.
- TLS stays on `ring`.

## Approaches

How the server and the runtime talk, beyond visitors' requests: the gate's questions, health, and revocation.

- **A. Internal FastCGI requests with a reserved parameter, plus an event socket.** The server's questions travel as
  FastCGI requests marked by a parameter only the server sets; revocations are pushed over a Unix socket the server
  listens on. Rejected by Jacob: it is a hidden way into the runtime, and the separate event socket is a second
  channel for little gain.
- **B. A control socket per worker for everything.** A framed protocol beside FastCGI for questions, health and
  events. Rejected: the runtime would need a second way to run PHP outside a request, and a passing health check
  would no longer show that the worker can serve.
- **C. Endpoints of the apps themselves (chosen).** The gate's questions, the revocation feed and the health checks
  are ordinary endpoints of the PHP apps, reached over the same FastCGI path as every visitor request. The server
  polls the revocation feed once a second instead of being pushed to. A poll is used rather than a request held
  open, because each worker serves one request at a time and a held request would tie up a worker. When live
  connections arrive, that project can replace the poll with a push.

## Design

### The PHP runtime as its own component

**Two binaries.** The server's `php` Cargo feature and its `php-worker` role are removed. The PHP crates move
behind a new binary crate, `speedyspec-php-runtime`, in the same workspace (my call: the same workspace keeps the
FastCGI and contract crates in one place; the CLI's copies keep their paths). The server depends only on the FastCGI
client and the contract crate.

**A new app kind, `runtime`.** A runtime release is like a server release: one binary per target triple, with its
manifest embedded and printed by `speedyspec-php-runtime manifest`. It is downloaded, verified, kept, switched and
rolled back like any release. Configuration:

```toml
[app.php-runtime]
kind = "runtime"
source = "https://releases.example.com/php-runtime/index.json"
keys = ["RWQ..."]
keep = 3
update = "manual"

[app.admin-api]
kind = "php"
runtime = "php-runtime"        # which runtime app runs this app; may be left out when only one is configured
site_dir = "/var/lib/speedyspec/sites/admin-api"
source = "https://releases.example.com/admin-api/index.json"
keys = ["RWQ..."]
```

- Each `php` app's release names the runtime versions it works with as `"runtime": "^1.4"` in its manifest's
  `php` section. This means "whichever runtime app this app is configured to run on", so the operator's choice of
  app ids never matters (my call). The existing requirement check treats it like a `requires` entry on that app,
  and refuses a switch that would break it. It is never ignored as "not configured": a `php` app always has a
  runtime.
- **Switching the runtime** starts new pools from the new runtime binary for every PHP app that uses it. Each pool
  is prepared, then all are activated in one step and the old pools drain. If any pool fails to prepare, nothing
  switches. It is a group switch, so every replica in a cluster switches together. A runtime switch never needs a
  new server.
- **A new runtime starts on trial, and rolls back on its own** (Jacob's choice).
  - The previous runtime's version stays recorded in the state file as the confirmed one, with the new one marked on
    trial, until `runtime.trial_period` (default 2 minutes) passes.
  - If any app on the new runtime turns unhealthy during the trial (its health checks reach `health.fail_after`,
    below), the server switches every app back to the confirmed runtime and marks the new version failed. It never
    switches to that version again on its own; an operator can retry it by hand.
  - With `[cluster]` configured, the leader decides the rollback, as in a server trial, and one replica's failure
    reverts them all. Without it, each replica rolls back only itself (see "Replicas").
  - Maintenance turns on only if the confirmed runtime fails its health checks too.
  - After the trial, the new runtime becomes the confirmed one, and later health failures lead to maintenance only.
- **Immutable mode:** the runtime is baked into the image with a `path`, like every app.

**The server–runtime contract.** It lives in its own crate, `speedyspec-runtime-contract`, which both binaries
depend on. It has a major version, and covers only:

1. **Starting a worker:** the program (the runtime release's binary), its arguments (`worker --spec <json>`), and the
   spec (app id, socket path, release directory, site directory, INI, bindings, and the contract version).
2. **Requests:** FastCGI 1.0 on the worker's private Unix socket (0700 directory, 0600 socket), as built in the PHP
   foundation.
3. **The per-request report:** the NUL-prefixed `FCGI_STDERR` record with memory, request count and the poisoned
   flag.

A runtime's manifest declares which contract majors it speaks. The server refuses to prepare a pool from a runtime
that doesn't speak its own major, and names both versions in the error. A worker refuses a spec whose contract major
it doesn't speak, and exits with a clear log line. Adding a field is a minor change that the other side ignores.
Anything else is a new major.

**Per-app credentials.** Each PHP app already has its own site directory. Each app also gets its own environment
(`env` and `env_file` in its config, beside the site directory's `.env`). That is how the admin API is given
stronger credentials than the public apps, for example a database user that can write and a writable `wp-content`,
while the front-end API and the blog get only what they need. Which credentials each app gets is the operator's
choice. The server only keeps them apart (my call).

### The admin gate

**Which app answers.** One PHP app is named as the gate's authority:

```toml
[gate]
app = "admin-api"
```

Every mount with `access = "signed-in"` is checked through the gate. `deny` and `dev-allow-all` remain for
development and testing, as before. The old built-in `/admin/api/` mount of the `server` app is removed: the admin
API is an ordinary `php` app with its own mount, and the `server` app has no mounts.

**Releases declare their access rules.** Every file and path behind a gated mount needs sign-in unless its release
says otherwise. In the manifest, the admin site's release lists its rules under `static.access` and the admin API's
release lists them under `php.access`:

```json
"access": [
    { "match": "", "public": "always" },
    { "match": "signin", "public": "always" },
    { "match": "signin/**", "public": "always" },
    { "match": "install/**", "public": "not-installed" },
    { "match": "components/{page}/**", "page": "{page}" },
    { "match": "contracts/{page}/**", "page": "{page}" }
]
```

- **Matching** (my call): rules are tested against the request path relative to the mount, *before* the static
  resolver tries `index.html` or `.html`. They are tested in list order, and the first match wins. The mount's own
  root (an empty relative path) matches `""`. Matching before resolution is what makes 401 the same for paths that
  exist and paths that don't. The admin site's release must therefore list the paths visitors use (`""`,
  `signin`, `signin/**`), not the files those paths resolve to.
- `public: "always"` means anyone may load it. `"not-installed"` means anyone, but only while the site isn't
  installed. `"maintenance"` means anyone, but only while maintenance is on for the app.
- **A conditional entry whose condition doesn't hold answers 404 for everyone**, signed-in users included: it is
  disabled, not just private.
- **`page`** marks a page's own files. Only a signed-in user whose session answer lists that page may load them.
  `{page}` captures the page id (`<namespace>/<name>`, two segments, as the admin site names its files).
- Anything else needs sign-in.
- The server reads these rules when it switches to the release, so a new public or page entry is a release, never a
  server change.

**What visitors get**, kept from the admin site's proxy:

- not signed in: **401** for every protected path, whether or not it exists;
- signed in, but the page isn't in their session answer: **403**, whether or not it exists;
- a disabled conditional entry: **404**;
- protected responses carry `Cache-Control: private, no-store` (my call), so no shared cache keeps them.

### The gate's endpoints on the admin API

The server calls these on the gate app, over the same FastCGI path as visitor requests. Their paths come from the
admin API's manifest, so the server hard-codes none of them:

```json
"gate": {
    "version": 1,
    "site": "/v1/gate/site",
    "session": "/v1/gate/session",
    "changes": "/v1/gate/changes"
}
```

- **Site state:** `GET site` → `{ "installed": bool, "maintenance": bool }`.
- **Session:** `GET session` with only the visitor's `Cookie` header →
  `200 { "user": { "id": <positive integer> }, "pages": ["<namespace>/<name>", ...] }` when the cookie is signed
  in, and `401` otherwise. A `200` with a malformed body is read as signed in with no pages (page files fail
  closed). Anything else (other statuses, an oversized or unparseable body, a timeout) counts as not signed in,
  as the admin site's proxy does.
- **Changes:** `GET changes?since=<cursor>` → `{ "cursor": "<opaque>", "users": [<id>, ...] }`, or
  `{ "cursor": "<opaque>", "reset": true }` when the admin API can't say which users changed (all remembered answers
  are dropped). An empty `since` gets the current cursor with no users.
- **`version`** is the gate contract's major version. The server refuses to switch to an admin API release whose
  gate version it doesn't speak.

**Where the server's calls go** (my call): the gate and health paths are relative to the app's root, not to any
mount. The server builds those requests itself as if from the app's first configured mount, with that mount's host
(or `localhost` for `*`). The request comes from no client address, and no visitor headers are copied except the
session call's `Cookie`.

**Internal calls have their own worker** (my call). Each PHP pool keeps one worker beyond `workers.max`, reserved
for the server's own calls (health probes, `site`, `changes`, `session`), so a pool saturated by visitors can't
fail its own health checks. Session checks for cookies with no remembered answer are capped at
`gate.max_pending` concurrent calls (default 16). Beyond the cap, a request gets 503 with `Retry-After` rather than
another PHP call. So a flood of made-up cookies can't fill the pool, and it isn't counted as a health failure.

**Visitors can't reach the server's calls.** The `changes` path is internal. The server answers 404 to any visitor
request for it and never forwards one. `site` and `session` are also ordinary endpoints the admin site may call from
the browser, for example to show the sign-in page. How the admin API works out its answers (the Rust WordPress API,
with PHP plugins able to override) is the runtime's business.

### Remembering answers, and revocation

- **Only positive session answers are remembered.** They are keyed by the SHA-256 of the whole `Cookie` header
  (never the cookie itself), with the user id and pages, for `gate.session_ttl` (default 60 s). A "no" is asked
  again every time, so a fresh sign-in is never shut out by an old answer. Concurrent checks for one cookie share
  one call. A check not answered within `gate.timeout` (default 5 s) fails closed.
- **The site state** is remembered for `gate.site_ttl` (default 2 s).
- **Sign-out takes effect at once.** When an admin API response to a request clears a cookie that the request
  carried (a `Set-Cookie` with `Max-Age=0` or an expiry in the past), the server forgets the remembered answer for
  that request's `Cookie` header before passing the response on.
- **Every other change takes effect within about a second.** The server polls `changes` every `gate.poll`
  (default 1 s) and forgets every remembered answer for the users it names, or all of them on `reset`. This covers
  changes the server never sees, such as WP-CLI, scheduled tasks, or one administrator removing another.
- **If the poll fails,** the server forgets every remembered answer and asks again per request until the poll
  succeeds (fail closed, my call). Repeated failure also shows up in the admin API's health (below).
- In a cluster, every replica polls and remembers on its own.

### Maintenance

Maintenance is **per app**. For each app, the server combines three planned sources, and maintenance is on when any
of them is. A fourth, health, is not maintenance: it makes the app answer a server error (below).

1. **The admin API's site state** says `maintenance: true`, for example during a plugin update. This applies to
   the apps named in `[gate] maintenance_apps` (default: every app; my call), but never to the admin itself (the gate
   app and every app behind a `signed-in` mount): admins sign in to turn it off, or use the CLI. The admin goes into
   maintenance only for a moment, while one of its own releases migrates or an operator turns it on; when the admin
   API is down it answers an error instead, and fixing that needs the CLI anyway (Jacob, 2026-11-20).
2. **The server's own upgrades.** A PHP app's release may declare that switching to it needs maintenance and a
   migration step: `"switch": { "maintenance": "site", "migrate": "/v1/migrate" }`. `"site"` means every app in
   `[gate] maintenance_apps`, and `"self"` means only this app, so no app ids are written into releases (my call).
   The server then:
   1. prepares the new pool;
   2. turns maintenance on;
   3. runs the migration;
   4. switches;
   5. turns maintenance off.

   **The migration** (my calls) is an ordinary internal `POST` to the new pool on its reserved worker. It has its
   own `switch.migrate_timeout` (default 30 minutes) in place of `request_timeout`. A 2xx answer is success;
   anything else, or a timeout, is failure. The server–runtime contract stays as it is.

   **Without `[cluster]`,** each replica runs the migration when it switches. So a migration must be safe to run
   more than once and from several replicas at the same time: the PHP app guards it (a database lock) and answers 2xx
   when it has already been applied.

   **With `[cluster]`,** only the leader runs it, once, against the shared database. That adds a "migrated" step to the
   group switch: the leader records it in the shared state after "prepared", and before any member activates.

   **If the migration fails,** the switch is abandoned, the old release stays live, and maintenance stays on.
   That's logged and shown in the release status (a half-migrated database must not take traffic). The operator
   then runs `speedyspec-server release retry-migration <app>`, or clears it with
   `speedyspec-server maintenance clear <app>`.

   **Rolling back across a migration** is allowed and instant, as every rollback is. A release that migrates must
   leave the database usable by the release it replaces (add first, remove in a later release). This rule is
   written into the release pipeline's documentation rather than enforced by the server.
3. **An operator:** `speedyspec-server maintenance on|off [<app>...]`. It goes in the state file, so it survives
   restarts. Without `[cluster]` it applies to that replica only. With `[cluster]` it applies to the whole cluster:
   followers refuse the command, as they refuse other release commands, and pick it up through the existing state
   poll. For a site-wide switch across independent replicas, the admin API's own maintenance (source 1) is the one
   to use, since it lives in the shared database. `maintenance off` clears only the operator's own
   source; `maintenance clear` also clears a failed migration's (my calls).
4. **Health.** Each PHP app's manifest names its probes (`"php": { "health": { "live": "/healthz", "ready":
   "/readyz" } }`). The server calls both on the app's pool every `health.interval` (default 2 s), as internal
   paths that visitors can't reach. A non-200 answer, or no answer within `health.timeout`, is a failure.
   `health.fail_after` failures in a row (default 3) mark the app unhealthy, and `health.pass_after` passes in a row
   (default 3) end it, so one slow answer doesn't flap it. An unhealthy app answers every visitor a plain
   `503 service unavailable`, with no `Retry-After` and no maintenance page, without reaching PHP. Failing health is
   exceptional and shouldn't happen, so it shows as an error rather than as planned maintenance (Jacob, 2026-11-20).
   An app both in maintenance and unhealthy shows its maintenance page.

**Health is per replica** (my call). Each replica checks its own pools. An unhealthy pod answers only its own
visitors with the error and reports not-ready on `/readyz`, so Kubernetes stops sending it traffic while the healthy
pods carry on. When every pod is unhealthy at once (a shared database is down), the load balancer has no ready pod and
answers its own 5xx, which is the answer this design wants for an outage (Jacob, 2026-11-20).

**The gate app's health covers its admin.** When the gate app is unhealthy, every mount that the gate checks answers
the same error: the admin site can't work without its API.

**What visitors get while an app is in maintenance:**

- **The admin site and the blog:** the maintenance page, with status 503 and `Retry-After`. Signed-in
  administrators included. The page is one self-contained HTML file (inline CSS and images, no scripts) that the admin
  site release names in its manifest (`"static": { "maintenance_page": "maintenance.html" }`). It is held in memory
  and served as the response body to the maintenance request itself, so it needs no other file, no condition and no
  PHP (my calls). Each app's config names which static app's page it shows (`maintenance_page = "admin"`).
- **The admin API and front-end API:** 503 with `Retry-After` and a small JSON body.
- **Probes and internal calls** are still made, so maintenance can end by itself.

### Replicas

Replicas never need to talk to each other: requests are independent, so several servers can run behind a load
balancer with no `[cluster]` configured (Jacob, 2026-11-19). When `[cluster]` is configured, the existing cluster
mode stays and these features use it ("if they can communicate, let them communicate").

| | Independent replicas (no `[cluster]`) | Cluster mode (`[cluster]`) |
|---|---|---|
| Switches, runtime switches | each replica on its own | the existing group switch: every replica together |
| Migration | every replica runs it; the PHP app makes it safe to repeat | the leader runs it once; members wait for "migrated" |
| Runtime trial rollback | each replica rolls back only itself | one replica's failure rolls every replica back |
| Operator maintenance | that replica | every replica, through the shared state |
| Admin API maintenance, health, sign-in answers, revocation | per replica in both modes | per replica in both modes |

### Transport roadmap

1. **Now:** the admin site talks to the admin API over REST only. An open browser learns that its access was
   revoked at its next REST call, which is refused, and the admin returns to sign-in.
2. **Next, the live-connections design:** WebSockets are added alongside REST, for a more responsive admin and an
   offline mode. The server can then push revocations to an open browser, and the `changes` poll can become a push.
3. **After that:** WebTransport is tested and then supported.

### What this changes in existing code

- `AppKind` gains `Runtime`. The `php` feature, the `php-worker` role and the PHP crates leave the server binary
  crate. `EmbeddedPhpPool` starts the runtime release's binary instead of `current_exe`.
- The `server` app loses its built-in `/admin/api/` mount and `AdminApiPlaceholder`.
- `AccessCheck` gets a `gate` implementation. The `AccessRequest` it sees gains whatever the manifest rules need
  (path within the mount and the release's access rules). Its identity output stays as it is, and nothing new is
  forwarded to PHP (the admin API checks its own permissions on every call).
- The CLI's copies (`copied-from.json`) keep their paths. Re-syncing them is the CLI session's job, as before.

## Testing

- **Runtime release:**
  - a fixture runtime binary is downloaded, verified, switched and rolled back;
  - a runtime switch moves every PHP app using it to new pools with no failed requests;
  - a runtime whose pool fails to prepare switches nothing;
  - a runtime that turns unhealthy during its trial rolls every app back and is marked failed; one that stays
    healthy is confirmed;
  - a contract-major mismatch is refused by both sides with both versions named.
- **No PHP in the server:** the server binary builds and passes its suite with no static PHP present.
- **Gate rules:**
  - every rule kind against a fixture admin site and admin API;
  - 401, 403 and 404 for existing and missing paths alike;
  - a conditional entry turns on and off with the site state.
- **Session answers:**
  - only positive answers are remembered, and concurrent checks share one call;
  - timeouts and malformed answers fail closed.
- **Revocation:**
  - a sign-out response drops its remembered answer before the response reaches the client;
  - a `changes` entry drops a user's answers within one poll;
  - `reset` drops everything;
  - a failing poll makes every check ask again.
- **Internal paths:** a visitor request for `changes` or a health path gets 404 and never reaches PHP.
- **Maintenance:**
  - each of the three sources turns it on and off per app, and per-app isolation holds;
  - an unhealthy app answers a plain 503 without the page, and an app also in maintenance keeps its page;
  - the gate app's health covers the admin site;
  - the page and the unhealthy answer are served with PHP busy;
  - a failed migration leaves maintenance on and the old release live;
  - `fail_after` and `pass_after` work without flapping.
- **Cluster mode:** a runtime switch and a maintenance-requiring switch land on every replica together.
- **Independent replicas:** two servers with separate data directories and no `[cluster]` each switch, migrate,
  roll back and keep maintenance on their own, with nothing passing between them.
- CI on Linux musl and macOS arm64, as today.

## Dependencies

- The admin API implements the gate endpoints and the health probes. The admin site's release declares its access
  rules and the maintenance page in its manifest, which replaces the proxy's `public-files.json` and
  `shared-files.json` when served by this server.
- The PHP apps' release pipelines produce archives with the new manifest fields. The runtime's pipeline produces one
  binary per target triple, as the server's does.

## Left to the builder

- Exact manifest key names and validation messages beyond the shapes above.
- The internal crate layout of `speedyspec-php-runtime` and how the server's pool code splits from the runtime's.
- How the server reads cookie deletion in `Set-Cookie`, within the rule above.
- The 503 JSON body's fields and the log event names.
