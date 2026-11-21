# AI answer engines and AI crawlers: what they require, honor, and what site-side controls exist (as of 17 Nov 2026)

Evidence labels used below: **[VENDOR]** = stated in the vendor's own documentation (fetched this session); **[DRAFT]** = unratified spec/proposal; **[STUDY]** = third-party measurement, usually by an SEO/vendor blog, correlational; **[SECONDARY]** = trade-press report of a primary source I could not open directly; **[UNVERIFIED]** = single weak source.

## 1. Documented AI user agents: purpose, robots.txt behavior, IP lists

### Takeaway
Every major vendor now splits its bots into three roles: training crawler, search-index crawler, and user-triggered fetcher. The training and search bots are documented as honoring robots.txt; the user-triggered fetchers (ChatGPT-User, Perplexity-User, meta-externalfetcher, Amzn-User) are documented as possibly or generally NOT bound by robots.txt. Google-Extended and Applebot-Extended are control tokens only, never appear in logs, and do not control Google Search AI features.

### Cited Findings

**OpenAI** [VENDOR]
- `GPTBot` (UA example ends `compatible; GPTBot/1.4; +https://openai.com/gptbot`): crawls content that may be used to train generative foundation models. "Disallowing GPTBot indicates a site's content should not be used in training". IPs: https://openai.com/gptbot.json — [OpenAI bots docs](https://developers.openai.com/api/docs/bots) (platform.openai.com/docs/bots 301-redirects here)
- `OAI-SearchBot` (`compatible; OAI-SearchBot/1.4; +https://openai.com/searchbot`): surfaces sites in ChatGPT search. Sites opted out "will not be shown in ChatGPT search answers" but "can still appear as navigational links". IPs: https://openai.com/searchbot.json — [OpenAI bots docs](https://developers.openai.com/api/docs/bots)
- `ChatGPT-User/1.0` (`+https://openai.com/bot`): user-initiated fetches in ChatGPT and Custom GPTs. "Because these actions are initiated by a user, robots.txt rules may not apply." Not used for automatic crawling and "not used to determine whether content may appear in Search". IPs: https://openai.com/chatgpt-user.json — [OpenAI bots docs](https://developers.openai.com/api/docs/bots)
- `OAI-AdsBot/1.0`: checks landing pages submitted as ChatGPT ads; only visits submitted ad pages; data not used for training. IPs: https://openai.com/adsbot.json — [OpenAI bots docs](https://developers.openai.com/api/docs/bots)
- robots.txt changes take about 24 hours to take effect; each bot's setting is independent; if both GPTBot and OAI-SearchBot are allowed OpenAI "may use the results from just one crawl for both use cases"; a `robots.txt` marker may be added to the UA when fetching robots.txt. The page does not mention noindex. — [OpenAI bots docs](https://developers.openai.com/api/docs/bots)

**Anthropic** [VENDOR]
- `ClaudeBot`: collects content that may contribute to model training. Blocking excludes future material from training datasets. — [Anthropic support](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)
- `Claude-SearchBot`: crawls to improve search result quality; blocking means content is not indexed for search and may reduce visibility. — [Anthropic support](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)
- `Claude-User`: fetches on behalf of a Claude user's question; blocking prevents retrieval for user queries. — [Anthropic support](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)
- Anthropic states its bots honor robots.txt (rules must be set per subdomain) and the non-standard `Crawl-delay`. IP list published at claude.com/crawling/bots.json, but Anthropic advises against IP blocking because it can prevent the bot reading robots.txt. — [Anthropic support](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)
- Anthropic says it honors a machine-readable "no AI training" preference signal, with robots.txt rules naming ClaudeBot taking priority over that general signal. (The fetched summary did not name the signal; likely the AIPREF `train-ai` / shared training-preference signal, but that is my inference.) — [Anthropic support](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)

