# Block library part 6: site blocks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the site area of the block library as a definitions package and a default outputs package.
- **Blocks:** navigation with links and submenus, a table of contents built from the page's headings, breadcrumbs from the host, and a search form posting to the host.
- **Post list:** a template, not a block type.
- **Heading anchors:** unique across a page, so the table of contents always links to the right heading.

**Architecture:**
- **One new contract** in `block-domain`, `SiteMapInterface`, says where a page sits in the host's site: its address, its breadcrumb trail, and where search goes. Memory and WordPress implementations are included.
- **`block-site-domain`** registers the block types, one `DocumentStepInterface` step and the `core/post-list` template.
  - The step sees the whole page before any block resolves and adds to the root context:
    - the page's headings and a unique id for each;
    - the page's address, trail and search target.
  - The template is a `core/repeat` over a collection the host registers, with editable children bound to the current item.
- **`block-site`** draws the blocks, with one inline stylesheet and no script. Submenus open on hover and keyboard focus with CSS alone.
- **Shared heading anchors.** The heading-anchor rules move into `block-domain` (`HeadingAnchor`). The text package's heading and the site package's step then share one algorithm, and the heading reads its unique id from the context when the site package is loaded.

**Tech Stack:** PHP 8.4, Pest, Composer path repositories; plain ES modules and `node --test`.

**Spec:** `docs/superpowers/specs/2026-11-18-block-library-design.md` (in `wordpress-develop`). The relevant sections:
- "The blocks" (Site);
- "Headings and the table of contents";
- the Part 2 note (repeated anchors are a known limit until the table of contents);
- the Part 3 to 5 notes (the "not set up" rule).

**Repository:** `wordpress-develop/packages/speedyspec-api-block`. Cut branch `feat/block-library-part-6` from `origin/main` once PR #13 (part 5) is merged; otherwise cut it from `feat/block-library-part-5`. Paths are relative to the repository's `packages/` directory.

## Global Constraints

- **Earlier constraints still hold.** Everything in parts 1 to 5's Global Constraints applies:
  - `declare(strict_types=1)`; 4 spaces;
  - braces of classes, methods and control structures on their own line;
  - docblocks per `wordpress-develop/.claude/rules/coding-style.md`; PHP lines at most 120 characters;
  - one commit per task with explicit paths, ending `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Dependencies.**
  - `block-site-domain` requires only `speedyspec/block-domain`.
  - `block-site` requires `speedyspec/block-domain` and `speedyspec/block-site-domain`.
  - Neither depends on `block-app`, another area's packages, or WordPress.
  - They name other areas' block types only as strings: `core/heading` in the step, and `core/repeat`, `core/heading`, `core/paragraph` and `core/button` in the template.
  - Only `block-infra-wp` touches WordPress, and only through its Ports.
- **Block names and settings.**
  - The blocks are `core/navigation`, `core/navigation-link`, `core/table-of-contents`, `core/breadcrumbs` and `core/search`, with category `site`. The package title is "Site".
  - The template is `core/post-list`.
  - Their settings are fixed by this plan.
  - Every block has a title, a description and an example that resolves without issues.
- **Context keys.** These are fixed by this plan, and the step owns them:
  - `core/headings`: a list of `{level, text, id}`, in page order;
  - `core/heading-ids`: a map from block id to id;
  - `core/page-url`: a string, empty when the host gives none;
  - `core/breadcrumbs`: a list of `{label, url}`;
  - `core/search`: `{url, parameter}` or null.
  - `core/nav-depth` is the exception: each navigation link provides it for its submenu (Task 3), so the step does not set it.
- **Links.**
  - A link is printed only through the attribute bag, so the escaper judges it.
  - A navigation link or crumb whose address the escaper refuses is drawn as plain text, never as a link.
- **No script.** Submenus open on hover and on keyboard focus within (`:focus-within`), and every link stays reachable by Tab with nothing hidden from it.
- **Not set up.** As in parts 3 to 5:
  - a search with no host search target, breadcrumbs with no trail, and a table of contents with no headings draw nothing for visitors and a prompt for editors;
  - a navigation link with no label or address is not set up in the same way.

## Review Focus

1. **Heading anchors on a busy page.**
   - Cases:
     - two headings with the same text;
     - a typed anchor equal to another heading's derived id;
     - a heading with no text;
     - markup in a heading;
     - headings in Greek;
     - a heading inside a `core/repeat`;
     - a heading in an included document.
   - Expected:
     - every id on the page is unique (`intro`, `intro-2`), and a typed anchor wins over a derived one;
     - the table of contents links to exactly those ids, as plain text;
     - repeated and included headings are left out of the table of contents;
     - without the site package, headings behave as before.
   - Tasks 1 and 3.
2. **Navigation addresses.**
   - Cases:
     - `javascript:alert(1)` as a link;
     - an external https link;
     - the current page matched against `/about`, `/about/`, `/about?x=1` and `/about#team`;
     - submenus three levels deep;
     - a keyboard user with no script.
   - Expected:
     - an unsafe address is drawn as text;
     - the current page is marked `aria-current="page"` when its path matches, ignoring a trailing slash, the query and the fragment;
     - submenus open on focus, and every link is reachable by Tab;
     - a link nested deeper than two submenu levels is unavailable, with an issue.
   - Tasks 3 and 4.
