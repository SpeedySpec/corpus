# ADR-007: Shared stores have file, Redis and database implementations

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** both (the project owner chose the three implementations; the designer set the defaults beyond the page cache)
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-008](008-background-job-runner-in-the-runtime.md), [ADR-009](009-database-port-requirements.md), [ADR-010](010-firewall-and-login-security.md), [ADR-011](011-page-cache-coarse-purge-first.md), [ADR-012](012-redirects-sitemaps-robots-feeds.md)

---

## Context

The features keep state: the page cache, firewall counters and bans, redirect rules, the 404 log and the image job
queue. Several runtime pools touch the same state. The admin API's pool, for one, purges cache entries that the blog's
pool serves. Replicas must work on their own behind a load balancer, and use the cluster's shared state when
`[cluster]` is configured.

### Why this matters

If each feature chose its own storage, pools would disagree: a purge in one pool would not reach the cache another
pool reads. Tying a store to one backend would force Redis on a single small host, or leave a cluster without shared
state.

---

## Decision

Every shared store is a contract with file, Redis and database implementations, chosen in configuration, plus an
in-memory implementation for tests.

### Rules

1. Each store (page cache, firewall counters and bans, redirect rules, 404 log, job queue) is a contract in its own
   package.
2. Each contract has file, Redis and database implementations, and an in-memory one for tests.
3. The page cache defaults to files, in a directory every pool of the app can reach.
4. Firewall counters and bans, redirect rules and the 404 log default to the database, through the ported database
   layer.
5. With `[cluster]`, Redis or the database is the shared choice; files are per replica.
6. A store that is down must not take the site with it. The feature degrades and logs a warning.

### What this looks like

✅ **Correct**

```rust
// The firewall's port knows nothing of the backend; any Caching Store backs it.
pub trait CounterStore {
    fn hit(&self, key: &str, window: Duration) -> Result<u64, FirewallError>;
}

let counters = CacheCounterStore::new(cache.clone(), clock.clone()); // memory, file, key-value or SQL
```

❌ **Not permitted**

```rust
// A feature hard-wired to one backend, unreachable from other pools or replicas.
static COUNTERS: LazyLock<Mutex<HashMap<String, u64>>> = LazyLock::new(Default::default);
```

### Patterns

- **Strategy:** the store backends, chosen in configuration.
- **Repository:** each store's contract.

---

## Consequences

### Positive

1. A single host runs with files and no extra service.
2. A cluster shares state through Redis or the database with a configuration change.
3. Tests run against the in-memory implementation, with no services.

### Negative

1. Three real implementations to keep for each store.
   **Mitigation:** shared conformance suites (for example `speedyspec-cache-conformance`) run every backend through
   the same checks.
2. With files and no shared directory, each replica has its own cache and its own counters.
   **Mitigation:** configure Redis or the database under `[cluster]`.
3. A store outage weakens the feature that uses it. The firewall, for one, lets requests through.
   **Mitigation:** a warning is logged.

### Trade-offs

**We prioritise:**
- replicas that work alone;
- backend choice by configuration, not code.

**We accept:**
- the cost of keeping three implementations;
- weaker protection while a store is down.

---

## Fleet impact

- **speedyspec-server (repository):** each store's contract and backends. The firewall keeps counters and bans in
  the Caching Store (`speedyspec-cache`), whose backends include memory, file, key-value (Valkey) and SQL.
- **speedyspec-php-runtime:** reads the store choice from configuration and gives every pool of an app the same
  store.
- **Blog, front-end API, admin API:** share the page cache across pools. Unverified until the cache exists.
- **speedyspec-cli:** unverified; it may need to write the store configuration.
- **speedyspec-server (binary), admin site:** unaffected.

---

## Enforcement

### Automated

In SpeedySpec/speedyspec-server, run by CI's `test` job on every pull request (cargo exits 101 on failure):

- `cargo test -p speedyspec-cache-memory --test conformance` and `cargo test -p speedyspec-cache-sql --test
  conformance` run the shared Caching Store suite.
- `cargo test -p speedyspec-firewall-infra-cache --test backends` runs the firewall's counters and bans on the
  memory, key-value and file caches (`counts_within_window_and_resets_in_next_on_every_backend`).
- `cargo test -p speedyspec-firewall-app` checks `store_errors_fail_open`.

None has been seen failing. That each store has all three implementations is enforced by review only.

### Review checklist

- [ ] The store's contract is in its own package.
- [ ] File, Redis and database implementations exist, plus an in-memory one.
- [ ] The default backend matches rules 3 and 4.
- [ ] A store error degrades the feature and logs; it never fails the request.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** the firewall's counters and bans built on the Caching Store, merged in
  SpeedySpec/speedyspec-server#20.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
