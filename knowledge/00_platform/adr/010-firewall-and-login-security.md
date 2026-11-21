# ADR-010: The firewall runs first; login protection lives in the auth API

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the designer
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-004](004-runtime-request-stages.md), [ADR-005](005-core-hooks-stay.md), [ADR-006](006-php-callback-registry-at-worker-start.md), [ADR-007](007-shared-stores-file-redis-database.md)

---

## Context

Wordfence and Solid Security filter requests before WordPress loads, and protect sign-in. Core already has the
`authenticate` filter, `wp_login_failed`, application passwords, XML-RPC, `?author=N` lookups and the REST users
endpoint. SpeedySpec wants the same protection without loading PHP for a request it is about to refuse.

### Why this matters

A firewall that runs inside PHP pays for PHP on every hostile request, which is the traffic a site least wants to pay
for. A firewall that can lock out the site's owner, or that fails closed when a store is down, turns a safety feature
into an outage.

---

## Decision

Request rules run as stage 1 of the runtime (ADR-004). Login protection belongs in the ported auth API, counting
failures through `wp_login_failed` and enforcing lockouts in `authenticate`.

### Rules

1. Request rules cover IP allow and block lists, rate limits, and patterns over the path and query string. Body
   patterns wait for ADR-004's body question.
2. An allow-listed client is never blocked or counted.
3. Switches close user enumeration and XML-RPC. Only HTTP Basic (application passwords) or a logged-in cookie with a
   REST nonce exempts a request from enumeration blocking.
4. Counters and bans sit behind store contracts (ADR-007).
5. The admin API runs in an admin scope with no rate limit and no IP bans. Lists, patterns and login lockout still
   apply.
6. IPv6 clients are counted and banned per /64 network.
7. Windows, bans and lockouts are capped at thirty days.
8. The firewall fails open: a store error, unreadable settings or a missing client address lets the request through
   and logs a warning.
9. Settings are `speedyspec_firewall_*` options, read through the ported options API. The firewall is off by default.
10. Kept or added hooks: `authenticate`, `wp_login_failed`, `xmlrpc_enabled`, `rest_endpoints`, and a hook for
    adding request rules.

### What this looks like

✅ **Correct**

```rust
// Rules in a fixed order; a failing rule is logged and treated as Continue (fail open).
pub fn check(&self, facts: &RequestFacts) -> Verdict {
    for rule in &self.rules {
        match rule.check(facts) {
            Ok(Verdict::Continue) => {}
            Ok(verdict) => return verdict,
            Err(error) => self.log.warn(&format!("{}: {error}", rule.id())),
        }
    }
    Verdict::Continue
}
```

❌ **Not permitted**

```rust
// Fails closed, and bans in the admin scope.
Err(_) => return Verdict::Block { status: 503, retry_after: None, reason: "store down".into() },
// ...
rules.push(Rc::new(RateLimitRule { limit, counters, bans })); // regardless of FirewallScope::Admin
```

### Patterns

- **Chain of Responsibility:** the rules, in order; `Allow` ends the chain and the first `Block` wins.
- **Specification:** IP sets and pattern rules.
- **Strategy:** the store backends.
- **Observer:** the `wp_login_failed` and `xmlrpc_enabled` callbacks.
- **Adapter:** `FirewallStage` wraps the firewall as a native stage.

---

## Consequences

### Positive

1. Blocked and rate-limited requests are refused in Rust, with 403 or 429 and `Retry-After`.
2. The owner can't lock themselves out of their own admin.
3. A typo in a setting can't ban a client for good.

### Negative

1. Until an auth port exists, lockouts are enforced at the request stage, by path, for `POST`s to the configured
   sign-in paths, not in `authenticate`.
   **Mitigation:** move enforcement into `authenticate` when the auth port and the admin API's sign-in endpoint exist.
2. A store outage turns the firewall off.
   **Mitigation:** a warning is logged.
3. Neighbours in one IPv6 /64 share a limit, and on Valkey and ReDB the rate counter can undercount by one under
   concurrency.
   **Mitigation:** none yet; an atomic increment in the cache contract if needed.

### Trade-offs

**We prioritise:**
- availability of the site and of its admin over strict blocking;
- refusing hostile requests without PHP.

**We accept:**
- no rate limit on the admin API;
- no permanent bans through these settings (the block list covers that).

---

## Fleet impact

- **speedyspec-server (repository):** holds the firewall's crates: `speedyspec-native-stage`, `speedyspec-firewall`,
  `-app`, `-infra-cache`, `-infra-options` and `-presentation`.
- **speedyspec-php-runtime:** must push `FirewallStage` first on every PHP app's chain and register its hooks at
  worker start. Not yet wired.
- **Blog, front-end API:** public scope, every rule. Unverified until wired.
- **Admin API:** admin scope; login protection on its sign-in endpoint once that exists. Unverified.
- **Admin site** (static files), **speedyspec-server (binary), speedyspec-cli:** not covered or unaffected.

---

## Enforcement

### Automated

`cargo test -p speedyspec-native-stage -p speedyspec-firewall -p speedyspec-firewall-app -p
speedyspec-firewall-infra-cache -p speedyspec-firewall-infra-options -p speedyspec-firewall-presentation` in
SpeedySpec/speedyspec-server runs the 47 tests from #20; CI's `test` job runs them on every pull request, and cargo
exits 101 on a failure. Rules map to tests: 2 `allow_list_wins_over_everything`, 3
`user_enumeration_blocked_for_anonymous_only` and `bearer_bare_cookie_or_nonce_alone_is_not_rest_credentials`, 5
`admin_scope_never_rate_limits_or_bans`, 6 `ipv6_clients_share_counters_and_bans_per_64`, 7
`lengths_are_capped_at_thirty_days`, 8 `store_errors_fail_open` and `missing_remote_addr_continues_and_logs`. None
has been seen failing on a rule violation. The domain crate's two dependencies were checked by hand with `cargo tree`,
not in CI.

### Review checklist

- [ ] New rules join the chain in the fixed order and respect `FirewallScope`.
- [ ] Every store or settings error is logged and returns `Continue`.
- [ ] Durations pass through the thirty-day cap; the domain crate depends only on `ipnet` and `regex`.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** built and merged in SpeedySpec/speedyspec-server#20. The build added the admin scope, IPv6 /64
  keys, the thirty-day cap, fail-open, stricter enumeration credentials and request-stage lockouts. Deferred: body
  patterns, rules added by PHP plugins (ADR-006), and `authenticate` enforcement.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
