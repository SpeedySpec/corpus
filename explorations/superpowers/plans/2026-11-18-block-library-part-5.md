# Block library part 5: interactive blocks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the interactive area of the block library as a definitions package and a default outputs package:
- accordion and accordion item;
- tabs and tab;
- slider and slide;
- countdown, progress, counter and modal.

Every block is readable and usable with no script. A small script, served as a package file, adds tabs, slider controls, a live countdown and a counting animation. The demo gains a visitor page that serves packages' asset files, so the script can be tried.

**Architecture:**
- **`block-interactive-domain`** registers the ten block types. Its behaviours turn settings into what the outputs need: element ids made from block ids, a countdown's instant in UTC, and values kept in range.
- **`block-interactive`** registers the outputs and two assets: an inline stylesheet `core/interactive` and a module script `core/interactive-script`, a file in the package (`assets/interactive.js`). Every output declares both.
- **Scripts are optional.** They only enhance markup that already works: accordions are `details` elements, the modal uses the browser's `popover` attribute, tabs show every panel in order, and the slider is a scrolling row that snaps to each slide.
- **The demo** gains `PackageFiles`, which locates package files for the asset locator and serves them, and a `/view` page that draws the home page for a visitor, with its assets.

**Tech Stack:** PHP 8.4, Pest, Composer path repositories; plain ES modules and `node --test`.

