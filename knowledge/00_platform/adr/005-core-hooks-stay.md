# ADR-005: WordPress core's hooks stay; new hooks only for what core lacks

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the designer
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md), [ADR-006](006-php-callback-registry-at-worker-start.md), [ADR-010](010-firewall-and-login-security.md), [ADR-011](011-page-cache-coarse-purge-first.md), [ADR-012](012-redirects-sitemaps-robots-feeds.md), [ADR-013](013-images-in-rust-on-the-job-runner.md)

---

## Context

PHP plugins and themes hook into exactly the areas the native features take over: redirects, sitemaps, `robots.txt`,
feeds, sign-in and image sizes. When those areas move to Rust, the plugins and themes must keep working. At the same
time, some features need extension points that core never had, such as redirect rules or cache tags.

### Why this matters

Renaming or reordering a core hook breaks every plugin that uses it, silently. Using a third-party plugin's hook or
option names would tie SpeedySpec to that plugin's choices and invite collisions. Without a compatibility check, a
port could drift from WordPress's output and nobody would see it.

---

## Decision

Core's hook names, and the order they fire in, stay unchanged. New hooks exist only for what core lacks, under the
project's prefix.

### Rules

1. A ported area fires every core hook it replaces, with the same name, arguments and order.
2. New hooks are added only for things core lacks (redirect rules, cache tags, request rules), and use the
   `speedyspec_` prefix.
3. Settings use the project's own option names (for example `speedyspec_firewall_*`), read through the ported options
   API.
4. Each feature names the core hooks it keeps (see ADR-010 to ADR-013).
5. Compatibility tests record WordPress's real output for the core parts (canonical redirects, sitemaps,
   `robots.txt`, feeds, image sub-sizes) and replay it against the ports.
6. Behaviour taken from plugins, not core, gets its own tests.

### What this looks like

✅ **Correct**

```rust
// The firewall registers on core's own hook names.
hooks.add_filter(&HookName::from("wp_login_failed"), failed, &HookValue::Int(10), 1)?;
hooks.add_filter(&HookName::from("xmlrpc_enabled"), xmlrpc, &HookValue::Int(99), 1)?;
```

❌ **Not permitted**

```rust
// A renamed core hook, and a plugin's option name.
hooks.do_action(&HookName::from("speedyspec_login_failed"), args)?;
let config = options.get::<String>("wordfence_config")?;
```

### Patterns

- **Observer**, through the Hook API: features and PHP plugins subscribe to the same named events.

---

## Consequences

### Positive

1. PHP plugins and themes that hook into these areas keep working.
2. Project hooks and options can't collide with WordPress or with third-party plugins.
3. Recorded compatibility shows when a port's output departs from WordPress's.

### Negative

1. A request that fires a hook with a PHP callback must go to PHP, which costs speed.
   **Mitigation:** ADR-006 records which hooks have PHP callbacks, so only those requests go to PHP.
2. Sites that used plugins' own hooks or option names get nothing from them.
   **Mitigation:** none yet. Importing settings is optional and not planned.

### Trade-offs

**We prioritise:**
- compatibility with the PHP ecosystem that hooks into core;
- a clear prefix boundary for anything new.

**We accept:**
- some requests leave the Rust path because a PHP callback is present;
- no support for the original plugins' APIs.

---

## Fleet impact

- **speedyspec-server (repository):** every port and feature keeps core hook names; new hooks and options take the
  `speedyspec_` prefix. The firewall follows this.
- **speedyspec-php-runtime:** fires the same hooks for PHP callbacks as stock WordPress would.
- **Blog, front-end API, admin API:** PHP plugins and themes keep their hooks.
- **Plugins:** a plugin that calls a third-party plugin's API (for example Yoast's) is unsupported.
- **speedyspec-cli, admin site:** unaffected.

---

## Enforcement

### Automated

`cargo test -p speedyspec-firewall-presentation --test hooks` in SpeedySpec/speedyspec-server, run by the CI `test`
job, checks that the firewall acts on core's `wp_login_failed` and `xmlrpc_enabled` (tests
`login_failed_hook_locks_out_after_max` and `xmlrpc_enabled_filter_follows_setting`); cargo exits 101 on failure.
It has not been seen failing. Recorded compatibility tests for canonical redirects, sitemaps, `robots.txt`, feeds and
image sub-sizes do not exist yet, because those ports are not built. The rest of this rule is enforced by review
only.

### Review checklist

- [ ] No core hook is renamed, dropped or fired in a new order.
- [ ] Every new hook and option uses the `speedyspec_` prefix.
- [ ] The feature lists the core hooks it keeps.
- [ ] Core-derived output has a recorded compatibility test; plugin-derived extras have their own tests.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** the firewall, keeping `wp_login_failed` and `xmlrpc_enabled`, merged in
  SpeedySpec/speedyspec-server#20.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
