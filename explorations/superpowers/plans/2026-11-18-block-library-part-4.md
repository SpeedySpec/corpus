# Block library part 4: actions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the actions area of the block library: buttons, call to action, testimonial, pricing table and plan, rating, and social links. It comes as a definitions package and a default outputs package with its own stylesheet. The reference editor shows that stylesheet on the canvas and gains a control for editing a list of text, such as a plan's features.

**Architecture:**
- `block-actions-domain` registers the nine block types and the behaviours that need logic: a social link checked against its service, a testimonial's optional photo, and a rating kept in range.
- `block-actions` registers the outputs and one stylesheet asset that the outputs declare through `DeclaresAssetsInterface`.
- The builder API's render and preview answers gain the resolved style assets, so the reference editor's canvas looks like the page.

**Tech Stack:** PHP 8.4, Pest, Composer path repositories; plain ES modules and `node --test`.

**Spec:** `docs/superpowers/specs/2026-11-18-block-library-design.md` (in `wordpress-develop`): "The blocks" (Actions), "Front-end assets", "Left to the builder" (the social services list), and the "Part 3" note.

**Repository:** `wordpress-develop/packages/speedyspec-api-block`. Cut branch `feat/block-library-part-4` from `origin/main` once PR #11 (part 3) is merged, otherwise from `feat/block-library-part-3`. Paths are relative to its `packages/` directory.

## Global Constraints

