# ADR-002: Native features run in the runtime's Rust side, before PHP

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the project owner
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md), [ADR-004](004-runtime-request-stages.md), [ADR-010](010-firewall-and-login-security.md)

---

## Context

SpeedySpec runs as two processes. The server (built on Salvo) holds no PHP and no WordPress knowledge. It only knows
app kinds. The PHP runtime embeds PHP, and the server reaches it over FastCGI. There is no side channel between them
besides FastCGI.

The native features (ADR-001) must answer some requests before any PHP runs: a cache hit, a blocked client, a
redirect. Something has to decide which process does that work.

### Why this matters

Putting the features in the server would give it database access and WordPress knowledge, and undo the boundary the
runtime design rests on. Splitting them across both processes would need the app to tell the server what to do, which
blurs the same boundary. Picking wrongly means rework in both binaries.

---

## Decision

All native features run in the PHP runtime's Rust side, and answer before the PHP script starts. The server is not
changed.

### Rules

1. No native feature code (cache, firewall, redirects, responders) goes into the server binary.
2. The server binary depends on no PHP crate.
3. The runtime and the server talk only over FastCGI. A feature adds no other channel.
4. A feature that answers a request does so in the runtime, before the PHP script runs.
5. A cache or firewall in front of FastCGI, in the server, is out of scope until the split experiment below is run.

### What this looks like

✅ **Correct**

```rust
// In the runtime's worker: Rust answers first, PHP only if nobody did.
match chain.answer(&StageRequest { params: &params }) {
    StageOutcome::Respond(response) => write_fastcgi_response(response),
    StageOutcome::Continue => run_php_script(&params, body),
}
```

❌ **Not permitted**

```rust
// In the server: WordPress knowledge and a database handle at the HTTP edge.
if let Some(page) = page_cache.get(&req.uri().to_string()) {
    return Ok(page);
}
```

### Patterns

- **Hexagonal Architecture:** the runtime is the application core; FastCGI is its only port to the server.

---

## Consequences

### Positive

1. The server stays small and free of WordPress knowledge.
2. Features share the runtime's process with the ported APIs, so they read data without crossing a boundary.

### Negative

1. Cached and blocked requests still cross FastCGI into the runtime. A cache hit costs a FastCGI round trip.
   **Mitigation:** approach A (split, below) is kept as a later experiment.
2. Static admin-site files are served by the server and never reach the runtime, so the firewall does not cover them.
   **Mitigation:** none yet. Revisited with approach A.

### Trade-offs

**We prioritise:**
- a clean boundary: the server knows app kinds, the runtime knows WordPress;
- seeing how well the runtime-only approach runs before optimising.

**We accept:**
- one FastCGI round trip on every cached or blocked request;
- an unprotected path for static admin-site files.

Alternatives weighed: **A. Split** (HTTP-level parts in the server, driven by the app) is fastest and kept as a later
experiment. **B. All in the server** is simplest to reason about but overturns the runtime design, so it was
rejected.

---

## Fleet impact

- **speedyspec-server (binary):** unchanged. Must not gain feature code or a PHP dependency.
- **speedyspec-php-runtime:** hosts every native feature. The firewall's crates exist; the worker is not yet wired.
- **Blog, front-end API, admin API:** reach the features through the runtime (ADR-004 says which stages each gets).
- **Admin site:** its static files bypass the features.
- **speedyspec-cli, plugins:** unaffected.

---

## Enforcement

### Automated

`scripts/check-no-php-in-server.sh` in SpeedySpec/speedyspec-server, run by the `test` job of
`.github/workflows/ci.yml` on every pull request and push to `main`. It exits 1 if `speedyspec-server` depends on
`speedyspec-php`, `speedyspec-php-ext`, `ext-php-rs`, `speedyspec-wp-hooks-php` or `speedyspec-container-php`.
It covers rule 2 only. It passed in the CI runs checked and has not been seen failing. Rules 1, 3, 4 and 5 are
enforced by review only.

### Review checklist

- [ ] The change adds no feature crate (`speedyspec-firewall*`, cache, redirects) to the server's dependencies.
- [ ] No new socket, file or shared-memory channel between server and runtime.
- [ ] The feature's entry point is a runtime stage, not a server handler.

---

## Exceptions

Only the split experiment (approach A), when the project owner chooses to run it. It needs its own ADR that
supersedes this one in part.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
