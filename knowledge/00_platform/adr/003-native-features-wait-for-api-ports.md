# ADR-003: Native features wait for proper Rust ports of the WordPress APIs

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the project owner
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md), [ADR-009](009-database-port-requirements.md), [ADR-010](010-firewall-and-login-security.md), [ADR-011](011-page-cache-coarse-purge-first.md), [ADR-012](012-redirects-sitemaps-robots-feeds.md), [ADR-013](013-images-in-rust-on-the-job-runner.md)

---

## Context

The native features need WordPress data: options, permalinks, posts, terms, users and attachments. When this was
decided, only the Hook API had been ported to Rust (`speedyspec-wp-hooks`). Database access still went through PHP's
`mysqli`. So the features had no Rust way to read anything.

### Why this matters

A shortcut here sets the shape of every feature. A Rust read layer over WordPress's tables would tie Rust to the table
layout and miss whatever PHP plugins filter. Calling into PHP for data would run PHP again, which is the thing the
features exist to avoid. Either shortcut would have to be torn out once the real ports exist.

---

## Decision

Port the WordPress APIs to Rust first. Build each feature once the APIs it needs exist, in this order:

| Step | Port | Unblocks |
|---|---|---|
| 1 | Database layer, options | Firewall (request rules, login protection) |
| 2 | Permalinks and rewrite | Page cache; redirects |
| 3 | Attachments, background job runner (cron) | Image optimization |
| 4 | Posts, a `WP_Query` subset, terms, users | Sitemaps, feeds, `robots.txt` contributions |

### Rules

1. A feature is not planned for building until every port it needs, per the table, has merged.
2. The ports are owned by separate porting work. The features' design only states what each port must include.
3. Each port follows the Hook API port's shape: commands and queries on the CQRS buses, PHP twins through
   `#[php_port]`, and recorded compatibility (`speedyspec-compat`) against WordPress's real outputs.
4. No feature gets a private read layer over WordPress tables, and none calls into PHP for data.

### What this looks like

This decision is an ordering of work, so it has no code shape of its own.

### Patterns

- **Repository** and **Data Mapper** behind each port, so features never see tables.
- **Command Query Responsibility Segregation:** the ports' commands and queries go over the buses.

---

## Consequences

### Positive

1. Each feature is built once, on the same APIs the rest of the port uses.
2. Recorded compatibility tests prove each port before a feature depends on it.
3. The order makes it plain which feature is next and what blocks it.

### Negative

1. Features arrive slowly. Nothing beyond the firewall can start until permalinks are ported.
   **Mitigation:** step 1 went first because it unblocks the firewall, the feature with the most to gain.
2. Feature work depends on other work streams.
   **Mitigation:** the native plugin features design lists what each port must include, so a port can be checked
   against it.

### Trade-offs

**We prioritise:**
- real ports over fast shortcuts;
- one data path for features and the rest of the port.

**We accept:**
- a long wait for the page cache, redirects, sitemaps, feeds and images;
- coordination with the porting work.

Alternatives weighed: **A. A Rust read layer over WordPress's tables** (fast, but tied to the table layout and blind
to PHP filters) and **C. Call into PHP for data** (defeats the purpose). Both rejected.

---

## Fleet impact

- **speedyspec-server (repository):** holds the ports and the features, built in the table's order. Step 1 is
  merged; steps 2 to 4 are not started.
- **speedyspec-php-runtime:** gains each feature as its step completes.
- **Blog, front-end API, admin API:** gain features step by step. Unverified until the runtime is wired.
- **speedyspec-cli, admin site, plugins:** unaffected.

---

## Enforcement

### Automated

None. This rule is enforced by review only.

### Review checklist

- [ ] Every port the feature needs, per the table, has merged.
- [ ] The feature reads data only through those ports.
- [ ] The ports it relies on have recorded compatibility tests.

---

## Exceptions

The project owner may reorder the steps. A reorder is recorded as a new revision entry here, or as a new ADR if it
changes which ports a feature needs.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-20 to 2026-11-21:** step 1 (database layer and options) merged in SpeedySpec/speedyspec-server#12 to
  #19.
- **2026-11-21:** the firewall, unblocked by step 1, merged in SpeedySpec/speedyspec-server#20.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