- **Earlier constraints still hold.** Everything in parts 1 to 3's Global Constraints applies:
  - `declare(strict_types=1)`; 4 spaces;
  - braces of classes, methods and control structures on their own line;
  - docblocks per `wordpress-develop/.claude/rules/coding-style.md`; lines at most 120 characters;
  - one commit per task with explicit paths, ending `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Dependencies.**
  - `block-actions-domain` requires only `speedyspec/block-domain`.
  - `block-actions` requires `speedyspec/block-domain` and `speedyspec/block-actions-domain`.
  - Neither depends on `block-app`, `block-media` or WordPress.
- **Block names and settings.**
  - The blocks are `core/button`, `core/buttons`, `core/call-to-action`, `core/testimonial`, `core/pricing-table`, `core/pricing-plan`, `core/rating`, `core/social-links` and `core/social-link`, with category `actions`. The package title is "Actions".
  - Their settings are fixed by this plan.
  - Every block has a title, a description and an example that resolves without issues.
- **Links.**
  - A link is printed only through the attribute bag, so `escaper->url()` judges it.
  - A link that opens in a new tab always carries `rel="noopener noreferrer"`.
  - A button or social link with no usable link is "not set up", as part 3 ruled for media: visitors get nothing, and editors get the element marked `aria-disabled="true"` with a note.
- **Social logos.** Social links are drawn as their service's name, not its logo. *My call:* the logos are other companies' trademarks, and a plugin that wants them ships its own outputs package.
- **Styles.**
  - The outputs' appearance comes from one stylesheet asset named `core/actions`, written with the design-token custom properties and fallbacks, such as `var(--color-primary, #2456d6)`.
  - The shared `core/style` settings still go on each block's outer element through the attribute bag.

## Review Focus

1. **A link that is not safe or not real.** Examples: `javascript:alert(1)`, ` JaVaScRiPt:x`, `data:text/html,…`, `//evil.test`, `/\evil.test`, and an empty link. Expected: never an `href` for visitors. The button or social link is not set up, so visitors get nothing and editors see the disabled element with its note. Tasks 2 and 3.
2. **A social link to another site.**
   - Examples: service `instagram` with `https://evil.test/instagram`, `https://instagram.com.evil.test/x`, or `http://instagram.com/x`.
   - Expected: unavailable with an issue naming the service.
   - `website` takes any https link, and `email` takes only `mailto:`.
   - Task 2.
3. **A rating outside its scale.** Examples: −1, 7 out of 5, 4.3 or 4.75, and an `outOf` of 0. Expected: kept within 0 and the scale (whose own range the schema holds) and rounded to the nearest half. The page always says "Rated 4.5 out of 5" in words, and the stars are decoration. Tasks 2 and 3.
4. **A pricing plan's features.**
   - Examples: an empty list, empty strings, 31 items, markup in a feature, and typing into `features.3` of a new plan.
   - Expected: empty items are skipped; the schema holds the length; markup is shown as text.
   - Typing on the page changes only that item, starting from the default features when none are stored, as part 2's table cells do.
   - Tasks 2, 3 and 5.
5. **The canvas stylesheets.** Examples: a style asset whose inline content contains `</style><script>`, an asset with a URL, and an asset the host's locator cannot place. Expected: inline content cannot leave its `<style>` element; a URL becomes a `<link>` only through the same URL check; an asset with no URL is left out, not drawn with an empty `href`. Scripts are never sent to the canvas. Task 1.

---

### Task 1: Style assets on the editor's canvas

**Files:**
- Modify: `block-presentation/src/BuilderApi.php` (the `render` and `preview` answers), `block-builder/src/view.js`, `block-builder/src/builder-element.js` (the canvas `srcdoc`), `block-builder/src/store.js` (keep `styles` with the preview)
- Test: `block-presentation/tests/StylesTest.php`, `block-builder/tests/view.test.js`

**Interfaces:**
- Consumes: part 1's `AssetRegistryInterface::resolve()`, `AssetLocatorInterface::url()`, `Asset`, `AssetKind`.
- Produces:
  - The answers of `GET /documents/{id}/render` and `POST /documents/{id}/preview` gain `styles`: `list<{name: string, href?: string, content?: string}>`.
    - These are the style assets among the result's assets, resolved in dependency order.
    - An inline asset gives `content`.
    - A URL asset gives `href`.
    - A file asset gives `href` from `AssetLocatorInterface` when one is bound and it answers a non-empty URL; otherwise the asset is left out.
    - Script and module assets are never included.
  - `view.js`: `canvasDocument(preview: {output, css, styles}): string`, the whole `srcdoc` the canvas uses today plus the stylesheets.
    - Each `href` passes `safeLink()` (from `inline.js`) or is dropped.
    - Each `content` has `</` written as `<\/`, as the collected CSS already is.
    - Linked stylesheets come before inline ones, and both before the collected CSS, so a block's own rules win.

- [ ] **Step 1: Write the failing tests.**
  - `StylesTest.php`: an outputs stub that declares an inline style `acme/look` depending on a URL style `acme/base`, and a script `acme/run`, gives `styles` `[{name: 'acme/base', href: …}, {name: 'acme/look', content: …}]` with no script. A file asset with no locator bound is left out; with a locator answering `''` it is left out; with a locator answering `/assets/x.css` it is included.
  - `view.test.js`: `canvasDocument` keeps the order; turns `</style><script>` in content into `<\/style><script>`; drops an `href` of `javascript:x` and of `//evil.test`; keeps `/assets/x.css` and `https://cdn.test/x.css`.
- [ ] **Step 2: Run** `cd block-presentation && vendor/bin/pest --filter=Styles` and `cd block-builder && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement.** The canvas builds its `srcdoc` with `canvasDocument(state.preview)`. The store keeps `styles` beside `output` and `css`.
- [ ] **Step 4: Run** the `block-presentation` suite and `npm test`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-builder): show the page's stylesheets on the editor's canvas`.

---

### Task 2: Actions block definitions

**Files:**
- Create: `block-actions-domain/`, containing `composer.json`, `phpunit.xml` and:
  - in `src/`: `ActionsTypesPackage.php`, `SocialServices.php`, `SocialLinkBehaviour.php`, `TestimonialBehaviour.php`, `RatingBehaviour.php`;
  - in `tests/`: `Pest.php`, `PackageContractTest.php`, `TypesTest.php`, `BehavioursTest.php`.

**Interfaces:**
- Consumes: part 3's `MediaLibraryInterface`, `MediaKind`, `UnavailableBlockException`, `IssueKind::UnavailableMedia`; `block-infra-memory`'s `InMemoryMediaLibrary` as a dev dependency.
- Produces:
  - **The package.** `ActionsTypesPackage(?MediaLibraryInterface $library = null)` and `static from(BlockServiceContainer $container): self`, which passes the library only when one is bound. Name `core/actions-types`, title "Actions". Settings:
    - **`core/button`:**
      - `text`: string, at most 80 characters, default "Learn more";
      - `link`: string, at most 2000 characters, default `""`;
      - `newTab`: boolean, default false;
      - `variant`: enum `primary`, `secondary`, `outline` or `plain`, default `primary`;
      - `size`: enum `small`, `medium` or `large`, default `medium`.
    - **`core/buttons`:** `align` (enum `start`, `center`, `end` or `between`, default `start`); children `core/button` only.
    - **`core/call-to-action`:**
      - `heading`: string, at most 200 characters, default "Ready to start?";
      - `level`: integer 2 to 4, default 2;
      - `text`: rich text;
      - children `core/buttons` and `core/button`.
    - **`core/testimonial`:**
      - `quote`: rich text;
      - `name` and `role`: strings, at most 120 characters each;
      - `media`: a media setting, kind image.
    - **`core/pricing-table`:** children `core/pricing-plan` only; `columns` (integer 1 to 4 or a responsive map, default 3, `x-control: responsive`).
    - **`core/pricing-plan`:**
      - `name`: string, at most 80 characters, default "Plan";
      - `price`: string, at most 20 characters, default "$0";
      - `period`: string, at most 40 characters, default "per month";
      - `description`: rich text;
      - `features`: a list of strings, each at most 200 characters, `maxItems` 30, default `["First feature", "Second feature"]`, `x-control: list`;
      - `highlighted`: boolean, default false;
      - `badge`: string, at most 40 characters, default `""`;
      - parents `core/pricing-table`; children `core/button` and `core/buttons`.
    - **`core/rating`:**
      - `value`: number, minimum 0, maximum 10, default 5;
      - `outOf`: integer 1 to 10, default 5;
      - `label`: string, at most 120 characters, default `""`.
    - **`core/social-links`:** `size` (enum `small`, `medium` or `large`, default `medium`); children `core/social-link` only.
    - **`core/social-link`:**
      - `service`: enum of `SocialServices::NAMES`, default `website`;
      - `url`: string, at most 2000 characters, default `""`;
      - parents `core/social-links`.
  - **`SocialServices`** has `NAMES`, `title(string $service): string` and `accepts(string $service, string $url): bool`. The list is *my call*:

    | Service | Title | Links accepted |
    |---|---|---|
    | `website` | Website | any https link |
    | `email` | Email | `mailto:` only |
    | `rss` | RSS feed | any https link |
    | `mastodon` | Mastodon | any https link, since instances vary |
    | `bluesky` | Bluesky | `bsky.app` |
    | `facebook` | Facebook | `facebook.com`, `www.facebook.com` |
    | `instagram` | Instagram | `instagram.com`, `www.instagram.com` |
    | `linkedin` | LinkedIn | `linkedin.com`, `www.linkedin.com` |
    | `x` | X | `x.com`, `twitter.com` |
    | `youtube` | YouTube | `youtube.com`, `www.youtube.com` |
    | `tiktok` | TikTok | `tiktok.com`, `www.tiktok.com` |
    | `github` | GitHub | `github.com` |
    | `threads` | Threads | `threads.net`, `www.threads.net` |
    | `pinterest` | Pinterest | `pinterest.com`, `www.pinterest.com` |

    Hosts are compared exactly after `parse_url`. A user, a password or a port is refused, and the scheme must be https except for `email`.
  - **Views.** These are what the outputs read:
    - **social link:** `{service, title, url}`. The `url` is empty when none is stored, so the block is not set up.
    - **testimonial:** `{quote, name, role, photo: {url, alt, width, height} | null}`. The photo is dropped when it is missing, of another kind, or when no library is bound.
    - **rating:** `{value, outOf, label, text}`.
      - `value` is kept within 0 and `outOf`, then rounded to the nearest half.
      - `text` is "Rated {value} out of {outOf}", with the value written without a trailing `.0`.
      - `label`, when given, goes before it, as in "Service: rated 4.5 out of 5".
    - Every other block has no behaviour; its view is its settings.
  - **Unavailable.** A social link whose stored `url` its service does not accept is unavailable: `UnavailableBlockException(IssueKind::UnavailableMedia, "This link is not to {title}.", 'url')`. For `email` the message is "Use a mailto: link for email.". *My call:* the issue kind is reused rather than adding `invalid-link`, since both mean "a chosen thing cannot be shown". Changing that later is a one-line migration of the enum value.

- [ ] **Step 1: Write the failing tests.**
  - `PackageContractTest` runs in definitions mode, once with a library and once without.
  - `TypesTest` checks:
    - the names, category, parents and children, and defaults;
    - the schema refuses 31 features, a feature of 201 characters, a `variant` of `giant`, `outOf` 0 and 11, and `value` −1;
    - every block has a title, description and example.
  - `BehavioursTest` checks:
    - each Review Focus 2 link is unavailable with its message;
    - `website` accepts `https://e.test` and refuses `http://e.test`;
    - `email` accepts `mailto:a@e.test` and refuses `https://e.test`;
    - a social link with no url resolves with an empty `url` and no issue;
    - the testimonial photo is present for an image id, and null for a missing id, a PDF id and no library;
    - the rating clamps 7 of 5 to 5, −1 to 0, 4.3 to 4.5 and 4.2 to 4, and gives "Rated 4.5 out of 5" and "Service: rated 4 out of 5".
- [ ] **Step 2: Run** `cd block-actions-domain && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement** the package, `SocialServices` and the three behaviours.
- [ ] **Step 4: Run** the `block-actions-domain` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-actions): add the actions block definitions`.

---

### Task 3: Actions block outputs and their stylesheet

**Files:**
- Create: `block-actions/`, containing `composer.json`, `phpunit.xml` and:
  - in `src/`: `ActionsPackage.php`, `ActionsOutput.php`, `ActionsStylesheet.php`;
  - in `tests/`: `Pest.php`, `PackageContractTest.php`, `OutputTest.php`.

**Interfaces:**
- Consumes: Task 2's views; Task 1 is not needed to render, only to see the styles in the editor.
- Produces:
  - **The package.** `ActionsPackage(StyleCollectorInterface $styles)` and `static from(BlockServiceContainer $container): self`. Name `core/actions`, title "Actions, drawn".
    - It registers `Asset::inline('core/actions', AssetKind::Style, ActionsStylesheet::CSS)` and `ActionsOutput` for the nine blocks.
    - `ActionsOutput` implements `DeclaresAssetsInterface` and answers `['core/actions']`.
  - **Markup.** The attribute bag always goes on the outermost element.
    - **Button:** `<a{attrs} class="b-button b-button--{variant} b-button--{size}" href>`, holding the text. When `newTab` is set, the link adds `target="_blank" rel="noopener noreferrer"`.
      - A button not set up draws nothing for visitors.
      - In admin mode it draws `<span{attrs} class="b-button …" aria-disabled="true">` with the text, then `<span class="b-note">Add a link in the block's settings.</span>`.
      - The text is marked `text` while editing.
    - **Buttons:** `<div{attrs} class="b-buttons b-buttons--{align}">`, holding the children.
    - **Call to action:** `<section{attrs} class="b-cta">`, holding `<h{level}>` (marked `heading`), then `<div class="b-cta-text">` (marked `text`), then the children.
    - **Testimonial:** `<figure{attrs} class="b-testimonial">`, holding:
      - `<blockquote>` (marked `quote`), with the quote as rich text;
      - `<figcaption>` with the photo `<img … alt="" loading="lazy">` (decorative: the name is beside it), then `<span class="b-testimonial-name">` (marked `name`) and `<span class="b-testimonial-role">` (marked `role`).
      - An empty role is left out for visitors.
    - **Pricing table:** `<div{attrs} class="b-pricing">`, holding the children. Its column count by breakpoint goes to the style collector as a grid, as part 3's gallery does.
    - **Pricing plan:** `<article{attrs} class="b-plan">`, with `b-plan--highlighted` when highlighted. It holds:
      - the badge `<p class="b-plan-badge">`, when there is one;
      - `<h3>` with the name (marked `name`);
      - `<p class="b-plan-price"><span>` with the price (marked `price`) and `<span>` with the period (marked `period`);
      - the description (marked `description`);
      - `<ul class="b-plan-features">`, each `<li>` marked `features.{n}`;
      - the children.
      - Empty features are skipped for visitors and kept while editing, so they can be typed into.
    - **Rating:** `<p{attrs} class="b-rating">`, holding `<span class="b-rating-stars" aria-hidden="true">` with `outOf` star SVGs (full, half or empty, drawn in this package), then `<span class="b-rating-text">` with the text. The text is visually small but never hidden, so the rating does not depend on seeing the stars.
    - **Social links:** `<ul{attrs} class="b-social b-social--{size}">`, holding the children.
    - **Social link:** `<li>` holding `<a href class="b-social-link" rel="me noopener">`, with the title as text. Not set up: nothing for visitors, and in admin mode the disabled span with "Add a link in the block's settings.".
  - **`ActionsStylesheet::CSS`.** The rules for the classes above, using the token custom properties `--color-primary` and `--space-md` with fallbacks. Each button variant has `:hover` and `:focus-visible` states; the focus outline is never removed. It respects `prefers-reduced-motion` by having no transitions in that case. There are no `@import` rules and no `url()` values, so the stylesheet loads nothing else.

- [ ] **Step 1: Write the failing tests.**
  - `PackageContractTest` runs in outputs mode, with `ActionsTypesPackage` and an `InMemoryMediaLibrary`.
  - `OutputTest` covers each block's markup and each Review Focus 1 link (no `href`, nothing for visitors, the disabled span and note in admin mode). It also checks that:
    - a new-tab link has `rel="noopener noreferrer"`;
    - the rating text is in words and the stars are `aria-hidden`;
    - empty features are skipped for visitors and marked while editing;
    - a feature holding `<b>x</b>` is shown as text;
    - the shared style settings land on each outer element;
    - every output declares `core/actions`, and rendering a page lists `core/actions` once in its assets;
    - the stylesheet has no `@import` and no `url(`.
- [ ] **Step 2: Run** `cd block-actions && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the `block-actions` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-actions): add the actions block outputs and their stylesheet`.

---

### Task 4: Editing a list of text in the inspector

**Files:**
- Modify: `block-builder/src/forms.js`, `block-builder/src/index.js`, `block-builder/src/builder-element.js`, `block-builder/src/styles.js`
- Test: `block-builder/tests/forms.test.js`

**Interfaces:**
- Consumes: part 2's `withTyped` (typing into `features.{n}` on the page already works through it).
- Produces:
  - **`forms.js` helpers:** `withListItem(list, at, text = '')`, `withoutListItem(list, at)`, `movedListItem(list, from, to)` and `withListText(list, at, text)`.
    - Each copies the list and never changes the one it was given.
    - A move past either end leaves the order as it was.
    - A list that is not an array is treated as empty.
  - **Inspector:** `x-control: list` draws one text input per item, each with Earlier, Later and Remove, and an Add item button at the end.
    - Each input is keyed `settings:{name}:{index}`, so focus and the cursor survive a redraw.
    - A change sends `changeSettings` with the item replaced, as part 3's picker does.
    - The list starts from the setting's default when nothing is stored.

- [ ] **Step 1: Write the failing tests** for the four helpers. They cover:
  - adding at the start, the middle and the end;
  - removing the only item, which leaves an empty list;
  - moving past either end;
  - replacing one item's text;
  - non-array input;
  - the input list never changing.
- [ ] **Step 2: Run** `npm test`. Expected: FAIL.
- [ ] **Step 3: Implement** the helpers and the control.
- [ ] **Step 4: Run** `npm test`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-builder): edit lists of text in the inspector`.

---

### Task 5: The demo and documentation

**Files:**
- Modify: `block-builder/demo/server.php`, `block-builder/demo/composer.json`, repository `README.md`, `block-builder/README.md`, `wordpress-develop/docs/superpowers/specs/2026-11-18-block-library-design.md` (a part 4 note)

**Interfaces:**
- Consumes: Tasks 1 to 4.
- Produces:
  - **Demo packages.** The demo loads `ActionsTypesPackage::from($container)` and `ActionsPackage::from($container)` after the media packages.
  - **Demo page.** It seeds the page with:
    - a call to action with two buttons, one primary and one outline opening in a new tab;
    - a testimonial with the first sample picture;
    - a pricing table of three plans, the middle one highlighted with a "Most popular" badge;
    - a rating of 4.5;
    - social links for website, email, GitHub and Mastodon.
  - **README sections.** "Actions" under the block areas, and in the builder README, the canvas stylesheets and the list control.

- [ ] **Step 1: Update** the demo, READMEs and design note.
- [ ] **Step 2: Run** every PHP suite and `npm test`. Expected: PASS.
- [ ] **Step 3: Check by hand.** Start the demo (`.claude/launch.json`, `block-builder-demo`) after deleting the demo's `documents/` so the page reseeds, and check that:
  - the buttons are styled on the canvas, and hover and focus are visible;
  - typing into a button, a plan's name and a feature changes only that setting;
  - a feature can be added, moved and removed in the inspector;
  - a button's link set to `javascript:alert(1)` is not set up: a disabled button and note for editors, nothing in `?mode=front`;
  - a social link to `https://evil.test` under GitHub shows its issue.
- [ ] **Step 4: Commit** `feat(block-actions): show the actions blocks in the demo`.

---

## Out of scope for this plan

- Brand logos for social links (a plugin's own outputs package).
- Choosing a button's link from the site's pages.
- Per-plan currency formatting (the price is text the person writes).
- Star ratings people can submit.
- The interactive and site areas (parts 5 and 6), and the remaining editor features (part 7).
