# ADR-001: Plugin features are behaviour of the ported WordPress APIs

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** both (proposed by the designer, accepted by the project owner)
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-002](002-native-features-run-in-the-runtime.md), [ADR-003](003-native-features-wait-for-api-ports.md), [ADR-005](005-core-hooks-stay.md), [ADR-011](011-page-cache-coarse-purge-first.md), [ADR-013](013-images-in-rust-on-the-job-runner.md)

---

## Context

SpeedySpec ports WordPress to Rust and embeds PHP only for what is left. The goal is to run as little PHP per
request as possible. Most WordPress sites run four kinds of large plugins, and each runs PHP on nearly every request,
often before WordPress loads:

- page caching (WP Rocket, W3 Total Cache, WP Super Cache);
- firewall and login security (Wordfence, Solid Security);
- redirects, `robots.txt`, sitemaps and feeds (Redirection, the sitemap parts of Yoast and Rank Math);
- image optimization (Smush, EWWW, Imagify).

Nearly all of this work already has a home in WordPress core. These plugins extend core subsystems: rewrite,
sitemaps, robots, feeds, auth and media. Core also calls its cache-clearing functions on every change.

### Why this matters

If each feature were its own package, each would need its own copy of "when did the data change" and "how is an
image made". Two copies drift. A cache that misses one write path serves stale pages, and nobody notices until a
reader does.

---

## Decision

Each feature becomes behaviour of the WordPress API it extends, added when that API is ported to Rust. The feature
runs in Rust and never enters the PHP interpreter.

### Rules

1. A feature lives in the package family of the API it extends (options, rewrite, sitemaps, robots, feeds, posts,
   terms, users, auth, media), not in a separate plugin package.
2. A feature reads WordPress data only through a ported API. Reading WordPress's tables directly is a defect.
3. Cache invalidation and image generation come from the API that changes the data. No second copy beside it.
4. Every feature sits behind a contract. The contract lives in its own package, apart from its implementations.
5. Every storage-backed contract has an in-memory implementation for tests and at least one real one.
6. The original plugins' PHP APIs, classes, hooks and option names are not supported. Importing their settings is
   optional.
7. Out of scope: CDN integration, CSS and JavaScript minification or bundling, critical CSS, database cleanup,
   malware scanning of files, two-factor authentication and SEO content scoring.

### What this looks like

✅ **Correct**

```rust
// The firewall reads its settings through the ported options API, behind its own port.
pub trait SettingsSource {
    fn load(&self) -> Result<RawSettings, FirewallError>;
}

impl SettingsSource for OptionsSettingsSource {
    fn load(&self) -> Result<RawSettings, FirewallError> {
        let d = RawSettings::default();
        Ok(RawSettings {
            enabled: self.read("enabled", d.enabled)?, // speedyspec_firewall_enabled
            // ... the other speedyspec_firewall_* options
        })
    }
}
```

❌ **Not permitted**

```rust
// A feature querying WordPress's tables itself.
let rows = mysql.query("SELECT option_value FROM wp_options WHERE option_name = 'wordfence_config'")?;
```

### Patterns

- **Ports and Adapters (Hexagonal Architecture):** every feature sits behind a contract in its own package.
- **Repository:** each storage-backed contract, with an in-memory and a real implementation.

---

## Consequences

### Positive

1. Invalidation and image generation are written once, in the APIs that already know when data changes.
2. PHP plugins that hook into core keep working, because the feature uses core's own subsystems.
3. Each implementation can be replaced partly or wholly without touching the contract.

### Negative

1. A feature can't ship before the API it extends is ported.
   **Mitigation:** the order of work in ADR-003 says which port unblocks which feature.
2. Sites that relied on a plugin's own PHP API lose it.
   **Mitigation:** none yet. Importing settings is allowed but not planned.

### Trade-offs

**We prioritise:**
- one source of truth for "data changed";
- less PHP per request.

**We accept:**
- features arrive later than a standalone package would;
- no compatibility with the original plugins' own APIs.

---

## Fleet impact

- **speedyspec-server (repository):** the port crates gain the features. The firewall's crates
  (`speedyspec-firewall`, `-app`, `-infra-cache`, `-infra-options`, `-presentation`) follow this rule.
- **speedyspec-php-runtime:** runs the features. Not yet wired for the firewall.
- **Server binary, speedyspec-cli, admin site:** unaffected.
- **Blog, front-end API, admin API:** gain the features through the runtime. Unverified until wired.
- **Plugins:** PHP plugins keep working through core hooks (ADR-005); the original plugins' options are not read.

---

## Enforcement

### Automated

None. This rule is enforced by review only.

### Review checklist

- [ ] The feature's code sits in the package family of the API it extends.
- [ ] No feature crate reads WordPress tables or raw SQL; data comes through a ported API.
- [ ] The contract is in its own package; the domain crate depends on no adapter.
- [ ] Each storage-backed contract has an in-memory implementation used by tests.
- [ ] No code reads or writes the original plugins' option names, except an optional importer.

---

## Exceptions

None. Adding a feature from the out-of-scope list needs a new ADR, decided by the project owner.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** the firewall, the first feature built this way, merged in SpeedySpec/speedyspec-server#20.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
