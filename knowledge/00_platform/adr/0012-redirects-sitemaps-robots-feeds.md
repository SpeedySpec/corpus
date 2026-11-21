# 0012. Redirects, sitemaps, `robots.txt` and feeds are answered in Rust

- **Status:** Accepted, 2026-11-19
- **Decided by:** the designer

## Context

Core has `redirect_canonical()`, `wp_old_slug_redirect()` and `wp_redirect()`; `WP_Sitemaps` with post, taxonomy and
user providers; `do_robots()` and `wp_robots()`; and RSS2, Atom and RDF feed templates. Redirection, Yoast and Rank
Math extend these.

## Decision

**Redirects**, in routing (stage 3 of 0004):
- Request resolution returns one of *route*, *redirect* or *not found*.
- Redirect rules are a third redirect source beside canonical and old-slug redirects, behind a rules-store contract:
  exact, prefix and regex matches, with a status per rule of 301, 302, 307 or 410.
- A *not found* is recorded in a 404 log behind its own contract, from which rules can be made.
- Kept: the `redirect_canonical`, `old_slug_redirect_url` and `wp_redirect` filters, and the `template_redirect`
  action.

**Sitemaps:**
- Port core's providers and add `lastmod` on every entry, image entries, exclusion of `noindex` content, and one
  canonical URL per entry, using the SEO API's rules.
- Kept: `wp_sitemaps_add_provider`, `wp_sitemaps_posts_query_args`, `wp_sitemaps_posts_entry` and the other
  `wp_sitemaps_*` filters.

**`robots.txt` and robots meta:**
- A builder with contributors, as planned in the SEO API's phase 3. Sitemaps, SEO settings and the firewall each
  contribute lines.
- Kept: the `robots_txt` and `wp_robots` filters.

**Feeds:**
- A Rust feed builder over the posts and query ports, with extras: full text or excerpt, a delay before new posts
  appear, per-category feeds.
- For hooks that print (`rss2_head`, `rss2_item`, `atom_entry` and the like), Rust callbacks return fragments and PHP
  callbacks get output buffering around just those hooks.

## Consequences

- A feed request enters PHP only when a PHP callback is registered on one of its printing hooks (0006).
- Sitemaps, feeds and `robots.txt` contributions wait for step 4 of 0003; redirects for step 2.

## Patterns

- **Specification:** redirect rules.
- **Builder:** sitemap, feed and `robots.txt` documents.
- **Repository:** the rules store and the 404 log.
