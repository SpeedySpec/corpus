# Site theme (Astro 7 and Starlight)

`site/src/layouts`, `site/src/components/starlight`, `site/src/model/starlight.ts`, `site/src/scripts/read-progress.ts`.
From SpeedySpec/speedyspec-guide#4.

## Each method's "Examples" heading repeats under "On this page"

- **Where:** reference pages, through `layouts/Reference.astro`'s `headingsIn()`. `components/Examples.astro` renders
  its heading as an `h2`, including inside each method's section on a class page.
- **What happens:** on a large class page the contents list alternates `::method()` and "Examples" many times, which
  makes it long and hard to scan.
- **Suggested fix:** render the examples heading as an `h3` inside a method section, or leave headings nested in
  `.method` out of the contents list.

## No 404 page in the new shell

- **Where:** `site/astro.config.mjs` (`disable404Route: true`), with no `site/src/pages/404.astro` on the branch.
- **What happens:** a missing page gets the host's default "not found" page instead of the guide's. Another session's
  uncommitted `404.astro` was written for the old `Base` layout, which no longer exists, so it won't build as it is.
- **Suggested fix:** render it through `layouts/Page.astro` with `template="splash"` and `search={false}`.

## A guide too short to scroll can never be ticked

- **Where:** `mount()` in `site/src/scripts/read-progress.ts`.
- **What happens:** a guide is counted as read only once the reader has scrolled and its end is on screen. A guide
  that fits on the screen without scrolling never gets its tick. None of today's guides is that short.
- **Suggested fix:** also count the guide when its end has stayed on screen for a few seconds, if very short guides
  appear.

## The read-progress script names the uncounted track itself

- **Where:** `progress()` in `site/src/scripts/read-progress.ts` skips the track named `'topics'`.
- **What happens:** the list of tracks lives in `TRACKS` in `site/src/model/starlight.ts`. If a new unordered track
  were added there, or topics were renamed, the script would count it anyway. The script only fills in bars the page
  already has, so the result stays hidden, but the rule is written in two places.
- **Suggested fix:** count only the tracks that have a `[data-progress-track]` bar on the page, and drop the
  `'topics'` check.

## Symbol page titles lost their monospace font and search name

- **Where:** `components/starlight/PageTitle.astro`. `components/Head.astro` used to render the title as
  `<h1 class="mono" data-pagefind-meta="name">`.
- **What happens:** symbol names such as `add_filter()` now show in the body font. Pagefind no longer gets a separate
  `name` field and uses the page title instead, which may change how search ranks or shows symbols.
- **Suggested fix:** in the `PageTitle` override, add `mono` and `data-pagefind-meta="name"` on symbol pages, for
  example by passing a flag through `layouts/Page.astro`.

## Every build warns that the docs collection is empty

- **Where:** the `docs` collection in `site/src/content.config.ts`, which exists only for its schema (the `chapter`
  field).
- **What happens:** `[WARN] The collection "docs" does not exist or is empty` appears in every build log. It's harmless
  noise that could hide a real warning.
- **Suggested fix:** silence it if Starlight offers a way, or put one hidden placeholder entry in the collection.