3. **Breadcrumbs from the host.**
   - Cases: an empty trail; 40 crumbs; a label with markup; a crumb whose address is unsafe; a last crumb that has an address.
   - Expected:
     - an empty trail is not set up;
     - labels are text;
     - an unsafe address becomes plain text;
     - the last crumb is always the current page (`aria-current="page"`), not a link;
     - the trail is capped at 20 crumbs, keeping the first and the last.
   - Tasks 2 and 4.
4. **The search form.**
   - Cases:
     - a host with no search;
     - a search URL of `javascript:x`;
     - a parameter name of `"><script>` or `q`;
     - an empty label;
     - two searches on one page.
   - Expected:
     - no search target is not set up, and an unsafe URL is not set up either;
     - the parameter must match `^[A-Za-z][A-Za-z0-9_-]{0,31}$`, or the block is not set up;
     - the input always has a label, falling back to "Search" when it is empty, and ids never clash;
     - the form is `role="search"`, `method="get"`.
   - Tasks 1, 3 and 4.
5. **The post-list template.**
   - Cases: inserted where the host registered no `core/recent-posts`; a post with no excerpt; a title with markup; the template on a host without the text or actions packages.
   - Expected:
     - with no collection, editors see the children once and visitors see nothing, as `core/repeat` already does;
     - a missing field uses the binding's fallback;
     - a title is text;
     - registering the template never fails, even where the types it uses are not loaded; inserting templates from the editor is part 7's.
   - Task 3.

---

### Task 1: The site map contract and shared heading anchors

**Files:**
- Create:
  - in `block-domain/src/`: `Contracts/SiteMapInterface.php`, `ValueObjects/Crumb.php`, `ValueObjects/SearchTarget.php`, `ValueObjects/HeadingAnchor.php`;
  - test: `block-domain/tests/ValueObjects/SiteTest.php`.
- Modify:
  - `block-text-domain/src/HeadingBehaviour.php`: `idFor()` delegates to `HeadingAnchor::fromText()`; the view's `id` is `core/heading-ids[$block->id]` when the context has it;
  - `block-text-domain/src/TextTypesPackage.php`: `core/heading` gains `uses: ['core/heading-ids']`;
  - test: `block-text-domain/tests/TypesTest.php`.

