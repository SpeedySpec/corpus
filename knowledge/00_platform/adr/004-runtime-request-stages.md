# ADR-004: The runtime answers requests in four stages before PHP

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the designer
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-002](002-native-features-run-in-the-runtime.md), [ADR-006](006-php-callback-registry-at-worker-start.md), [ADR-010](010-firewall-and-login-security.md), [ADR-011](011-page-cache-coarse-purge-first.md), [ADR-012](012-redirects-sitemaps-robots-feeds.md)

---

## Context

With every native feature in the runtime (ADR-002), each request needs a fixed order in which the features get a
chance to answer. Stock WordPress has two such points. Wordfence loads through `auto_prepend_file`, before anything
else. A page cache loads through `advanced-cache.php` when `WP_CACHE` is set. Everything else happens once WordPress
has loaded.

### Why this matters

Order decides behaviour. A cache asked before the firewall serves pages to banned clients. A redirect checked before
the cache misses cached redirects. Without one agreed order, each feature would pick its own hook point, and the
result would depend on which was built last.

---

## Decision

The runtime handles each request in four stages, all in Rust, before any PHP runs. The first stage that answers wins.

1. **Request rules** (the firewall), where `auto_prepend_file` would run.
2. **Response cache** lookup, where `advanced-cache.php` would run.
3. **Routing:** route, redirect, or not found.
4. **Native responders** (sitemaps, `robots.txt`, feeds), or failing those, the PHP script.

### Rules

1. Stages run in the order above. A new feature joins an existing stage or needs a new ADR.
2. Request rules run for every PHP app: the blog, the front-end API and the admin API.
3. The response cache and native responders run for the blog and the front-end API only. The admin API is never
   cached.
4. Login protection attaches to the admin API's sign-in endpoint, where SpeedySpec authenticates.
5. A stage sees only the request's FastCGI params, never its body, because PHP may still need the body.

### What this looks like

✅ **Correct**

```rust
// speedyspec-native-stage: a stage reads params only, and the chain stops at the first Respond.
impl NativeStage for FirewallStage {
    fn name(&self) -> &str { "firewall" }
    fn answer(&self, request: &StageRequest<'_>) -> StageOutcome {
        let facts = match RequestFacts::from_params(request.params) {
            Ok(facts) => facts,
            Err(error) => { self.log.warn(&format!("request: {error}")); return StageOutcome::Continue; }
        };
        // ... check facts, return Respond(403/429) or Continue
    }
}
```

❌ **Not permitted**

```rust
// A stage that consumes the body, taking it away from PHP.
fn answer(&self, request: &StageRequest<'_>) -> StageOutcome {
    let body = read_to_end(request.stdin);
    if body.contains("<script") { return block(403); }
    StageOutcome::Continue
}
```

### Patterns

- **Chain of Responsibility:** `StageChain` asks each `NativeStage` in order; the first `Respond` wins.
- **Adapter:** each feature wraps its own domain in a `NativeStage` (for example `FirewallStage`).

---

## Consequences

### Positive

1. A cache hit, a blocked request, a redirect, a sitemap, `robots.txt` or a feed served from Rust never starts the
   PHP script.
2. Each feature has a known place and a small contract, so it can be tested alone.

### Negative

1. Rules over the request body can't run as a stage.
   **Mitigation:** none yet. Waits for a design that can inspect the body without taking it from PHP.
2. The chain is not yet wired into the runtime's worker, so no request is answered in Rust today.
   **Mitigation:** a wiring request was handed to the runtime's owner with the firewall.

### Trade-offs

**We prioritise:**
- a predictable order that mirrors where stock WordPress plugins hook in;
- PHP keeping the body intact.

**We accept:**
- no body inspection for now;
- the admin API never cached.

---

## Fleet impact

- **speedyspec-php-runtime:** must build a `StageChain` per worker and ask it before starting PHP. Not yet done.
- **Blog, front-end API:** all four stages.
- **Admin API:** request rules only, and login protection on its sign-in endpoint once that exists.
- **speedyspec-server (binary), speedyspec-cli, admin site:** unaffected. Admin-site static files never reach the
  stages.

---

## Enforcement

### Automated

`cargo test -p speedyspec-native-stage` in SpeedySpec/speedyspec-server, run by the CI `test` job. The test
`first_respond_wins_and_later_stages_are_not_called` fails (cargo exits 101) if the chain asks a stage after one has
responded. `cargo test -p speedyspec-firewall-presentation --test stage` checks the firewall stage continues on a
missing client address (`missing_remote_addr_continues_and_logs`). Neither has been seen failing. Nothing checks the
order of the four stages or rule 5; those are enforced by review only.

### Review checklist

- [ ] The new stage is pushed onto the chain in the documented position.
- [ ] The stage reads only `StageRequest::params`.
- [ ] The cache and responders are not added to the admin API's chain.

---

## Exceptions

None. Body inspection, when designed, needs a new ADR.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** stage 1 built as `speedyspec-native-stage` (`StageChain`) with the firewall, merged in
  SpeedySpec/speedyspec-server#20.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
