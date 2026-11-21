# Bing and other non-Google search engines: technical requirements (state as of November 2026)

Research note on source quality: Bing's help pages (bing.com/webmasters/help/...) are JavaScript-rendered and returned no body text to the fetch tool; web.archive.org and searchadvisor.naver.com could not be fetched. Bing guideline content below therefore comes from Bing Webmaster Blog posts (primary) and trade-press coverage of the guidelines (secondary, flagged). Yandex, Seznam, Brave, DuckDuckGo, IndexNow, Cloudflare, Rank Math and the WordPress.org plugin page were read directly (primary).

## What do the Bing Webmaster Guidelines require and recommend?

### Takeaway
Bing rewrote its Webmaster Guidelines in late February 2026 around AI/Copilot "grounding" eligibility; the practical CMS-level differences from Google are (a) robots directives NOARCHIVE and NOCACHE now control Copilot use, (b) accurate sitemap `lastmod` is a crawl-scheduling input while `changefreq`/`priority` are ignored, (c) IndexNow is "highly recommended", and (d) hreflang is a weak signal versus content-language. The exact current wording of the guidelines could not be read directly.

### Cited Findings
- The guidelines were substantially rewritten; Search Engine Roundtable reported the change on Feb 26, 2026, with no separate Microsoft announcement, two weeks after Bing launched its AI Performance dashboard — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/); [Search Engine Roundtable](https://www.seroundtable.com/bing-webmaster-guidelines-updated-41002.html)
- New intro text (quoted by SER): the guidelines "describe how Bing discovers, crawls, indexes, evaluates, and surfaces content"; listed outcomes include "Indexing and ranking", "Grounding results and citations", and "Sustained visibility and qualified traffic" — [Search Engine Roundtable](https://www.seroundtable.com/bing-webmaster-guidelines-updated-41002.html)
- Robots directives and Copilot (secondary report of the 2026 guidelines): NOARCHIVE "prevents content from being used in Copilot responses and grounding results"; NOCACHE limits Copilot to URL, title and snippet; NOSNIPPET and data-nosnippet may limit citation quality; Bing advises against NOCACHE if richer citations are wanted — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- The 2026 rewrite names "Generative Engine Optimization (GEO)", adds a "Prompt Injection and AI Manipulation" abuse section, renames keyword stuffing to "Keyword Stuffing and Artificially Engineered Language", and says content produced without oversight "may be excluded from indexing" — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- Sections removed in the 2026 rewrite (per SEJ): detailed sitemap format support, JavaScript rendering guidance, SafeSearch content marking, social media schemes — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- Bing added support for the `data-nosnippet` HTML attribute on October 15, 2025; it works on any HTML element; marked content "is still indexed normally, but it will be excluded from snippets and AI summaries" (Bing Search and Copilot) — [Bing Webmaster Blog](https://blogs.bing.com/webmaster/October-2025/Bing-Introduces-Support-for-the-data-nosnippet-HTML-Attribute)
- The same Bing post lists page-level directives Bing supports: `noindex` ("Prevents a page from being indexed"), `nosnippet` ("Blocks all text and preview thumbnails from appearing in snippets"), and `max-snippet`, `max-image-preview`, `max-video-preview` ("Limit the size or duration of preview content") — [Bing Webmaster Blog](https://blogs.bing.com/webmaster/October-2025/Bing-Introduces-Support-for-the-data-nosnippet-HTML-Attribute)
- data-nosnippet support was a reversal: Bing had earlier said it had no near-term plans to support it — [Search Engine Land](https://searchengineland.com/microsoft-bing-supports-data-nosnippet-html-attribute-463411)
- Sitemap `lastmod` (Bing blog, Fabrice Canel, Feb 1, 2023): must be "the date the linked page was last modified, not when the sitemap is generated"; Bing disregards dates that consistently equal the current date; Bing was revamping crawl scheduling to rely on lastmod (rollout planned by June 2023); W3C Datetime format (YYYY-MM-DD or YYYY-MM-DDThh:mm:ssTZD); regenerate sitemaps at least daily — [Bing Webmaster Blog](https://blogs.bing.com/webmaster/february-2023/The-Importance-of-Setting-the-lastmod-Tag-in-Your-Sitemap)
- Bing "largely disregards" `changefreq` and `priority` — [Bing Webmaster Blog](https://blogs.bing.com/webmaster/february-2023/The-Importance-of-Setting-the-lastmod-Tag-in-Your-Sitemap)
- Bing says it is "highly recommended to adopt IndexNow" — [Bing Webmaster Blog](https://blogs.bing.com/webmaster/february-2023/The-Importance-of-Setting-the-lastmod-Tag-in-Your-Sitemap)
- hreflang: Fabrice Canel (Bing) stated in 2020 that "hreflang is indeed a far weaker signal than content-language at Bing" — [Search Engine Roundtable](https://www.seroundtable.com/bing-hreflang-weak-signal-30099.html)
- Crawl-delay (secondary sources only): Bingbot honors `Crawl-delay` in robots.txt; third-party pages say Bing recommends 1–30 seconds, manages it per subdomain, and that a robots.txt value overrides the Crawl Control setting in Bing Webmaster Tools — [WebProNews](https://www.webpronews.com/bing-details-how-to-control-bingbot-crawling-behavior); [Rank Math glossary](https://rankmath.com/seo-glossary/crawl-delay/)
- Site verification: Bing Webmaster Tools offers XML file, meta tag (`msvalidate.01`), DNS record, Domain Connect (DNS-provider integration) and import from Google Search Console; imported sites are periodically re-validated by syncing with the GSC account — [Bing Webmaster Blog, Sept 2019](https://blogs.bing.com/webmaster/september-2019/Import-sites-from-Search-Console-to-Bing-Webmaster-Tools); [Bing Webmaster Blog, Aug 2019](https://blogs.bing.com/webmaster/august-2019/Bing-Webmaster-Tools-simplifies-site-verification-using-Domain-Connect); [AIOSEO docs](https://aioseo.com/documentation/bing-webmaster-verification)

### Inferences
- A CMS SEO API needs NOARCHIVE and NOCACHE as distinct, individually settable robots directives (meta and `X-Robots-Tag`), because on Bing they now have different Copilot effects; Google treats noarchive as legacy and has no nocache equivalent.
- `lastmod` must be driven by real content-modified time, never by sitemap generation time; emitting `changefreq`/`priority` has no value for Bing.
- For language targeting on Bing, emitting `<html lang>` plus a `content-language` meta tag or HTTP header alongside hreflang is the safe output; hreflang alone is weak.
- A filterable hook for adding verification meta tags (`msvalidate.01`) is the minimum; file/DNS methods are outside CMS scope except serving a root file.

### Gaps
- The live text of the Bing Webmaster Guidelines and Bing's "which robots meta tags does Bing support" help page could not be read (JS-rendered). Not verified from a primary source: the full directive list, canonical guidance, redirect guidance, the current hreflang/content-language wording, crawl-delay wording, and whether the NOARCHIVE/NOCACHE Copilot wording is exactly as SEJ reports.
- From memory, unverified this session: the verification file is named `BingSiteAuth.xml`, the meta tag is `<meta name="msvalidate.01" content="...">`, and the DNS method is a CNAME pointing to `verify.bing.com`. Confirm before relying on them.
- The hreflang statement is from 2020; no 2024–2026 primary restatement was found.

## What is IndexNow (protocol, participants, CMS integration, Google's position)?

### Takeaway
IndexNow is a simple push protocol (one GET or one JSON POST plus a key file on the host) where a single submission to `api.indexnow.org` is shared with all participants: Bing, Yandex, Naver, Seznam, Yep, plus Internet Archive and Amazonbot. Google still does not support it as of 2026.

### Cited Findings
- Single URL: `GET https://<searchengine>/indexnow?url=<url-encoded-url>&key=<key>`, optional `&keyLocation=<key file URL>` — [IndexNow documentation](https://www.indexnow.org/documentation)
- Bulk: `POST /indexnow`, `Content-Type: application/json; charset=utf-8`, body fields `host`, `key`, optional `keyLocation`, `urlList`; up to 10,000 URLs per POST; http and https URLs may be mixed — [IndexNow documentation](https://www.indexnow.org/documentation)
- Key: 8–128 characters; a–z, A–Z, 0–9 and dashes. Key file is UTF-8 text named `{key}.txt` containing the key, at the host root (recommended), or elsewhere on the same host via `keyLocation`, in which case it only covers URLs under that path — [IndexNow documentation](https://www.indexnow.org/documentation)
- Response codes: 200 OK (received); 202 Accepted (received, key validation pending); 400 Bad Request (invalid format); 403 Forbidden (key invalid / not found); 422 Unprocessable Entity (URLs do not belong to host or key does not match schema); 429 Too Many Requests (potential spam) — [IndexNow documentation](https://www.indexnow.org/documentation)
- Participating engines "agree to automatically share" submitted URLs with all other participants — [IndexNow documentation](https://www.indexnow.org/documentation)
- Endpoints listed in the official registry file: Bing, Yandex, Seznam, Naver, Yep, Internet Archive, Amazonbot — [indexnow.org/searchengines.json](https://www.indexnow.org/searchengines.json)
- FAQ: submit to the global `api.indexnow.org` endpoint or one engine's endpoint; only one is needed — [IndexNow FAQ](https://www.indexnow.org/faq)
- What to submit: new, updated, removed content; redirected URLs; deleted URLs returning 404/410. Not for routinely submitting every URL (acceptable after a migration), and not for changes made before adoption (use sitemaps with accurate `lastmod` for those) — [IndexNow FAQ](https://www.indexnow.org/faq)
- Rate limits are not disclosed and are per engine; each submitted URL counts toward crawl quota; wait at least 5 minutes before resubmitting a frequently changing URL and at least 10 minutes after a 429; honor `Retry-After` — [IndexNow FAQ](https://www.indexnow.org/faq)
- Each subdomain is a separate host and needs its own key file — [IndexNow FAQ](https://www.indexnow.org/faq)
- Submission does not guarantee crawling or indexing — [IndexNow FAQ](https://www.indexnow.org/faq)
- CMS/CDN adopters listed by indexnow.org: WordPress, Shopify, Wix, GoDaddy, Duda, XenForo, Milestone, Joomla, Drupal, Bitrix24, PrestaShop, MODX, Shopware, OpenCart, Typo3, Umbraco; CDN: Cloudflare — [IndexNow FAQ](https://www.indexnow.org/faq)
- Yandex endpoint is `https://yandex.com/indexnow` (GET single, POST multiple); Yandex checks the key on every request and says to send only changes made since adopting IndexNow, using sitemaps for the rest — [Yandex Webmaster](https://yandex.com/support/webmaster/en/indexing-options/index-now)
- Microsoft's WordPress "IndexNow Plugin" (author bingwebmastertools): generates and hosts the key, detects page create/update/delete and submits in the background to `https://api.indexnow.org/indexnow`; supports path exclusions, manual submission, a retryable submission log, and skips noindex content (meta robots, X-Robots-Tag, WordPress search-visibility setting); 100,000+ active installs; version 1.0.4 — [WordPress.org](https://wordpress.org/plugins/indexnow/)
- Rank Math: IndexNow is in the free plugin (Instant Indexing module); auto-submits on publish, modify and delete for selected post types; skips noindex URLs; generates the key and serves it dynamically at the site root (manual `{key}.txt` fallback); offers manual and bulk submission and a history with response codes — [Rank Math KB](https://rankmath.com/kb/how-to-use-indexnow/)
- Cloudflare Crawler Hints: a zone-wide toggle (all plans) that sends IndexNow signals when cache status MISS suggests content changed; pages are excluded by serving `X-Robots-Tag: noindex` or a noindex meta tag — [Cloudflare docs](https://developers.cloudflare.com/cache/advanced-configuration/crawler-hints/)
- All in One SEO has built-in IndexNow support (search-result summary) — [OptinMonster](https://optinmonster.com/what-is-indexnow-for-seo)
- Google: said in October 2021 it would test IndexNow; no 2025–2026 report of adoption was found; a third-party article updated April 2026 states Google does not support it — [Search Engine Roundtable (2021)](https://www.seroundtable.com/google-indexnow-32396.html); [Blck Alpaca](https://blckalpaca.at/en/knowledge-base/seo-geo/technical-seo/indexnow-35-billion-urls-daily-without-google)
- Unverified secondary claims: IndexNow was the source of 22% of clicked Bing URLs (Dec 2024 Bing data); Shopify launched native integration in May 2025 — [Blck Alpaca](https://blckalpaca.at/en/knowledge-base/seo-geo/technical-seo/indexnow-35-billion-urls-daily-without-google)

### Inferences
- A core implementation needs: key generation and storage; a dynamically routed `/{key}.txt` response at the host root; a hook on publish/update/trash/delete and status transitions; batching into a POST of at most 10,000 URLs to `api.indexnow.org`; skipping noindex and non-public URLs; back-off on 429 with `Retry-After`; a debounce of about 5 minutes per URL; and a per-site off switch (staging, discouraged-search-engines).
- In multisite/subdomain installs, each host needs its own key file.
- Cloudflare and plugins can double-submit; a core API should expose a filter so one sender wins.
- IndexNow complements but does not replace sitemaps: both IndexNow and Yandex direct that historic content be communicated via sitemap `lastmod`.

### Gaps
- Yoast's IndexNow help page returned 404; whether Yoast's integration is still Premium-only in 2026 and how it submits was not verified.
- Google has made no primary statement since 2021 that was found; "does not support" rests on absence of adoption plus secondary sources.
- Baidu is not in the IndexNow registry file; no statement from Baidu about IndexNow was found.
- Naver's own IndexNow page could not be fetched; its participation is confirmed only via the indexnow.org registry and FAQ.

## Which engines have their own index vs. syndicate Bing or Google?

### Takeaway
Supporting Bing (Bingbot access, Bing Webmaster Tools, IndexNow) covers most Western alternative engines, since DuckDuckGo, Yahoo, Ecosia and partly Qwant draw on Bing; Brave is a genuinely independent index but exposes no distinct crawler and mirrors Googlebot's permissions. Ecosia and Qwant have been building their own European index (Staan) since August 2025.

### Cited Findings
- DuckDuckGo: traditional links and images are "largely sourced from Bing"; it also maintains "our own crawler (DuckDuckBot) and many indexes to support our results" — [DuckDuckGo Help](https://duckduckgo.com/duckduckgo-help-pages/results/sources)
- Brave Search: does not advertise a dedicated user agent; will not crawl any page Googlebot cannot crawl; robots.txt is not used to prevent indexing, a robots `noindex` directive is required; URL re-fetch/delisting via a submit-url form; partly fed by the opt-in Web Discovery Project in the Brave browser — [Brave Search help](https://search.brave.com/help/brave-search-crawler)
- Brave describes its index as first-party/independent — [Brave](https://brave.com/learn/best-search-api-2026)
- Third-party directories list a "BraveBot/1.0" user agent, which conflicts with Brave's own statement of no differentiated agent — [Dark Visitors](https://darkvisitors.com/agents/bravebot); contradicted by [Brave Search help](https://search.brave.com/help/brave-search-crawler)
- A November 2026 lawsuit by the New York Times and other publishers concerns Brave's unidentified crawler and index (secondary source) — [Relevant Audience](https://www.relevantaudience.com/geo/new-york-times-sues-brave-search-index-crawler/)
- Ecosia and Qwant: joint venture European Search Perspective (EUSP) has operated its own index, Staan (Search Trusted API Access Network), since August 2025 — [TechCrunch](https://techcrunch.com/2025/08/06/qwant-and-ecosia-debut-staan-a-european-search-index-that-aims-to-take-on-big-tech); [heise](https://heise.de/-11208524)
- Yahoo Search is powered by Bing's index; Qwant uses its own index supplemented by Bing; Ecosia uses Bing; Mojeek runs its own crawler and index (search-result summaries of secondary lists) — [Search Engine World, updated 4-26-2026](https://www.searchengineworld.com/the-big-search-engine-list-who-indexes-what); [Search Engine Journal](https://www.searchenginejournal.com/seo/meet-search-engines/)

### Inferences
- There is nothing engine-specific to emit for DuckDuckGo, Yahoo or Ecosia; Bing indexing is the lever.
- For Brave, anything that blocks Googlebot blocks Brave, and a robots.txt Disallow does not de-index; `noindex` must be reachable. No verification tag or sitemap submission exists.
- CMS robots.txt defaults should not whitelist only named big-engine bots, or smaller independent crawlers (DuckDuckBot, MojeekBot, the Staan crawler) are excluded.

### Gaps
- Startpage: no source read this session confirms its supply; it is commonly described as serving Google results (unverified here).
- Staan's crawler user agent, its robots.txt behaviour, and what share of Ecosia/Qwant queries it serves in late 2026 were not established; reported targets and rollout dates conflict between sources.
- Whether Ecosia still mixes Google results in 2026 was not confirmed.

## What Yandex-specific requirements exist?

### Takeaway
Yandex honors a distinct robots.txt directive (`Clean-param`), no longer honors `Crawl-delay` or `Host` (both dropped in 2018), supports hreflang only as `<link>` elements in `<head>` (not in sitemaps), participates in IndexNow, and discontinued Turbo pages in April 2025.

### Cited Findings
- Supported robots.txt directives listed by Yandex: `User-agent` (mandatory), `Disallow`, `Allow`, `Sitemap`, `Clean-param`; robots.txt must be within 500 KB or the site is treated as fully open — [Yandex Webmaster](https://yandex.com/support/webmaster/en/controlling-robot/robots-txt)
- `Clean-param` syntax: `Clean-param: p0[&p1&p2&..&pn] [path]`; max rule length 500 characters; case-sensitive; path prefix allows A-Za-z0-9 and `*`, with an implicit trailing `*`; it is cross-section ("can be indicated anywhere within the file"); Disallow takes priority. Examples: `Clean-param: utm`, `Clean-param: sid&sort /forum/*.php` — [Yandex Webmaster](https://yandex.com/support/webmaster/en/robot-workings/clean-param)
- If a `User-agent: Yandex` group exists, the `User-agent: *` group is ignored for Yandex, so all Yandex rules must be in one section — [Yandex Webmaster](https://yandex.com/support/webmaster/en/robot-workings/clean-param)
- `Crawl-delay`: "From February 22, 2018, Yandex doesn't take into account the Crawl-delay directive"; crawl rate is set in Yandex Webmaster (search-result quote of Yandex documentation) — [Yandex Webmaster (TR mirror)](https://yandex.com.tr/support/webmaster/en/controlling-robot/robots-txt)
- `Host` directive: deprecated by Yandex in 2018 in favour of 301 redirects (secondary) — [ctrl.blog](https://ctrl.blog/entry/arcane-robotstxt-directives)
- Verification: `<meta name='yandex-verification' content='...' />` in the home page `<head>`; or an HTML file with a Yandex-supplied unique name and content in the site root; or a DNS TXT record. www and non-www are separate sites — [Yandex Webmaster](https://yandex.com/support/webmaster/en/service/rights)
- hreflang: `<link rel="alternate" hreflang="..." href="..."/>` in `<head>` of every version, each linking all alternatives; `x-default` supported; ISO 639-1 language plus optional ISO 3166-1 alpha-2 region; Yandex no longer supports "using the Sitemap file for language versions" — [Yandex Webmaster](https://yandex.com/support/webmaster/en/yandex-indexing/locale-pages)
- IndexNow endpoint `https://yandex.com/indexnow` — [Yandex Webmaster](https://yandex.com/support/webmaster/en/indexing-options/index-now)
- Turbo pages: Yandex Webmaster changelog records them as discontinued on April 1, 2025; they stopped appearing in search from April 2, 2025 — [Yandex Webmaster "New features"](https://yandex.com/support/webmaster/en/service/about); [Yandex Turbo docs](https://yandex.com/dev/turbo/doc/en/index.md); [postium.ru](https://postium.ru/yandeks-prekrashhaet-podderzhku-turbo-stranic/)

### Inferences
- A robots.txt API must allow arbitrary extra directives (`Clean-param`) and must be careful when generating a `User-agent: Yandex` group, since that makes Yandex ignore `*` rules.
- hreflang must be emitted in HTML `<head>` to work for Yandex; sitemap-only hreflang (valid for Google) is insufficient.
- No Turbo RSS feed output is needed; emitting `Host:` or `Crawl-delay:` for Yandex is dead weight.

### Gaps
- Yandex's own page for the Host deprecation was not read (secondary source only).
- Yandex-specific meta robots values (e.g. `noyaca`, the `<noindex>` element) and HTTP-header hreflang support were not checked.

## What Baidu-specific requirements exist?

### Takeaway
Reliable primary documentation was not reachable; secondary sources agree that Baidu needs core content in server-rendered HTML, benefits from mainland hosting with an ICP filing, uses its own push API rather than IndexNow, and has legacy meta tags for mobile adaptation and anti-transcoding. Treat tag-level details as low-confidence.

### Cited Findings
- JavaScript: "Baidu struggles with complex JavaScript"; keep essential content in HTML (published 2024-12-17) — [Chinafy](https://www.chinafy.com/blog/technical-seo-checklist-for-baidu)
- User agents reported by third parties: `Baiduspider/2.0` and a rendering crawler `Baiduspider-render/2.0`; robots.txt token `Baiduspider` (unverified against Baidu) — [Centinel Analytica](https://centinelanalytica.com/crawlers/baiduspider-render); [Dark Visitors](https://api.darkvisitors.com/agents/baiduspider)
- Submission channels (vendor guide, limits unconfirmed): XML sitemap in the Search Resource Platform; manual submission (max 20 URLs at a time); API push (1000 URLs per submit, 10 submits/day per site); auto-push JS snippet — [x-cmd](https://x-cmd.com/seo/baidu)
- Push API format (community tutorials): POST plain text, one URL per line, to `data.zz.baidu.com/urls` with `site` and `token` query parameters; JSON response includes `success`, `remain` (daily quota left), `not_same_site`, `not_valid` — [cnblogs](https://www.cnblogs.com/duanguyuan/p/16205184.html); [Aliyun developer](https://developer.aliyun.com/article/945676)
- `applicable-device` meta tag values `pc`, `mobile`, `pc,mobile` tell Baidu the target device and help avoid transcoding — [Dragon Metrics](https://www.dragonmetrics.com/how-to-succeed-in-baidu-mobile-seo/); [China Internet Watch](https://chinainternetwatch.com/19792/baidu-mobile-seo)
- Anti-transcoding: `<meta http-equiv="Cache-Control" content="no-siteapp">` is described in older Chinese-language sources — [php.cn](https://php.cn/faq/357338.html)
- Canonical is supported by Baidu but not fully obeyed (secondary) — [Dragon Metrics](https://www.dragonmetrics.com/how-to-succeed-in-baidu-mobile-seo/)
- Set `lang="zh-CN"` and UTF-8; Baidu's structured-data support is more limited than Google's; .cn domain / China hosting is supportive, not critical — [Chinafy](https://www.chinafy.com/blog/technical-seo-checklist-for-baidu)
- ICP filing with mainland hosting is "strongly recommended for serious Baidu SEO"; HTTPS is standard practice in 2026 (vendor sources) — [x-cmd](https://x-cmd.com/seo/baidu); [SEO Sherpa](https://seosherpa.com/baidu-seo/)

### Inferences
- The CMS-relevant Baidu surface is small: server-rendered HTML, arbitrary meta tags (`applicable-device`, `Cache-Control: no-siteapp/no-transform`, a verification tag), a sitemap, and a pluggable "URL changed" event that a Baidu push adapter can consume alongside IndexNow.
- ICP is a hosting/legal matter, not CMS output, apart from displaying the filing number in the footer.

### Gaps
- No Baidu primary documentation (ziyuan.baidu.com) was read. Unverified: the `baidu-site-verification` meta tag name (from memory), `no-transform`, hreflang support, crawl-delay support, MIP's current status, current push API quotas, and any 2024–2026 changes.
- Sources disagree on the quality of Baidu's JS rendering; none is authoritative.

## What do Naver and Seznam require?

### Takeaway
Both are IndexNow participants with their own webmaster consoles. Seznam's distinctive item is the non-standard robots.txt `Request-rate` directive; Naver's are Search Advisor verification and sitemap plus RSS submission. Naver detail is from secondary guides only.

### Cited Findings
- Naver and Seznam both have IndexNow endpoints (`searchadvisor.naver.com/indexnow`, `search.seznam.cz/indexnow`) — [indexnow.org/searchengines.json](https://www.indexnow.org/searchengines.json)
- Naver verification: `naver-site-verification` meta tag in `<head>`, or HTML file upload (secondary guides) — [inblog](https://inblog.ai/docs/seo-tutorial-domain-setup/registering-with-naver-search-advisor); [The Egg](https://www.theegg.com/seo/korea/naver-webmaster-tools-guide)
- Naver: sitemap and RSS are submitted under the Search Advisor "request" menu; crawler is Yeti (`User-agent: Yeti`), respects Allow/Disallow and robots.txt Sitemap lines (secondary) — [inblog](https://inblog.ai/docs/seo-tutorial-domain-setup/registering-with-naver-search-advisor); [siteline.ai](https://siteline.ai/ai-agent-directory/naverbot/)
- Seznam: SeznamBot follows the robots exclusion standard; supports `Allow`, `*` and `$` wildcards, `Sitemap: <absolute URL>` (sitemaps.org protocol), and `Request-rate: <documents>/<time>` (e.g. `Request-rate: 300/1m`, optional time-of-day window; minimum 1 document per 10 seconds); robots.txt changes take days to weeks to apply — [Seznam help](https://o-seznam.cz/napoveda/vyhledavani/en/crawling-control/)
- Seznam: to de-index, crawling must stay allowed so HTML-level restrictions can be read — [Seznam help](https://o-seznam.cz/napoveda/vyhledavani/en/crawling-control/)
- Third-party pages claim SeznamBot honors `Crawl-delay`; Seznam's crawling-control page does not mention it — [Dark Visitors](https://darkvisitors.com/agents/seznambot); not confirmed by [Seznam help](https://o-seznam.cz/napoveda/vyhledavani/en/crawling-control/)

### Inferences
- An RSS/Atom feed remains a first-class discovery input for Naver, so feed URLs should be exposed by the API alongside sitemaps.
- IndexNow via `api.indexnow.org` covers both engines with no extra work.

### Gaps
- Naver's official docs were unreachable; DNS verification, file name pattern, and any Naver-specific IndexNow rules are unverified.
- Seznam's verification method (from memory, a `seznam-wmt` meta tag; unverified), supported meta robots values, and sitemap submission flow were not confirmed.

## What site-verification meta tags/files does each engine use?

### Takeaway
Every console with verification accepts a `<meta name="..." content="token">` on the home page, so one generic "verification meta" registry covers them; only Yandex's and (partly) Bing's were confirmed from primary sources.

### Cited Findings
- Bing: meta tag `msvalidate.01`, XML file, DNS record, Domain Connect, or GSC import — [AIOSEO docs](https://aioseo.com/documentation/bing-webmaster-verification); [Bing Webmaster Blog](https://blogs.bing.com/webmaster/september-2019/Import-sites-from-Search-Console-to-Bing-Webmaster-Tools)
- Yandex: `<meta name='yandex-verification' content='...'>`, root HTML file, or DNS TXT — [Yandex Webmaster](https://yandex.com/support/webmaster/en/service/rights)
- Naver: `naver-site-verification` meta tag or HTML file (secondary) — [inblog](https://inblog.ai/docs/seo-tutorial-domain-setup/registering-with-naver-search-advisor)
- Brave: no webmaster console, only a URL submit form — [Brave Search help](https://search.brave.com/help/brave-search-crawler)
- DuckDuckGo: help page describes no site submission or verification mechanism — [DuckDuckGo Help](https://duckduckgo.com/duckduckgo-help-pages/results/sources)
- IndexNow ownership proof is the `{key}.txt` file, independent of console verification — [IndexNow documentation](https://www.indexnow.org/documentation)

### Inferences
- The API should model verification as a list of (meta name, content) pairs plus the ability to serve small root-level text/XML/HTML files, not hard-coded per-engine fields.

### Gaps
- Baidu (`baidu-site-verification`), Seznam (`seznam-wmt`), Bing file name (`BingSiteAuth.xml`) are recalled from memory and unverified this session.
- Ecosia/Qwant (Staan) verification: nothing found.

## Which robots.txt extensions are honored by which engine?

### Takeaway
`Sitemap` is universal; `Crawl-delay` is honored by Bing (and reportedly Seznam) but ignored by Yandex since 2018 (and by Google); `Clean-param` is Yandex-only; `Host` is dead; `Request-rate` is Seznam-only.

### Cited Findings
- Yandex: supports `Sitemap`, `Clean-param`; ignores `Crawl-delay` since Feb 22, 2018 — [Yandex Webmaster](https://yandex.com/support/webmaster/en/controlling-robot/robots-txt); [Yandex Webmaster (TR mirror)](https://yandex.com.tr/support/webmaster/en/controlling-robot/robots-txt)
- Yandex `Host`: deprecated 2018; Googlebot never used it — [ctrl.blog](https://ctrl.blog/entry/arcane-robotstxt-directives)
- Bing: honors `Crawl-delay` (secondary) — [WebProNews](https://www.webpronews.com/bing-details-how-to-control-bingbot-crawling-behavior)
- Seznam: `Sitemap`, `Allow`, wildcards, `Request-rate` — [Seznam help](https://o-seznam.cz/napoveda/vyhledavani/en/crawling-control/)
- Naver Yeti: Allow/Disallow and `Sitemap` (secondary) — [siteline.ai](https://siteline.ai/ai-agent-directory/naverbot/)
- Brave: follows Googlebot's permissions; robots.txt does not prevent indexing — [Brave Search help](https://search.brave.com/help/brave-search-crawler)

### Inferences
- A robots.txt builder should support per-user-agent groups with free-form directive lines, since the non-standard directives are engine-scoped (`Crawl-delay` under bingbot, `Clean-param` for Yandex, `Request-rate` under SeznamBot).

### Gaps
- Baidu's and DuckDuckBot's handling of `Crawl-delay` and `Sitemap` was not verified.
- Bing's primary documentation of `Crawl-delay` was not read.