**Spec:** `docs/superpowers/specs/2026-11-18-block-library-design.md` (in `wordpress-develop`): "The blocks" (Interactive), "Front-end assets" (especially "No JavaScript is needed for content to be readable"), "Deferred from part 1" (the demo's locator), and the Part 3 and Part 4 notes.

**Repository:** `wordpress-develop/packages/speedyspec-api-block`. Cut branch `feat/block-library-part-5` from `origin/main` once PR #12 (part 4) is merged; otherwise cut it from `feat/block-library-part-4`. Paths below are relative to its `packages/` directory.

## Global Constraints

- **Earlier constraints still hold.** Everything in parts 1 to 4's Global Constraints applies:
  - `declare(strict_types=1)`; 4 spaces;
  - braces of classes, methods and control structures on their own line;
  - docblocks per `wordpress-develop/.claude/rules/coding-style.md`; PHP lines at most 120 characters;
  - one commit per task with explicit paths, ending `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Dependencies.**
  - `block-interactive-domain` requires only `speedyspec/block-domain`.
  - `block-interactive` requires `speedyspec/block-domain` and `speedyspec/block-interactive-domain`.
  - Neither depends on `block-app`, another area's packages, or WordPress.
- **Block names and settings.**
  - The blocks are `core/accordion`, `core/accordion-item`, `core/tabs`, `core/tab`, `core/slider`, `core/slide`, `core/countdown`, `core/progress`, `core/counter` and `core/modal`, with category `interactive`.
  - The package title is "Interactive".
  - Their settings are fixed by this plan.
  - Every block has a title, a description and an example that resolves without issues.
- **No script is needed for content.** Every output draws complete, usable markup.
  - The script never creates content from settings with `innerHTML`. It moves the elements the output drew, sets text with `textContent`, and reads numbers from `data-` attributes.
  - Anything it cannot read, it leaves as drawn.
- **Motion.** When the visitor asks for reduced motion, the slider never plays by itself and the counter shows its final value at once. A slider that plays by itself always has a visible Pause button, and pauses while hovered or focused.
- **Element ids** come from block ids only, as `b-` followed by the block id with anything outside `[A-Za-z0-9_-]` turned into `-`, then a suffix. Two blocks on one page therefore never share an id.
- **Not set up.** A countdown with no target date follows the parts 3 and 4 "not set up" rule: visitors get nothing, and editors get a prompt.

## Review Focus

1. **Reading the page with no script.**
   - Cases: every block with the script missing, blocked, or failing to load.
   - Expected:
     - accordion items open and close;
     - every tab's label and panel are shown in order;
     - slides can be scrolled through;
     - the countdown names its date and time in words, with the time zone;
     - the counter shows its final number;
     - progress shows its value;
     - the modal opens and closes through `popover`;
     - nothing is hidden that only the script could show.
   - Tasks 3 and 5.
2. **A countdown's date.**
   - Cases:
     - an unknown time zone;
     - a local time skipped by daylight saving (2027-03-28T01:30 in Europe/London);
     - a repeated one (2027-10-31T01:30 in Europe/London);
     - a target already past when the page is drawn;
     - a target 50 years away;
     - a visitor whose clock is wrong.
   - Expected:
     - an unknown zone is unavailable with an `invalid-settings` issue (its pattern only keeps the name well formed, so the zone itself is checked when the page is drawn);
     - a skipped time moves to the next valid instant, and a repeated one uses the earlier;
     - a past target shows the "after" text and no timer;
     - the script uses the `data-target` instant and never counts below zero.
   - Tasks 2, 3 and 4.
3. **Keys and screen readers.**
   - Cases:
     - tabs driven with Left, Right, Home and End;
     - the slider's Previous, Next and Pause buttons;
     - the modal's Escape key and Close button;
     - an accordion set to one item open at a time.
   - Expected:
     - the tab, tabpanel and tablist roles follow WAI-ARIA's tabs pattern, with roving `tabindex`;
     - the slider is a region named by its label, and each slide is announced as "slide 2 of 5";
     - Escape closes the modal and focus returns to its button;
     - opening one accordion item closes the others only when `single` is set.
   - Tasks 3 and 4.
4. **Many on one page.**
   - Cases: two tab sets; tabs inside a tab; two sliders; the script loaded twice; a block added after the script ran.
   - Expected:
     - ids never clash;
     - each set of tabs owns only its own tabs, never a nested set's;
     - each element is set up exactly once (`data-b-ready`);
     - a second load does nothing.
   - Tasks 3 and 4.
5. **Numbers the script reads.**
   - Cases: an interval of `0`, `-1` or `abc`; a duration of `1e9`; a counter whose `from` is greater than its `to`; `decimals` of 3; a `data-target` that is not a date.
   - Expected: everything is kept to its range (the interval 3 to 30 seconds, the duration 0 to 10 seconds); a counter can count down; an unreadable target leaves the countdown as drawn.
   - Tasks 2 and 4.

---

### Task 1: Serving package assets in the demo, and a visitor page

**Files:**
- Create: `block-builder/php/PackageFiles.php`
- Modify: `block-builder/demo/server.php`
- Test: `block-builder/tests/PackageFilesTest.php` (in `block-builder`'s Pest suite)

**Interfaces:**
- Consumes: part 1's `AssetLocatorInterface`, `AssetRegistryInterface`, `Asset`.
- Produces:
  - **`PackageFiles`** is `PackageFiles(array $roots, string $prefix = '/packages/')` and implements `AssetLocatorInterface`.
    - `$roots` maps a package name such as `speedyspec/block-builder` to its directory.
    - `url(string $package, string $path): string` gives `"{$prefix}{$package}/{$path}"` for a known package, and `''` otherwise.
    - `file(string $urlPath): ?array{path: string, type: string}` gives the file for a URL path. It returns null:
      - when the package is unknown;
      - when the real path is outside that package's directory;
      - when the extension is not `js`, `mjs` or `css`;
      - when the file does not exist.
    - The type is `text/javascript` or `text/css`.
  - **Demo files and locator.** The demo builds `PackageFiles` from its `vendor/speedyspec/*` packages and its own `block-builder`. It uses `PackageFiles` as the asset locator in place of today's stub, and serves `/packages/...` through `file()`, with `Cache-Control: no-store` and `X-Content-Type-Options: nosniff`. The editor's own script moves under `/packages/speedyspec/block-builder/src/…`.
  - **`/view` page.** The demo's `/view` draws the home document for a visitor (`Mode::Front`) as a whole HTML page. Its head holds the collected CSS and the resolved assets: styles as `<link>` or `<style>`, scripts and modules as `<script>`. A Content Security Policy goes with it: `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; frame-src https://www.youtube-nocookie.com https://player.vimeo.com https://www.openstreetmap.org`.

- [ ] **Step 1: Write the failing tests.**
  - `url()` for a known and an unknown package.
  - `file()` for a js file, a css file, `../` out of the package, an encoded `%2e%2e`, a `.php` file, a missing file and an unknown package.
  - A file reached through a symbolic link that leads outside the package is refused.
- [ ] **Step 2: Run** `cd block-builder && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement `PackageFiles` and wire the demo.** `/` still opens the editor, and `/view` is the visitor page.
- [ ] **Step 4: Run** the `block-builder` Pest suite and `npm test`. Start the demo, check that `/` loads the editor and `/view` draws the page, then check by hand that `/packages/../demo/server.php` is a 404. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-builder): serve package assets and a visitor page in the demo`.

---

### Task 2: Interactive block definitions

**Files:**
- Create: `block-interactive-domain/`, holding `composer.json` and `phpunit.xml`, plus:
  - in `src/`: `InteractiveTypesPackage.php`, `ElementIds.php`, `AccordionItemBehaviour.php`, `CountdownBehaviour.php`, `ProgressBehaviour.php`, `CounterBehaviour.php`, `IdBehaviour.php`;
  - in `tests/`: `Pest.php`, `PackageContractTest.php`, `TypesTest.php`, `BehavioursTest.php`.

**Interfaces:**
- Consumes: `UnavailableBlockException`, `IssueKind`.
- Produces:
  - **The package.** `InteractiveTypesPackage(?Closure $now = null)` takes `Closure(): DateTimeImmutable`, which tells the countdown the time. It defaults to the system clock. The package's name is `core/interactive-types` and its title is "Interactive". The settings are:
    - **`core/accordion`:** `single` (boolean, default false: one item open at a time). Its children are `core/accordion-item` only. It provides `core/accordion-group` from its view's `group`.
    - **`core/accordion-item`:** `title` (string, at most 200, default "Question") and `open` (boolean, default false). Its parent is `core/accordion`, and it uses `core/accordion-group`.
    - **`core/tabs`:** no settings. Its children are `core/tab` only.
    - **`core/tab`:** `label` (string, at most 80, default "Tab"). Its parent is `core/tabs`.
    - **`core/slider`:**
      - `label` (string, at most 120, default "Slides");
      - `autoplay` (boolean, default false);
      - `interval` (integer 3 to 30 seconds, default 6).
      - Its children are `core/slide` only.
    - **`core/slide`:** `label` (string, at most 120, default `""`). Its parent is `core/slider`.
    - **`core/countdown`:**
      - `label` (string, at most 120, default "Starts in");
      - `target` (string, pattern `^(\d{4}-\d{2}-\d{2}T\d{2}:\d{2})?$`, default `""`), a local date and time;
      - `timeZone` (string, at most 64, pattern `^[A-Za-z]+(?:[/_+-][A-Za-z0-9]+)*$`, default `UTC`);
      - `after` (rich text, default "It has started.").
    - **`core/progress`:**
      - `label` (string, at most 120, default "Progress");
      - `value` (number, minimum 0, default 40);
      - `max` (number, minimum 1, default 100).
    - **`core/counter`:**
      - `from` and `to` (numbers, defaults 0 and 100);
      - `prefix` and `suffix` (strings, at most 10 each);
      - `duration` (integer, 0 to 10000 milliseconds, default 2000);
      - `decimals` (integer, 0 to 3, default 0).
    - **`core/modal`:** `trigger` (string, at most 80, default "Open") and `title` (string, at most 200, default "Details"). Children are any.
    - **Children elsewhere.** `core/accordion-item`, `core/tab`, `core/slide` and `core/modal` take any children.
  - **`ElementIds::for(string $blockId, string $suffix): string`.** It gives `b-{clean id}-{suffix}`, as Global Constraints describes.
  - **Views.** The outputs read these:
    - **Ids.** `IdBehaviour` serves tabs, tab, slider, slide and modal. Each view is its settings plus an `id`, which is `ElementIds::for($block->id, 'x')` for the block's own element.
    - **Accordion and its items.** The accordion's view adds `group` (`ElementIds::for($block->id, 'group')`) when `single` is set, and `''` otherwise. An item's view is `{title, open, group}`, with `group` taken from the context.
    - **Countdown.** `{label, after, target, ended, when, timeZone}`, or not set up (`target` is `''`):
      - `target` is the instant in UTC, written `Y-m-d\TH:i:s\Z`.
      - `when` is the local date and time in words, such as "31 December 2026, 23:59 (Europe/London)".
      - `ended` compares `target` with `now`.
      - A skipped local time moves to the next valid instant, and a repeated one uses the earlier offset. That is what `DateTimeImmutable` does with a zone, and the tests pin it.
      - A zone that `DateTimeZone` does not know is unavailable: `UnavailableBlockException(IssueKind::InvalidSettings, "Unknown time zone.", 'timeZone')`.
    - **Progress.** `{label, value, max, percent}`. `value` is kept within 0 and `max`, and `percent` is an integer from 0 to 100.
    - **Counter.** `{from, to, prefix, suffix, duration, decimals, text}`. `text` is the final value, formatted with `decimals` places, between the prefix and suffix.

- [ ] **Step 1: Write the failing tests.**
  - `PackageContractTest` in definitions mode.
  - `TypesTest`:
    - names, category, parents and children, and defaults;
    - the schema refuses an interval of 2 and of 31, a duration of 10001, decimals of 4, a target of `2026-13-01T10:00`, a time zone of `../etc`, and a `max` of 0;
    - every block has a title, a description and an example.
  - `BehavioursTest`, with a fixed clock at 2026-11-18T12:00:00Z:
    - **Countdown.**
      - 2026-12-31T23:59 in Europe/London gives `2026-12-31T23:59:00Z`, not ended.
      - 2026-07-01T09:00 in Europe/London gives `2026-07-01T08:00:00Z`, ended, because that is before the clock.
      - The skipped 2027-03-28T01:30 in Europe/London gives `2027-03-28T01:30:00Z`, the next valid instant.
      - The repeated 2027-10-31T01:30 in Europe/London gives `2027-10-31T00:30:00Z`, the earlier offset.
      - An unknown zone stored past the editor is unavailable with its message.
      - No target means not set up, with no issue.
    - **Progress.** 150 of 100 gives 100%; 1 of 3 gives 33%.
    - **Counter.** The text gives "$1,234.50+" for a prefix of `$`, a suffix of `+`, a `to` of 1234.5 and 2 decimals.
    - **Ids.** A block id `a b/c` gives `b-a-b-c-x`. The accordion's group is shared by its items only when `single` is set.
- [ ] **Step 2: Run** `cd block-interactive-domain && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the `block-interactive-domain` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-interactive): add the interactive block definitions`.

---

### Task 3: Interactive block outputs

**Files:**
- Create: `block-interactive/`, holding `composer.json`, `phpunit.xml` and `assets/interactive.js` (an empty module for now; Task 4 fills it), plus:
  - in `src/`: `InteractivePackage.php`, `InteractiveOutput.php`, `InteractiveStylesheet.php`;
  - in `tests/`: `Pest.php`, `PackageContractTest.php`, `OutputTest.php`.

**Interfaces:**
- Consumes: Task 2's views.
- Produces:
  - **The package.** `InteractivePackage` is `new InteractivePackage()`, with `PACKAGE = 'speedyspec/block-interactive'`. Its name is `core/interactive` and its title is "Interactive, drawn". It registers:
    - `Asset::inline('core/interactive', AssetKind::Style, InteractiveStylesheet::CSS)`;
    - `Asset::file('core/interactive-script', AssetKind::Module, self::PACKAGE, 'assets/interactive.js')`;
    - an `InteractiveOutput` for each of the ten blocks.

    `InteractiveOutput` implements `DeclaresAssetsInterface` and declares both assets.
  - **Markup.** The attribute bag always goes on the outermost element. While editing, every panel, slide and modal body is drawn open and in place, so it can be typed into and selected. These outputs are for visitors:
    - **Accordion.** `<div{attrs} class="b-accordion">` holds the children. Each item is `<details class="b-accordion-item"{ name=group}{ open}>` holding `<summary>` (marked `title`) and `<div class="b-accordion-panel">` with the children inside. While editing, every item has `open` and no `name`.
    - **Tabs.**
      - The set is `<div{attrs} class="b-tabs" data-b-tabs id>`, holding the children.
      - Each tab is `<section{attrs} class="b-tab" data-b-tab id>`, holding `<h3 class="b-tab-label">` (marked `label`) and `<div class="b-tab-panel">` with the children.
      - With no script, this reads as headed sections in order.
    - **Slider.**
      - The slider is `<section{attrs} class="b-slider" aria-roledescription="carousel" aria-label data-b-slider data-autoplay="0|1" data-interval id>`, holding `<div class="b-slides">` with the children.
      - Each slide is `<div{attrs} class="b-slide" role="group" aria-roledescription="slide">`, with `aria-label` when the slide has a label. The script adds "n of m" otherwise.
      - The stylesheet makes `.b-slides` a horizontal scroll that snaps to each slide, so it works with no script.
    - **Countdown.**
      - The countdown is `<div{attrs} class="b-countdown" data-b-countdown data-target>`, holding `<p class="b-countdown-label">` and `<p class="b-countdown-time"><time datetime>` with `when` inside.
      - When `ended`, it holds `<div class="b-countdown-after">` with the after text instead. Otherwise that element is drawn `hidden`, so the script can show it when time runs out.
      - Not set up: nothing for visitors, and "Choose a date and time in the block's settings." for editors.
    - **Progress.** `<div{attrs} class="b-progress">` holds `<span class="b-progress-label" id>`, then `<progress max value aria-labelledby>` with the percentage inside, then `<span class="b-progress-value">40%</span>`.
    - **Counter.**
      - The counter is `<p{attrs} class="b-counter" data-b-counter>` with `data-from`, `data-to`, `data-duration`, `data-decimals`, `data-prefix` and `data-suffix`.
      - Inside are `<span class="b-counter-number" aria-hidden="true">` and `<span class="b-visually-hidden">`, both holding `text`.
    - **Modal.**
      - The modal is `<div{attrs} class="b-modal">`, holding `<button type="button" class="b-modal-open" popovertarget>` with the trigger text, then `<div class="b-modal-panel" id popover role="dialog" aria-labelledby>`.
      - The panel holds `<h2 id>` with the title, the children, and `<button type="button" popovertarget popovertargetaction="hide">Close</button>`.
      - While editing, the panel is drawn without `popover`, so its content shows.
  - **`InteractiveStylesheet::CSS`.**
    - It styles the classes above, including a `.b-visually-hidden` helper.
    - The slider snaps with `scroll-snap-type: x mandatory`, and its smooth scrolling is turned off for reduced motion.
    - The modal panel is centred, with a backdrop through `::backdrop`.
    - It has no `@import` and no `url()`.

- [ ] **Step 1: Write the failing tests.**
  - `PackageContractTest` runs in outputs mode.
  - `OutputTest` checks each block's markup for a visitor and while editing (Review Focus 1). It also checks:
    - `single` accordions give their items one `name`;
    - two tab sets have distinct ids;
    - a slide is announced as a slide;
    - a countdown already ended shows the after text and no `<time>`;
    - the progress text is drawn twice, as the bar's fallback and as the visible value;
    - the counter's visible number is `aria-hidden`, with the same text for screen readers;
    - the modal panel uses `popover` for visitors and not while editing;
    - shared style settings land on each outer element;
    - every output declares both assets, and a page lists each once;
    - the stylesheet has no `@import` and no `url(`.
- [ ] **Step 2: Run** `cd block-interactive && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the `block-interactive` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-interactive): add the interactive block outputs`.

---

### Task 4: The enhancement script

**Files:**
- Create: `block-interactive/assets/logic.js`, `block-interactive/package.json` (`{"type": "module", "scripts": {"test": "node --test tests/js/"}}`, no dependencies), `block-interactive/tests/js/logic.test.js`
- Modify: `block-interactive/assets/interactive.js`

**Interfaces:**
- Consumes: Task 3's markup and `data-` attributes.
- Produces:
  - **`logic.js` (no DOM):**
    - `clamp(value, min, max, fallback)`: a number within its bounds, or the fallback when the value is not a finite number.
    - `nextTab(index, count, key)`: the index after Left, Right, Home or End, wrapping around, or null for any other key.
    - `remaining(targetMs, nowMs)`: `{days, hours, minutes, seconds, ended}`, never negative.
    - `countAt(from, to, elapsedMs, durationMs)`: the eased value. It reaches `to` exactly at the end, and works when `from` is greater than `to`.
    - `formatNumber(value, decimals, prefix, suffix)`: uses `toLocaleString('en', {minimumFractionDigits: decimals, maximumFractionDigits: decimals})`.
    - `slideLabel(index, count)`: gives "Slide 2 of 5".
  - **`interactive.js`** imports `logic.js`. On `DOMContentLoaded`, or at once if the page has already loaded, it sets up each `[data-b-tabs]`, `[data-b-slider]`, `[data-b-countdown]` and `[data-b-counter]` that has no `data-b-ready`. It then marks the element `data-b-ready`. A second load of the script does nothing, because a `window.__bInteractive` flag is set on the first.
    - **Tabs.**
      - It builds a `role="tablist"` of buttons, whose text is copied from each direct child tab's label.
      - Each panel gets `role="tabpanel"`, `aria-labelledby` and `hidden`.
      - The label heading becomes visually hidden, not removed.
      - It uses roving `tabindex` and the `nextTab` keys.
      - It owns only its direct children with `[data-b-tab]`.
    - **Slider.**
      - It adds Previous and Next buttons that scroll by one slide.
      - With autoplay, it adds a Pause/Play button, advances every `clamp(interval, 3, 30, 6)` seconds, and pauses on hover, focus or reduced motion.
      - It labels slides without a label with `slideLabel`.
    - **Countdown.**
      - Once a second it updates `days`, `hours`, `minutes` and `seconds` in a `<span>` beside the `<time>`. The `<time>` itself stays.
      - When the target passes, it reveals the after element and stops.
      - An unreadable `data-target` leaves the countdown as drawn.
    - **Counter.**
      - When the counter first scrolls into view (`IntersectionObserver`), it animates the visible number with `countAt` over `clamp(duration, 0, 10000, 2000)` milliseconds.
      - With reduced motion, or no `IntersectionObserver`, it does nothing, so the final value stays.

- [ ] **Step 1: Write the failing tests** for each `logic.js` function:
  - `clamp` of 0, −1, `'abc'`, `1e9` and `NaN`;
  - `nextTab` at both ends and for an unknown key;
  - `remaining` before, at and after the target, and 50 years out;
  - `countAt` at 0, halfway and the end, and counting down;
  - `formatNumber` with 0 and 3 decimals and a prefix and suffix;
  - `slideLabel`.
- [ ] **Step 2: Run** `cd block-interactive && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement** `logic.js` and `interactive.js`.
- [ ] **Step 4: Run** `npm test`, and `node --check assets/interactive.js`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-interactive): enhance tabs, sliders, countdowns and counters with a small script`.

---

### Task 5: The demo, a hand check and documentation

**Files:**
- Modify: `block-builder/demo/server.php`, `block-builder/demo/composer.json`, the repository `README.md`, `block-builder/README.md`, `wordpress-develop/docs/superpowers/specs/2026-11-18-block-library-design.md` (a part 5 note)

**Interfaces:**
- Consumes: Tasks 1 to 4.
- Produces:
  - **Demo packages.** The demo loads `InteractiveTypesPackage` and `InteractivePackage` after the actions packages.
  - **Demo page.** The demo seeds the page with:
    - an accordion of three questions, set to one open at a time;
    - tabs with three tabs, one holding a nested set of tabs;
    - a slider of three slides with autoplay;
    - a countdown to 2027-01-01T00:00 in Europe/London;
    - a progress bar at 65 of 100;
    - a counter from 0 to 1200 with the suffix `+`;
    - a modal holding a paragraph.
  - **Documentation.** The repository README gains an "Interactive" section. The builder README gains the visitor page.

- [ ] **Step 1: Update** the demo, READMEs and design note.
- [ ] **Step 2: Run** every PHP suite, the builder's `npm test` and the interactive package's `npm test`. Expected: PASS.
- [ ] **Step 3: Check by hand.** First delete the demo's `documents/` so the page reseeds.
  - On `/view`:
    - tabs work with the keys from Review Focus 3, and the nested set is separate;
    - the slider's buttons work, autoplay pauses on hover and with its button, and emulated reduced motion stops it;
    - the countdown ticks;
    - the counter counts once when scrolled to;
    - the modal opens, closes with Escape and gives focus back.
  - On `/view` with scripts blocked (the browser's JavaScript turned off for the page, or the script's URL failing): every block is readable and usable, as Review Focus 1 says.
  - In the editor (`/`): every panel, slide and modal body is visible and can be typed into.
- [ ] **Step 4: Commit** `feat(block-interactive): show the interactive blocks in the demo`.

---

## Out of scope for this plan

- Saving which tab or accordion item a visitor opened.
- Slider transitions other than scrolling.
- Countdowns that repeat.
- A modal opened by anything other than its own button.
- Analytics events.
- The site area (part 6) and the remaining editor features (part 7).
