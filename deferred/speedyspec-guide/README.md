# speedyspec-guide: deferred findings

Repository: [SpeedySpec/speedyspec-guide](https://github.com/SpeedySpec/speedyspec-guide).

## Where these came from

The guide site's move to Astro 7 and a Starlight docs theme (SpeedySpec/speedyspec-guide#4) was read as a whole branch
by a fresh reviewer. It found no Critical findings, one Important finding and seven Minor ones. The Important one and
five of the Minor ones were fixed in the same pull request:

- `mount()` and `browserStore()` in the read-progress script had no tests.
- A short guide was ticked as read as soon as it opened.
- On phones, the home page had no theme selector, and its header title was cut short.
- The read key came from the browser's address, so a host that dropped the trailing slash never ticked a guide.
- The track ids and labels were written in two places.
- The test helper ran Pagefind itself, which hid a broken search index.
- `headingsIn` read `data-id` as an id.

What follows is what is still open: the reviewer's remaining notes, and trade-offs the implementer chose and recorded.

Recorded 2026-11-22.

## Index

| File | Area | Findings |
|---|---|---|
| [site-theme.md](site-theme.md) | `site/src` layouts, Starlight overrides, read-progress script | 6 |
