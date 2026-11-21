# Google Search: technical requirements and recommendations for a website / CMS (state as of November 2026)

Research date: 2026-11-17. Google's documentation pages do not carry visible "last updated" dates in the fetched content; dates below come from the Search Central documentation changelog (https://developers.google.com/search/updates) or dated blog posts. Short source labels used below:

- [Changelog] = https://developers.google.com/search/updates

## 1. Search Essentials technical requirements and spam policies relevant to a CMS

### Takeaway
Google has only three minimum technical requirements (Googlebot not blocked, HTTP 200, indexable content in a supported file type that does not violate spam policies); meeting them makes a page eligible, never guaranteed. The spam policies a CMS can technically influence are cloaking (paywall exception), hidden text, sneaky redirects, unqualified paid/UGC links, user-generated spam, hacked content, and (new April 2026) back-button hijacking.

### Cited Findings
- The three technical requirements are: "Googlebot isn't blocked", the page returns "HTTP 200 (success) status code", and "the page has indexable content". Meeting them makes a page eligible for indexing but does not guarantee it. — [Search technical requirements](https://developers.google.com/search/docs/essentials/technical)
- Login-protected pages are not crawled. Pages blocked by robots.txt are unlikely to show in results but their URLs can still appear; to keep a page out of the index use `noindex` and allow crawling. — [Search technical requirements](https://developers.google.com/search/docs/essentials/technical)
- Google does not index client-error or server-error pages. Indexable content means text in a supported file type that does not violate spam policies. — [Search technical requirements](https://developers.google.com/search/docs/essentials/technical)
- Cloaking = showing search engines different content from users. Paywalls/content gating are not cloaking if Google can see the full gated content and the site follows Flexible Sampling guidance. — [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- Hidden text/link abuse covers white-on-white text, zero font size, off-screen CSS; accordions, tabs, sliders, tooltips and screen-reader-only text are explicitly allowed. — [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- Sneaky redirects = redirecting users or search engines deceptively to substantially different content; site moves, page consolidation and post-login routing are legitimate. — [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- Link spam: paid/sponsored links do not violate policy if qualified with `rel="nofollow"` or `rel="sponsored"` on the `<a>` element. — [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- User-generated spam (forum posts, comments, account registrations, file uploads) is the site owner's responsibility to control. — [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- Scaled content abuse = mass-generating pages (AI, scraping, automated rewording) mainly to manipulate rankings; if a site hosts such content it should exclude it from Search. — [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- Other listed policies: doorway abuse, site reputation abuse, expired domain abuse, hacked content, thin affiliation, machine-generated traffic. — [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- CHANGED 2024-03-05: expired domain abuse, scaled content abuse and site reputation abuse policies were added. — [Changelog](https://developers.google.com/search/updates)
- CHANGED 2024-11-19: site reputation abuse policy extended to cover third-party content exploitation regardless of first-party involvement; 2025-01-21 guidance clarified; 2026-08-28 policy adjusted for enforcement in the EEA (affected pages may be categorized separately from the main domain rather than receive a manual action). — [Changelog](https://developers.google.com/search/updates); [Spam policies](https://developers.google.com/search/docs/essentials/spam-policies)
- NEW 2026-04-13: a "back button hijacking" policy was added under malicious practices. — [Changelog](https://developers.google.com/search/updates)
- NEW 2026-05-15: spam policies explicitly apply to generative AI responses in Search. — [Changelog](https://developers.google.com/search/updates)
- 2026-04-14 / 2026-04-23: Google may use spam report submissions to take manual action. — [Changelog](https://developers.google.com/search/updates)
- 2026-10-26: the spam-prevention guide added advice to block indexing of link-shared content and monitor abuse in Search Console. — [Changelog](https://developers.google.com/search/updates)
- 2025-08-28: new guidance on JavaScript-based paywall considerations was published. — [Changelog](https://developers.google.com/search/updates)

### Inferences
- A CMS SEO API has to guarantee correct status codes (real 200 only for real content), must not block Googlebot by default, and should default UGC links (comments) to `rel="ugc"`/`nofollow` and offer `sponsored` for paid links.
- CMS features that inject history entries or intercept the back button now carry spam-policy risk (April 2026).
- "Block indexing of link-shared content" (Oct 2026) implies a need for per-URL `noindex` on private/share-link content types.

### Gaps
- The exact wording and scope of the back-button-hijacking policy and the October 2026 spam-prevention additions were only seen as changelog one-liners; the full policy text was not retrieved.
- The list of "supported file types" was not retrieved (changelog notes EPUB added 2024-05-23 and AVIF image support 2024-08-30).

## 2. Meta tags, link tags, attributes and HTTP headers Google supports (and ignores)

### Takeaway
Google supports a small fixed set: `<title>`, meta description, robots/googlebot meta and `X-Robots-Tag` (noindex, nofollow, none, nosnippet, indexifembedded, max-snippet, max-image-preview, max-video-preview, notranslate, noimageindex, unavailable_after), `data-nosnippet`, `rel=canonical`, hreflang, `rel` link qualifiers, viewport, charset, rating, google-site-verification, nopagereadaloud. It ignores meta keywords, the HTML `lang` attribute (for language detection), `rel=next/prev`, `noarchive`, `nocache` and `nositelinkssearchbox`.

### Cited Findings

Supported meta tags / attributes
- Supported meta tags: `description`; `robots` and `googlebot` (defaults `index, follow`; the more restrictive rule wins on conflict); `googlebot` `notranslate`; `google` `nopagereadaloud`; `google-site-verification` (name/content case-sensitive); `Content-Type`/`charset` (UTF-8 recommended); `http-equiv="refresh"` (server-side 301 preferred); `viewport` (signals mobile-friendliness); `rating` (adult content labelling for SafeSearch). — [Meta tags and attributes Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)
- Supported HTML attributes: `src`/`href` for discovery, `rel` link qualifiers, `data-nosnippet` on `div`, `span`, `section`. — [Meta tags and attributes Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)
- Explicitly unsupported/ignored: `<meta name="keywords">` (no effect on indexing or ranking); HTML `lang` attribute (Google detects language from the text); `rel="next"`/`rel="prev"` (no longer used); `<meta name="google" content="nositelinkssearchbox">` (feature no longer exists). — [Meta tags and attributes Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)
- Google ignores unsupported meta tags (harmless). It advises against injecting/changing meta tags with JavaScript where possible. — [Meta tags and attributes Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)

Robots meta / X-Robots-Tag
- Rules: `all` (default, no effect); `noindex`; `nofollow`; `none` (= noindex, nofollow); `nosnippet` (no text snippet or video preview; static thumbnail may still show); `indexifembedded` (only works together with `noindex`; allows indexing of content embedded via iframe); `max-snippet:[n]` (0 = nosnippet, -1 = no limit); `max-image-preview:[none|standard|large]`; `max-video-preview:[n]` seconds (0 = static image at most, -1 = no limit); `notranslate`; `noimageindex`; `unavailable_after:[date]` (RFC 822, RFC 850, ISO 8601 accepted; crawl rate also reduced after the date). — [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)
- Deprecated/ignored: `noarchive` (cached links no longer exist; moved to historical section 2024-10-02), `nocache` (not used), `nositelinkssearchbox` (archived 2024-11-29). — [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag); [Changelog](https://developers.google.com/search/updates)
- Rules can be comma-separated or split across multiple meta tags; on conflict the more restrictive rule applies (e.g. `nosnippet` beats `max-snippet:50`); a generic rule plus a Googlebot-specific rule are combined for Googlebot. Crawler tokens accepted in the meta `name` are `googlebot` and `googlebot-news`; `X-Robots-Tag` can prefix a user agent. — [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)
- Robots meta and `X-Robots-Tag` are only seen if the URL is crawled; if robots.txt disallows the URL, the rules are ignored. — [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)
- `X-Robots-Tag` carries the same rules for non-HTML files. — [Meta tags and attributes Google supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)
- `nosnippet` applies to AI Overviews and AI Mode and prevents content being used as direct input; `max-snippet` limits how much content may be used as direct input (limit does not apply where the publisher separately granted permission, e.g. via structured data or licence). AI Mode information was added to this page 2025-03-05. — [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag); [Changelog](https://developers.google.com/search/updates)
- 2026-03-24: a note was added on how robots meta tags outside the HTML `<head>` are processed (documented as no behaviour change). — [Changelog](https://developers.google.com/search/updates)
- `data-nosnippet`: valid only on `span`, `div`, `section`; boolean attribute (value ignored); HTML must be valid/closed; must not be added or removed via JavaScript (extraction can happen before and after rendering); structured data inside it still works. — [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)

Title links
- Sources Google uses for title links: `<title>`, main visual title, headings such as `<h1>`, `og:title` (added as a documented source 2024-08-26), other prominent text, anchor text on the page, text of links pointing to the page, `WebSite` structured data. — [Title links](https://developers.google.com/search/docs/appearance/title-link); [Changelog](https://developers.google.com/search/updates)
- Best practices: every page has a `<title>`; descriptive, concise, unique per page; no boilerplate or keyword stuffing; concise branding with a delimiter; make the main title the most prominent `<h1>`; title language/script must match content. No length limit is specified; title links are truncated to device width. — [Title links](https://developers.google.com/search/docs/appearance/title-link)
- Google rewrites when titles are half-empty, obsolete (old date), inaccurate, micro-boilerplate, lack a clear main heading, mismatch language/script, or duplicate the site name. — [Title links](https://developers.google.com/search/docs/appearance/title-link)

Snippets / meta description
- Snippets are built primarily from page content (clarified 2024-01-05); the meta description is used when it describes the page better. No length limit for meta descriptions; snippets truncated to device width. Controls: `nosnippet`, `max-snippet`, `data-nosnippet`. — [Snippets](https://developers.google.com/search/docs/appearance/snippet); [Changelog](https://developers.google.com/search/updates)
- Unique descriptions per page are recommended; programmatically generated descriptions are acceptable if human-readable and diverse. — [Snippets](https://developers.google.com/search/docs/appearance/snippet)
- NEW 2026-04-20: snippet docs added a "Read more" deep-links section: keep content visible (not in hidden tabs/expandable sections), do not force scroll position on load, do not strip the hash fragment from the URL. — [Snippets](https://developers.google.com/search/docs/appearance/snippet); [Changelog](https://developers.google.com/search/updates)

rel=canonical
- `rel="canonical"` is supported as an HTML `<link>` in `<head>` (ignored elsewhere) or an HTTP `Link` header (web search only; suited to PDFs). Use absolute URLs; self-referential canonical recommended; use one method, not both; never a fragment URL. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
- `rel="canonical"` annotations combined with `hreflang`, `lang`, `media` or `type` attributes are ignored for canonicalization (documented 2024-02-15). — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls); [Changelog](https://developers.google.com/search/updates)

hreflang
- Three equivalent methods: HTML `<link rel="alternate" hreflang>` in `<head>`, HTTP `Link` header, XML sitemap `<xhtml:link>`; using more than one gives no benefit. — [Localized versions](https://developers.google.com/search/docs/specialty/international/localized-versions)
- Requirements: each version lists itself and all others; return links are mandatory (non-reciprocal annotations are ignored); fully qualified URLs including protocol; cross-domain allowed; `x-default` for fallback/language-selector pages. — [Localized versions](https://developers.google.com/search/docs/specialty/international/localized-versions)
- Codes: ISO 639-1 language, optional ISO 3166-1 alpha-2 region, ISO 15924 script (e.g. `zh-Hant`); region alone is invalid; `es-419` unsupported; `EU`, `UN`, `UK` ignored (use `GB`). — [Localized versions](https://developers.google.com/search/docs/specialty/international/localized-versions)
- Google does not use hreflang or the HTML `lang` attribute to detect page language. — [Localized versions](https://developers.google.com/search/docs/specialty/international/localized-versions)
- 2024-06-12: clarified that `link` attributes for hreflang must not be combined in one tag. — [Changelog](https://developers.google.com/search/updates)

Link qualifiers
- `rel="sponsored"` for ads/paid placements (preferred; `nofollow` still acceptable); `rel="ugc"` for user-generated content; `rel="nofollow"` when neither applies. Values can be combined, space- or comma-separated (`rel="ugc nofollow"`). Links so marked "will generally not be followed". For internal links Google recommends robots.txt disallow rather than nofollow. — [Qualify outbound links](https://developers.google.com/search/docs/crawling-indexing/qualify-outbound-links)

Pagination
- Give each page a unique URL (e.g. `?page=n`); each page gets its own canonical (do not canonicalize all to page 1); link pages sequentially with `<a href>`; do not use `#` fragments for page numbers; `rel=next/prev` is no longer used by Google; filter/sort variants can be kept out with `noindex` or robots.txt. — [Pagination and incremental page loading](https://developers.google.com/search/docs/specialty/ecommerce/pagination-and-incremental-page-loading)
- Google only follows `<a href>` links; it does not click "load more" buttons or trigger scroll-driven JS requiring user action. — [Pagination and incremental page loading](https://developers.google.com/search/docs/specialty/ecommerce/pagination-and-incremental-page-loading)

### Inferences
- A sensible CMS default robots output for public content is `max-image-preview:large` (needed for Discover, see section 6) with optional `max-snippet:-1, max-video-preview:-1`; these are the only preview levers and, since 2025, the same levers bound AI Overviews/AI Mode usage.
- The CMS should emit canonical, hreflang and robots in server-rendered `<head>` HTML, never via JS, and offer `X-Robots-Tag` for attachments/PDFs/feeds.
- No reason to emit meta keywords, `rel=next/prev` (for Google), `noarchive` or `nositelinkssearchbox`; other engines may still use some of these (out of scope here).

### Gaps
- No documented maximum lengths exist for title or meta description; pixel/character limits circulated in the industry are not Google-documented.
- The details of the March 2026 note on robots meta tags outside `<head>` were not retrieved.

## 3. robots.txt, sitemaps, redirects and status codes

### Takeaway
robots.txt follows RFC 9309 with a 500 KiB limit and only four fields (user-agent, allow, disallow, sitemap); XML sitemaps are capped at 50,000 URLs / 50 MB uncompressed, with `priority` and `changefreq` ignored and `lastmod` used only if accurate; the ping endpoint is dead. 301/308 are canonical signals, 302/303/307 are not; 4xx (except 429) removes content, 429 and 5xx slow crawling.

### Cited Findings

robots.txt
- Google follows the Robots Exclusion Protocol as in RFC 9309. File must sit at the host root; applies per protocol + host + port; subdomains need their own. — [How Google interprets robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt)
- Must be UTF-8 plain text; size limit 500 KiB, content beyond is ignored. — [How Google interprets robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt)
- Supported fields: `user-agent`, `allow`, `disallow`, `sitemap`. Unsupported fields such as `crawl-delay` are ignored (clarified 2024-10-07). Sitemap URLs must be absolute and may be on another host. — [How Google interprets robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt); [Changelog](https://developers.google.com/search/updates)
- Matching: `*` wildcard and `$` end anchor; paths case-sensitive; longest matching rule wins; on allow/disallow tie the least restrictive applies; a crawler uses only the most specific matching user-agent group (not merged with `*`). — [How Google interprets robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt)
- Status handling for the robots.txt fetch: 2xx processed; 3xx followed for at least five hops then treated as 404; 4xx (except 429) treated as no robots.txt (no restrictions); 5xx: crawling halts for the first 12 hours, cached copy used up to 30 days, after 30 days treated as no robots.txt if the site is otherwise available. Error handling docs consolidated 2024-12-18. — [How Google interprets robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt); [Changelog](https://developers.google.com/search/updates)
- robots.txt is cached generally up to 24 hours; `Cache-Control: max-age` can influence this. — [How Google interprets robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt)
- robots.txt must not be used for canonicalization or to keep pages out of the index; a disallowed URL can still be indexed. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls); [Search technical requirements](https://developers.google.com/search/docs/essentials/technical)

Sitemaps
- Formats: XML, RSS 2.0, Atom 1.0, mRSS (video), plain text. Limits: 50 MB uncompressed or 50,000 URLs per file; split and optionally use a sitemap index. UTF-8, entity-escaped, fully-qualified absolute URLs, only canonical URLs. — [Build and submit a sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap)
- A sitemap not submitted through Search Console only affects URLs at or below its directory; root placement recommended. — [Build and submit a sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap)
- Google ignores `<priority>` and `<changefreq>`. `<lastmod>` is used only if consistently and verifiably accurate, and should reflect significant changes (main content, structured data, links), not e.g. a copyright year. — [Build and submit a sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap)
- Submission: Search Console, Search Console API, `Sitemap:` line in robots.txt, WebSub for feeds. Submission is a hint only. — [Build and submit a sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap)
- DEPRECATED: the sitemap ping endpoint was announced deprecated 2023-06-26, to stop working within 6 months (i.e. by about end of 2023) and return 404; the post is now bannered as complete. Existing ping code is harmless but useless. — [Sitemaps ping endpoint is going away](https://developers.google.com/search/blog/2023/06/sitemaps-lastmod-ping)
- Sitemap inclusion is a weak canonical signal. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
- Image sitemap extension: namespace `http://www.google.com/schemas/sitemap-image/1.1`; only `<image:image>` and `<image:loc>` remain; up to 1,000 images per `<url>`; `<image:caption>`, `<image:geo_location>`, `<image:title>`, `<image:license>` were deprecated (announcement dated May 2022). — [Image sitemaps](https://developers.google.com/search/docs/crawling-indexing/sitemaps/image-sitemaps)
- News sitemap extension: namespace `http://www.google.com/schemas/sitemap-news/0.9`; required `<news:publication>` (`<news:name>`, `<news:language>`), `<news:publication_date>` (W3C format), `<news:title>`; only articles from the last two days; max 1,000 `<news:news>` entries per sitemap; update the same file rather than creating new ones. — [News sitemaps](https://developers.google.com/search/docs/crawling-indexing/sitemaps/news-sitemap)
- hreflang can be expressed in sitemaps via `<xhtml:link>`. — [Localized versions](https://developers.google.com/search/docs/specialty/international/localized-versions)

Redirects
- Permanent (target becomes canonical): HTTP 301, HTTP 308, instant (0 s) meta refresh, 0 s HTTP `Refresh` header; JavaScript `location` redirects are treated as permanent but only seen if rendering succeeds; "crypto" redirects (a link + notice) are last resort. — [Redirects and Google Search](https://developers.google.com/search/docs/crawling-indexing/301-redirects)
- Temporary (followed, not a canonical signal): HTTP 302, 303, 307, delayed (>0 s) meta refresh / `Refresh` header. — [Redirects and Google Search](https://developers.google.com/search/docs/crawling-indexing/301-redirects)
- Googlebot follows up to 10 redirect hops. — [HTTP status codes and network errors](https://developers.google.com/search/docs/crawling-indexing/http-network-errors)

Status codes
- 200/201 content processed; 202 waits limited time; 204 nothing to process; 2xx pages that look like errors or are empty are reported as soft 404. — [HTTP status codes and network errors](https://developers.google.com/search/docs/crawling-indexing/http-network-errors)
- 301/308 strong signal; 302/303/307 weak signal; 304 = unchanged, no indexing effect. — [HTTP status codes and network errors](https://developers.google.com/search/docs/crawling-indexing/http-network-errors)
- 400, 401, 403, 404, 410, 411 are treated alike: content ignored, URL removed from the index if indexed, crawl frequency gradually reduced. 4xx other than 429 does not affect crawl rate; do not use 401/403 to limit crawling. — [HTTP status codes and network errors](https://developers.google.com/search/docs/crawling-indexing/http-network-errors)
- 429 is treated as a server-overload signal, like a server error. 500/502/503 slow crawling in proportion to the number of failing URLs; URLs that persistently error are eventually dropped from the index. — [HTTP status codes and network errors](https://developers.google.com/search/docs/crawling-indexing/http-network-errors)
- 2025-11-20 and 2025-12-18: crawling documentation (crawl budget, faceted navigation, HTTP status codes, DNS errors, crawler lists) was migrated to a separate Google crawling documentation site (developers.google.com/crawling). — [Changelog](https://developers.google.com/search/updates)

Crawler infrastructure
- Google crawlers by default process only the first 15 MB of a file; individual crawlers may set other limits, with the docs giving "a smaller size limit (for example, 2MB)" as a Googlebot example; PDFs may get a larger limit. File size limits were moved to the crawler docs and made more precise 2026-02-03. — [Overview of Google crawlers](https://developers.google.com/crawling/docs/crawlers-fetchers/overview-google-crawlers); [Changelog](https://developers.google.com/search/updates)
- Crawlers support HTTP/1.1 and HTTP/2, gzip/deflate/Brotli, `ETag`/`If-None-Match` and `Last-Modified`/`If-Modified-Since` (ETag preferred), and optional `Cache-Control: max-age`. Crawling is mainly from US IPs. — [Overview of Google crawlers](https://developers.google.com/crawling/docs/crawlers-fetchers/overview-google-crawlers)

### Inferences
- A CMS should emit `lastmod` only from a real content-modified timestamp, omit `priority`/`changefreq` (or accept they are no-ops for Google), split at 50,000 URLs, and advertise sitemaps via robots.txt; any "ping Google" code is dead.
- 404 and 410 are equivalent for Google per the current doc; a CMS need not distinguish them for Google's sake.
- For maintenance mode, 503 (or 429) is the correct signal; a 200 maintenance page risks soft-404 or indexing of the notice.
- Supporting conditional requests (ETag / Last-Modified, 304) is directly beneficial for crawl efficiency.
- Keep rendered HTML well below the Googlebot size limit; the docs' "2MB" example suggests the effective HTML limit for Googlebot may be much lower than the generic 15 MB.

### Gaps
- The dedicated soft-404 page (old URL) returned HTTP 404 after the docs migration; detailed soft-404 guidance was not retrieved beyond the status-code page.
- The exact Googlebot-specific size limits (HTML vs PDF) could not be confirmed: the dedicated Googlebot page at the guessed new URL returned 404; only the overview's "for example, 2MB" wording was seen.
- How Google handles 429 specifically on the robots.txt fetch is not described on the robots.txt page (only excluded from the general 4xx rule).
- Video sitemap extension rules were not retrieved.
- Network/DNS error handling details and removal timeframes are not specified in the fetched content ("eventually").

## 4. Canonical URL selection, duplicates, URL parameters, trailing slashes, HTTPS, site moves

### Takeaway
Canonical selection is signal-based: redirects (strong), `rel=canonical` (strong), sitemap inclusion (weak), plus HTTPS preference and hreflang-cluster membership; Google may override. URLs should follow RFC 3986 with hyphens, minimal `key=value&` parameters and no content-changing fragments; site moves need permanent server-side redirects kept at least one year.

### Cited Findings
- Signal strength: permanent redirects (strong) > `rel="canonical"` link element/HTTP header (strong) > sitemap inclusion (weak); signals stack. Without signals Google picks what it considers the best version. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
- Google prefers HTTPS over HTTP as canonical unless there are conflicts: invalid certificate, insecure non-image dependencies, HTTPS-to-HTTP redirect, or HTTPS page canonicalizing to HTTP. Keep HTTP URLs out of sitemaps and hreflang. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
- URLs in a reciprocal hreflang cluster are preferred as canonical; the canonical should be in the same language. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
- Don'ts: robots.txt for canonicalization; URL removal tool; `noindex` to choose a canonical within a site; conflicting canonicals across methods; fragment URLs as canonical. Link internally to the canonical consistently. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
- Canonical should be in the HTML source; if JS sets it, it must match and be the only canonical, and JS should not change a server-set canonical (JS canonical best practices added 2025-12-17). — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls); [Changelog](https://developers.google.com/search/updates)
- A canonical can consolidate tracking-parameter URLs (e.g. `gclid`) to the clean URL. — [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)
- URL structure: follow RFC 3986 (STD 66); percent-encode non-ASCII; hyphens not underscores (clarified again 2026-10-23); URLs are case-sensitive, normalize to one case; parameters as `key=value` joined by `&`, multiple values comma-separated; as few parameters as possible; no fragments to change content (use History API). URL parameter best practice added 2024-10-23; docs reorganized 2025-06-18. — [URL structure best practices](https://developers.google.com/search/docs/crawling-indexing/url-structure); [Changelog](https://developers.google.com/search/updates)
- Common crawl problems: additive filter combinations, referral/sort parameters, session IDs, infinite calendars, broken relative links. Fixes: cookies not session IDs, `nofollow` on infinite calendar links, root-relative links, correct status codes for nonexistent pages, robots.txt blocks for search results/filter/sort URLs. A separate faceted-navigation crawling guide was published 2024-12-17. — [URL structure best practices](https://developers.google.com/search/docs/crawling-indexing/url-structure); [Changelog](https://developers.google.com/search/updates)
- Site moves with URL changes: use server-side 301/308; redirect to final destination (Googlebot follows up to 10 hops; aim for 3 or fewer, always under 5); do not mass-redirect to the home page; keep redirects at least 1 year; permanent redirects do not lose PageRank; update self-referencing canonicals, hreflang, sitemaps, internal links; if there is no robots.txt return a proper 404. — [Site moves with URL changes](https://developers.google.com/search/docs/crawling-indexing/site-move-with-url-changes)
- Change of Address tool is for domain/subdomain changes only, not HTTP-to-HTTPS or path changes; the guide was updated 2026-06-17 to cover Change of Address for www and non-www variants. — [Site moves with URL changes](https://developers.google.com/search/docs/crawling-indexing/site-move-with-url-changes); [Changelog](https://developers.google.com/search/updates)
- 2026-07-10: canonicalization troubleshooting guide clarified re-evaluation timing. — [Changelog](https://developers.google.com/search/updates)

### Inferences
- A CMS should emit exactly one absolute, self-referential `rel=canonical` per indexable page, server-side, on the HTTPS URL, and keep sitemap, hreflang and internal links consistent with it; inconsistency weakens the signal.
- A CMS redirect manager should default to 301 (or 308) for permanent slug/URL changes and retain them indefinitely.
- Trailing-slash and case variants are separate URLs to Google; the CMS should pick one form and 301 the other.

### Gaps
- The fetched Google pages do not address trailing slashes directly (the URL-structure and canonicalization pages are silent); the inference above rests on general URL-equivalence rules, not a cited Google statement.
- HSTS is not mentioned on the site-move page; the linked HTTPS best-practices page was not retrieved.
- The legacy Search Console URL Parameters tool's removal (2022) was not re-verified here.

## 5. Core Web Vitals, page experience, mobile-first indexing, JavaScript rendering

### Takeaway
Core Web Vitals (LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1 at the 75th percentile) are the only page-experience aspect Google says ranking systems use; there is no single page-experience signal. Mobile-first indexing has been complete since October 2023, and Google renders JavaScript with evergreen Chromium but may skip rendering for `noindex` or non-200 pages.

### Cited Findings
- Thresholds: LCP within 2.5 s; INP under/at 200 ms; CLS below/at 0.1. — [Understanding Core Web Vitals and Google Search](https://developers.google.com/search/docs/appearance/core-web-vitals); [web.dev Web Vitals](https://web.dev/articles/vitals)
- Assessment is at the 75th percentile of page loads, segmented by mobile and desktop; a page passes if all three metrics meet targets. — [web.dev Web Vitals](https://web.dev/articles/vitals)
- INP replaced FID as a Core Web Vital in 2024; Google's docs were updated 2024-03-12. — [web.dev Web Vitals](https://web.dev/articles/vitals); [Changelog](https://developers.google.com/search/updates)
- Google "highly recommend[s] site owners achieve good Core Web Vitals for success with Search". — [Understanding Core Web Vitals and Google Search](https://developers.google.com/search/docs/appearance/core-web-vitals)
- There is no single "page experience signal"; core ranking systems look at several signals. Beyond Core Web Vitals, other page-experience aspects do not directly help ranking. Good CWV does not guarantee top rankings; relevance dominates. — [Page experience](https://developers.google.com/search/docs/appearance/page-experience)
- Page-experience self-assessment covers HTTPS, mobile display, no intrusive interstitials, no excessive ads, distinguishable main content. — [Page experience](https://developers.google.com/search/docs/appearance/page-experience)
- Mobile-first indexing was declared complete 2023-10-31; a very small set of sites that do not work on mobile is still crawled with desktop Googlebot. — [Mobile-first indexing has landed](https://developers.google.com/search/blog/2023/10/mobile-first-is-here)
- Google uses the mobile version crawled with the smartphone agent for indexing and ranking. Mobile and desktop must match on: robots meta tags, primary content, headings, structured data, titles and meta descriptions, image alt text, robots.txt rules; primary content must not be lazy-loaded on user interaction. Responsive design is recommended. — [Mobile-first indexing best practices](https://developers.google.com/search/docs/crawling-indexing/mobile/mobile-sites-mobile-first-indexing)
- Pages go missing from the index if the mobile version serves an error, uses fragment URLs, is `noindex`, or is blocked by robots.txt. — [Mobile-first indexing best practices](https://developers.google.com/search/docs/crawling-indexing/mobile/mobile-sites-mobile-first-indexing)
- JavaScript: crawl, render, index phases; rendering uses an evergreen Chromium; pages with 200 status are queued for rendering and rendering may be skipped for non-200 pages (note added 2025-12-18). — [JavaScript SEO basics](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics); [Changelog](https://developers.google.com/search/updates)
- Google may skip rendering/JS execution when it sees `noindex` in the original HTML, so removing `noindex` via JS may not work (clarified 2025-12-15). — [JavaScript SEO basics](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics); [Changelog](https://developers.google.com/search/updates)
- Links are discovered only from `<a>` elements with `href`; use History API routing not fragments; SPAs should redirect to a real 404 URL or inject `noindex` for error views. — [JavaScript SEO basics](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics)
- Googlebot caches aggressively and may ignore caching headers; content fingerprinting of JS/CSS filenames is recommended. Server-side rendering or pre-rendering is recommended. — [JavaScript SEO basics](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics)
- Dynamic rendering was marked a deprecated workaround on 2024-02-06. — [Changelog](https://developers.google.com/search/updates)
- 2026-07-01: outdated AMP viewer, AMP Cache and signed-exchange references were removed; Search links directly to publisher AMP pages. — [Changelog](https://developers.google.com/search/updates)

### Inferences
- SEO-critical tags (title, robots, canonical, hreflang, structured data) should be in server-rendered HTML; a CMS that emits `noindex` initially and removes it client-side risks never being indexed.
- A CMS must emit a viewport meta tag and identical head metadata for all device classes (responsive output satisfies this by construction).
- CWV is mostly a theme/front-end concern, but the SEO layer influences it through image dimension attributes (CLS), LCP image priority and script injection.

### Gaps
- "Needs improvement" / "poor" boundary values (commonly cited as LCP 4 s, INP 500 ms, CLS 0.25) appear only in images on web.dev and were not textually confirmed in this research.
- A cited July 2024 announcement that sites not accessible on mobile would stop being indexed was not verified in this session.
- No evidence was found of any change to CWV metrics or thresholds in 2025-2026; the web.dev page's last update shown was 2024-10-31.

## 6. Site names, favicons, sitelinks search box, breadcrumbs, dates/bylines, Discover and News eligibility

### Takeaway
Site name needs `WebSite` structured data (`name`, `url`, optional `alternateName`) on the domain/subdomain home page only; favicon needs a `<link rel="icon">` on the home page with a square image (min 8x8, recommended >48x48). Sitelinks search box is gone (Nov 2024), breadcrumb rich results are desktop-only (Jan 2025), and Discover needs large images (≥1200 px wide) enabled via `max-image-preview:large`.

### Cited Findings

Site names
- `WebSite` structured data on the home page is the most important signal; required `name` and `url`, recommended `alternateName` (ordered by preference); JSON-LD, RDFa or microdata. Google also considers `og:site_name`, `<title>`, headings and other home page text. — [Site names](https://developers.google.com/search/docs/appearance/site-names)
- Supported at domain and subdomain level only, not subdirectories; one site name per domain/subdomain; `www` and `m.` are treated as the domain-level home page. — [Site names](https://developers.google.com/search/docs/appearance/site-names)
- Duplicate home pages (HTTP/HTTPS, www/non-www) need identical structured data; the home page must be crawlable (not blocked by robots.txt, noindex or login). Fallback: add the lowercase domain as last `alternateName`. The Rich Results Test does not support site names. — [Site names](https://developers.google.com/search/docs/appearance/site-names)

Favicons
- Add `<link rel="icon" href="...">` in the home page header. Supported `rel` values: `icon`, `shortcut icon`, `apple-touch-icon`, `apple-touch-icon-precomposed` (rel values defined 2024-04-24). — [Favicon in Search](https://developers.google.com/search/docs/appearance/favicon-in-search); [Changelog](https://developers.google.com/search/updates)
- Image must be square (1:1), minimum 8x8 px, recommended larger than 48x48 px (requirement wording updated October 2024). Formats explicitly listed since 2026-08-28: BMP, GIF, ICO, PNG, JPEG, PPM, TIFF. — [Favicon in Search](https://developers.google.com/search/docs/appearance/favicon-in-search); [Changelog](https://developers.google.com/search/updates)
- One favicon per hostname; subdirectory-level home pages unsupported; home page must be crawlable by Googlebot and the icon file by Googlebot-Image; favicon URL should be stable; href may be relative or absolute and may be on a CDN. — [Favicon in Search](https://developers.google.com/search/docs/appearance/favicon-in-search)

Sitelinks search box
- DEPRECATED: on 2024-11-29 Google removed the sitelinks search box documentation, stating the feature "is no longer available in Google Search results", and archived the `nositelinkssearchbox` rule. — [Changelog](https://developers.google.com/search/updates); [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)

Breadcrumbs
- Breadcrumb rich result is available on desktop only (changed 2025-01-22). `BreadcrumbList` needs `itemListElement` with at least two `ListItem`s, each with `position`, `name` and `item` (`item` optional on the last). — [Breadcrumb structured data](https://developers.google.com/search/docs/appearance/structured-data/breadcrumb); [Changelog](https://developers.google.com/search/updates)

Dates and bylines
- To influence the byline date: show a prominent, labelled visible date ("Published…", "Last updated…") and add `datePublished` and/or `dateModified` on a `CreativeWork` subtype (Article, BlogPosting, etc.) in ISO 8601 with timezone. Visible and structured dates must agree; no future dates; if Google picks wrong dates, remove other dates from the page. Not guaranteed to display. — [Publication dates](https://developers.google.com/search/docs/appearance/publication-dates)

Discover
- Eligibility: indexed and compliant with Discover content policies; no special tags or structured data required. — [Discover](https://developers.google.com/search/docs/appearance/google-discover)
- Images: at least 1200 px wide, more than 300,000 total pixels, 16:9 aspect ratio, enabled by `max-image-preview:large` (or AMP). Specify the preferred image with schema.org markup or `og:image` (preferred-image guidance added 2026-03-02); avoid site logos and text-heavy images. — [Discover](https://developers.google.com/search/docs/appearance/google-discover); [Changelog](https://developers.google.com/search/updates)
- Titles must not be clickbait or misleading. — [Discover](https://developers.google.com/search/docs/appearance/google-discover)
- REMOVED 2025-11-19: Discover "Follow" feature guidance (RSS/Atom feed linking) was removed from the docs. — [Changelog](https://developers.google.com/search/updates)
- 2026-02-05: guidance added for a February 2026 Discover Core Update. — [Changelog](https://developers.google.com/search/updates)

News and sources
- `Googlebot-News` is a robots.txt token (no separate user agent) affecting Google News (news.google.com and the app); description corrected 2025-04-25 to say it has no effect on the News tab in Search. — [Google common crawlers](https://developers.google.com/crawling/docs/crawlers-fetchers/google-common-crawlers); [Changelog](https://developers.google.com/search/updates)
- News sitemaps (see section 3) cover only the last two days and max 1,000 entries. — [News sitemaps](https://developers.google.com/search/docs/crawling-indexing/sitemaps/news-sitemap)
- NEW 2026: "Preferred sources" documentation for site owners (2026-01-30), available in all Search languages (2026-04-30), in AI Overviews and AI Mode (2026-05-27), with a custom interactive button (2026-08-20). A "Search profile badge" guide was added 2026-09-16. — [Changelog](https://developers.google.com/search/updates)

### Inferences
- A CMS SEO API should output, on the front page only: `WebSite` JSON-LD (`name`, `alternateName`, `url`), `og:site_name`, and a `rel="icon"` link to a stable square icon ≥ 48 px (larger is safe).
- `SearchAction` / sitelinks-search-box markup no longer has any Google function.
- BreadcrumbList is still worth emitting (desktop), but mobile results will not show it.
- Emitting `max-image-preview:large` by default plus a ≥1200 px `og:image` is the concrete Discover prerequisite a CMS controls.
- Any CMS feature built to support Discover's Follow via feed `<link>` is no longer documented by Google.

### Gaps
- Google News eligibility/content-policy details (Publisher Center help) were not retrieved; I found no fetched primary source describing technical News inclusion requirements beyond the sitemap and crawler token.
- The "Preferred sources" and "Search profile badge" pages were only seen as changelog entries; what markup/button a site must emit was not retrieved.
- Whether favicon size must be a multiple of 48 px: the current page no longer says so (it says recommended larger than 48x48); the older "multiple of 48px" wording appears to be superseded as of October 2024.

## 7. What changed 2024-2026 (deprecations, AI controls, new directives)

### Takeaway
The biggest 2026 change is a site-level Search Console "Search generative AI" opt-out for AI Overviews / AI Mode (announced 2026-06-03, global by 2026-08-31), which sits alongside page-level `nosnippet` / `max-snippet` / `data-nosnippet`; no new meta directive was introduced and Google states llms.txt is not used. Many structured-data features were retired (sitelinks search box 2024, seven types in 2025, practice problems and FAQ rich results in 2026).

### Cited Findings

AI features and controls
- To appear in AI Overviews / AI Mode a page must be indexed and snippet-eligible; "There are no additional technical requirements", and no special markup, schema or AI text files are needed. — [AI features and your website](https://developers.google.com/search/docs/appearance/ai-features)
- Page-level controls for AI features are the existing preview controls: `nosnippet`, `data-nosnippet`, `max-snippet`, and `noindex`. — [AI features and your website](https://developers.google.com/search/docs/appearance/ai-features); [Robots meta tag specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)
- `Google-Extended` is a robots.txt token only (no separate user agent). It controls use of content for training future Gemini models and for grounding in Gemini Apps and Grounding with Google Search on Vertex AI. It does not affect inclusion in Google Search and is not a ranking signal. (Description clarified 2024-02-08 and 2025-04-25.) — [Google common crawlers](https://developers.google.com/crawling/docs/crawlers-fetchers/google-common-crawlers); [Changelog](https://developers.google.com/search/updates)
- Google states it intends to support the AIPREF training preference signal (`user-agent: __usage_ai-training`) in early 2027. — [Google common crawlers](https://developers.google.com/crawling/docs/crawlers-fetchers/google-common-crawlers)
- NEW 2026-06-03: Google announced a Search Console toggle letting site owners choose whether their site appears in and grounds generative AI Search features (AI Overviews, AI Mode, AI Overviews in Discover). Opted-out sites get no impressions or traffic from those features. Google: "This control will not be used as a ranking signal for search results outside of these generative AI Search features." Initially a UK subset; an update dated 2026-08-31 says it is rolled out to all websites worldwide. A generative AI performance report was added to Search Console at the same time. — [Google blog: New controls for website owners](https://blog.google/products-and-platforms/products/search/new-controls-website-owners)
- Google's "Control what you share" page (updated 2026-10-21) lists the "Search generative AI control" as a site-wide control set in Search Console, alongside robots meta, `data-nosnippet` and `X-Robots-Tag`. — [Control what you share with Google](https://developers.google.com/search/docs/crawling-indexing/control-what-you-share); [Changelog](https://developers.google.com/search/updates)
- Secondary coverage reports the setting has Include (default) / Exclude / "Inherit from parent" options per Search Console property, and that it followed a UK CMA decision; it does not affect the Gemini app. — [Herdl](https://herdl.com/search-generative-ai-controls-rolling-out-beyond-the-uk/); [9to5Google](https://9to5google.com/2026/06/02/google-ai-mode-overviews-opt-out/)
- Google's generative-AI optimization guide (published 2026-05-15) lists as a requirement that the site be included in Search generative AI features in Search Console, and says llms.txt and similar AI-specific files are not used by Google Search (also changelog 2026-06-15), no special schema or Markdown is needed, and content "chunking" is unnecessary. — [Optimizing for generative AI features](https://developers.google.com/search/docs/fundamentals/ai-optimization-guide); [Changelog](https://developers.google.com/search/updates)
- AI Overviews / AI Mode traffic is counted in Search Console Performance under "Web" (AI Mode counted since 2025-06-16). — [AI features and your website](https://developers.google.com/search/docs/appearance/ai-features); [Changelog](https://developers.google.com/search/updates)
- 2025-05-21: Google published dedicated docs on AI features; 2024-05-14: first AI Overviews page. — [Changelog](https://developers.google.com/search/updates)

Structured data / feature deprecations (site-level view only)
- 2024-06-11 home activity docs removed; 2024-08-09 Notes removed; 2024-09-24 `cache:` operator docs removed; 2024-10-02 `noarchive` historical; 2024-11-29 sitelinks search box removed. — [Changelog](https://developers.google.com/search/updates)
- 2025-06-12 phase-out banners for book actions, course info, estimated salary, ClaimReview, learning video, special announcement, vehicle listing; 2025-09-09 docs for course info, estimated salary, learning video, special announcement and vehicle listing deleted (special announcement deprecated effective 2025-07-31). — [Changelog](https://developers.google.com/search/updates)
- 2025-11-05 practice problem phased out (docs removed 2026-01-06); Dataset markup clarified as used for Dataset Search only. — [Changelog](https://developers.google.com/search/updates)
- 2026-05-08: FAQ rich result deprecation notice — "This feature will no longer appear in Google Search starting May 7, 2026"; documentation removed 2026-06-15. — [Changelog](https://developers.google.com/search/updates)
- 2025-01-22 breadcrumbs desktop-only. 2026-07-14 package tracking early adopters programme closed to new partners. — [Changelog](https://developers.google.com/search/updates)

Other 2024-2026 changes relevant to a CMS
- 2024-03-12 INP replaced FID. 2024-02-06 dynamic rendering deprecated as a workaround. — [Changelog](https://developers.google.com/search/updates)
- 2024-02-15 `rel=canonical` with hreflang/lang/media/type attributes ignored. 2024-08-26 `og:title` documented as a title-link source. — [Changelog](https://developers.google.com/search/updates)
- 2024-10-07 robots.txt unsupported fields clarified; 2024-12-09 crawler cache-control handling documented; 2024-09-16 content-encoding support documented. — [Changelog](https://developers.google.com/search/updates)
- 2025-03-07 and 2024-11-20: Page Annotations opt-out information added then removed from the control docs. — [Changelog](https://developers.google.com/search/updates)
- 2025-06-11: the section on blocking automatically translated pages via robots.txt was removed from multilingual docs. — [Changelog](https://developers.google.com/search/updates)
- 2025-11 new user-triggered fetchers `Google-Pinpoint`, `Google-CWS`; 2025-10-09 `Google-NotebookLM`; 2024-08-20 `Google-CloudVertexBot`; 2024-05-16 `GoogleOther-Image`/`-Video`. — [Changelog](https://developers.google.com/search/updates)
- 2026-11-12: new page on identifying Google Search traffic via HTTP Referer headers. — [Changelog](https://developers.google.com/search/updates)

### Inferences
- There is still no page-level or markup-level "AI opt-out" distinct from snippets: per-page control is `nosnippet`/`max-snippet`/`data-nosnippet` (which also degrade classic snippets), site-level control is a Search Console setting outside the CMS's output, and training control is the `Google-Extended` robots.txt token. A CMS can only directly emit the first and third.
- A CMS robots.txt API should support arbitrary product tokens (e.g. `Google-Extended`) and be ready for the AIPREF-style `__usage_ai-training` token Google says it will support in early 2027.
- FAQPage and SearchAction markup have no remaining Google Search rich-result benefit as of mid-2026; emitting them by default is unnecessary for Google.
- A CMS need not generate llms.txt for Google's benefit.

### Gaps
- The Search Console help article for the "Search generative AI" control was not retrieved; the Include/Exclude/Inherit option names come from secondary coverage only, and whether the control can be set by API or any on-site signal (meta tag, robots.txt) is unknown — the fetched Google pages mention only Search Console.
- The exact text of Google's AIPREF commitment (timing, syntax) was seen only as summarized from the common-crawlers page; treat the "early 2027" date and token syntax as needing confirmation before implementation.
- The changelog was read only for 2024 through November 2026 (first 100,000 characters); nothing earlier was reviewed, and individual entries were not all verified against their target pages.
- No reliable source was found in this session for any new robots meta directive introduced in 2024-2026; the supported list appears unchanged apart from the removals noted.