**Perplexity** [VENDOR, with a documented dispute]
- `PerplexityBot/1.0` (`+https://perplexity.ai/perplexitybot`): surfaces and links sites in Perplexity search; "not used to crawl content for AI foundation models". Honors robots.txt. IPs: https://www.perplexity.ai/perplexitybot.json — [Perplexity docs](https://docs.perplexity.ai/guides/bots)
- `Perplexity-User/1.0` (`+https://perplexity.ai/perplexity-user`): user-triggered fetch; "generally ignores robots.txt rules" because a user requested it; not used for training. IPs: https://www.perplexity.ai/perplexity-user.json — [Perplexity docs](https://docs.perplexity.ai/guides/bots)
- Perplexity's WAF guidance: match UA plus published IP range; refresh the IP lists automatically. — [Perplexity docs](https://docs.perplexity.ai/guides/bots)
- Dispute: in Aug 2025 Cloudflare reported Perplexity used undeclared crawlers with a generic Chrome-on-macOS UA and rotating IPs/ASNs outside its published ranges when its declared bots were blocked, and de-listed Perplexity from its verified-bots list. Perplexity rebutted that user-initiated assistant fetches are not crawling. Cloudflare contrasted OpenAI, whose crawler stopped when disallowed. [SECONDARY] — [Search Engine Journal](https://www.searchenginejournal.com/cloudflare-delists-and-blocks-perplexity-from-crawling-websites/552899/), [Neowin](https://www.neowin.net/news/perplexitys-stealth-crawlers-exposed-in-new-cloudflare-report/)

**Google** [VENDOR]
- `Google-Extended` is "a standalone product token"; it "doesn't have a separate HTTP request user agent string" (crawling uses existing Google UAs). It controls whether crawled content may be used to train future Gemini models (Gemini Apps, Vertex AI API for Gemini) and for grounding in Gemini Apps and Grounding with Google Search on Vertex AI. "Google-Extended does not impact a site's inclusion in Google Search" and is not a ranking signal. — [Google common crawlers](https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers)
- The same page now says Google "intends to support the AIPREF" shared training-preference signal, expressed as `user-agent: __usage_ai-training`, "in early 2027". — [Google common crawlers](https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers)
- `Google-CloudVertexBot`: crawls requested by site owners for building Vertex AI Agents; "no effect on Google Search or other products". `GoogleOther`, `GoogleOther-Image/1.0`, `GoogleOther-Video/1.0`: generic crawler for product teams and one-off R&D crawls. — [Google common crawlers](https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers)
- AI Overviews and AI Mode are fed by Googlebot, so Google-Extended does not cover them. [SECONDARY, consistent with the Google docs above] — [TechCrunch, 1 Jul 2026](https://techcrunch.com/2026/07/01/cloudflares-new-policy-pushes-ai-companies-to-pay-for-publishers-content/)

**Apple** [VENDOR]
- `Applebot` (UA contains `(Applebot/0.1; +http://www.apple.com/go/applebot)`): powers search in Spotlight, Siri, Safari; data may also be used to train Apple generative models. — [Apple support 119829](https://support.apple.com/en-us/119829)
- `Applebot-Extended`: control token to opt out of training Apple's foundation models. "Applebot-Extended does not crawl webpages." — [Apple support 119829](https://support.apple.com/en-us/119829)
- Applebot respects robots.txt, "does not follow crawl-delay", and "If robots instructions don't mention Applebot but mention Googlebot, the Apple robot will follow Googlebot instructions." Verify via reverse DNS in `*.applebot.apple.com`; IPs at http://search.developer.apple.com/applebot.json. — [Apple support 119829](https://support.apple.com/en-us/119829)

**Meta** [VENDOR]
- `meta-externalagent/1.1`: "training foundation AI models" or improving products by indexing content directly; blockable via Disallow. — [Meta web crawlers](https://developers.facebook.com/docs/sharing/webmasters/web-crawlers/)
- `meta-webindexer/1.1`: used to "improve Meta AI search result quality"; allowing it helps Meta cite and link content in Meta AI responses. — [Meta web crawlers](https://developers.facebook.com/docs/sharing/webmasters/web-crawlers/)
- `meta-externalfetcher/1.1`: fetches individual links at a user's request; "may bypass robots.txt". — [Meta web crawlers](https://developers.facebook.com/docs/sharing/webmasters/web-crawlers/)
- `meta-externalads/1.1`: advertising/business products. `facebookexternalhit/1.1`: link-share previews; "might bypass robots.txt when performing security or integrity checks". robots.txt changes can take up to 24 hours. The page recommends allow-listing by UA or IP but the fetched page did not list IPs. — [Meta web crawlers](https://developers.facebook.com/docs/sharing/webmasters/web-crawlers/)

**Amazon** [VENDOR]
- `Amazonbot`: improves Amazon products/services; "may be used to train Amazon AI models". IPs: https://developer.amazon.com/amazonbot/ip-addresses/ — [Amazonbot](https://developer.amazon.com/amazonbot)
- `Amzn-SearchBot`: search experiences such as Alexa; "does not crawl content for generative AI model training"; with no rules of its own it follows rules for other search bots you allow. IPs: https://developer.amazon.com/amazonbot/searchbot-ip-addresses/ — [Amazonbot](https://developer.amazon.com/amazonbot)
- `Amzn-User`: user-initiated (e.g. Alexa live queries); not for training; "may not follow all robots.txt directives". IPs: https://developer.amazon.com/amazonbot/live-ip-addresses/ — [Amazonbot](https://developer.amazon.com/amazonbot)
- Amazon honors robots.txt (cached copy up to 30 days old), `rel=nofollow`, and page-level robots meta: `noarchive` = "do not use the page for model training", `noindex`/`none` = do not index. Crawl-delay not supported. — [Amazonbot](https://developer.amazon.com/amazonbot)

**Common Crawl** [VENDOR]
- `CCBot/2.0 (https://commoncrawl.org/faq/)`: builds an open crawl repository (widely used as LLM training data). Block with `User-agent: CCBot` / `Disallow: /`. Dedicated IP ranges with reverse DNS; JSON at https://index.commoncrawl.org/ccbot.json. Common Crawl warns other crawlers spoof CCBot. — [Common Crawl CCBot](https://commoncrawl.org/ccbot)

**You.com** [VENDOR via search summary]
- `YouBot/1.0` (`+https://docs.you.com/youbot`): powers You.com search; docs state it "fully respects robots.txt" including crawl-delay; requests originate from 68.67.112.0/24. — [You.com docs](https://docs.you.com/youbot)

**Brave** [SECONDARY, not verified against Brave's own page]
- Brave's crawler reportedly has no differentiated user agent and follows Googlebot's robots.txt rules, so a site cannot block Brave while allowing Google; removal requires noindex. Brave resells its index to AI companies via the Brave Search API ("AI Grounding"). — [Search Engine Journal](https://www.searchenginejournal.com/brave-sells-its-search-index-to-ai-and-your-website-is-probably-in-it/)
- A Nov 2026 report says the New York Times sued Brave over its index and crawler. [UNVERIFIED, single source] — [Relevant Audience](https://www.relevantaudience.com/geo/new-york-times-sues-brave-search-index-crawler/)

**ByteDance** [no vendor docs]
- `Bytespider`: ByteDance publishes no bot documentation. Third-party reports conflict: some document it requesting disallowed paths (2023), others say it stops after re-fetching robots.txt. Compliance cannot be confirmed from a primary source. — [search summary incl. knownagents.com](https://knownagents.com/agents/bytespider)

### Inferences
- A CMS robots.txt API needs a registry of agent tokens grouped by purpose (training / search index / user fetch / control-only token), because the correct default for each group differs: blocking training tokens has no documented visibility cost, blocking search tokens (OAI-SearchBot, Claude-SearchBot, PerplexityBot, meta-webindexer, Amzn-SearchBot) removes the site from that product's answers.
- robots.txt cannot reliably stop user-triggered fetchers; four vendors document this. Enforcement, if wanted, has to be at the server/WAF using UA plus the published IP JSON files. A CMS could expose the IP-list URLs as data but enforcement belongs to the host/CDN.
- There is no token that opts out of Google AI Overviews / AI Mode while staying in Google Search; only snippet controls do that (see section 2).
- Applebot falls back to Googlebot rules and Brave reportedly mirrors Googlebot, so Googlebot rules have side effects on other engines.
- The list changes often (OAI-AdsBot, Amzn-SearchBot, meta-webindexer are recent). A hard-coded list in core will go stale; it should be filterable/updatable.

### Gaps
- Microsoft: I did not retrieve a Microsoft page documenting a separate AI-training token; Copilot grounding uses the Bing index (bingbot). Not verified this session.
- Bytespider: no primary documentation exists. DuckAssistBot, MistralAI-User, cohere-ai, Diffbot, Google NotebookLM / Google user-triggered fetchers were not researched.
- Meta IP ranges: Meta's page did not list them in the fetched content.
- Brave's own crawler help page was not opened; statements are from trade press.

## 2. Google and Bing controls over AI answers, and their optimization guidance

### Takeaway
Google documents no AI-specific requirement or opt-out: eligibility for AI Overviews/AI Mode equals being indexed and snippet-eligible, and the only controls are the existing preview controls (nosnippet, data-nosnippet, max-snippet, noindex). Bing reinterprets existing robots meta values for Copilot: NOARCHIVE excludes content from Copilot answers and grounding, NOCACHE limits it to URL/title/snippet.

### Cited Findings

**Google** [VENDOR]
- "a page must be indexed and eligible to be shown in Google Search with a snippet" to appear as a supporting link; "There are no additional requirements to appear in AI Overviews or AI Mode, nor other special optimizations necessary." — [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- "You don't need to create new machine readable files, AI text files, or markup to appear in these features." and "There's also no special schema.org structured data that you need to add." — [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- Both features may use "query fan-out": "issuing multiple related searches across subtopics and data sources". — [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- Controls: nosnippet, data-nosnippet, max-snippet, noindex. Google-Extended is only "To limit AI training and grounding in some of Google's other systems." — [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- AI feature traffic is counted in Search Console's Performance report under the "Web" search type (not separated). — [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- Google's 2026 AI optimization guide lists llms.txt, content chunking, AI-specific rewriting and special schema as not needed for generative AI features; llms.txt "should be skipped". [SECONDARY] — [Search Engine Journal, 20 May 2026](https://www.searchenginejournal.com/googles-llms-txt-guidance-depends-on-which-product-you-ask/575431/)

**Bing / Copilot** [SECONDARY report of Bing Webmaster Guidelines; the Bing page itself is JS-rendered and returned no body to my fetch]
- Bing added GEO (generative engine optimization) to its Webmaster Guidelines, described as "focused on content eligibility for grounding and reference in AI responses"; GEO does not guarantee citations. Reported 27 Feb 2026. — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- "NOARCHIVE prevents content from being used in Copilot responses and grounding results." "NOCACHE limits Copilot to using only the URL, title, and snippet." NOSNIPPET and data-nosnippet "may limit citation quality". — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- Guidance: single-topic pages are more likely to be selected for grounding; state facts directly rather than implying them. — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- New abuse categories: "Keyword Stuffing and Artificially Engineered Language" (content engineered to trigger AI citations), "Prompt Injection and AI Manipulation", and large-scale generated content "without oversight, quality control, or editorial review". — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- Bing Webmaster Tools "AI Performance" report (public preview, Feb 2026) shows Copilot citation counts by grounding query and by page; it does not show clicks. — [Search Engine Land](https://searchengineland.com/bing-webmaster-tools-testing-new-ai-performance-report-468039), [Search Influence](https://www.searchinfluence.com/blog/bing-ai-performance-report-copilot-citations/)

### Inferences
- The per-post robots meta API must support `nosnippet`, `max-snippet:N`, `noarchive`, `nocache`, plus an easy way to emit `data-nosnippet` on blocks. These are the only vendor-documented page-level AI-answer controls for Google and Bing, and `noarchive` has three different vendor meanings (Bing: no Copilot use; Amazon: no model training; classic: no cached copy).
- Using nosnippet to leave Google AI Overviews also removes the classic text snippet; there is no documented AI-only switch.
- Prompt-injection text hidden in content is now an explicit Bing guideline violation; a CMS should not ship features that embed instructions aimed at LLMs.

### Gaps
- Exact Bing guideline wording should be re-read from https://www.bing.com/webmasters/help/webmaster-guidelines-30fba23a in a browser; my fetch got only the title. SEJ's write-up mentions a "data-snippet" attribute, which may be a typo for data-nosnippet.
- I did not open Google's 2026 AI optimization guide directly, nor verify whether Google has added any AI-specific preview control during 2026.
- Bing's statements on IndexNow/lastmod specifically for Copilot were not retrieved.

## 3. llms.txt / llms-full.txt status and adoption

### Takeaway
llms.txt remains an informal proposal (Jeremy Howard, Sept 2024; spec page revised Aug 2026). File counts are growing fast, largely because plugins generate it by default, but multiple log studies show AI crawlers almost never request it, and Google Search explicitly says to skip it. No answer-engine vendor documentation I retrieved says it reads llms.txt for ranking or citation.

### Cited Findings
- Proposed by Jeremy Howard, published 3 Sept 2024, page last modified 10 Aug 2026 ("version 2"). Format: Markdown at `/llms.txt`; only an H1 is required; then optional blockquote summary, optional prose, and H2 sections of `[name](url): notes` links; an H2 named "Optional" marks skippable links. Recommends Markdown twins at `page.html.md` / `page.md` and `rel="alternate" type="text/markdown"` links (HTML or HTTP Link header). Intended "mainly useful for inference rather than training". Self-described as a proposal. The page does not define llms-full.txt. [DRAFT] — [llmstxt.org](https://llmstxt.org/)
- Google Search: llms.txt is not a Google initiative, is not needed for AI Overviews/AI Mode and "should be skipped". John Mueller compared it to the keywords meta tag and said no AI services used it. Gary Illyes confirmed Google is not pursuing it. [SECONDARY] — [Search Engine Journal, 20 May 2026](https://www.searchenginejournal.com/googles-llms-txt-guidance-depends-on-which-product-you-ask/575431/), [SE Roundtable](https://www.seroundtable.com/google-ai-llms-txt-39607.html)
- Counter-signal inside Google: Lighthouse 13.3 added an experimental "Agentic Browsing" category (on by default) that audits WebMCP integration and presence of llms.txt, calling it "an emerging convention". It targets browser agents, not rankings or citations. [SECONDARY] — [Search Engine Journal, 20 May 2026](https://www.searchenginejournal.com/googles-llms-txt-guidance-depends-on-which-product-you-ask/575431/)
- Ahrefs, 137,210 domains: 97% of llms.txt files received zero requests in May 2026; of requests that occurred, under a fifth came from named AI tools (led by GPTBot and Claude Code); most came from SEO tools and validators. [STUDY, as summarized by third parties] — [Optimizely](https://www.optimizely.com/field-notes/articles/does-llms-txt-work/), [DesignRush](https://news.designrush.com/llmstxt-ai-search-visibility)
- EZY Research, 83 sites over 12 weeks: OpenAI fetched llms.txt 7 times versus 3,990 robots.txt fetches; PerplexityBot zero; Meta's crawler 193 times. [STUDY] — [Something Inc.](https://somethinginc.com/blog/llms-txt-ai-crawlers-fetch-data/)
- Originality.ai (June 2026): llms.txt instances up 8.8x in 12 months (4,088 to 36,120) with near-zero evidence of AI systems reading them. [STUDY] — [PPC Land](https://ppc.land/llms-txt-adoption-rises-8-8x-but-97-of-files-get-zero-ai-requests/)
- Adoption estimates vary by method: SEOmator Sept 2026 scan found 8.6% of the top 100,000 domains (and warns naive 200-OK checks overcount); Thunderbit found 5.86% of Tranco top 10,000. [STUDY] — [SEOmator](https://seomator.com/blog/llms-txt-adoption), [DEV Community](https://dev.to/searchless_ai/llmstxt-adoption-in-2026-only-586-of-top-sites-use-it-but-early-data-reveals-a-clear-signal-39na)
- Common Crawl's analysis of its July 2026 archive found about two-thirds of llms.txt files examined were plugin-generated. [STUDY] — [Common Crawl blog](https://commoncrawl.org/blog/a-content-analysis-of-llms-txt-files-from-the-july-2026-crawl-archive)
- Generators: Yoast SEO for WordPress added llms.txt in v25.3 (June 2025; not available on multisite); Rank Math has an "LLMS Txt" module (default cap reported as 100 links per post type); standalone plugin "Website LLMs.txt" exists on wordpress.org. [SECONDARY] — [theguidex](https://theguidex.com/insights/llms-txt.md), [wordpress.org plugin](https://wordpress.org/plugins/website-llms-txt/)
- Yoast SEO for Shopify added llms.txt on 31 Mar 2026 (top 10 products, up to five collections, policies, homepage). — [Yoast](https://yoast.com/yoast-seo-for-shopify-march-31-2026/)
- Claim that Shopify itself rolled out llms.txt, llms-full.txt and agents.md to all stores in spring 2026. [UNVERIFIED, single third-party blog] — [Crocoblock](https://crocoblock.com/blog/llms-txt-for-wordpress-full-guide/)
- Critique from Joost de Valk that SEO plugins market llms.txt beyond the evidence. — [joost.blog](https://joost.blog/standards-dont-prove-themselves/)

### Inferences
- No evidence supports llms.txt as an AI-search visibility feature. Its plausible use is for coding/browsing agents fetching on demand (the requests that do occur include Claude Code), which matters for documentation sites more than typical CMS content.
- If a CMS offers it, it is cheap to generate from the same data as an XML sitemap, but it should be opt-in and described neutrally; the Markdown-twin (`.md` + `rel=alternate type=text/markdown`) part of the proposal is a separable feature.
- I found no evidence that WordPress core generates llms.txt.

### Gaps
- I did not open the Ahrefs, Originality.ai or EZY primary write-ups; numbers are from summaries.
- No vendor (OpenAI, Anthropic, Perplexity, Microsoft) statement on llms.txt was retrieved. Anthropic and others publish llms.txt for their own docs, but that is not the same as their crawlers consuming it; not verified this session.
- llms-full.txt has no specification on llmstxt.org; its origin (documentation platforms) was not researched.

## 4. Emerging AI-usage-preference standards

### Takeaway
Nothing is ratified. The IETF AIPREF drafts (robots.txt `Content-Usage:` rule plus `Content-Usage` HTTP header) are the closest to a real standard and Google has announced intent to support a training-preference signal in early 2027. Cloudflare's `Content-Signal:` and RSL's `License:` robots.txt lines are deployed at scale on the publisher side but have no documented honoring by major AI vendors.

### Cited Findings

**IETF AIPREF** [DRAFT]
- Two working-group documents, both Internet-Drafts intended as Proposed Standard, no RFC published: `draft-ietf-aipref-attach-05` (18/19 Aug 2026, expires 20 Feb 2027) and `draft-ietf-aipref-vocab-08` (13 Sept 2026). — [IETF datatracker aipref documents](https://datatracker.ietf.org/wg/aipref/documents/)
- attach draft (authors Gary Illyes of Google, Martin Thomson of Mozilla) defines: an HTTP structured-field header, e.g. `Content-Usage: train-ai=n`; and a robots.txt rule inside a user-agent group with optional path, e.g. `Content-Usage: train-ai=n` and `Content-Usage: /ai-ok/ train-ai=y`. Uses RFC 9309 longest-match path rules; preferences apply only to paths that are crawlable; it would update RFC 9309. — [draft-ietf-aipref-attach-05](https://www.ietf.org/archive/id/draft-ietf-aipref-attach-05.html)
- The original WG milestone was Aug 2026; it passed without an RFC. — [datatracker draft page](https://datatracker.ietf.org/doc/draft-ietf-aipref-attach/)
- Google "intends to support the AIPREF" shared training-preference signal as `user-agent: __usage_ai-training` in robots.txt "in early 2027". Note this syntax (a pseudo user-agent) differs from the `Content-Usage:` rule in attach-05; a related individual draft "training-preference-signal-01" is listed on the WG page. — [Google common crawlers](https://developers.google.com/search/docs/crawling-indexing/google-common-crawlers), [datatracker](https://datatracker.ietf.org/wg/aipref/documents/)
- Anthropic says it honors a machine-readable no-AI-training preference signal. — [Anthropic support](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)

**Cloudflare Content Signals Policy** [vendor-published, not a standard]
- Published 24 Sept 2025. robots.txt line `Content-Signal: search=yes, ai-train=no` within a user-agent group. Signals: `search` (index, links, short excerpts; explicitly "does not include providing AI-generated search summaries"), `ai-input` (RAG/grounding, real-time use for AI answers), `ai-train` (training/fine-tuning). Omitted signal = no preference expressed. — [Cloudflare blog](https://blog.cloudflare.com/content-signals-policy/)
- Framed as an express reservation of rights under Article 4 of EU Directive 2019/790. Cloudflare says signals "are not technical countermeasures" and "Some companies might simply ignore them." Managed robots.txt on "over 3.8 million domains" gets `search=yes, ai-train=no` (no ai-input signal). — [Cloudflare blog](https://blog.cloudflare.com/content-signals-policy/)
- An individual draft `draft-romm-aipref-contentsignals-00` (Oct 2025) brought this to the IETF; it is not a WG document. — [IETF archive](https://www.ietf.org/archive/id/draft-romm-aipref-contentsignals-00.html)

**Cloudflare enforcement and payment** [SECONDARY]
- From 15 Sept 2026 Cloudflare defaults block "mixed-use" crawlers (search plus training/agent use in one bot) on ad-supported pages, for new customers, new sites and existing free customers; adjustable. Pay Per Crawl is succeeded by "Pay Per Use" (payment when content is used in an AI result), launched with Ceramic.ai and You.com. Cloudflare says over 50% of AI crawler traffic re-fetches unchanged pages. — [TechCrunch, 1 Jul 2026](https://techcrunch.com/2026/07/01/cloudflares-new-policy-pushes-ai-companies-to-pay-for-publishers-content/)

**RSL (Really Simple Licensing)** [published spec, industry consortium]
- "an open standard for declaring how AI systems and automated agents may access, use, and license digital content"; site states RSL 1.0 is finalized. Declared via a robots.txt `License:` directive pointing at an XML license file, and an `rsl` RSS namespace. Usage values include `ai-train`, `ai-input`, `all`; payment types `free`, `attribution`, `subscription`, `use`, `purchase`. Open License Protocol (OLP) servers issue license tokens. Supporters listed: Akamai, Cloudflare, Creative Commons, Fastly, Internet Brands, O'Reilly, People Inc., Reddit, Vox Media, Yahoo, Ziff Davis. The site names no AI company that honors it. — [rslstandard.org](https://rslstandard.org/)

**TDMRep** [W3C Community Group final report, not a W3C Recommendation]
- W3C TDM Reservation Protocol CG (launched Jan 2021, grounded in EU DSM Directive Art. 4). Three carriers: HTTP headers (`tdm-reservation`, `tdm-policy`), a `/.well-known/tdmrep.json` file, and HTML meta tags. Deployment looks small: a 2026 CG activity page reports declarations on 28 of 7,282 observed hosts. — [W3C TDMRep spec](https://w3c.github.io/tdm-reservation-protocol), [W3C CG](https://www.w3.org/groups/cg/tdmrep)

**noai / noimageai** [informal]
- Informal robots meta values (originating with DeviantArt, 2022); not part of Google's robots meta spec. Originality.ai counted 88,000+ domains carrying them as of June 2026, which measures publisher adoption, not crawler compliance. No source found confirming which AI crawlers honor them. — [Originality.ai](https://originality.ai/blog/noai-noimageai-adoption-dashboard), [IPTC metawatch](https://metawatch.iptc.org/ai-policy/)

### Inferences
- Four competing robots.txt extensions now exist (`Content-Usage:`, `Content-Signal:`, `License:`, and Google's planned `__usage_ai-training` pseudo-agent). A CMS robots.txt API should allow arbitrary extra directive lines inside and outside user-agent groups rather than model only Allow/Disallow/Sitemap, and should let HTTP headers (`Content-Usage`, `tdm-reservation`) be attached per response.
- A neutral internal model of three purposes (search, AI input/grounding, AI training) maps onto Cloudflare, RSL and AIPREF vocabularies and onto the vendor bot-role split in section 1; serialization formats can then be swapped as standards settle.
- In the EU, a machine-readable reservation has legal weight under Art. 4 regardless of crawler compliance, which is a reason to offer the declaration even without enforcement.
- Hosts behind Cloudflare may already have robots.txt rewritten (managed robots.txt); a CMS-generated file is not necessarily what crawlers see.

### Gaps
- AIPREF vocabulary category names in vocab-08 (beyond `train-ai`) were not retrieved; attach-05 only shows `train-ai`. Whether the WG has adopted the `__usage_ai-training` form versus `Content-Usage:` needs a direct read of the training-preference-signal draft.
- contentsignals.org returned no body to my fetch; details come from Cloudflare's blog.
- RSL: release date of 1.0, HTTP Link header / HTML `<link rel="license">` carriers, and any AI-vendor adoption were not confirmed.
- ai.txt (Spawning) and C2PA: no sources retrieved this session. I found no evidence either is consumed by an answer engine for crawl control; C2PA is a provenance-metadata standard for media, not a usage-preference signal. Treat as unresearched.
- Microsoft's and OpenAI's positions on AIPREF were not found.

## 5. JavaScript rendering by AI crawlers

### Takeaway
The best available evidence (one large log study from Dec 2024, widely repeated) says the dedicated AI crawlers fetch but do not execute JavaScript; Applebot documents that it may render, and Google's AI features use Googlebot's rendered index. Server-rendered HTML is therefore the only form of content and structured data that all AI crawlers can see.

### Cited Findings
- Vercel/MERJ analysis (Dec 2024): none of the major AI crawlers studied (OpenAI's, Anthropic's ClaudeBot, Meta-ExternalAgent, Bytespider) rendered JavaScript. GPTBot made 569M and Claude 370M requests on Vercel's network in a month; they do download JS files (reported 11.5% of GPTBot and 23.84% of ClaudeBot requests) without executing them. [STUDY, single source, two years old, not independently replicated in my results] — [Vercel](https://vercel.com/blog/the-rise-of-the-ai-crawler)
- Apple: "Applebot may render the content of your website within a browser"; blocking JS/CSS may prevent proper rendering. [VENDOR] — [Apple support 119829](https://support.apple.com/en-us/119829)
- Google AI Overviews / AI Mode draw on the Google Search index (page must be indexed and snippet-eligible), i.e. Googlebot's pipeline. [VENDOR] — [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- OpenAI, Anthropic and Perplexity bot documentation says nothing about JavaScript rendering. — [OpenAI](https://developers.openai.com/api/docs/bots), [Anthropic](https://support.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler), [Perplexity](https://docs.perplexity.ai/guides/bots)

### Inferences
- Meta tags, canonical, JSON-LD and main content must be in the initial HTML response. JSON-LD injected client-side is likely invisible to GPTBot/ClaudeBot/PerplexityBot. A server-side SEO API (as in a PHP CMS) is the right layer; block themes and interactivity that hydrate content client-side are the risk.
- PerplexityBot was not in the Vercel study; its non-rendering is repeated by secondary blogs only.

### Gaps
- No 2026 vendor statement or replication found; agentic browsers (ChatGPT agent/Atlas, Gemini in Chrome, Claude in Chrome) do render pages, so "AI does not run JS" applies to crawlers, not to browser agents. Not sourced this session.

## 6. Evidence on what improves citation in AI answers

### Takeaway
Vendor-documented levers are few: be crawlable by the right search bot, be indexed and snippet-eligible (Google), avoid NOARCHIVE/NOCACHE (Bing), write single-topic pages with explicit facts (Bing). Everything else (schema, freshness, headings) rests on correlational industry studies with conflicting magnitudes.

### Cited Findings
- **Strong (vendor-documented):** ChatGPT search inclusion requires not blocking OAI-SearchBot. — [OpenAI bots docs](https://developers.openai.com/api/docs/bots)
- **Strong (vendor-documented):** Google: indexed + snippet-eligible is the whole requirement; no special schema or files. — [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- **Moderate (vendor guideline via trade press):** Bing: single-topic pages more likely selected for grounding; state facts directly; NOCACHE/NOARCHIVE/NOSNIPPET reduce or remove Copilot use. — [Search Engine Journal](https://www.searchenginejournal.com/bing-adds-geo-to-official-guidelines-expands-ai-abuse-definitions/568442/)
- **Moderate (vendor statements, extent undisclosed):** ChatGPT search sends queries to third-party providers such as Bing, and an OpenAI VP of Engineering confirmed Bing is an important part of the stack; OpenAI also runs its own index via OAI-SearchBot and reportedly aimed to serve most queries from its own index. Share served by Bing is undisclosed. — [OpenAI Help Center](https://help.openai.com/en/articles/10093903), [SE Roundtable](https://www.seroundtable.com/bing-powers-chatgpt-search-38345.html)
- **Weak-to-moderate (correlational):** Freshness. Ahrefs analysis of ~17M citations (July 2025) found AI-cited content about 25.7% fresher than organically ranked content; a Scrunch/Stacker dataset reported half of AI citations go to content under 13 weeks old. Cosmetic date changes without content changes reportedly do not help. [STUDY, via secondary summaries] — [Writesonic](https://writesonic.com/blog/how-content-freshness-affects-ai-citations), [Radiant Elephant](https://www.radiantelephant.com/?p=51451)
- **Weak (conflicting correlational):** Structured data. One write-up claims pages with comprehensive schema are 3.2x more likely to be cited in AI Overviews; a Surfer study of 10,000 keywords reported a far smaller association (+21.6%). Google states no special schema is needed. — [Digital Applied](https://www.digitalapplied.com/blog/ai-search-citations-drop-38-percent-top-10-pages); contradicted in emphasis by [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)
- **Weak-to-moderate (single study):** Ahrefs reported the share of AI Overview citations drawn from Google's top-10 organic results fell from 76% to 38% between mid-2025 and early 2026 (consistent with query fan-out). [STUDY via secondary] — [Digital Applied](https://www.digitalapplied.com/blog/ai-search-citations-drop-38-percent-top-10-pages)
- **Tooling:** Bing's AI Performance report is the only first-party citation telemetry found; Google folds AI traffic into the Web search type. — [Search Engine Land](https://searchengineland.com/bing-webmaster-tools-testing-new-ai-performance-report-468039), [Google AI features](https://developers.google.com/search/docs/appearance/ai-features)

### Inferences
- For a CMS, the defensible "AI visibility" features are the same as sound technical SEO: server-rendered content, accurate `dateModified`/sitemap `lastmod` that change only on real edits, not blocking search-role AI bots by accident, and valid robots meta. Features marketed as AI-specific (llms.txt, AI schema) lack vendor backing.
- Because ChatGPT search partly depends on Bing, Bing index inclusion (and by extension IndexNow, which feeds Bing) plausibly helps ChatGPT visibility, but that chain is inference; no vendor states IndexNow affects AI citations.

### Gaps
- No vendor statement or controlled study found on IndexNow, lastmod or heading structure affecting AI citations.
- Primary Ahrefs/Surfer/Scrunch reports were not opened; magnitudes are second-hand.
- No evidence found on Perplexity, Claude or Meta AI ranking/citation factors beyond bot access.

## 7. Agent-oriented standards (NLWeb, MCP, agents.json, WebMCP)

### Takeaway
All are early. WebMCP is a W3C Community Group draft with a Chrome origin trial and Gemini in Chrome as the only reported consumer; NLWeb is a Microsoft open-source project with a handful of named early adopters. None is documented as an input to any search or answer engine's ranking or citation.

### Cited Findings
- **WebMCP** [DRAFT]: W3C Draft Community Group Report (sources give latest publication as 23 Apr 2026 or 26 Aug 2026), co-developed by Google and Microsoft; exposes page tools to in-browser agents via `navigator.modelContext`. Shipped behind a flag in Chrome 146 (Mar 2026); public origin trial reported for Chrome 149 through 156, announced at Google I/O 2026. Gemini in Chrome reported as the only consumer; site adoption near zero; Firefox and Safari have not committed. [SECONDARY, third-party blogs with conflicting dates] — [studiomeyer reality check](https://dev.to/studiomeyer_io/webmcp-reality-check-where-the-spec-actually-stands-4gh1), [eWeek](https://www.eweek.com/news/google-webmcp-chrome-ai-web-standard-preview/), [W3C CG notes](https://w3c-cg.github.io/aikr/webMCP/webmcp-technical-notes.html)
- Lighthouse 13.3's experimental Agentic Browsing category checks WebMCP integration and llms.txt; John Mueller reportedly pointed site owners toward WebMCP rather than llms.txt. [SECONDARY] — [Search Engine Journal](https://www.searchenginejournal.com/googles-llms-txt-guidance-depends-on-which-product-you-ask/575431/)
- **NLWeb** [vendor open-source project]: announced by Microsoft at Build 2025; lets a site answer natural-language queries over its schema.org/RSS data via an `/ask` endpoint; each instance is also an MCP server; discovery reportedly via `<link rel="nlweb">`. Early adopters named in 2025: TripAdvisor, Shopify, Eventbrite, Hearst, O'Reilly. Described as early and mostly Microsoft-driven. No 2026 adoption figures found. — [Yoast explainer](https://yoast.com/what-is-nlweb/), [specification.website](https://specification.website/spec/agent-readiness/nlweb/)
- **Agent payments/marketplaces**: Cloudflare Pay Per Use (Ceramic.ai, You.com) is the only deployed machine-to-site commercial mechanism found. [SECONDARY] — [TechCrunch](https://techcrunch.com/2026/07/01/cloudflares-new-policy-pushes-ai-companies-to-pay-for-publishers-content/)

### Inferences
- These address agents acting on a site (forms, search, checkout), not citation in answers. For a built-in SEO API they are out of scope beyond leaving room for `<link rel>` discovery tags and well-known endpoints. NLWeb's reliance on schema.org JSON-LD and feeds means good structured data and feeds are the reusable foundation.
- WordPress-relevant: the repo's own Abilities API / MCP work is the natural home for agent endpoints, separate from SEO.

### Gaps
- agents.json, `.well-known/mcp` discovery, A2A agent cards, and OpenAI/Stripe Agentic Commerce Protocol: no usable sources retrieved; status unknown from this research.
- WebMCP dates conflict between sources; verify against the Chrome developer blog and the W3C CG repo.
- No source shows any answer engine (Google, Bing, OpenAI, Perplexity, Anthropic) using NLWeb or WebMCP for discovery or citation.