**Interfaces:**
- Produces:
  - **`SiteMapInterface`:**
    - `url(string $documentId): ?string` gives the page's address, or null when the host has none.
    - `trail(string $documentId): array`, documented `list<Crumb>`, runs from the site's top to the page itself.
    - `search(): ?SearchTarget` says where the site's search form posts, or null.
  - **`Crumb(string $label, string $url = '')`.** The URL is empty for the current page.
  - **`SearchTarget(string $url, string $parameter)`.**
  - **`HeadingAnchor`:**
    - `fromText(string $text): string` is today's `HeadingBehaviour::idFor` rules, moved here.
    - `unique(array $wanted): array` takes a `list<string>` and returns a `list<string>`. It keeps each first use. A later repeat becomes `{id}-2`, `{id}-3` and so on, skipping any that are already wanted or taken.

- [ ] **Step 1: Write the failing tests.**
  - `SiteTest`:
    - `fromText` gives the same answers as part 2's heading tests;
    - `unique(['intro', 'intro', 'a', 'intro-2', 'intro'])` gives `['intro', 'intro-3', 'a', 'intro-2', 'intro-4']`;
    - `Crumb` and `SearchTarget` carry their values.
  - `block-text-domain` `TypesTest`:
    - `core/heading` uses `core/heading-ids`;
    - with that context key set, the view's `id` is the context's;
    - without it, the id is derived as before.
