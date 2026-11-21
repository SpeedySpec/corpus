# SEO API design

Date: 2026-11-17
Status: approved 2026-11-17; amended after the phase 1 final review
Research: [`research/search-engine-seo-requirements-2026/report.md`](../../research/search-engine-seo-requirements-2026/report.md), with its notes beside it

## Intent

Give the project one SEO API that owns a page's search-facing output, so that SEO plugins build on a shared model
instead of each shipping its own incompatible one.

The research behind this design found that search engines require very little, and that what hurts a site is its own
outputs disagreeing: a canonical link that differs from the sitemap entry, the `og:url`, or the structured data URL.
Today that output is produced by unrelated functions spread across at least eight packages (`wp-theme`, `wp-route`,
`wp-query`, `wp-support`, `wp-sitemap`, `wp-rest-api`, `wp-customizer`, `wp-deprecated`), hooked separately from
`wp-plugin-legacy/functions/default-filters.php`, and nothing emits a meta description, social card tags or
structured data at all.

## Goals

- One canonical URL per page, resolved once and reused by every output.
- A single model of the document head that plugins contribute to and override.
- Every piece of behaviour sits behind a contract, with contracts and implementations in separate packages.
- An implementation can be replaced one contract at a time, or as a whole.
- Existing WordPress functions and filters keep working unchanged.
- All output is in the server-rendered HTML or HTTP response (AI crawlers mostly do not execute JavaScript).

## Non-goals

- Content scoring, keyword analysis, readability checks. These are what SEO plugins add on top.
- Core Web Vitals measurement or optimisation.
- Rich-result generators for content types that need plugin-held data (Product, Event, Recipe, JobPosting).
- llms.txt, meta keywords, sitemap `priority`/`changefreq`, sitemap pings, FAQ/HowTo/SearchAction markup, AMP.
- Enforcing blocks against AI crawlers. That is host and CDN work.

## Anti-goals

- A contracts package that contains replaceable behaviour. If swapping an implementation requires forking the
  contracts package, the design has failed.
- A default implementation that other packages reach into directly instead of through a contract.
- Hard-coded lists of engine names, bot tokens, structured data types or verification tags. Research showed these
  lists change every year.
- Breaking a site that has an existing SEO plugin installed and hooked onto today's filters.

## Constraints

