# Exploration: What search engines require from a site in 2026

**Status:** complete
**Created:** 2026-11-17
**Author:** Claude, for the project owner
**Context:** Research into what search and AI answer engines read, to scope a core SEO API for SpeedySpec's WordPress fork.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

The project wanted one SEO API, so that SEO plugins build on a shared model instead of each shipping its own. Nobody had checked what search engines and AI answer engines actually read as of November 2026. Without that, the API could not be scoped: what core must own, what plugins should extend, and what nobody should build. The answer had to hold for the project's porters and plugin authors, not only for Google.

## Current State

What the engines do today, as the research found it. Findings marked **secondary only** rest on trade press, vendor guides or memory, not the engine's own documentation.

**The floor is small and server-side.**

- Google has three hard requirements: Googlebot is not blocked, the page returns HTTP 200, and it has indexable content ([technical requirements](https://developers.google.com/search/docs/essentials/technical)).
- AI Overviews and AI Mode add none, and need no special schema or text file ([AI features](https://developers.google.com/search/docs/appearance/ai-features)).
- Google reads a closed set of tags. It ignores meta keywords, `rel=next/prev`, the `lang` attribute for language detection, and `nositelinkssearchbox` ([supported meta tags](https://developers.google.com/search/docs/crawling-indexing/special-tags)).
- No engine documents a length limit for titles or descriptions.
- A robots.txt disallow hides a page's `noindex`, because the page is never crawled. Google, Seznam and Brave all say so ([robots meta spec](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)).
- One December 2024 log study found that no dedicated AI crawler ran JavaScript ([Vercel](https://vercel.com/blog/the-rise-of-the-ai-crawler)). It is a single, unreplicated study.

**Agreement between outputs is what matters.**

- A canonical is a stack of signals. Permanent redirects and `rel=canonical` are strong; sitemap entries are weak ([canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)).
- 301 and 308 move the canonical; 302, 303 and 307 do not ([redirects](https://developers.google.com/search/docs/crawling-indexing/301-redirects)).
- An error page that returns 200 is flagged as a soft 404.
- Google gives no rule on trailing slashes. "Pick one and redirect" is an inference.

**Sitemaps and robots.txt.**

- Google reads only `user-agent`, `allow`, `disallow` and `sitemap`, caps the file at 500 KiB, and ignores `crawl-delay` ([robots.txt](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt)).
- A sitemap holds at most 50,000 URLs or 50 MB. Google and Bing ignore `priority` and `changefreq` and discard inaccurate `lastmod` ([sitemaps](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap); [Bing blog](https://blogs.bing.com/webmaster/february-2023/The-Importance-of-Setting-the-lastmod-Tag-in-Your-Sitemap)).
- Engines disagree on extensions: Yandex reads `Clean-param`, Seznam reads `Request-rate`, Bing reportedly honours `Crawl-delay` (**secondary only**).
- A group named for Yandex makes Yandex ignore the `*` group ([Yandex](https://yandex.com/support/webmaster/en/robot-workings/clean-param)).
- Yandex accepts hreflang only as head `<link>` elements, the one form Google also accepts ([Yandex locales](https://yandex.com/support/webmaster/en/yandex-indexing/locale-pages)).

**Other engines.**

- DuckDuckGo, Yahoo and Ecosia lean on Bing. Brave and Applebot fall back to Googlebot's robots rules ([Brave](https://search.brave.com/help/brave-search-crawler); [Apple](https://support.apple.com/en-us/119829)).
- IndexNow reaches Bing, Yandex, Seznam, Naver, Yep, the Internet Archive and Amazonbot in one call ([IndexNow](https://www.indexnow.org/documentation)). Google does not take part; that rests on absence of evidence.
- **Secondary only:** Bing's February 2026 guideline rewrite, which trade press reports gives `noarchive` and `nocache` Copilot meanings. Bing's guideline pages render in JavaScript and could not be read.
- **Secondary only:** all of Baidu, Naver's tags and crawler, and every verification tag name except Yandex's.

**Rich results keep shrinking.**

- Google removed HowTo (2023), the sitelinks search box (November 2024), seven types (from June 2025) and FAQ (May 2026) ([changelog](https://developers.google.com/search/updates)).
- WebSite, Organization or Person, BreadcrumbList, Article and ProfilePage can be filled from data a CMS already holds. Product, Event, Recipe and JobPosting need a content plugin's data.
- Yoast's single `@graph` of flat pieces linked by `@id` is the mature model ([Yoast schema API](https://developer.yoast.com/features/schema/api/)).
- Google reads Open Graph for title links, site names and Discover images; Discover wants 1200 px images and `max-image-preview:large` ([Discover](https://developers.google.com/search/docs/appearance/google-discover)).
- **Secondary only:** X Cards. X's developer pages returned HTTP 402.

**AI engines add bot roles, not markup.**

- Vendors split agents into training crawlers, search crawlers and user-started fetchers ([OpenAI](https://developers.openai.com/api/docs/bots); [Anthropic](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)). Blocking a search crawler removes the site from that product's answers; user-started fetchers may ignore robots.txt.
- `noarchive` means nothing to Google, no Copilot use to Bing (**secondary only**), and no training to Amazon.
- Google says llms.txt is not used by Search. AI-usage preference standards (AIPREF, `Content-Signal`, RSL) are unratified and disagree.

**WordPress today.** The report did not audit the codebase. The SEO API design that followed found the head output spread across at least eight packages, and no meta description, social tags or structured data emitted at all.

## Proposed Solution

A core API that owns the outputs that must agree, and leaves every engine's list of names to plugins.

### Architecture

```
                 +--------------------------+
                 |  canonical URL resolver  |
                 +------------+-------------+
                              |
   +-----------+-----------+--+--------+-----------+------------+
   v           v           v           v           v            v
rel=canonical og:url   sitemap <loc> hreflang  JSON-LD url  redirects
                                                   
 head registry --> <title>, meta (name|property), link tags
 robots resolver --> one value per response --> meta tag + X-Robots-Tag
 robots.txt builder --> agent groups + free-form lines
 URL-changed event --> push senders (IndexNow etc., plugins)
```

### Data Model

- Robots directives: an open set of names, with `noarchive`, `nocache` and `nosnippet` as distinct values.
- Meta tags carry either a `name` or a `property` attribute.
- JSON-LD: one `@graph`, flat nodes, stable `@id`s as the public contract.
- AI controls: three purposes (search, AI input, AI training), with the agent-token list as filterable data.

## Design Decisions

1. **One canonical resolver feeds every output.** Separate plugins cannot keep `rel=canonical`, `og:url`, sitemap entries, hreflang and structured data aligned, and Google weakens disagreeing signals. Sitemaps and robots follow these rules, per [ADR 0012](../../adr/0012-redirects-sitemaps-robots-feeds.md).
   *Trade-off:* every output depends on one service, so a resolver bug shows up everywhere.
2. **Robots directives are an open set, emitted as meta and header.** Bing and Amazon give `noarchive` and `nocache` meanings Google does not, and attachments and feeds need the header form. Warn when a URL is both disallowed and `noindex`.
   *Trade-off:* core cannot validate unknown directive names.
3. **robots.txt has agent groups plus free-form lines.** That absorbs `Clean-param`, `Request-rate`, `Content-Signal` and future syntaxes without core endorsing any. Named groups copy the general rules.
   *Trade-off:* free-form lines are unchecked.
4. **A small identity graph in core; content types in a plugin registry.** The core types use data core holds; the rest track a gallery Google keeps cutting.
   *Trade-off:* sites without a content-type plugin get no Product or Event markup.
5. **One URL-changed event; push senders are replaceable.** Google ignores IndexNow, Baidu uses another API, and Cloudflare or a plugin may already submit.
   *Trade-off:* IndexNow is not on by default in core.
6. **All output is server-rendered.** It is the only form every crawler sees.
   *Trade-off:* no client-side SEO features.

## Fleet Impact

- The SEO API design in wordpress-develop is built on this research ([SEO API exploration](2026-11-17-seo-api.md)). Its packages now live in [SpeedySpec/speedyspec-api-seo](https://github.com/SpeedySpec/speedyspec-api-seo); confirmed with `gh`.
- ADR 0012 applies these rules to sitemaps and robots.
- No other repository is known to depend on it. Unverified beyond that.

## Open Questions

1. **What does Bing's live guideline text say about `noarchive` and `nocache`?** — read the page in a browser.
2. **What is Google's exact AI-training preference syntax and date?** — the full Google crawler page, read directly.
3. **What does X require for cards today?** — X's developer pages, once reachable.
4. **What are Baidu's and Naver's real requirements?** — their own documentation.
5. **What sizes do Slack, Discord, WhatsApp and iMessage previews use, and how do Bluesky and Threads behave?** — platform documentation; the guides found contradict each other.
6. **What does the codebase already emit?** — an audit of the head output.

## Success Criteria

- [x] Every finding labelled by evidence quality — the notes tag vendor, draft, study, secondary and unverified sources.
- [x] Every capability sorted into core, extension or skip — the report's conclusion.
- [ ] The three unconfirmed items re-read from primary sources — still open.
- [ ] A gap analysis against the existing code — not done.

---

## Non-Goals

- **Ranking advice or content scoring** — the question was technical requirements.
- **Core Web Vitals work** — set by themes, images and scripts, not a metadata API.
- **Blocking AI fetchers** — host and CDN work.

## Alternatives Considered

### Option: Per-type rich-result generators in core
- **Pros:** Rich markup for every site out of the box.
- **Cons:** Google removed many types in three years; required fields need plugin data.
- **Verdict:** Rejected. A piece registry instead.

### Option: Hard-coded engine, bot and tag lists
- **Pros:** Simple settings screens.
- **Cons:** The lists changed repeatedly in 2025 and 2026, and most tag names are unconfirmed.
- **Verdict:** Rejected. Filterable data and registries.

### Option: A strict four-field robots.txt model
- **Pros:** Matches what Google reads.
- **Cons:** Cannot express Yandex, Seznam or AI-preference lines.
- **Verdict:** Rejected. Free-form lines added.

## Dependencies

- Read access to engine documentation. Bing's help pages, X's developer pages and Baidu's documentation were not reachable.

## References

- Source: report "Search engine SEO requirements 2026"; notes `google_search.md`, `bing_and_other_engines.md`, `structured_data_and_social_cards.md`, `ai_answer_engines.md`.
- [ADR 0012: redirects, sitemaps, robots and feeds](../../adr/0012-redirects-sitemaps-robots-feeds.md)
- [SEO API exploration](2026-11-17-seo-api.md)
- [SpeedySpec/speedyspec-api-seo](https://github.com/SpeedySpec/speedyspec-api-seo)
- [Google: technical requirements](https://developers.google.com/search/docs/essentials/technical)
- [Google: AI features](https://developers.google.com/search/docs/appearance/ai-features)
- [Google: Search Central changelog](https://developers.google.com/search/updates)
- [IndexNow documentation](https://www.indexnow.org/documentation)
- [Yoast schema API](https://developer.yoast.com/features/schema/api/)
