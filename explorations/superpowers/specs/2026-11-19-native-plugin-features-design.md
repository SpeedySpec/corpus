# Native plugin features design

Date: 2026-11-19
Status: approved 2026-11-19

## Intent

Make the work of four kinds of popular plugins part of SpeedySpec's Rust port of WordPress, so that a site needs no
PHP plugin and runs no PHP for them:

- page caching (WP Rocket, W3 Total Cache, WP Super Cache);
- firewall and login security (Wordfence, Solid Security);
- redirects, `robots.txt`, sitemaps and feeds (Redirection, the sitemap parts of Yoast and Rank Math);
- image optimization (Smush, EWWW, Imagify).

The why is to minimize how much PHP is in flight per request. The software is for the project owner's personal use.

Nearly everything these plugins do already has a home in WordPress core: plugins mostly extend core subsystems. So the
features are not built as separate Rust plugins. Each one becomes behaviour of the WordPress API it extends, added
when that API is ported to Rust.

## Goals

- Each feature runs in Rust and never enters the PHP interpreter.
- Each feature is part of the ported WordPress API it extends (rewrite, sitemaps, robots, feeds, options, posts,
  terms, users, auth, media). Nothing is written twice.
- WordPress core's hook names and the order they fire in stay unchanged, so PHP plugins and themes that hook into
  these areas keep working.
- New hooks exist only for what core lacks (redirect rules, cache tags, request rules), under the project's prefix.
- Every feature sits behind a contract, with contracts and implementations in separate packages, each implementation
  replaceable partly or wholly. Each storage-backed contract has an in-memory implementation for tests.

## Non-goals

- Compatibility with the original plugins' own PHP APIs, classes, hooks or option names. Importing their settings is
  optional and out of scope here.
- CDN integration, minification or bundling of CSS and JavaScript, critical CSS, database cleanup, malware scanning of
  files, two-factor authentication, SEO content scoring.
- A server-level cache or firewall in front of FastCGI (see Approaches). That is a later experiment.
- Designing the API ports themselves. Each port gets its own design; this document says only what each port must
  include so the features fit.

## Anti-goals

- Running PHP for a request that a feature can answer: a cache hit, a blocked request, a redirect, a sitemap,
  `robots.txt` or a feed served from Rust must not start the PHP script.
- A feature that reads WordPress's tables directly instead of through a ported API.
- Cache invalidation logic kept apart from the APIs that change the data. If a write path can change content without
  the cache hearing about it through core's own functions, the design has failed.
- WordPress or plugin knowledge in the SpeedySpec server. The server knows only app kinds.

## Constraints

- **Stock WordPress never runs.** WordPress is ported to Rust with PHP embedded for what remains; WordPress's global
  functions stay as shims over the Rust ports (`2026-11-18-speedyspec-php-foundation-design.md`).
- **The server holds no PHP and no WordPress knowledge** (`2026-11-19-speedyspec-php-runtime-and-admin-gate-design.md`).
  These features therefore live in the runtime.
- **No side channel** between runtime and server beside FastCGI (same design).
- Package layering: `-domain`, `-app`, `-infra-*`, `-presentation`, `-legacy`; never `-core`. Contracts in their own
  package.
- Ports follow the Hook API port's pattern: commands and queries on the CQRS buses, PHP twins via `#[php_port]`, and
  recorded compatibility (`speedyspec-compat`) against WordPress's real outputs.