- [ ] **Step 2: Run** `cd block-domain && vendor/bin/pest` and `cd block-text-domain && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `block-domain`, `block-text-domain` and `block-text` suites. Expected: PASS, and part 2's heading tests unchanged.
- [ ] **Step 5: Commit** `feat(block-domain): add the site map contract and shared heading anchors`.

---

### Task 2: Site maps and an array collection

**Files:**
- Create:
  - `block-infra-memory/src/InMemorySiteMap.php` and `block-infra-memory/src/ArrayCollectionSource.php`;
  - `block-infra-wp/src/Ports/WpSiteInterface.php`, `block-infra-wp/src/WordPress/WordPressSite.php` and `block-infra-wp/src/WpSiteMap.php`.
- Modify: `block-infra-memory/src/MemoryServiceProvider.php` (no default site map: a host without one has none), `block-infra-wp/src/WpServiceProvider.php`
- Test: `block-infra-memory/tests/SiteMapTest.php`, `block-infra-wp/tests/SiteMapTest.php`

**Interfaces:**
- Consumes: Task 1.
- Produces:
  - **`InMemorySiteMap(array $urls = [], array $trails = [], ?SearchTarget $search = null)`.**
  - **`ArrayCollectionSource(string $name, array $items)`** implements `CollectionSourceInterface`. Its arguments schema is `{type: object, properties: {}}`, and it gives the items as they are.
  - **`WpSiteInterface`:** `permalink(int $id): ?string`, `title(int $id): string`, `ancestors(int $id): list<int>` (nearest last), `homeUrl(): string`, `siteName(): string`.
  - **`WordPressSite`** implements `WpSiteInterface` with `get_permalink`, `get_the_title`, `get_post_ancestors` (reversed), `home_url('/')` and `get_bloginfo('name')`. It is not unit-tested, as the other `WordPress/` adapters are not.
  - **`WpSiteMap(WpSiteInterface $site)`:**
    - The URL is the permalink. A non-numeric document id has none.
    - The trail is the site name linked to the home URL, then each ancestor linked, then the page's own title with no URL.
    - The search target is `SearchTarget(homeUrl, 's')`.
  - **`WpServiceProvider`** binds `SiteMapInterface` to `WpSiteMap`.

- [ ] **Step 1: Write the failing tests.**
  - `InMemorySiteMap` gives what it was given, and nothing for unknown ids.
  - `ArrayCollectionSource` gives its items.
  - `WpSiteMap` with a fake port:
    - a page with two ancestors gives four crumbs, the last without a URL;
    - an id of `abc` gives a null URL and an empty trail;
    - the search target uses `s`.
- [ ] **Step 2: Run** each package's `vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `block-infra-memory` and `block-infra-wp` suites. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-infra): add site maps for memory and WordPress, and an array collection`.

---

### Task 3: Site block definitions, the page step and the post-list template

**Files:**
- Create `block-site-domain/`:
  - `composer.json` and `phpunit.xml`;
  - in `src/`: `SiteTypesPackage.php`, `SitePageStep.php`, `NavigationLinkBehaviour.php`, `TableOfContentsBehaviour.php`, `BreadcrumbsBehaviour.php`, `SearchBehaviour.php`, `PostListTemplate.php`;
  - in `tests/`: `Pest.php`, `PackageContractTest.php`, `TypesTest.php`, `PageStepTest.php`, `BehavioursTest.php`.

**Interfaces:**
- Consumes: Task 1's contract, `HeadingAnchor`, `DocumentStepInterface`, `Binding::to()`, `BlockTemplate`; Task 2's `InMemorySiteMap` and `ArrayCollectionSource` as dev dependencies.
- Produces:
  - **The package.** `SiteTypesPackage(?SiteMapInterface $siteMap = null)` and `static from(BlockServiceContainer $container): self`. Name `core/site-types`, title "Site". Settings:
    - **`core/navigation`:**
      - `label` (string, at most 80, default "Main");
      - `orientation` (enum `horizontal` or `vertical`, default `horizontal`);
      - children `core/navigation-link` only;
      - it uses `core/page-url`.
    - **`core/navigation-link`:**
      - `label` (string, at most 80, default "Link");
      - `url` (string, at most 2000, default `""`);
      - `newTab` (boolean, default false);
      - parents `core/navigation` and `core/navigation-link`; children `core/navigation-link`;
      - it uses `core/page-url` and `core/nav-depth`, and provides `core/nav-depth` from its view's `depth`.
      - A link's `depth` is one more than the `core/nav-depth` it inherits, and 1 when it inherits none: a top-level link is 1, a submenu link 2, a sub-submenu link 3.
      - A link deeper than 3 is unavailable: `UnavailableBlockException(IssueKind::DisallowedChild, 'Menus go two levels deep at most.')`. Parent and child policies cannot count depth.
    - **`core/table-of-contents`:**
      - `title` (string, at most 80, default "On this page");
      - `from` and `to` (integers 1 to 6, defaults 2 and 3);
      - it uses `core/headings`.
    - **`core/breadcrumbs`:**
      - `label` (string, at most 80, default "Breadcrumb");
      - it uses `core/breadcrumbs`.
    - **`core/search`:**
      - `label` (string, at most 80, default "Search");
      - `placeholder` (string, at most 80, default `""`);
      - `button` (string, at most 40, default "Search");
      - it uses `core/search`.
  - **`SitePageStep(?SiteMapInterface $siteMap)` implements `DocumentStepInterface`.** It walks the document's tree once, in order, and skips the children of `core/repeat` blocks. Included documents are separate documents, so they are never in the tree.
    - For each `core/heading` it reads `level`, `text` and `anchor` from its stored settings.
    - The id it wants is the anchor when there is one, otherwise `HeadingAnchor::fromText($text)`.
    - Typed anchors are made unique first, then derived ids, so a typed anchor keeps its exact value.
    - It sets `core/headings`, `core/heading-ids`, `core/page-url`, `core/breadcrumbs` and `core/search` (as listed in Global Constraints). Heading `text` is plain: tags stripped, entities decoded.
  - **Views:**
    - **Navigation link:** `{label, url, newTab, current, depth}`. `current` is true when the path of `url` equals the path of `core/page-url`, with trailing slashes ignored and query and fragment ignored. Two empty paths never match.
    - **Table of contents:** `{title, items: list<{level, text, id}>}`, keeping only levels between `from` and `to`. If `from` is greater than `to`, the two are swapped.
    - **Breadcrumbs:** `{label, crumbs: list<{label, url, current}>}`.
      - The last crumb is current and has no URL.
      - More than 20 crumbs keep the first and the last 19.
    - **Search:** `{label, placeholder, button, url, parameter, id}`.
      - `url` is empty when there is no target, or when the parameter does not match its pattern. Either way the block is not set up.
      - `id` comes from the block id, as part 5's element ids do: `'b-' . $encoded . '-search'`, with part 5's encoding repeated in this package. *My call:* that avoids a dependency on another area's package.
      - An empty label becomes "Search".
  - **`PostListTemplate::template(): BlockTemplate`.** It gives `core/post-list`, titled "Post list", in category `site`:
    - a `core/repeat` with source `core/recent-posts` and limit 5;
    - holding a `core/section`;
    - holding a `core/heading` (level 3), whose `text` is bound to `Binding::to('core/current-item', ['field' => 'title'])` with fallback "Post title";
    - then a `core/paragraph` bound to `excerpt`, with fallback `""`;
    - then a `core/button`, its `text` "Read more" and its `link` bound to `url`, with fallback `""`.

- [ ] **Step 1: Write the failing tests.**
  - `PackageContractTest` runs in definitions mode, with and without a site map.
  - `TypesTest` checks:
    - names, category, parents and children, uses, and defaults;
    - the template's name and tree shape;
    - registering the package on a host without the text package does not fail.
  - `PageStepTest` covers each Review Focus 1 heading case, with exact ids. It also checks the order of `core/headings`, that a heading inside a `core/repeat` is left out, and the site map values copied into the context.
  - `BehavioursTest` checks:
    - each Review Focus 2 current-page case;
    - the depth refusal at a third level;
    - the table of contents' `from` and `to`, and the swap;
    - breadcrumbs capped at 20, with the last crumb current;
    - each Review Focus 4 search case;
    - no site map leaves search, breadcrumbs and the page URL empty, with no issue.
- [ ] **Step 2: Run** `cd block-site-domain && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the `block-site-domain` suite, then `block-text-domain` (to confirm headings read their ids from the step). Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-site): add the site block definitions, the page step and the post-list template`.

---

### Task 4: Site block outputs

**Files:**
- Create `block-site/`:
  - `composer.json` and `phpunit.xml`;
  - in `src/`: `SitePackage.php`, `SiteOutput.php`, `SiteStylesheet.php`;
  - in `tests/`: `Pest.php`, `PackageContractTest.php`, `OutputTest.php`.

**Interfaces:**
- Consumes: Task 3's views.
- Produces:
  - **The package.** `SitePackage()`, with name `core/site` and title "Site, drawn". It registers `Asset::inline('core/site', AssetKind::Style, SiteStylesheet::CSS)` and a `SiteOutput` for the five blocks. `SiteOutput` declares `core/site`.
  - **Markup.** The attribute bag always goes on the outermost element.
    - **Navigation:** `<nav{attrs} class="b-nav b-nav--{orientation}" aria-label>` holding `<ul>` with the children.
    - **Navigation link:** `<li class="b-nav-item">` holding `<a href>`, marked `label`, with `aria-current="page"` when current. A new-tab link adds `target="_blank" rel="noopener noreferrer"`.
      - Children go in `<ul class="b-nav-sub">`, and the item gains `b-nav-item--has-sub`.
      - An unsafe address is drawn as `<span class="b-nav-text">`.
      - Not set up: nothing for visitors, and the label with "Add a link in the block's settings." for editors.
    - **Table of contents:** `<nav{attrs} class="b-toc" aria-labelledby>` holding `<h2 id>` (marked `title`), then `<ol>` with `<li class="b-toc-l{level}"><a href="#{id}">` and the text inside.
      - No items: nothing for visitors, and "Add headings to the page to list them here." for editors.
    - **Breadcrumbs:** `<nav{attrs} class="b-crumbs" aria-label>` holding `<ol>`.
      - Each crumb is `<li>` holding `<a href>`, or `<span>` for an unsafe address.
      - The current crumb is `<span aria-current="page">`.
      - No crumbs: nothing for visitors, and "The site gives this page no trail." for editors.
    - **Search:** `<form{attrs} class="b-search" role="search" method="get" action>` holding:
      - `<label for={id}>` (marked `label`);
      - `<input type="search" id name={parameter} placeholder>`;
      - `<button type="submit">` (marked `button`).

      Not set up: nothing for visitors, and "This site has no search to send to." for editors.
  - **`SiteStylesheet::CSS`:**
    - horizontal and vertical menus;
    - submenus hidden until `.b-nav-item:hover` or `:focus-within`, positioned under their item, and shown in place when the menu is vertical or narrow (a `max-width: 40em` media query);
    - table-of-contents indentation by level;
    - breadcrumbs separated by a `/` drawn with `::before` content, not markup;
    - the search row;
    - visible focus throughout;
    - no `@import` and no `url()`.

- [ ] **Step 1: Write the failing tests.**
  - `PackageContractTest` runs in outputs mode.
  - `OutputTest` checks:
    - each block's markup for visitors and while editing;
    - the Review Focus 2, 3 and 4 cases as drawn;
    - submenus nest as `ul` inside `li`;
    - the current link is marked;
    - a table of contents' links match the page's heading ids when `block-text-domain`'s heading is loaded beside it (a dev dependency for this one test);
    - shared style settings land on each outer element;
    - every output declares `core/site`;
    - the stylesheet has no `@import` and no `url(`, and has `:focus-within`.
- [ ] **Step 2: Run** `cd block-site && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the `block-site` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-site): add the site block outputs`.

---

### Task 5: The demo, a hand check and documentation

**Files:**
- Modify: `block-builder/demo/server.php`, `block-builder/demo/composer.json`, repository `README.md`, `wordpress-develop/docs/superpowers/specs/2026-11-18-block-library-design.md` (a part 6 note)

**Interfaces:**
- Consumes: Tasks 1 to 4.
- Produces:
  - **Demo packages.** The demo loads `SiteTypesPackage::from($container)` and `SitePackage` after the interactive packages.
  - **Demo host.** The demo binds:
    - an `InMemorySiteMap`: `home` is at `/view`, with the trail Demo (`/`) → Pages (`/view#pages`) → Home; search goes to `/view` with parameter `q`;
    - an `ArrayCollectionSource('core/recent-posts', …)` of three sample posts.
  - **Demo page.** At the top of the page it seeds a navigation: Home (`/view`), Blocks (with a submenu of Text, Media and Actions, linking to anchors on the page), and an external link opening in a new tab. Next it seeds breadcrumbs, a search, and a table of contents of levels 1 to 3. At the end it seeds the post-list template's tree.
  - **Demo headings.** It adds a second heading whose text repeats an earlier one, to show unique ids.
  - **Documentation.** The README gains a "Site" section. It covers the site map contract and its implementations, the step's context keys, the post-list template, and that heading ids are unique when the site package is loaded.

- [ ] **Step 1: Update** the demo, README and design note.
- [ ] **Step 2: Run** every PHP suite and both `npm test`s. Expected: PASS.
- [ ] **Step 3: Check by hand.** Delete the demo's `documents/` first so the page reseeds. On `/view`:
  - Tab reaches every menu and submenu link, and each submenu opens on focus;
  - the Home link is marked current;
  - every table-of-contents link scrolls to its heading, including the repeated heading's `-2` id;
  - breadcrumbs end in the current page;
  - search submits `?q=` to `/view`;
  - the post list shows three posts.

  In the editor (`/`), every block is visible, and a navigation link's label can be typed into.
- [ ] **Step 4: Commit** `feat(block-site): show the site blocks in the demo`.

---

## Out of scope for this plan

- A mobile menu button: the menu wraps and its submenus show in place on narrow screens.
- Menus stored apart from the page, such as WordPress menus.
- Search results drawn by the block system.
- Editing the post list's collection options in a dedicated control (the inspector's JSON field for `arguments` serves for now).
- Inserting templates from the editor (part 7).
- The remaining editor features (part 7).
