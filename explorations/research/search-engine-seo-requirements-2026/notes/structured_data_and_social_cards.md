# Structured data (schema.org) and social / link-preview metadata — state as of November 2026

Research date: 2026-11-17. Primary sources were fetched live (Google Search Central, ogp.me, oembed.com, Yoast and Rank Math developer docs, Meta, LinkedIn, Mastodon, Pinterest, Yandex). Where only secondary sources were reachable (X Cards, Bing, Baidu, chat-app unfurlers) this is flagged inline and in Gaps.

## 1. Which schema.org types does Google support for rich results, and with which required / recommended properties?

### Takeaway
Google's Search Gallery in November 2026 lists 25 features; FAQ, HowTo, sitelinks search box, practice problems, fact check, course info, learning video, estimated salary, special announcement, vehicle listing and book actions are all gone. For a general CMS the durable core is Article, BreadcrumbList, Organization, WebSite (site name), ProfilePage/Person, Product (snippet + merchant listing), Review/AggregateRating, LocalBusiness, Event, Recipe, VideoObject, ImageObject metadata, QAPage, DiscussionForumPosting and paywall markup; several of these have no "required" properties at all, only recommended ones.

### Cited Findings

**Current Search Gallery (complete list as fetched Nov 2026)**
- The gallery lists: Article, Breadcrumb, Carousel (must be combined with Recipe, Course list, Restaurant or Movie), Course list, Dataset, Discussion forum, Education Q&A, Employer aggregate rating, Event, Image metadata, Job posting, Local business, Math solver, Movie, Organization, Product, Profile page, Q&A, Recipe, Review snippet, Software app, Speakable, Subscription and paywalled content, Vacation rental, Video. FAQ and HowTo are not in the list. — [Google Search Gallery](https://developers.google.com/search/docs/appearance/structured-data/search-gallery)

**Article / NewsArticle / BlogPosting**
- Supported types: `Article`, `NewsArticle`, `BlogPosting`. "There are no required properties; instead, add the properties that apply to your content." — [Google Article docs](https://developers.google.com/search/docs/appearance/structured-data/article)
- Recommended: `author` (Person or Organization) with `author.name` (name only, no titles/honorifics/"posted by") and `author.url` (page uniquely identifying the author; `sameAs` accepted alternative); `dateModified`; `datePublished` (ISO 8601, timezone recommended, otherwise Googlebot's timezone is assumed); `headline`; `image` (repeated ImageObject or URL). — [Google Article docs](https://developers.google.com/search/docs/appearance/structured-data/article)
- Image rules: crawlable and indexable, relevant to the article (not logos or captions), Google Images-supported format, multiple high-resolution images with at least 50K pixels (width x height) each, in 16x9, 4x3 and 1x1 aspect ratios. — [Google Article docs](https://developers.google.com/search/docs/appearance/structured-data/article)
- Author best practice: include all on-page authors, one `author` entry per author (do not merge names), use correct `@type` (Person vs Organization), provide `url`/`sameAs`, do not put the publisher in `author` (use `publisher`). — [Google Article docs](https://developers.google.com/search/docs/appearance/structured-data/article)

**BreadcrumbList**
- Required: `itemListElement` (ordered ListItems, at least two); each `ListItem` needs `position` (integer, 1 = start of trail), `name`, and `item` (URL or Thing with id; optional on the last item, where Google uses the containing page URL). No recommended properties listed. — [Google Breadcrumb docs](https://developers.google.com/search/docs/appearance/structured-data/breadcrumb)
- Breadcrumb rich result is desktop only (documented 22 Jan 2025), all regions/languages. Breadcrumbs should reflect a typical user path, not mirror URL structure; the root domain and current page do not need ListItems. data-vocabulary.org markup is ineligible. — [Google Breadcrumb docs](https://developers.google.com/search/docs/appearance/structured-data/breadcrumb); [Search Central updates log](https://developers.google.com/search/updates)

**Organization**
- No required properties. Place on the home page or a single page such as "About us"; not needed on every page. Use the most specific subtype (e.g. `OnlineStore`); local businesses should use `LocalBusiness` subtypes. — [Google Organization docs](https://developers.google.com/search/docs/appearance/structured-data/organization)
- Recommended: `name` (match site name), `alternateName`, `legalName`, `url`, `description`, `foundingDate`, `numberOfEmployees`, `telephone`, `email`, `contactPoint` (`contactType`, `telephone`, `email`), `address` (PostalAddress: `addressCountry` 2-letter ISO, `addressLocality`, `addressRegion`, `postalCode`, `streetAddress`), `sameAs`, `logo`. — [Google Organization docs](https://developers.google.com/search/docs/appearance/structured-data/organization)
- Logo: at least 112x112 px, crawlable/indexable URL, Google Images-supported format, must look right on a pure white background; an ImageObject needs valid `contentUrl` or `url`. — [Google Organization docs](https://developers.google.com/search/docs/appearance/structured-data/organization)
- Identifiers: `iso6523Code` preferred (ICD:identifier; 0060 = DUNS, 0088 = GLN, 0199 = LEI); also `duns`, `globalLocationNumber`, `leiCode`, `vatID`, `taxID`, `naics`. Merchant properties: `hasMerchantReturnPolicy`, `hasShippingService` (ShippingService), `hasMemberProgram` (MemberProgram). — [Google Organization docs](https://developers.google.com/search/docs/appearance/structured-data/organization)
- Organization docs were introduced 29 Nov 2023 (merging the old Logo docs); org-level return policy docs added 11 Jun 2024; loyalty program docs 10 Jun 2025; merchant-level shipping policy docs 12 Nov 2025. — [Search Central updates log](https://developers.google.com/search/updates)

**WebSite (site name)**
- Add `WebSite` to the home page only. Required: `name`, `url` (canonical home URL of the domain or subdomain). Recommended: `alternateName` (may be several, in preference order). — [Google site names docs](https://developers.google.com/search/docs/appearance/site-names)
- Must be on the root of a domain or subdomain; `www`/`m` are equivalent to the bare domain; subdomains supported; subdirectories (`example.com/news`) are not supported. One site name per domain/subdomain. Duplicate home pages (http/https, www/non-www) must all carry the same markup. Add to the existing WebSite node rather than creating a second one. — [Google site names docs](https://developers.google.com/search/docs/appearance/site-names)
- Google also considers `og:site_name`, `<title>`, headings and other home page text, but WebSite markup carries the most weight. Rich Results Test does not support site names; use a generic schema validator. — [Google site names docs](https://developers.google.com/search/docs/appearance/site-names)

**ProfilePage / Person**
- `ProfilePage` requires `mainEntity` (Person or Organization; default to Person if unknown). Recommended: `dateCreated`, `dateModified`. — [Google Profile page docs](https://developers.google.com/search/docs/appearance/structured-data/profile-page)
- mainEntity requires `name` (or `alternateName`). Recommended: `alternateName` (handle), `description`, `identifier`, `image` (real profile image, same 16x9/4x3/1x1 and 50K px guidance), `sameAs`, `interactionStatistic` (FollowAction, LikeAction, BefriendAction counts received on this platform), `agentInteractionStatistic` (FollowAction, LikeAction, WriteAction = post count, ShareAction). — [Google Profile page docs](https://developers.google.com/search/docs/appearance/structured-data/profile-page)
- Profile page and discussion forum docs were added 27 Nov 2023. — [Search Central updates log](https://developers.google.com/search/updates)

**Product: product snippets vs merchant listings**
- Product docs were split into three pages on 1 May 2024. — [Search Central updates log](https://developers.google.com/search/updates)
- Product snippet (editorial/review pages): Product requires `name` plus one of `review`, `aggregateRating`, `offers`. Offer requires `price` (or `priceSpecification.price`); recommended `availability`, `priceCurrency`, `priceValidUntil` (a past date may suppress the snippet). `AggregateOffer` requires `lowPrice`, `priceCurrency`; recommended `highPrice`, `offerCount`. Price-drop eligibility requires `Offer`, not `AggregateOffer`. Pros/cons: `positiveNotes`/`negativeNotes` ItemLists on an editorial Review, at least two statements in total. — [Google Product snippet docs](https://developers.google.com/search/docs/appearance/structured-data/product-snippet)
- Merchant listing (page where the product can be bought): Product requires `name`, `image`, `offers` (must be `Offer`). Recommended: `aggregateRating`, `audience`, `brand.name`, `category`, `color`, `description`, GTIN (`gtin`/`gtin8`/`gtin12`/`gtin13`/`gtin14`/`isbn`), `hasAdultConsideration`, `hasCertification`, `inProductGroupWithID`, `isVariantOf`, `material`, `mpn`, `pattern`, `review`, `size`, `sku`, `subjectOf`. — [Google Merchant listing docs](https://developers.google.com/search/docs/appearance/structured-data/merchant-listing)
- Merchant listing Offer requires `price` (> 0) and `priceCurrency` (directly or via `priceSpecification`); recommended `availability`, `hasMerchantReturnPolicy`, `itemCondition`, `priceValidUntil`, `shippingDetails`, `url`, `validFrom`, `validThrough`. If both `offers.price` and `offers.priceSpecification` are present, `offers.price` wins. — [Google Merchant listing docs](https://developers.google.com/search/docs/appearance/structured-data/merchant-listing)
- `UnitPriceSpecification`: required `price`, `priceCurrency`; recommended `priceType` (only `StrikethroughPrice` supported), `validForMemberTier`, `membershipPointsEarned` (beta), `referenceQuantity`; `priceType` and `validForMemberTier` cannot be combined. — [Google Merchant listing docs](https://developers.google.com/search/docs/appearance/structured-data/merchant-listing)
- `OfferShippingDetails`: required `deliveryTime`, `shippingDestination` (DefinedRegion with `addressCountry`), `shippingRate` (`currency` plus `value` or `maxValue`); handling/transit times as QuantitativeValue with `minValue`, `maxValue`, `unitCode` DAY. `MerchantReturnPolicy`: required `applicableCountry`, `returnPolicyCategory`; recommended `merchantReturnDays`, `returnFees`, `returnMethod`, `returnShippingFeesAmount`. `Certification`: required `issuedBy`, `name`. — [Google Merchant listing docs](https://developers.google.com/search/docs/appearance/structured-data/merchant-listing)
- Product-related additions by date: `suggestedAge` 9 Jan 2024; product variants / `isVariantOf` (ProductGroup) 20 Feb 2024; 3D model 25 Mar 2024; `priceType` sale pricing 23 Sep 2024; `Certification` 3 Oct 2024 (`EnergyConsumptionDetails` removed 24 Apr 2025); member pricing `validForMemberTier` 13 Feb 2025; `hasAdultConsideration` 20 May 2026; `Product.category` with `CategoryCode` and sale-duration section 7 Jul 2026. — [Search Central updates log](https://developers.google.com/search/updates)

**Review snippet (Review / AggregateRating / Rating)**
- Eligible parents: Book, Course list, Event, Local business, Movie, Product, Recipe, Software App, plus CreativeWorkSeason, CreativeWorkSeries, Episode, Game, MediaObject, MusicPlaylist, MusicRecording, Organization. — [Google Review snippet docs](https://developers.google.com/search/docs/appearance/structured-data/review-snippet)
- Self-serving rule: if the entity controls reviews about itself (including via third-party widgets), pages with LocalBusiness or Organization markup are ineligible for stars. — [Google Review snippet docs](https://developers.google.com/search/docs/appearance/structured-data/review-snippet)
- `Review` requires `author` (Person/Organization, name under 100 chars), `itemReviewed` (omit when nested), `reviewRating.ratingValue`; recommended `datePublished`, `bestRating` (default 5), `worstRating` (default 1). `AggregateRating` requires `ratingValue` and `ratingCount` or `reviewCount`, plus `itemReviewed` unless nested. Decimal separator must be a dot. — [Google Review snippet docs](https://developers.google.com/search/docs/appearance/structured-data/review-snippet)

**LocalBusiness**
- Required: `name`, `address` (PostalAddress). Recommended: `geo` (lat/long with at least 5 decimals), `openingHoursSpecification` (`dayOfWeek`, `opens`, `closes` hh:mm:ss; `validFrom`/`validThrough` for seasonal), `priceRange` (< 100 chars), `telephone`, `url`, `department`, `menu`, `servesCuisine`; `aggregateRating`/`review` only for sites reviewing other businesses. Use the most specific subtype; multiple types as an array; `additionalType` not supported. 24h = opens 00:00 closes 23:59; closed all day = both 00:00. — [Google Local business docs](https://developers.google.com/search/docs/appearance/structured-data/local-business)

**Event**
- Required: `name`, `startDate` (ISO 8601), `location` (Place with `address`). Recommended: `description`, `endDate`, `eventStatus`, `previousStartDate`, `image`, `offers` (`availability`, `price`, `priceCurrency`, `validFrom`, `url`), `organizer` (`name`, `url`), `performer.name`. — [Google Event docs](https://developers.google.com/search/docs/appearance/structured-data/event)
- Online-only events are no longer supported ("Events must take place in a physical location"); online event properties were removed from the docs on 5 Jun 2025. Available only in listed regions/languages (AU, BR, CA, DE, IN, Latin America, ES, UK, US). — [Google Event docs](https://developers.google.com/search/docs/appearance/structured-data/event); [Search Central updates log](https://developers.google.com/search/updates)

**Recipe**
- Required: `name`, `image`. Recommended: `aggregateRating`, `author`, `cookTime`+`prepTime` (paired), `totalTime`, `datePublished`, `description`, `keywords`, `nutrition.calories` (needs `recipeYield`), `recipeCategory`, `recipeCuisine`, `recipeIngredient`, `recipeInstructions` (HowToStep with `text`; HowToSection for multi-part), `recipeYield`, `video`. Host carousel: `ItemList` with `ListItem.position` and unique canonical `ListItem.url`. Guided recipes were removed 6 Mar 2024. — [Google Recipe docs](https://developers.google.com/search/docs/appearance/structured-data/recipe); [Search Central updates log](https://developers.google.com/search/updates)

**VideoObject**
- Required: `name`, `thumbnailUrl`, `uploadDate` (ISO 8601). Recommended: `contentUrl` (preferred) or `embedUrl`, `creator`/`author` (added 24 Sep 2026), `description`, `duration`, `expires`, `hasPart` (Clip), `ineligibleRegion`/`regionsAllowed`, `interactionStatistic` (WatchAction, LikeAction, CommentAction, ShareAction), `publication` (BroadcastEvent). Markup must be on a page where users can watch the video. — [Google Video docs](https://developers.google.com/search/docs/appearance/structured-data/video); [Search Central updates log](https://developers.google.com/search/updates)
- LIVE badge: `publication` BroadcastEvent with `isLiveBroadcast: true`, `startDate`, `endDate`. Key moments: `Clip` (`name`, `startOffset`, `url`; recommended `endOffset`) or `potentialAction` SeekToAction (`target` with `{seek_to_second_number}`, `startOffset-input: "required name=seek_to_second_number"`). — [Google Video docs](https://developers.google.com/search/docs/appearance/structured-data/video)

**ImageObject / image license metadata**
- Required: `contentUrl` plus at least one of `creator`, `creditText`, `copyrightNotice`, `license`. Recommended: `license` (needed for the Licensable badge), `acquireLicensePage`, `creator.name`, `creditText`, `copyrightNotice`. Alternative: embedded IPTC photo metadata (Web Statement of Rights required for the badge); structured data wins on conflict. Must be on every page where the image appears. — [Google Image metadata docs](https://developers.google.com/search/docs/appearance/structured-data/image-license-metadata)

**QAPage**
- `QAPage.mainEntity` = one `Question` (one QAPage per page). Question requires `name`, `answerCount`, and `acceptedAnswer` and/or `suggestedAnswer`; Answer requires `text` (`url` strongly recommended); Comment requires `text`. Recommended on all: `author`+`author.url`, `datePublished`, `dateModified`, `upvoteCount`, `comment`, `commentCount`, `image`, `video`, `digitalSourceType`. Valid only when users can submit answers to a single question; site-authored FAQs, multi-question pages, blog posts and how-tos are invalid. — [Google Q&A docs](https://developers.google.com/search/docs/appearance/structured-data/qapage)

**DiscussionForumPosting / SocialMediaPosting**
- Required: `author` (+`author.name`), `datePublished`, and one of `text`/`image`/`video`. Recommended: `headline`, `url`, `author.url`, `comment`, `commentCount`, `interactionStatistic`, `sharedContent`, `isPartOf`, `dateModified`, `creativeWorkStatus` (`Deleted`), `digitalSourceType`. Comment requires `author`, `datePublished`, and text/image/video. — [Google Discussion forum docs](https://developers.google.com/search/docs/appearance/structured-data/discussion-forum)
- Only for user-generated posts; publisher-authored articles/blogs with comments must use Article instead. Google recommends Microdata or RDFa over JSON-LD here to avoid duplicating large text blocks (JSON-LD still supported). `SocialMediaPosting` support confirmed 4 Jun 2024; more comment-thread properties added 24 Mar 2026. — [Google Discussion forum docs](https://developers.google.com/search/docs/appearance/structured-data/discussion-forum); [Search Central updates log](https://developers.google.com/search/updates)

**JobPosting**
- Required: `datePosted`, `description`, `hiringOrganization`, `title`, `jobLocation` (unless fully remote with `applicantLocationRequirements`; `jobLocationType: TELECOMMUTE` required for 100% remote). Recommended: `baseSalary`, `directApply`, `identifier`, `validThrough`, `employmentType`. Google recommends the Indexing API for job URLs. — [Google Job posting docs](https://developers.google.com/search/docs/appearance/structured-data/job-posting)

**Course list**
- Course requires `name`, `description` (60-char display limit); recommended `provider`. Needs an `ItemList` with at least 3 courses. English only. (The separate "Course info" feature was deprecated in 2025; see section 2.) — [Google Course list docs](https://developers.google.com/search/docs/appearance/structured-data/course)

**SoftwareApplication**
- Types: `SoftwareApplication`, `MobileApplication`, `WebApplication` (`VideoGame` only when co-typed). Required: `name`, `offers.price` (0 for free), and `aggregateRating` or `review`. Recommended: `applicationCategory` (enumerated list), `operatingSystem`. — [Google Software app docs](https://developers.google.com/search/docs/appearance/structured-data/software-app)

**Dataset**
- Required: `name`, `description` (50–5,000 chars). Many recommended properties (`creator`, `license`, `identifier`, `distribution.contentUrl`, `temporalCoverage`, `spatialCoverage`, `includedInDataCatalog`, etc.). Since 5 Nov 2025 Google states Dataset markup is used only by Dataset Search, not main Search. — [Google Dataset docs](https://developers.google.com/search/docs/appearance/structured-data/dataset); [Search Central updates log](https://developers.google.com/search/updates)

**Speakable**
- Still beta; US users with Google Home devices in English, English-language publishers. `speakable` SpeakableSpecification with `cssSelector` or `xPath` (not both) on `Article` or `WebPage`; target 20–30 seconds of content per section. — [Google Speakable docs](https://developers.google.com/search/docs/appearance/structured-data/speakable)

**Paywalled content**
- On a `CreativeWork` subtype (Article, NewsArticle, Blog, Comment, Course, HowTo, Message, Review, WebPage): `isAccessibleForFree: false` (required) plus `hasPart` `{ "@type": "WebPageElement", "isAccessibleForFree": false, "cssSelector": ".paywall" }` (recommended; array for multiple sections). Only `.class` selectors; do not nest sections; JSON-LD and microdata accepted; apply to all page versions. — [Google paywalled content docs](https://developers.google.com/search/docs/appearance/structured-data/paywalled-content)

### Inferences
- A CMS core graph can safely emit, with no site-owner input, the types that have zero or trivially satisfiable required properties: WebSite (home page), Organization/Person (home page), WebPage, BreadcrumbList, Article/BlogPosting, ProfilePage on author archives, ImageObject. Everything else (Product, Event, Recipe, JobPosting, LocalBusiness, Video) needs content-type-specific data and is best exposed through an extension API.
- WordPress comments on publisher-authored posts should be modelled as `Comment` on the Article, not as DiscussionForumPosting; forum plugins (bbPress-type content) are the DiscussionForumPosting/QAPage case.
- The Article image guidance (16x9, 4x3, 1x1, at least 50K pixels) implies a CMS should generate or select three crops of the featured image.
- Because FAQ and HowTo no longer produce Google rich results, built-in FAQ/HowTo blocks should not be justified on rich-result grounds; `HowToStep`/`HowToSection` remain relevant only inside Recipe.

### Gaps
- Not fetched individually: Movie, Math solver, Education Q&A, Employer aggregate rating, Vacation rental, Carousel (beta), product variants (ProductGroup) detail page, loyalty program, shipping policy and return policy detail pages. Listed as existing in the gallery only.
- No Google page defines a "WebPage" rich result; WebPage is used by Yoast-style graphs as glue, and Google's consumption of plain WebPage nodes is undocumented.

## 2. Which rich result types were deprecated, restricted or removed in 2023–2026?

### Takeaway
Google has steadily pruned rich results: HowTo (2023), sitelinks search box (2024), seven types in mid-2025, practice problems (Jan 2026) and finally FAQ entirely (May–June 2026). Google states leftover markup is harmless and removals do not affect ranking.

### Cited Findings
- 8 Aug 2023: "FAQ rich results will only be shown for well-known, authoritative government and health websites"; HowTo rich results limited to desktop. Not a ranking change; unused markup need not be removed. — [Search Central blog, Aug 2023](https://developers.google.com/search/blog/2023/08/howto-faq-changes)
- 13/14 Sep 2023: "Google Search no longer shows How-to rich results on desktop" — HowTo fully deprecated; Search Console report and Rich Results Test support dropped 30 days later, Search Console API support 180 days later. — [Search Central blog, Aug 2023 (Sept update)](https://developers.google.com/search/blog/2023/08/howto-faq-changes)
- 6 Mar 2024: guided recipes removed from recipe docs. 11 Jun 2024: home activity structured data docs removed. — [Search Central updates log](https://developers.google.com/search/updates)
- 21 Nov 2024: sitelinks search box removed from Search; the structured data is unsupported but "won't cause issues in Search" and "there's no need" to remove it; WebSite markup for site names continues to be supported. Docs removed 29 Nov 2024. — [Search Central blog, Oct 2024](https://developers.google.com/search/blog/2024/10/sitelinks-search-box); [Search Central updates log](https://developers.google.com/search/updates)
- 22 Jan 2025: breadcrumb rich result documented as desktop only. — [Search Central updates log](https://developers.google.com/search/updates)
- 5 Jun 2025: online event properties removed from Event docs. — [Search Central updates log](https://developers.google.com/search/updates)
- 12 Jun 2025: Google announced phase-out of Book Actions, Course Info, Claim Review, Estimated Salary, Learning Video, Special Announcement and Vehicle Listing "over the coming weeks and months"; "This update won't affect how pages are ranked." Search Console reporting removed from 9 Sep 2025 (six types, Book Actions not in that list); Search Console API support through Dec 2025; bulk export fields NULL by 1 Oct 2025. — [Search Central blog, Jun 2025](https://developers.google.com/search/blog/2025/06/simplifying-search-results)
- Special announcement had its own earlier notice (23 Apr 2025, effective 31 Jul 2025). Docs for course info, estimated salary, learning video, special announcement and vehicle listing were removed 9 Sep 2025. — [Search Central updates log](https://developers.google.com/search/updates)
- 5 Nov 2025: Google said it is phasing out more lesser-used features and "Starting in January 2026, we'll remove support for the structured data types in Search Console and its API." The updates log for that date records the practice problem deprecation notice, Dataset markup clarified as Dataset Search only, and removal of the Book actions deprecation banner. — [Search Central blog, Nov 2025](https://developers.google.com/search/blog/2025/11/update-on-our-efforts); [Search Central updates log](https://developers.google.com/search/updates)
- 6 Jan 2026: practice problem documentation removed. — [Search Central updates log](https://developers.google.com/search/updates)
- 8 May 2026: FAQ rich result deprecation notice added, stating the feature no longer appears in Google Search from 7 May 2026. 15 Jun 2026: FAQ rich result documentation removed. — [Search Central updates log](https://developers.google.com/search/updates)
- Secondary reporting of the Nov 2025 round also names nutrition facts, nearby offers and other SERP features among the removals. — [Search Engine Land](https://searchengineland.com/google-to-remove-more-search-features-including-practice-problems-nutrition-facts-nearby-offers-and-more-464255)

### Inferences
- The trend is contraction to commerce, media, UGC and identity types. A CMS should treat rich-result eligibility as volatile and keep per-type generators modular and removable.
- Because Google says obsolete markup is harmless, a CMS need not emit migrations to strip FAQPage/HowTo/SearchAction, but should stop presenting them as producing Google features. Other consumers (Bing, Yandex Q&A, AI systems) may still read them.

### Gaps
- The Nov 2025 blog post does not itself enumerate the affected features; the full list comes from the changelog and secondary press.
- The "Book actions deprecation banner removed" entry is ambiguous (page removed or deprecation reversed); Book actions is absent from the current gallery, suggesting removal.
- The 2023 portion of the updates log before mid-Nov 2023 was not read (page truncated).
- Fact check (ClaimReview) is absent from the current gallery; I did not fetch a dedicated page confirming its final removal date beyond the June 2025 announcement.

## 3. Format preference, general guidelines, and how to build a connected @graph (Yoast, Rank Math and others)

### Takeaway
Google supports JSON-LD, Microdata and RDFa and recommends JSON-LD; the binding policies are visibility, accuracy, completeness of required properties and specificity. Yoast's model — one JSON-LD script with an `@graph` of flat top-level "pieces" cross-referenced by stable `@id`s, extended through piece classes and filters — is the reference design; Rank Math exposes a single array filter plus per-type entity filters.

### Cited Findings

**Google format and policy**
- JSON-LD recommended; Microdata and RDFa supported. — [Google structured data policies](https://developers.google.com/search/docs/appearance/structured-data/sd-policies)
- "In general, we recommend using a format that's easiest for you to implement and maintain (in most cases, that's JSON-LD)"; Google can read JSON-LD "when it is dynamically injected into the page's contents". — [Google intro to structured data](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data)
- Google's docs, not schema.org, are "definitive for Google Search behavior"; all required properties are needed for eligibility; "it is more important to supply fewer but complete and accurate recommended properties". — [Google intro to structured data](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data)
- Policies: do not block marked-up pages from Googlebot; do not mark up content not visible to readers; no irrelevant/misleading content or fake reviews; keep time-sensitive data current; structured data must reflect the page; use the most specific type; images must be crawlable, indexable and relevant. — [Google structured data policies](https://developers.google.com/search/docs/appearance/structured-data/sd-policies)
- Location: put structured data on the page it describes; for duplicate pages put the same markup on all duplicates, not only the canonical. — [Google structured data policies](https://developers.google.com/search/docs/appearance/structured-data/sd-policies)
- Multiple items: either nest related items under the main item or emit separate items; items that belong together can be linked by using the same `@id`; include the main type matching the page's primary purpose; mark up all visible items (e.g. all reviews shown). — [Google structured data policies](https://developers.google.com/search/docs/appearance/structured-data/sd-policies)
- Google switched its `@id` examples to hash-fragment style on 9 Jan 2024. — [Search Central updates log](https://developers.google.com/search/updates)
- Violations can lead to a manual action making the page ineligible for rich results. — [Google Organization docs](https://developers.google.com/search/docs/appearance/structured-data/organization)

**Yoast SEO schema graph**
- One JSON-LD block with `"@context": "https://schema.org"` and an `@graph` array of top-level pieces that reference each other by `@id`; base script on every page = Organization (or Person), WebSite, WebPage. — [Yoast functional specification](https://developer.yoast.com/features/schema/functional-specification/)
- Relationships: WebPage `isPartOf` WebSite; WebSite `publisher` Organization; WebPage `breadcrumb` BreadcrumbList; home WebPage `about` Organization; Article `isPartOf` and `mainEntityOfPage` WebPage, `author` Person, `publisher` Organization, `image`/`video` references; Product `mainEntityOfPage` WebPage, `offers` Offer (Offer `seller` Organization). WooCommerce product pages type the page `["WebPage","ItemPage"]`. — [Yoast functional specification](https://developer.yoast.com/features/schema/functional-specification/)
- ID convention in the spec: `{{website}}/#/schema/{{type}}/{{ID}}`; identifier cascade = globally unique stable system ID, else stable ID synthesized from context, else page-local sequential integer. Exceptions: `ImageObject` @id is the absolute image URL; `WebPage` @id is the unmodified canonical permalink; site Organization and WebSite use identifier `1`; Person identifiers must be obfuscated. Hybrid types concatenate alphabetically with hyphens (e.g. `organization-person`). — [Yoast technology and approach](https://developer.yoast.com/features/schema/technology-approach/)
- Principles: each URL has one primary entity; avoid deep nesting, reference by `@id`; each page outputs all relevant pieces rather than relying on cross-page ID references; third parties may add separate `<script>` blocks that join the graph by reusing `@id`s (Yoast WooCommerce SEO does this from the footer). — [Yoast technology and approach](https://developer.yoast.com/features/schema/technology-approach/)
- Extension API: `wpseo_schema_graph_pieces` (`$pieces, $context`) to add/remove pieces (custom pieces extend `Abstract_Schema_Piece` and receive a `Meta_Tags_Context`); `wpseo_schema_needs_<class>` to force a piece on/off; `wpseo_schema_<class>` (e.g. `wpseo_schema_webpage`, `wpseo_schema_article`) to alter a piece's data; `wpseo_schema_webpage_type`; `wpseo_schema_graph` (`$data, $context`) for whole-graph changes; `wpseo_schema_block_<block-type>` (`$graph, $block, $context`) for block-driven schema; `wpseo_schema_person_user_id`; `wpseo_json_ld_output` returning false disables output; `yoast_seo_development_mode` pretty-prints. `Schema_IDs` holds fixed ID constants (e.g. `ORGANIZATION_HASH`). Schema also appears in REST as `yoast_head_json`. — [Yoast Schema API](https://developer.yoast.com/features/schema/api/)

**Rank Math**
- Central filter `rank_math/json_ld` (`$data, $jsonld`, typically priority 99) receives the whole entity array; returning an empty array removes output. Per-type filters `rank_math/snippet/rich_snippet_{$schema}_entity` (`$entity`) modify one entity; `rank_math/snippet/rich_snippet_{$schema_type}` can suppress a type; `rank_math/json_ld/breadcrumbs_enabled`, `rank_math/json_ld/disable_search`, `rank_math/json_ld/search_url`, `rank_math/schema/add_global_entities`, `rank_math/snippet/breadcrumb`, `rank_math/settings/snippet/type` and `.../article_type` (defaults per post type), block content filter `rank_math/schema/block/{$block}/content`. — [Rank Math filters and hooks](https://rankmath.com/kb/filters-hooks-api-developer/)
- Rank Math social hooks: `rank_math/opengraph/type`, `rank_math/opengraph/url`, `rank_math/opengraph/image_sizes` (default full, large, medium_large), `rank_math/opengraph/{$network}/image`, `rank_math/opengraph/{$network}/{$og_property}` (network = facebook|twitter), `rank_math/opengraph/twitter_card`, `rank_math/opengraph/twitter/card_type`, `rank_math/woocommerce/og_price` (controls `product:price:amount` / `product:price:currency`). — [Rank Math filters and hooks](https://rankmath.com/kb/filters-hooks-api-developer/)

### Inferences
- A core design consistent with both Google's guidance and Yoast's model: a registry of "pieces" each with `is_needed( context )` + `generate( context )`, deterministic `@id`s (site-level `#/schema/...` fragments, page URL for WebPage, file URL for images), a single `<script type="application/ld+json">` with `@graph`, and three filter layers (piece list, per-piece data, whole graph).
- Google's "same @id links items" guidance and Yoast's "reuse an @id from a second script" pattern mean plugins can enrich core nodes without core needing to know about them, provided core's IDs are stable and documented as public API.
- Google's duplicate-page rule (same markup on duplicates) means the graph should be emitted on non-canonical variants too, with `@id`/`url` pointing at the canonical.
- Rank Math's `@id`-keyed associative array passed through one filter is simpler but less structured than Yoast's class-based pieces; the Yoast approach is better suited to a core API.

### Gaps
- The Yoast API page fetched did not show `Abstract_Schema_Piece::is_needed()` / `generate()` signatures; these method names are from prior knowledge and should be verified in the Yoast source.
- Yoast's shipped plugin historically used IDs such as `{site}/#website`, `{site}/#organization`, `{url}#breadcrumb`, `{url}#primaryimage`, `{url}#article`; the developer docs fetched show only the `/#/schema/{type}/{id}` pattern. Whether the plugin output has fully migrated was not verified.
- No primary documentation fetched for other plugins (SEOPress, All in One SEO, The SEO Framework, Slim SEO Schema) describing their graph model or extension API.
- Rank Math's internal graph structure (its @id conventions and how entities are keyed) is not described on the hooks page.

## 4. What does Bing support, and what do Yandex and Baidu consume?

### Takeaway
Bing reads schema.org in JSON-LD, Microdata and RDFa plus Microformats and Open Graph; Yandex reads schema.org (Microdata and JSON-LD), Open Graph and microformats; Baidu's JSON-LD/schema.org support could not be confirmed from primary sources. Current primary documentation for Bing was not retrievable, so details are dated.

### Cited Findings
- Bing's Markup Validator "supports six markup languages, including Schema.org, HTML Microdata, Microformats, Open Graph and RDFa", with JSON-LD added in August 2018. — [Bing Webmaster Blog, Aug 2018](https://blogs.bing.com/webmaster/august-2018/Introducing-JSON-LD-Support-in-Bing-Webmaster-Tools)
- Bing co-founded schema.org with Google and Yahoo in June 2011. — [Bing blog, Jun 2011](https://blogs.bing.com/search/June-2011/Introducing-Schema-org-Bing,-Google-and-Yahoo-Uni)
- A Microsoft Advertising help page states Bing "only supports Offer and Product microdata" in that (shopping/ads) context; this conflicts with the 2018 announcement and appears dated or scoped to Merchant Center crawling. — [Microsoft Advertising help](https://help.ads.microsoft.com/apex/index/3/en/56885); contradicted by [Bing Webmaster Blog, Aug 2018](https://blogs.bing.com/webmaster/august-2018/Introducing-JSON-LD-Support-in-Bing-Webmaster-Tools)
- Yandex announced schema.org support in November 2011. — [schema.org blog](https://blog.schema.org/2011/11/04/yandex-now-supports-schema-org-markup/)
- Yandex states JSON-LD information is used by Yandex Mail, for breadcrumbs in search snippets, and for Q&A in mobile results, and links to a JSON-LD validator. — [Yandex Webmaster JSON-LD](https://yandex.ru/support/webmaster/en/json-ld/about)
- Yandex Metrica documentation lists Microdata or JSON-LD as the two markup methods for schema.org; Yandex's structured data validator checks microformats, Schema.org and Open Graph (per search-result summaries of Yandex help pages, not fetched in full). — [Yandex Metrica schema.org help](https://www.yandex.com/support/metrica/en/publishers/schema-org/schema-org); [Yandex validator help](https://www.yandex.com/support/webmaster/en/yandex-indexing/validator.md)
- Baidu: no official documentation found; a Drupal.org comparison describes Baidu's JSON-LD status as uncertain. — [Drupal.org schema.org metatag docs](https://www.drupal.org/docs/7/modules/schemaorg-metatag/json-ld-microdata-or-rdfa)

### Inferences
- Emitting schema.org JSON-LD plus Open Graph covers Google, Bing and Yandex. Yandex still surfaces Q&A and breadcrumbs from JSON-LD, so FAQ/QAPage/BreadcrumbList markup retains value outside Google even after Google's 2026 FAQ removal.
- Bing's explicit Open Graph consumption makes OG tags dual-purpose (social previews and a secondary search signal).

### Gaps
- The current Bing Webmaster help page "Marking Up Your Site with Structured Data" returned no content (JS-rendered); the list of Bing-supported schema.org types as of 2026, and whether the standalone Markup Validator still exists versus being folded into URL Inspection, are unverified.
- No sourced statement found on how Bing/Copilot uses structured data for AI answers.
- Yandex's `supported-schemas` index page returned 404; the full list of Yandex-consumed types (products, recipes, organizations, video, etc.) was not obtained.
- Baidu: no reliable source. Treat as unknown.

## 5. Open Graph protocol: properties and image sizes per platform

### Takeaway
Four OG properties are formally required (`og:title`, `og:type`, `og:image`, `og:url`); in practice `og:description`, `og:site_name`, `og:locale` and `og:image:width/height/alt` should always be emitted. A single 1200x630 (1.91:1) image under about 5 MB satisfies Facebook and LinkedIn; chat-app specifics are only secondarily sourced.

### Cited Findings
- Required: `og:title`, `og:type`, `og:image`, `og:url`. Optional: `og:audio`, `og:description`, `og:determiner`, `og:locale`, `og:locale:alternate`, `og:site_name`, `og:video`. — [ogp.me](https://ogp.me/)
- Image structured properties: `og:image:url`, `og:image:secure_url`, `og:image:type`, `og:image:width`, `og:image:height`, `og:image:alt`. Video: `og:video:secure_url`, `:type`, `:width`, `:height`. Audio: `og:audio:secure_url`, `:type`. — [ogp.me](https://ogp.me/)
- `og:type` values: `website`, `article`, `book`, `profile`, `music.song`, `music.album`, `music.playlist`, `music.radio_station`, `video.movie`, `video.episode`, `video.tv_show`, `video.other`. — [ogp.me](https://ogp.me/)
- `article:*`: `article:published_time`, `article:modified_time`, `article:expiration_time` (datetime), `article:author` (profile array), `article:section` (string), `article:tag` (string array). `profile:*`: `first_name`, `last_name`, `username`, `gender`. `book:*`: `author`, `isbn`, `release_date`, `tag`. — [ogp.me](https://ogp.me/)
- Facebook: minimum image 200x200 px; max file size 8 MB; "Use images that are at least 1200 x 630 pixels" for high-resolution displays; at least 600x315 px for a large-image link post; keep "as close to 1.91:1 aspect ratio as possible". Providing `og:image:width` and `og:image:height` lets the crawler render the image on first share without pre-fetching. — [Meta sharing docs: images](https://developers.facebook.com/docs/sharing/webmasters/images/)
- LinkedIn: requires `og:title`, `og:image`, `og:description`, `og:url`; image minimum 1200 (w) x 627 (h) px, recommended ratio 1.91:1, max 5 MB. — [LinkedIn Help](https://www.linkedin.com/help/linkedin/answer/a521928)
- Google uses `og:site_name` as one signal for the site name. — [Google site names docs](https://developers.google.com/search/docs/appearance/site-names)
- Rank Math emits `product:price:amount` and `product:price:currency` OG product tags for WooCommerce. — [Rank Math filters and hooks](https://rankmath.com/kb/filters-hooks-api-developer/)
- Secondary sources (not platform docs) agree 1200x630 is the common recommendation for Slack, Discord, WhatsApp and iMessage, but conflict on WhatsApp's file-size limit (300 KB vs 600 KB) and on minimum sizes for Discord. — [DEV Community OG sizes guide](https://dev.to/corbanware/og-image-sizes-the-complete-guide-for-2026-4l6d); [ogpixel guide](https://ogpixel.pages.dev/guide/og-image-size-guide)

### Inferences
- Default CMS output per singular page: `og:title`, `og:type` (`article` for posts, `website` for home/archives, `profile` for author pages), `og:url` (canonical), `og:description`, `og:site_name`, `og:locale`, `og:image` + `og:image:width` + `og:image:height` + `og:image:alt` + `og:image:type`; for posts add `article:published_time`, `article:modified_time`, `article:author`, `article:section`, `article:tag`.
- A dedicated registered image size of 1200x630 covers Facebook and LinkedIn (LinkedIn's 627 height is within tolerance of the 1.91:1 ratio); keeping the file small (ideally a few hundred KB) hedges against unverified chat-app limits.
- OG uses the `property` attribute (RDFa-style) while Twitter and `fediverse:creator` use `name`; a meta-tag API needs to support both attributes.

### Gaps
- No primary documentation obtained for Slack, Discord, WhatsApp or iMessage unfurl requirements; the WhatsApp size limit and Discord `theme-color` behaviour are unverified.
- Threads link-preview behaviour: no source found (presumably Meta's OG crawler; unverified).
- ogp.me lists the profile reference for `article:author` as a profile URL array; Facebook's historical use of `article:publisher` is not in ogp.me and was not sourced.

## 6. X/Twitter Cards and equivalents (Mastodon, Bluesky, Pinterest, Threads)

### Takeaway
X Cards still use the legacy `twitter:*` meta names with four card types (summary, summary_large_image, app, player) and fall back to Open Graph, but X's official markup docs were paywalled/unreachable (HTTP 402) and X has removed much card documentation, so current behaviour rests on secondary sources. Mastodon adds `fediverse:creator`; Pinterest Rich Pins reuse Open Graph and schema.org.

### Cited Findings
- developer.x.com card markup pages returned HTTP 402 Payment Required when fetched on 2026-11-17. — [X developer cards markup](https://developer.x.com/cards/markup)
- Search snippets of X's docs: card type "will be one of 'summary', 'summary_large_image', 'app', or 'player'"; `twitter:title` limit 70 chars; `twitter:description` 200 chars; each tag names its Open Graph fallback. — [X developer cards markup](https://developer.x.com/cards/markup) (via search snippet only)
- The SEO Framework KB (published 17 May 2026, revised 13 Oct 2026): X "removed much of its Twitter Card documentation"; the `twitter:` names remain in use; tags used are `twitter:card`, `twitter:site`, `twitter:creator`, `twitter:title`, `twitter:description`, `twitter:image`, `twitter:image:alt`; `twitter:site`/`twitter:creator` may no longer change the visible preview; OG fallback for title/description/image is described as historical behaviour. — [The SEO Framework KB](https://kb.theseoframework.com/kb/twitter-cards-and-x-sharing/)
- Image limits as X documented them: summary card square, min 144x144; large image card near 2:1, min 300x157; max 4096x4096 and under 5 MB; JPG, PNG, WEBP, GIF (first frame only), no SVG; `twitter:image:alt` up to 420 chars; card content cached about seven days. — [The SEO Framework KB](https://kb.theseoframework.com/kb/twitter-cards-and-x-sharing/)
- Mastodon: `<meta name="fediverse:creator" content="@Gargron@mastodon.social" />` makes the author byline on link previews link to the fediverse account; handle may be on any fediverse platform; only the first tag is used; the author must list the site's domain under Edit Profile > Verification > Author Attribution. Announced July 2024. — [Mastodon blog, Jul 2024](https://blog.joinmastodon.org/2024/07/highlighting-journalism-on-mastodon/)
- Pinterest Rich Pins: types are Article, Product and Recipe; "the most common formats are Open Graph and Schema.org"; priority when multiple are present is Product, then Recipe, then Article. — [Pinterest developer docs](https://developers.pinterest.com/docs/web-features/rich-pins-overview/)

### Inferences
- Minimal robust output: `<meta name="twitter:card" content="summary_large_image">` plus complete OG tags; add `twitter:image:alt` because it has no OG fallback according to secondary guides; `twitter:site`/`twitter:creator` are optional and low-value in 2026.
- `fediverse:creator` should be a per-author field (fediverse handle in the user profile) emitted on singular posts; multiple authors cannot be represented (first tag wins).
- Pinterest needs nothing beyond OG `article` tags and Product/Recipe schema that the graph already emits.

### Gaps
- X's official current card spec (required tags, whether `twitter:site` is still required, whether app/player cards still function, validator status) could not be read from a primary source.
- Bluesky: docs page redirected to bsky.network and returned empty content. How link cards (`app.bsky.embed.external`) are built from OG tags, and image size limits, are unverified from primary sources.
- Threads: no source found.
- The Mastodon version number that shipped `fediverse:creator` was not stated in the announcement.

## 7. oEmbed discovery links and their role

### Takeaway
oEmbed is a provider-side embed API, discovered via `<link rel="alternate">` tags (JSON and XML) or an HTTP Link header; it governs rich embeds in consumers that support it and is separate from, and complementary to, Open Graph link previews.

### Cited Findings
- Discovery: `<link rel="alternate" type="application/json+oembed" href="...">` and/or `type="text/xml+oembed"` in `<head>`, or an HTTP `Link` header with `rel="alternate"` and the same types; `href` should contain the full endpoint URL including the target `url` and format. — [oembed.com](https://oembed.com/)
- Response must include `type` (`photo`, `video`, `link`, `rich`) and `version` (`"1.0"`); optional `title`, `author_name`, `author_url`, `provider_name`, `provider_url`, `cache_age`, `thumbnail_url` + `thumbnail_width` + `thumbnail_height` (all three together). — [oembed.com](https://oembed.com/)
- Type requirements: photo = `url`, `width`, `height`; video and rich = `html`, `width`, `height`; link = none. Providers must respect `maxwidth`/`maxheight`. — [oembed.com](https://oembed.com/)
- Security: consumers should restrict URL schemes to http/https/mailto and render provider HTML in a cross-domain iframe. — [oembed.com](https://oembed.com/)

### Inferences
- WordPress core already emits oEmbed discovery links and serves a `rich` response; a unified head-metadata layer should treat the oEmbed discovery links as part of the same "preview metadata" output group as OG/Twitter tags so they stay consistent (same title, author, thumbnail).
- oEmbed `author_name`/`author_url`/`thumbnail_url` duplicate OG data; they should be derived from the same source values.

### Gaps
- Which major platforms in 2026 prefer oEmbed over OG for unfurls (e.g. Slack, Discord, Pinterest) was not verified from primary sources.

## 8. Validation tools

### Takeaway
Google's Rich Results Test (plus Search Console URL Inspection and rich result reports) checks Google eligibility; the Schema Markup Validator checks generic schema.org validity (and is what Google points to for site-name markup); Bing and Yandex have their own validators; X's validator status is unknown.

### Cited Findings
- Google: "The Rich Results Test is an easy and useful tool for validating your structured data"; use it during development and monitor with Search Console rich result status reports; URL Inspection confirms crawlability. — [Google intro to structured data](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data); [Google structured data policies](https://developers.google.com/search/docs/appearance/structured-data/sd-policies)
- Rich Results Test does not support site names; Google says to validate WebSite markup with a schema testing tool. — [Google site names docs](https://developers.google.com/search/docs/appearance/site-names)
- Rich Results Test and Search Console drop support for types as they are deprecated (HowTo 30 days after 14 Sep 2023; practice problems January 2026). — [Search Central blog, Aug 2023](https://developers.google.com/search/blog/2023/08/howto-faq-changes); [Search Central updates log](https://developers.google.com/search/updates)
- Schema Markup Validator (validator.schema.org) is the successor to Google's Structured Data Testing Tool (deprecated August 2020) and validates against schema.org generally, not Google eligibility (secondary sources). — [Impression Digital](https://www.impressiondigital.com/blog/structured-data-testing-vs-rich-results-test-how-do-they-compare/); [Huckabuy](https://huckabuy.com/structured-data-testing-tool-alternatives/)
- Bing Webmaster Tools Markup Validator (Diagnostics and Tools) validates JSON-LD, Microdata, RDFa, Microformats, Open Graph, schema.org (2018 statement). — [Bing Webmaster Blog, Aug 2018](https://blogs.bing.com/webmaster/august-2018/Introducing-JSON-LD-Support-in-Bing-Webmaster-Tools)
- Yandex Webmaster provides a structured data validator covering microformats, schema.org and Open Graph, and a JSON-LD validator. — [Yandex validator help](https://www.yandex.com/support/webmaster/en/yandex-indexing/validator.md); [Yandex Webmaster JSON-LD](https://yandex.ru/support/webmaster/en/json-ld/about)
- A 2026 third-party X Cards guide mentions no working X card validator and relies on source inspection. — [The SEO Framework KB](https://kb.theseoframework.com/kb/twitter-cards-and-x-sharing/)

### Inferences
- CI-style validation for a core implementation should target schema.org validity (Schema Markup Validator semantics) plus Google's per-type required-property tables encoded as unit tests, since Rich Results Test coverage shrinks with each deprecation.

### Gaps
- Not verified from primary sources: Meta Sharing Debugger, LinkedIn Post Inspector, Pinterest Rich Pins validator current status, and whether Bing's standalone Markup Validator still exists in 2026.
