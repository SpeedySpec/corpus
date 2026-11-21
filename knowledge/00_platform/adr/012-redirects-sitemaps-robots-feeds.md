# ADR-012: Redirects, sitemaps, `robots.txt` and feeds are answered in Rust

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the designer
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-003](003-native-features-wait-for-api-ports.md), [ADR-004](004-runtime-request-stages.md), [ADR-005](005-core-hooks-stay.md), [ADR-006](006-php-callback-registry-at-worker-start.md), [ADR-007](007-shared-stores-file-redis-database.md)

---

## Context

Core already has the pieces that Redirection, Yoast and Rank Math extend:

- `redirect_canonical()`, `wp_old_slug_redirect()` and `wp_redirect()`;
- `WP_Sitemaps`, with post, taxonomy and user providers;
- `do_robots()` and `wp_robots()`;
- RSS2, Atom and RDF feed templates.

Each of these answers a request that needs little or no theme output, so each can be served from Rust once the data
it reads is ported.

### Why this matters

These are high-volume, crawler-facing requests. Running PHP for each one is wasted work. But themes and plugins hook
into all of them, so serving them from Rust without keeping those hooks would quietly change a site's output.

---

## Decision

Each subsystem is ported with core's hooks, plus the extras the plugins add. Redirects are decided in routing (stage
3); sitemaps, `robots.txt` and feeds are native responders (stage 4).

### Rules

1. **Redirects:** request resolution returns *route*, *redirect* or *not found*. Redirect rules are a third source
   beside canonical and old-slug redirects, behind a rules-store contract, with exact, prefix and regex matches and a
   status per rule of 301, 302, 307 or 410.
2. A *not found* is recorded in a 404 log behind its own contract, from which rules can be made.
3. Kept: the `redirect_canonical`, `old_slug_redirect_url` and `wp_redirect` filters, and the `template_redirect`
   action.
4. **Sitemaps:** core's providers are ported, adding `lastmod` on every entry, image entries, exclusion of `noindex`
   content, and one canonical URL per entry, using the SEO API's rules. Kept: `wp_sitemaps_add_provider`,
   `wp_sitemaps_posts_query_args`, `wp_sitemaps_posts_entry` and the other `wp_sitemaps_*` filters.
5. **`robots.txt` and robots meta:** a builder with contributors, as planned in the SEO API's phase 3. Sitemaps, SEO
   settings and the firewall each contribute lines. Kept: the `robots_txt` and `wp_robots` filters.
6. **Feeds:** a Rust feed builder over the posts and query ports, with full text or excerpt, a delay before new posts
   appear, and per-category feeds.
7. For hooks that print (`rss2_head`, `rss2_item`, `atom_entry` and the like), Rust callbacks return fragments and
   PHP callbacks get output buffering around just those hooks.
8. A feed request enters PHP only when a PHP callback is registered on one of its printing hooks (ADR-006).

### What this looks like

✅ **Correct**

```rust
// robots.txt: contributors add lines; core's filter still runs over the result.
let mut robots = RobotsBuilder::new();
for contributor in &self.contributors {          // sitemaps, SEO settings, firewall
    contributor.contribute(&mut robots);
}
let text = hooks.apply_filters(&HookName::from("robots_txt"), vec![robots.build().into(), public.into()])?;
```

❌ **Not permitted**

```rust
// A hard-coded document that skips contributors and the robots_txt filter.
StageOutcome::Respond(text_response("User-agent: *\nDisallow: /wp-admin/\n"))
```

### Patterns

- **Specification:** redirect rules.
- **Builder:** sitemap, feed and `robots.txt` documents.
- **Repository:** the redirect rules store and the 404 log.

---

## Consequences

### Positive

1. Redirects, sitemaps, `robots.txt` and most feeds are served without starting PHP.
2. Themes and plugins that hook into these subsystems keep working.
3. A site gets Redirection-style rules and Yoast-style sitemap extras without those plugins.

### Negative

1. Sitemaps, feeds and `robots.txt` contributions wait for step 4 of ADR-003; redirects wait for step 2.
   **Mitigation:** none; follows from ADR-003.
2. A PHP callback on a printing feed hook sends the whole feed request to PHP.
   **Mitigation:** output buffering is limited to those hooks once in PHP.

### Trade-offs

**We prioritise:**
- core's hooks and output, proven by recorded compatibility;
- plugin extras built into core's subsystems.

**We accept:**
- later delivery than a standalone read layer would give;
- feed requests falling back to PHP when a printing hook has a PHP callback.

---

## Fleet impact

- **speedyspec-server (repository):** gains the rewrite, sitemaps, robots and feeds ports with these features, and
  the rules store and 404 log contracts. Not started.
- **speedyspec-php-runtime:** runs redirects in routing and the responders in stage 4. Not built.
- **Blog, front-end API:** serve these responses. The admin API does not run responders (ADR-004).
- **Plugins:** keep core's hooks; Redirection's and Yoast's own APIs are not supported.
- **speedyspec-server (binary), speedyspec-cli, admin site:** unaffected.

---

## Enforcement

### Automated

None. This rule is enforced by review only. Recorded compatibility tests for canonical redirects, sitemaps,
`robots.txt` and feeds are planned but do not exist yet.

### Review checklist

- [ ] Every kept hook in rules 3 to 5 fires with core's name and arguments.
- [ ] Redirect rules and the 404 log are reached only through their contracts.
- [ ] Native responders check the PHP callback registry for every hook they fire.
- [ ] Core-derived output has a recorded compatibility test; extras have their own tests.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