- Features must work per replica with no `[cluster]`, and use the cluster's shared state when it is configured.
- Rust 1.99, edition 2024.
- **Design principles** (project owner's standing rule): every port and feature draws on Clean Code, Clean Architecture,
  Hexagonal Architecture (ports and adapters), Fowler's Patterns of Enterprise Application Architecture and the Gang of
  Four patterns. Here that means, for example: Repository and Unit of Work for the stores; Strategy for store backends
  and image encoders; Chain of Responsibility for the request stages; Specification for redirect and firewall rules;
  Builder for sitemap, feed and `robots.txt` documents; Observer through the Hook API for cache purging.
- Only the Hook API is ported today (`speedyspec-wp-hooks`). Database access is still PHP's `mysqli`.

## Approaches considered

### Where the features run

1. **A. Split.** HTTP-level parts (response cache, rate limiting, IP blocking, request rules) in the server, driven by
   what the app tells it; WordPress-specific parts in the runtime. Keeps the server rule and gives the fastest path.
2. **B. All in the server.** Simplest to reason about, but gives the server database access and WordPress knowledge,
   which overturns the runtime design.
3. **C. All in the runtime's Rust side,** answering before the PHP script runs. Leaves the server untouched; cached
   and blocked requests still cross FastCGI into the runtime.

**Chosen: C**, by the project owner, to see how well it runs. A is kept as a later experiment.

### How the features get WordPress data

1. **A. A Rust read layer over WordPress's tables,** per feature. Fast, but ties Rust to the table layout and misses
   anything plugins filter in PHP.
2. **B. Wait for proper Rust ports** of the WordPress APIs (database layer, options, permalinks, posts, `WP_Query`,
   terms, users, attachments, cron).
3. **C. Call into PHP for data.** Runs PHP again, defeating the purpose.

**Chosen: B**, by the project owner. So porting the WordPress APIs comes before the features.

### How the features are shaped

1. **Separate Rust plugin packages** that hook into the ported APIs from outside.
2. **Behaviour of the ported APIs themselves,** extending what core already has.

**Chosen: 2** (proposed and accepted in conversation): core already has a home for almost every feature, and core
already calls its cache-clearing functions on every change, so invalidation and image generation fall out of the
ports instead of being reimplemented beside them.

## Order of work

Features arrive as the APIs they need are ported:

| Step | Port | Unblocks |
|---|---|---|
| 1 | Database layer, options | Firewall (request rules, login protection) |
| 2 | Permalinks and rewrite | Page cache; redirects |
| 3 | Attachments, background job runner (cron) | Image optimization |
| 4 | Posts, a `WP_Query` subset, terms, users | Sitemaps, feeds, `robots.txt` contributions |

The ports themselves are built elsewhere (see Ownership).

## How each API absorbs its feature

### Runtime request stages

The runtime handles each request in this order, all in Rust before any PHP:

1. **Request rules** (firewall).
2. **Response cache** lookup.
3. **Routing**: route, redirect or not found.
4. Native responders (sitemaps, `robots.txt`, feeds) or, failing those, the PHP script.

Stage 1 sits where Wordfence loads via `auto_prepend_file`; stage 2 where `advanced-cache.php` loads when `WP_CACHE`
is set.

**Which apps run which stages** (designer's call): request rules run for every PHP app (blog, front-end API, admin
API). The response cache and native responders run for the blog and the front-end API only; the admin API is never
cached. Static admin-site files are served by the server and never reach the runtime, so the firewall does not cover
them; that is accepted under approach C and revisited with approach A. Login protection attaches to the admin API's
sign-in endpoint, since that is where authentication happens in SpeedySpec.

### Rewrite and routing (absorbs Redirection)

- Core has `redirect_canonical()`, `wp_old_slug_redirect()` with `_wp_old_slug` meta, and `wp_redirect()`.
- Request resolution returns one of: *route*, *redirect*, *not found*. Redirect rules are a third redirect source
  beside canonical and old-slug redirects, behind a rules-store contract: exact, prefix and regex matches, per-rule
  status 301, 302, 307 or 410.
- A *not found* result is recorded in a 404 log behind its own contract, from which rules can be made.
- Kept: `redirect_canonical`, `old_slug_redirect_url` and `wp_redirect` filters; the `template_redirect` action.

### Sitemaps

- Core has `WP_Sitemaps` with post, taxonomy and user providers.
- Port the providers and add: `lastmod` on every entry, image entries, exclusion of `noindex` content using the SEO
  API, and one canonical URL per entry using the SEO API's canonical rule (`2026-11-17-seo-api-design.md`).
- Kept: `wp_sitemaps_add_provider`, `wp_sitemaps_posts_query_args`, `wp_sitemaps_posts_entry` and the other
  `wp_sitemaps_*` filters.

### `robots.txt` and robots meta

- Core has `do_robots()` with the `robots_txt` filter, and `wp_robots()`.
- Use the builder-and-contributors shape planned in phase 3 of the SEO API design. Sitemaps, SEO settings and the
  firewall each contribute lines.
- Kept: `robots_txt` and `wp_robots` filters.

### Feeds

- Core has RSS2, Atom and RDF templates.
- A Rust feed-document builder over the posts and query ports, plus plugin extras: full text or excerpt, a delay
  before new posts appear, per-category feeds.
- Feed hooks that print (`rss2_head`, `rss2_item`, `atom_entry` and the like): Rust callbacks return fragments; PHP
  callbacks get output buffering around just those hooks. Only when a PHP callback is registered on one of these hooks
  does a feed request enter PHP.

### Page cache

- Core has no page cache beyond the `advanced-cache.php` drop-in, but calls `clean_post_cache()`,
  `clean_term_cache()`, `clean_user_cache()` and `wp_cache_flush()` on every change.
- Each cached response is tagged with what it depends on: post IDs, term IDs, author, home page, feeds, sitemaps.
- The ported `clean_*_cache` functions, `transition_post_status`, `switch_theme` and `updated_option` purge by tag.
  Because these are Rust hooks in the shared Hook API, purges happen in whichever runtime pool made the change (usually
  the admin API's), so the cache store must be reachable by every pool that serves or changes content.
- Bypass rules: logged-in and comment-author cookies, configurable query strings, non-GET requests,
  `DONOTCACHEPAGE`.
- Kept or added: a cache-key filter; a purge-by-URL-or-tag command for plugins that change content behind core's back.
- **Coarse purging first** (designer's call): the cache ships at step 2 with purge-everything on any content change.
  Tagging and purging by tag arrive with the step 4 ports, which are what call the `clean_*_cache` functions.

### Firewall and login security (absorbs Wordfence and Solid Security)

- Core has the `authenticate` filter, `wp_login_failed`, application passwords, XML-RPC, `?author=N` lookups and the
  REST users endpoint.
- **Request rules** (stage 1): IP allow and block lists, rate limits, and patterns over path, query string and body.
- **Login protection**, inside the ported auth API: `wp_login_failed` counts failures, the `authenticate` filter
  enforces lockouts; switches close user enumeration and XML-RPC.
- Counters and bans sit behind a store contract (see Shared state).
- Kept or added: `authenticate`, `wp_login_failed`, `xmlrpc_enabled`, `rest_endpoints`; a hook for adding request
  rules.

### Media and images (absorbs Smush, EWWW and Imagify)

- Core has `WP_Image_Editor` (GD, Imagick), resumable `wp_create_image_subsizes()`, `image_editor_output_format`
  (WebP since 6.1, AVIF since 6.5), `big_image_size_threshold`, and srcset and lazy loading via
  `wp_filter_content_tags()`.
- A Rust image-editor backend (the `image` crate with WebP and AVIF encoders) beside GD and Imagick.
- Sub-size generation moves to the background job runner, so uploads return at once; core's resumable sub-size
  bookkeeping tracks progress.
- Added: re-compressing the existing library, stripping metadata, quality per format.
- Kept: `wp_image_editors`, `intermediate_image_sizes_advanced`, `image_editor_output_format`,
  `wp_generate_attachment_metadata`, `wp_calculate_image_srcset`.

## Shared rules

- Core hook names stay; new hooks use the project's prefix.
- Settings use the project's own option names, read through the ported options API.
- Every store (redirect rules, 404 log, cache, firewall counters, image job queue) is a contract with an in-memory
  implementation and a real one.
- Compatibility tests record WordPress's real output for the core parts (canonical redirects, sitemaps, `robots.txt`,
  feeds, image sub-sizes) and replay it against the ports; plugin-derived extras get their own tests.

## PHP callbacks on native paths

Decided by the project owner: **build a registry when a worker starts.** PHP callbacks normally exist only for one
request, registered as PHP loads plugins and the theme. So a request answered in Rust would never see them. Instead,
each runtime worker loads its app's plugins and theme once at startup and records which hooks have PHP callbacks.

- A native path (redirect, sitemap, `robots.txt`, feed, cache-key check, firewall rule) checks the registry for every
  hook it fires. If none of them has a PHP callback, the request stays in Rust. If any does, the request goes to PHP,
  and the PHP path fires the hook as usual.
- The registry is rebuilt whenever a worker is replaced, which covers plugin activation, deactivation and new
  releases, because those recycle workers.
- A PHP callback registered conditionally (only on some requests) is still recorded, so the registry may send to PHP
  more often than needed, but never less. Correctness wins over speed.

## Shared state

Decided by the project owner: every shared store is a contract with **file, Redis and database** implementations,
chosen in configuration.

- **The page cache** defaults to files in a directory every pool of the app can reach, because the admin API's pool
  purges entries the blog's pool serves.
- **Firewall bans and counters, redirect rules and the 404 log** default to the database, through the ported database
  layer (designer's call, following the same three-implementation rule).
- With `[cluster]`, Redis or the database is the shared choice; files are per replica.
- Every store also has an in-memory implementation for tests.

## Background job runner

Decided by the project owner: **a long-lived process inside the runtime**, beside the request workers, supervised the
same way. It runs queued jobs (image sub-sizes, library re-compression) and scheduled events, and is expected to
replace WordPress's request-driven cron. It never serves requests. The job queue is a store like the others (file,
Redis or database). With `[cluster]`, only one replica runs a given job. How that is arranged inside the runtime,
without giving the server WordPress knowledge, is left to the job runner's own design.

## Requirements handed to the database port

Stated by the project owner for step 1; the port's own design settles how:

- **Several databases behind repository contracts:** MySQL, PostgreSQL, SQLite and ReDB, with room for others.
- **Engines, in the manner of SQLAlchemy:** each database is reached through an engine that knows its dialect and can
  apply engine-specific optimizations. Several databases can be used at once.
- **A query builder in the manner of Eloquent's,** which also understands raw SQL: raw SQL can be parsed into a
  builder instance and manipulated with partials (adding or replacing clauses, fragments).
- **A unit of work** that groups changes and commits them together.
- **Plain objects, not Active Record:** entities are plain old PHP objects and plain Rust structs that know nothing
  about storage. Data Mappers and Repositories load and save them. "In the manner of Eloquent" refers to the query
  builder's style only, not to Eloquent's models.

Two consequences the port must address:

- WordPress plugins send raw MySQL-flavoured SQL through `$wpdb`. Parsing raw SQL into the builder is what lets that
  SQL run on PostgreSQL or SQLite, by re-emitting it in the target engine's dialect.
- ReDB is a key-value store with no SQL. Raw SQL and the builder can only target the SQL engines; ReDB is reached
  through repository implementations written for it.

## Ownership

Decided by the project owner: **the API ports in steps 1 to 4 are not part of this work.** Other sessions own them,
most likely a new session devoted to porting. This design is the specification of what each port must include so the
features fit, and each feature is planned and built once the ports it needs exist.

## Left to the builder

- Crate and package names within the layering rules.
- Choice of Rust crates for regex matching, image encoding and rate limiting.
- The exact tag vocabulary for cached responses, as long as it covers the dependencies listed above.
- Settings defaults.
