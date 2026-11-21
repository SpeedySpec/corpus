# speedyspec-server: deferred findings

Repository: [SpeedySpec/speedyspec-server](https://github.com/SpeedySpec/speedyspec-server).

## Where these came from

### The data-and-options port

The data-and-options port to Rust ran as eight plans, merged as #12 to #19 in November 2026. A fresh reviewer read
each plan's whole branch. Findings graded Critical or Important were fixed before merge. These are the Minor ones,
deferred.

Seven of the original 72 were fixed afterwards in #25 (SpeedySpec/speedyspec-server#25), so they are not listed
here:
- `SqlCache`'s duplicate `add` inside a PostgreSQL transaction (not a bug: the store already uses savepoints).
- SQLite writing to the file on every open.
- SQLite refusing MySQL's `LIMIT o, 18446744073709551615`.
- `FileCache` locks inside the group directory, and reads that removed expired files without the lock.
- `KvCache` increment and decrement races.
- A failed `settings_put` being lost silently.
- No cross-process test of transients on the persistent object cache.

One more is left out: it concerned the wording of a review ledger that has since been deleted, not the code.

Recorded 2026-11-21. Each finding was written at review time; where later code changed part of one, the text says
what is still true.

### The server's own plans

The server itself was built from seven plans in November 2026: the server foundation (merged before #1), managed
releases (#1), the keeper and socket handoff (#2), the cluster (#3), the PHP foundation (#4), the PHP runtime and admin
gate (#6), and the release pipeline (#11). A fresh reviewer read each plan's whole branch. Findings graded Critical or
Important were fixed before merge; the Minor ones were deferred, and are listed in the files below. A few more were
found on 2026-11-21 while releasing server 0.1.1, and are recorded with them.

Recorded 2026-11-21. Each finding was checked against main at 0d398a3. Where later code changed part of one, the text
says what is still true. Findings fixed or ruled on since are left out and listed below.

Left out, because they were fixed or ruled on:

**Fixed:**

- A session check in flight across a sign-out or changes poll could re-insert a stale answer: fixed with the forget
  counter in #10 (SpeedySpec/speedyspec-server#10).
- `acquire_internal` could respawn a worker in a tight loop when spawning failed fast: start backoff added in #10.
- The default `maintenance_apps` included the gate app, so site maintenance could not be ended from the admin: since
  #10 the gate app and every app behind the gate are never put in site maintenance (signed-in admins are exempt).
- Workers inherited the server's whole environment: since #6 a worker gets only an allowlist of the server's
  environment plus its app's own (`WORKER_ENV_ALLOWLIST`). The quotes part of that finding is still listed.
- A request routed to the old handler just before a retire got 503: since #6 a draining pool keeps lending idle
  workers for a grace period (`DRAIN_GRACE`, 1 s).
- PHP callback values converted PHP to Rust to PHP: #5 passes zvals through. The eval part of that finding is still
  listed.
- Connect and params write were not under the request deadline, and a leftover `ended` variable: both fixed in passing
  by the PHP foundation's I4 restructure, before #4 merged.
- `docs/cluster.md` read as absolute fencing: it now says "The fencing is not absolute" and explains the late-write
  window (#3).
- Release pull requests opened with `GITHUB_TOKEN` did not run CI before merge (release pipeline M9): since #27
  release-please uses `RELEASE_TOKEN`, and CI ran on release pull request #31. The README still describes the old
  token; that is listed in release-pipeline.md.
- Part of the control socket finding (managed releases M6): the socket is now bound under a staging name and made
  owner-only before it appears at its real path. What remains is listed in keeper-and-handoff.md.
- Part of release pipeline M5: the release build now runs `check-no-php-in-server.sh`. What remains is listed.

**Ruled by the owner:**

- Runtime trials need `keep >= 2` for the runtime app: the config must not check it; prune now keeps a trial's
  fallback instead (#10).
- Runtime trials use wall-clock time, so a clock step can shorten or lengthen a trial: accepted, as long as workers
  are recreated and no zombies are left.

**Not carried:**

- "Missing tests noted by the reviewer beyond those added in the fix pass" (cluster plan): the review did not name the
  tests, so there is nothing specific to record.

## Index

| File | Area | Findings |
|---|---|---|
| [data-layer.md](data-layer.md) | `speedyspec-data`, `-data-query`, `-data-uow`, `-data-mysql` | 9 |
| [wpdb.md](wpdb.md) | `speedyspec-wp-db`, `-wp-db-php` | 11 |
| [cache-and-object-cache.md](cache-and-object-cache.md) | `speedyspec-cache-*`, `-php-serialize`, `-wp-object-cache*` | 11 |
| [options-api.md](options-api.md) | `speedyspec-wp-options*` | 10 |
| [sqlite-engine.md](sqlite-engine.md) | `speedyspec-data-sqlite`, `-data-translate` | 6 |
| [postgres-engine.md](postgres-engine.md) | `speedyspec-data-postgres` | 6 |
| [nosql-and-direct-stores.md](nosql-and-direct-stores.md) | `speedyspec-data-valkey`, `-data-redb`, `-data-file`, `-cache-file` | 5 |
| [persistent-cache-transients-network-options.md](persistent-cache-transients-network-options.md) | `speedyspec-wp-options*`, `-wp-object-cache` | 7 |
| [http-and-static.md](http-and-static.md) | `speedyspec-server-presentation`, `-server-static`, `-server-domain`, `-server-app` | 7 |
| [releases.md](releases.md) | `speedyspec-server-app`, `-server-infra-fs`, `-server-infra-http`, `-server-infra-minisign` | 7 |
| [keeper-and-handoff.md](keeper-and-handoff.md) | `speedyspec-server`, `-server-handoff` | 15 |
| [cluster.md](cluster.md) | `speedyspec-server-app`, `-server-infra-fs`, `-server-infra-kubernetes`, `deploy/kubernetes` | 10 |
| [php-runtime-and-pool.md](php-runtime-and-pool.md) | `speedyspec-php`, `-php-runtime`, `-server` (`php/`) | 14 |
| [admin-gate-and-maintenance.md](admin-gate-and-maintenance.md) | `speedyspec-server-gate`, `-server-presentation` | 3 |
| [release-pipeline.md](release-pipeline.md) | `.github/workflows`, `scripts/release-*`, `README.md` | 10 |
| [tests-and-deploy.md](tests-and-deploy.md) | `speedyspec-server` and `-server-infra-fs` tests, `deploy/kubernetes` | 4 |