- PHP 8.4, `declare(strict_types=1)`, 4-space indentation, and the docblock rules in `.claude/rules/coding-style.md`.
- Contracts package has zero dependencies and uses no WordPress functions.
- Namespace root `SpeedySpec\WP\Seo\`.
- Tests use Pest, as the hook packages and `speedyspec-support` do.
- **Confirmed by the project owner:** the packages are built and tested standalone. Today
  `src/wp-includes/` is empty, the root `composer.json` requires no package, and nothing calls any package's
  `ServiceProvider`, so the site does not boot from `packages/` yet. Wiring packages into a running site, and
  passing `tests/phpunit`, is therefore outside this design. Phase 1 is accepted on its own Pest suites.

## Approaches considered

1. **One `wp-seo` package holding contracts and implementation.** Simplest, but rejected by the project owner:
   contracts must always be their own package, and implementations must be replaceable partially and completely.
2. **One package per capability, each with its own contracts** (head, robots, canonical, structured data,
   redirects). Clean boundaries, but the capabilities share value objects (URL, page context) and the canonical
   resolver, so the shared pieces would need yet another package, and agreement between outputs becomes a
   coordination problem.
3. **Plain hooks beside the existing functions.** Cheapest, but nothing gains a single view of the page.
4. **Chosen: Domain-Driven Design layers, one contracts package, per-contract binding.** All contracts live in
   `wp-seo-domain`. Default behaviour lives in the other layer packages. Each contract is bound individually in a container,
   so one binding can be overridden (partial replacement) or an entire implementation package can be left out and
   another installed in its place (complete replacement). The project owner confirmed that keeping all contracts
   together in `wp-seo` is acceptable.

## Package layout

The package follows the project's Domain-Driven Design layering (`.claude/skills/ddd-refactor.md`). The domain and
infrastructure layers are the core. The application, presentation and legacy layers ship in the same `wp-seo`
directory, as separate packages, and each can be replaced.

```
packages/wp-seo/
├── wp-seo-domain/         Domain layer: contracts, entities, value objects, events, exceptions, service container.
├── wp-seo-infra-wp/       Infrastructure layer: WordPress-backed adapters and repositories.
├── wp-seo-infra-memory/   Infrastructure layer: in-memory adapters for tests and non-WordPress hosts.
├── wp-seo-app/            Application layer: use cases that orchestrate the domain through its contracts.
├── wp-seo-legacy/         Legacy layer: existing WordPress functions, rewritten as wrappers over the contracts.
├── wp-seo-presentation/   Presentation layer: REST endpoints for the admin screen (later phase).
├── wp-seo-contract-tests/ Shared Pest contract tests that any implementation runs.
└── wp-seo-indexnow/       Optional extension: IndexNow sender (later phase).
```

Where logic lives:

- **Domain** holds every contract, plus value objects and entities with the behaviour that is intrinsic to them
  (a `Url` validating itself, `RobotsDirectives` merging two sets). It holds no replaceable behaviour.
- **Application** holds the use cases behind the resolver and builder contracts (resolve the title, resolve the
  canonical URL, build the head document). It is framework-free, so a Laravel host reuses it unchanged.
- **Infrastructure** holds everything that touches a host system: reading content and settings, storing overrides,
  dispatching hooks, sending headers.

Putting use-case logic in `wp-seo-app` and not in the domain package is my call, made so that the project owner's
rule holds: contracts in their own package, with no implementation beside them.

### Dependency rules

- `wp-seo-domain` requires only PHP.
- Every other package requires `wp-seo-domain`. `wp-seo-app` requires nothing else. Infrastructure packages do not
  require each other.
- `wp-seo-legacy` and `wp-seo-presentation` require `wp-seo-domain` and resolve everything through the container.
  They never name a concrete class from the application or infrastructure packages.
- `wp-seo-legacy` also requires two virtual packages, `speedyspec/wp-seo-infra-implementation` and
  `speedyspec/wp-seo-app-implementation`, following the `psr/log-implementation` pattern. Each infrastructure
  package provides the first and `wp-seo-app` provides the second, so Composer refuses an install with a layer
  missing, yet any layer can be supplied by another package.

These rules are stricter than the existing packages and the DDD skill, on purpose. In `wp-option`, the
infrastructure package requires the application package and the legacy package requires both concrete packages,
which makes none of them replaceable. The virtual packages are new to this repository.

### How replacement works

`wp-seo-domain` contains `SeoServiceContainer`: a registry keyed by interface name with `add()`, `remove()`, `has()`
and `get()`. It plays the role `HookServiceContainer` plays for hooks but is written fresh, not copied: the
existing containers have no `has()`, are `final` (which the domain layer rules forbid), keep stale instances after
a late `add()`, and `OptionServiceContainer` references the wrong class for its own singleton.

- Each implementation package ships a `ServiceProvider` whose `register()` binds every contract it implements, one
  `add()` call per contract.
- **Partial replacement:** a plugin calls `add(SomeContract::class, ...)` after the defaults are registered. Only
  that contract changes. Because implementations receive their collaborators from the container, the replacement
  is used everywhere.
- **Complete replacement:** install a different package that provides the same virtual package for that layer and
  registers its own bindings; do not install the default. This works per layer: swap the infrastructure, the
  application layer, the legacy wrappers or the presentation layer independently, or all of them.
- Bindings must be final before the first `get()` for that contract. The container caches instances, so a late
  `add()` for an already-resolved contract throws. The existing containers silently keep the stale instance; this
  one must not.
- Every contract has a reusable Pest contract test. These ship in an extra package, `wp-seo-contract-tests`, which
  implementations add as a dev dependency, because package `tests/` directories are excluded from Composer archives
  and the domain package must not depend on Pest. Any implementation, default or third-party, runs the same tests
  to prove it is a valid replacement.

## Contracts

Names are indicative; the builder may adjust signatures while keeping the responsibilities and the one-contract,
one-responsibility split.

Added during planning and implementation of phase 1: `SiteInfoProviderInterface`, `ContentSourceInterface` and
`HeadDocumentBuilderInterface`; an `extend()` method on the container for decorating one binding; and
`PageContext::hasPost()`.

### Shared value objects

`Url` (absolute, validated), `PageContext` (what is being rendered: kind, object identifier, page number, locale,
HTTP status), `RobotsDirectives`, `MetaTag`, `LinkTag`, `HeadDocument`, `SchemaNode`, `SchemaGraph`, `SocialCard`,
`RobotsTxtDocument`, `Redirect`, `SeoOverrides` (per-object title, description, canonical, robots, image).

### Phase 1 contracts: context, canonical, head, robots

| Contract | Responsibility | Default layer |
|---|---|---|
| `PageContextProviderInterface` | Describe the current request as a `PageContext`. | infra-wp |
| `CanonicalUrlResolverInterface` | Resolve the one canonical `Url` for a `PageContext`, or none. | app, using `PermalinkSourceInterface`; infra-wp adds the `get_canonical_url` filter |
| `PermalinkSourceInterface` | Supply the raw URL for a `PageContext`. | infra-wp |
| `SeoOverridesRepositoryInterface` | Read and write per-object overrides. | infra-memory in phase 1; infra-wp in phase 5 |
| `TitleResolverInterface` | Produce the document title. | app; infra-wp binds the WordPress algorithm and filters |
| `DescriptionResolverInterface` | Produce the meta description. | app |
| `RobotsDirectivesResolverInterface` | Produce one `RobotsDirectives` per response. | app; infra-wp binds one that applies the `wp_robots` filter |
| `HeadContributorInterface` | Add or change tags in a `HeadDocument`. Many may be registered. | app ships title, description, robots, canonical contributors |
| `HeadContributorRegistryInterface` | Hold the ordered contributors. | app |
| `HeadRendererInterface` | Turn a `HeadDocument` into HTML. | app |
| `ResponseHeaderEmitterInterface` | Send `X-Robots-Tag` and `Link` headers. | infra-wp |
| `EventDispatcherInterface` | Publish SEO events and apply legacy filters. | infra-wp |

### Later-phase contracts

| Phase | Contracts |
|---|---|
| 2. Structured data and social cards | `SchemaPieceInterface` (`isNeeded`, `generate`), `SchemaPieceRegistryInterface`, `SchemaIdFactoryInterface`, `SchemaGraphBuilderInterface`, `SocialCardResolverInterface`, `SocialImageSourceInterface` |
| 3. robots.txt, sitemaps, hreflang | `RobotsTxtBuilderInterface`, `RobotsTxtContributorInterface`, `SitemapProviderInterface`, `SitemapIndexInterface`, `AlternateLanguageProviderInterface` |
| 4. Redirects and change events | `RedirectRepositoryInterface`, `RedirectMatcherInterface`, `UrlChangeListenerInterface`, event `UrlChanged` |
| 5. Extensions and admin | `VerificationTagRegistryInterface`, `CrawlerCatalogInterface`, IndexNow sender, REST endpoints, admin screen |

## Behaviour decided by the research

These are requirements on the default implementations, not on the contracts.

- **Canonical:** always absolute and HTTPS when the site is HTTPS; paginated views are self-canonical; the same
  resolved value feeds `rel=canonical`, `og:url`, sitemap `<loc>`, hreflang targets and structured data `url`.
- **Robots directives:** an open-ended set, not an enum. `noarchive` and `nocache` are distinct from each other and
  from `nosnippet`, because Bing and Amazon reportedly give them meanings Google does not. Default for public
  content includes `max-image-preview:large`. Emitted as a meta tag for HTML and as `X-Robots-Tag` for everything
  else.
- **Title and description:** no length limits enforced and no meta keywords emitted; no engine documents a limit.
- **Head tags:** `MetaTag` supports both `name` and `property` attributes (Open Graph uses `property`).
- **Structured data (phase 2):** one server-rendered JSON-LD `@graph`; nodes are flat and reference each other by
  stable `@id`. The `@id` scheme is a public contract. Core pieces: `WebSite` (home page only), `Organization` or
  `Person`, `WebPage`, `BreadcrumbList`, `Article`/`BlogPosting`, `ProfilePage`.
- **Open Graph (phase 2):** `og:title`, `og:type`, `og:url`, `og:description`, `og:site_name`, `og:locale`,
  `og:image` with width, height and alt, `article:*` times; a registered 1200x630 image size.
- **robots.txt (phase 3):** per-user-agent groups plus free-form lines inside and outside groups, so unratified
  syntaxes (`Content-Signal:`, `Content-Usage:`, `Clean-param`) need no core change. Output stays under 500 KiB.
- **Sitemaps (phase 3):** only canonical, indexable URLs; `lastmod` from real content modification; no `priority`
  or `changefreq`.
- **Redirects (phase 4):** 301 or 308 by default for slug and URL changes; 404 or 410 for missing content.

Three research items could not be confirmed from primary sources and must be re-read before the phase that depends
on them: Bing's `noarchive`/`nocache` wording (phase 1 treats them only as distinct pass-through values, so it is
not blocked), Google's AIPREF robots.txt syntax (phase 3), and X card requirements (phase 5).

## Backward compatibility

- The functions `wp_get_document_title`, `_wp_render_title_tag`, `wp_robots` and its helpers, `rel_canonical` and
  `wp_get_canonical_url` move from `wp-theme-legacy` into `wp-seo-legacy` and become wrappers that resolve their
  contract from the container. How `wp-seo-legacy` is loaded alongside `wp-theme-legacy` belongs to the site-wide
  package wiring, which does not exist yet (see Constraints).
- Existing filters keep firing with the same arguments: `pre_get_document_title`, `document_title_parts`,
  `document_title_separator`, `document_title`, `wp_robots`, `get_canonical_url`.
- **Where WordPress filters live:** only in `wp-seo-infra-wp`. The application layer never names a WordPress
  filter. `wp-seo-app` holds plain default resolvers for hosts without WordPress; `wp-seo-infra-wp` binds its own
  resolvers that reproduce WordPress's current algorithms, including the points where filters fire mid-way.
- **Robots defaults stay as filter callbacks.** `wp_robots_noindex`, `wp_robots_noindex_search`,
  `wp_robots_max_image_preview_large` and the others remain callbacks on the `wp_robots` filter, so a plugin's
  `remove_filter()` keeps working. The WordPress robots resolver starts empty and applies that filter, exactly as
  today.
- **No new output on existing pages in phase 1, with one exception.** Title, robots and canonical output are
  unchanged; canonical is still emitted only for singular content, though the resolver contract accepts any
  `PageContext`. Widening canonical to archives and paginated views is decided in the phase 3 addendum.
- **The exception is the meta description.** One new function, `wp_seo_render_head()`, hooked to `wp_head`, renders
  the tags that no legacy function prints, which in phase 1 is the description only. A plugin that prints its own
  description removes that action, the same way plugins remove `rel_canonical` today.
- **Each legacy function resolves only its own part, at its own hook time** (amended after the final review of
  phase 1). `_wp_render_title_tag`, `wp_robots` and `rel_canonical` each call their own resolver contract when they
  run, so every legacy filter fires exactly when and as often as it does today. `wp_seo_render_head()` builds a head
  document with the title, robots and canonical contributors skipped. Building one document for the whole head is
  deferred to the phase that collapses the `wp_head` registrations; an eager, cached build was tried and changed
  filter timing and scope.
- **Titles and descriptions are display text, never trusted markup.** Resolvers may return text that already contains
  HTML entities; every output path escapes without double-encoding. `_wp_render_title_tag` therefore prints through
  the renderer contract. Known difference from WordPress: a `pre_get_document_title` filter that returns raw `<`, `&`
  or quotes now has them entity-encoded.
- **Canonical follows post identity, not page kind.** A static front page is a front page that also carries a post,
  so it keeps its canonical link, as today.
- **Known phase 1 limit:** under WordPress, the title and robots resolvers do not consult the overrides repository,
  because they reproduce WordPress's own algorithms. This is revisited when WordPress-backed overrides arrive.
- **Per-object overrides are not read from WordPress storage in phase 1.** The `SeoOverridesRepositoryInterface`
  contract and its in-memory implementation exist; the WordPress-backed repository, its meta keys, and who may set
  a canonical or `noindex` arrive with the admin screen. When they do, overrides apply before legacy filters, so a
  plugin's filter still wins.
- **`ResponseHeaderEmitterInterface` is contract and adapter only in phase 1.** The existing hard-coded
  `X-Robots-Tag` `header()` calls in `wp-support`, `wp-rest-api` and `wp-customizer` are untouched, and no
  `Link: rel=canonical` header is sent.
- Functions outside phase 1 (`redirect_canonical`, `do_robots`, sitemap classes) are untouched until their phase.

## Phasing

This is several projects. Each phase gets its own implementation plan, and phases 2 to 5 each get a short design
addendum before planning, because their details depend on what phase 1 teaches.

1. **Contracts and the head core:** the domain, app, infra-wp, infra-memory, legacy and contract-tests packages scaffolded; shared value objects; phase 1
   contracts; defaults for context, canonical, title, description, robots, head rendering; legacy wrappers;
   contract tests. Meta description becomes available for the first time.
2. **Structured data and social cards.**
3. **robots.txt, sitemaps, hreflang.**
4. **Redirect store and the URL-changed event.**
5. **Extensions and admin:** IndexNow package, verification registry, crawler catalogue, REST endpoints, and the
   SEO screen in `speedyspec-admin-astro` backed by `speedyspec-admin-backend`.

Only phase 1 is planned next.

## Left to the builder

- Exact method signatures and value object internals.
- Whether `wp-seo-domain` defines its own `EventDispatcherInterface` or reuses one from `speedyspec-wp-hook-domain`
  (the contracts package must stay dependency-free either way).
- How the description default is derived when no override exists (excerpt, then trimmed content, is the expected
  starting point).
- Internal structure of each implementation package.

## Deferred from phase 1

Minor findings from the phase 1 reviews that were left unfixed. Each later phase's addendum should check this list.

- **Before phase 2 emits `og:url` or a `Link` header:** `Url::fromString` accepts whitespace and control characters;
  `Url::equals` compares raw strings, so scheme and host case differ.
- **Before phase 2 adds Open Graph tags:** `HeadDocument` keys meta tags by key alone, so a `name` and a `property`
  tag with the same key overwrite each other.
- **Before phase 5 stores overrides:** an override holding an empty string is not treated as "no override"; object
  type names (`post`, `term`, `user`) become storage keys; under WordPress, title and robots do not consult overrides.
- **Skipping contributors:** `BuildHeadDocument` honours the skip list by cloning the registry, which assumes a
  third-party registry is cloneable by value. A registry contract that exposes ids would remove the assumption. A
  missing binding needed only by a skipped contributor still silences `wp_seo_render_head()`.
- **Robots values containing a comma are dropped,** so `unavailable_after` with an RFC 822 date is lost.
- **Container:** no circular-dependency guard in `get()`; a factory returning a non-object is cached before the type
  error.
- **Site info:** `usesHttps()` reads `is_ssl()`, which describes the request, not the site; `name()` and `tagline()`
  throw if a `bloginfo` filter returns a non-string.
- **Tests and style:** a few unpinned cases (override canonical https upgrade, overrides lookup for three resolvers,
  `wp_robots` with services bound but no directives), missing constructor docblocks, unsorted imports, and the
  resolver contract test comparing only `Url` and `RobotsDirectives` by value.
- **Site wiring (out of scope for the package work):** `src/wp-settings.php` still requires the deleted
  `robots-template.php`; `default-filters.php` hooks `wp_seo_render_head` and other legacy packages call the moved
  functions, but no package requires `wp-seo-legacy` yet; nested `.gitignore` files are ignored by the root rule, so
  `vendor/` under `packages/wp-seo/` is only excluded locally.
