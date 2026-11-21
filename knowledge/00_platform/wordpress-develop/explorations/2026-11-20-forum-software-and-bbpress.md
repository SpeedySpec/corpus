# Exploration: Forum software and bbPress

**Status:** complete
**Created:** 2026-11-20
**Author:** Claude, for the project owner
**Context:** Research into free and open-source forum software, commercial forum suites and bbPress, to scope a forum built on SpeedySpec's WordPress core with every feature of the popular open-source forums.
**Approved by:** not recorded
**Socialized with:** not socialized

---

## Problem Statement

The project owner wants a forum built on WordPress core. It should have every feature of the popular free and open-source forum software, plus whichever features of commercial suites an open-source version could reasonably add. Three things were unknown:

- what the open-source forums actually ship in late 2026, in PHP and in other languages;
- how bbPress, the incumbent WordPress forum, is built, how healthy it is, and how it scales;
- whether SpeedySpec should port bbPress, build a forum natively, or leave forums to plugins.

The answer has to fit the port's rules: a Rust domain behind ports and adapters, plain entities persisted through repositories, and no PHP Hook API to lean on.

## Current State

Findings as the research left them. **Secondary only** marks findings that rest on trade press, aggregators, community threads or memory rather than a project's own repository or documentation.

**The open-source PHP forums are alive, and all are mid-rewrite.**

- phpBB 3.3.19 (September 2026) is stable; 4.0 is in alpha on Symfony and PHP 8.2 ([phpBB tags](https://github.com/phpbb/phpbb/tags)).
- MyBB 1.8.41 shipped September 2026; its 1.9 branch moves to Illuminate and Twig ([MyBB releases](https://github.com/mybb/mybb/releases)).
- SMF 2.1.8 shipped October 2026; SMF 3.0 is an object-oriented rewrite in alpha ([SMF releases](https://github.com/SimpleMachines/SMF/releases)).
- Flarum 2.0 shipped 9 October 2026 on PHP 8.3 and Illuminate 13, bundling messages, realtime, likes, mentions, flags, approval, audit and Akismet as first-party extensions ([Flarum extensions](https://github.com/flarum/framework/tree/v2.0.2/extensions)).
- Discourse (Ruby, GPL) releases monthly. On 10 July 2025 it bundled about 29 plugins into core, disabled by default: AI, Assign, Events, Data Explorer, Gamification, Reactions, Solved, Subscriptions, Templates, Topic Voting and others ([announcement](https://meta.discourse.org/t/bundling-more-popular-plugins-with-discourse-core/373574)).
- Dead: FluxBB (2014), PunBB (2017), FeatherBB (2019), and Vanilla's open-source edition, which Higher Logic discontinued ([update](https://open.vanillaforums.com/discussion/39624/an-update-on-vanilla-open-source)). Waterhole is commercial despite a 0BSD badge: production use is paid and its core package is marked proprietary ([pricing](https://waterhole.dev/pricing)).

The schemas barely changed in twenty years while every codebase is being rewritten. What carries over is a domain shape, not code.

**Open-source forums outside PHP add ideas the PHP forums lack.**

- NodeBB (GPL-3.0) and Lemmy and Mbin federate over ActivityPub using FEP-1b12: a category is a `Group` actor that re-announces posts ([NodeBB ActivityPub](https://docs.nodebb.org/activitypub/)).
- Lemmy publishes its ranking math: a time-decayed log of score, plus "Active" and community-size "Scaled" sorts ([ranking](https://join-lemmy.org/docs/contributors/07-ranking-algo.html)).
- PieFed adds private votes that are never federated, image-hash blocking, keyword filters and low-reputation indicators ([features](https://join.piefed.social/features/)).
- Apache Answer unlocks privileges by reputation; Misago and Talkyard have best-answer topics; Talkyard doubles as embedded blog comments; Zulip makes topics resolvable and lets messages move between them ([Zulip topics](https://zulip.com/help/introduction-to-topics)).
- Dead or stalled: /kbin (2023), Question2Answer (no release since July 2023).

**Commercial suites sell bundles and operations, not better forums.**

- XenForo, Invision Community and vBulletin sell an all-in-one package: gallery, downloads, CMS, clubs, paid upgrades, ad slots and warning points. Their sites refused direct fetches, so parts of their feature lists are **secondary only**.
- Hosted Discourse and Higher Logic Vanilla sell meters: analytics retention, automation counts, SSO connections, audit-log retention and AI credits ([Discourse pricing](https://www.discourse.org/pricing)). An open-source project can ship the feature without the meter.
- AI arrived in 2024–2026: Discourse AI (bring your own model when self-hosted), Invision's plan-gated image scanner, Vanilla's retrieval search. No first-party AI was found in XenForo or vBulletin.
- WordPress already supplies much of the bundle: a CMS, the block editor, the media library, a REST API and commerce through WooCommerce.

**bbPress is hooks over postmeta.**

- Forums, topics and replies are three post types; four custom statuses (`closed`, `spam`, `orphan`, `hidden`); one `topic-tag` taxonomy. Parent IDs sit in `post_parent` and are copied into meta ([bbpress.php](https://github.com/bbpress/bbPress/blob/trunk/src/bbpress.php)).
- Counters and freshness live in postmeta and are kept current by hook callbacks whose priorities matter. About 20 repair tools rebuild them when they drift.
- Topic lists sort on the `_bbp_last_active_time` meta value, which cannot use an index.
- Five forum roles are injected at runtime and never stored. Theme compatibility edits the global `$wp_filter` to strip and restore filters ([theme-compat.php](https://github.com/bbpress/bbPress/blob/trunk/src/includes/core/theme-compat.php)).
- Trunk has about 250 action names and 780 filter names. It registers no REST routes of its own; posting and subscribing are form handlers and admin-ajax.
- Two parts are already ports in disguise: the Engagements API picks a storage strategy, and the importer is a declarative field map with converters for 22 platforms.

**bbPress's health rests on one person.**

- Version 2.6.22 (16 November 2026), still on the 2.6 branch from 2019; 2.7 is an alpha with no date.
- The Trac timeline shows no commits from 2022 to early 2024, then 1,158 in about 90 days from August 2026, all by John James Jacoby ([timeline](https://bbpress.trac.wordpress.org/timeline)). The releases were mostly security and access-control fixes, plus block-theme support.
- 100,000+ installs, rated 3.9/5 ([plugin page](https://wordpress.org/plugins/bbpress/)). Patchstack lists nine vulnerabilities, all patched.
- WordPress.org's own forums run bbPress only by switching parts of it off: they sort by ID instead of freshness, limit views by date, cap pagination, send search to Google and disable whole-forum subscriptions ([performance class](https://github.com/WordPress/wordpress.org/blob/trunk/wordpress.org/public_html/wp-content/plugins/support-forums/inc/class-performance-optimizations.php)).
- Ticket reports of 5-second sorts at 250,000 topics and 30-second posting at 11 million meta rows are **secondary only**: Trac pages blocked the fetcher. No independent benchmark exists.

## Proposed Solution

Build a native forum module on dedicated tables. Treat bbPress as an import source and a behaviour specification, not as code to port.

### Architecture

```
   HTTP / JSON API adapter        blocks and templates (render via the API)
              |                              |
              v                              v
   +--------------------------------------------------------+
   |  forum application: commands and queries (use cases)   |
   +--------------------------------------------------------+
              |                              |
              v                              v
   +--------------------------+   +--------------------------+
   |  forum domain (plain     |   |  forum contracts (ports) |
   |  entities and policies)  |   |  repositories, SearchIndex,
   +--------------------------+   |  SpamCheck, LanguageModel,
                                  |  PushTransport, JobQueue, |
                                  |  PaymentProvider,         |
                                  |  Federation, InboundMail, |
                                  |  EventStream, Importer    |
                                  +-------------+------------+
                                                |
         +-----------+-----------+--------------+---------+------------+
         v           v           v              v         v            v
     database    database FTS  Akismet,     Web Push   WooCommerce  phpBB, SMF,
     mappers     / OpenSearch  AI classifier           / Stripe     bbPress ... importers
```

Committed domain events (`PostCreated`, `PostSpammed`, `TopicMoved`, ...) drive notifications, mail, search indexing, webhooks and federation.

### Data Model

- `forum`: adjacency list with cached depth and path; visibility (public, private, hidden, password) inherited down the tree; direct counts only.
- `topic`: `forum_id`, an archetype (discussion, question, private message, article), state (open, closed, sticky, resolved), `last_active_at`, `reply_count`, `highest_post_number`, optional `hot_rank` and `active_rank`. Index on `(forum_id, sticky, last_active_at)`.
- `post`: `topic_id`, a per-topic `post_number`, `reply_to`.
- `topic_user`: `last_read_post_number`, notification level (muted, normal, tracking, watching), bookmarked, liked and posted flags. Unread count is `highest_post_number - last_read_post_number`. A `forum_user` row or a global mark handles "mark all read".
- `review_item`, `warning`, `vote`, `reaction` and a moderator log with a public-visibility flag.

### Interfaces

- **Repositories per aggregate** for forum, topic, post, topic-user, review item and warning.
- **Ports:** `SearchIndex`, `SpamCheck`, `LanguageModel` and `Embeddings`, `PushTransport`, `PaymentProvider`, `Federation`, `InboundMail`, `JobQueue`, `EventStream`, `Importer`, `RankingPolicy`.
- **One adapter shipped per port first:** database full-text search, Akismet, an OpenAI-compatible model client, Web Push, WooCommerce, a database job queue, and a bbPress importer.

## Design Decisions

1. **Dedicated tables, not posts and postmeta.** Every mature forum stores typed, indexed counters and freshness on topic and forum rows. The WordPress forum plugins that compete with bbPress (wpForo, Asgaros, Simple:Press) also use custom tables (**secondary only**). WordPress.org's overrides show what postmeta costs.
   *Trade-off:* revisions, core REST, search and export must be rebuilt as adapters.
2. **Counters change in the same transaction as the write.** A Unit of Work commits the post and relative counter updates (`reply_count = reply_count + 1`) together. Rebuilding from source rows is a first-class command. bbPress patched concurrent overwrites with a retry loop in 2.6.17; that is the bug to avoid.
   *Trade-off:* hot topics serialise on their row.
3. **Read state by per-topic post number.** This is the Flarum and Discourse model: integer, monotonic, unread count by subtraction. It folds subscription level and bookmarks into the same row.
   *Trade-off:* posts need a stable per-topic number, so moving posts between topics renumbers them.
4. **Ranking is a Strategy over an indexed column.** `last_active_at` is the default sort. Hot and active ranks come from a scheduled job, with Lemmy's formula as the reference implementation.
   *Trade-off:* scheduled ranks lag behind live votes.
5. **Side effects come from domain events, not hook priorities.** bbPress's correctness depends on callbacks running in a fixed order. An Observer over committed events removes that coupling.
   *Trade-off:* extensions see events after commit and cannot veto them; vetoes need explicit policy ports.
6. **Private messages are a topic archetype.** Discourse does this, and it gives private messages read tracking, notifications and moderation for free. Misago's private threads (author plus invited users inside a forum) fall out of the same model.
   *Trade-off:* every topic query must filter on archetype.
7. **Spam checks form a Chain of Responsibility feeding one review queue.** Captcha, then a question-and-answer check, then Akismet, then an AI classifier. Flags, held posts and new-user posts all land in the same queue, as Discourse's reviewables do.
   *Trade-off:* the order of checks becomes configuration that sites can get wrong.
8. **AI sits behind a bring-your-own-model port.** Discourse's self-hosted model, not a metered service.
   *Trade-off:* sites must configure a provider before any AI feature works.
9. **Keep bbPress's semantics where sites depend on them.** URL shapes, private, hidden and password inheritance, and the keymaster-to-blocked role ladder. The 2026 security releases show these inheritance rules are where the bugs are, so they go into tests from the start.
   *Trade-off:* some of bbPress's quirks become contract.
10. **Replicas never coordinate.** Real-time delivery needs a shared broker or a database-backed event stream, never in-process subscriber lists, and counters are database updates, never cached increments.
    *Trade-off:* real-time needs infrastructure beyond one process.

## Implementation Phases

1. **The FOSS baseline.** Hierarchy, topics and posts with sticky, lock, move, split and merge; read tracking; subscriptions; in-app notifications; mentions; likes; private messages; polls; search; spam checks; the review queue; warnings, suspensions and bans; per-forum permissions; drafts and bookmarks; moderator logs; the bbPress importer.
2. **The modern bar.** Tags with subscriptions, trust levels and automatic promotion, slow mode, Q&A accepted answers, topic and post voting, the JSON API and webhooks, staff notes on users.
3. **Ports with their first adapter.** Real-time event stream, reply by email, AI, Web Push, paid upgrades through WooCommerce, automation rules.
4. **First-party extensions on the same ports.** Chat, events with RSVP, assign, canned replies, policy acceptance, a resource manager, a gallery on the media library, clubs as forum subtrees with their own membership.
5. **Federation**, once the licence question is settled.

Commercial features worth carrying into the open-source version, by phase: warning points with automatic sanctions (1), topic-type conversion and featured content (2), paid upgrades and web push (3), resource manager, gallery, clubs, digest emails and analytics with retention (4).

## Fleet Impact

- **wordpress-develop:** the forum would be new work; nothing in the fork depends on bbPress today. The fork has already removed WordPress's PHP Hook API in favour of the Rust port, which is the main reason a faithful bbPress port is costly.
- **speedyspec-server:** the forum's tables, job queue and event stream would follow the [database layer](../../speedyspec-server/explorations/2026-11-19-database-layer.md) and [native plugin features](../../speedyspec-server/explorations/2026-11-19-native-plugin-features.md) work. Per [ADR-003](../../adr/003-native-features-wait-for-api-ports.md), native features wait for the API ports they need. Unverified beyond that.
- **SEO:** forum URLs, canonicals and sitemaps would go through the [SEO API](2026-11-17-seo-api.md).

## Open Questions

1. **How fast is postmeta storage really, against dedicated tables?** No independent benchmark exists; the dev4press figures (about 30× at 100,000 posts) are the author's own. A reproducible benchmark would settle it.
2. **Can Lemmy's AGPL federation library be used?** A licence decision before federation work.
3. **Do bbPress's 22 converters still work against current phpBB, XenForo and vBulletin?** Test runs against current dumps.
4. **What do XenForo, Invision and vBulletin ship today?** Their sites refused direct fetches; read them in a browser.
5. **Does Flarum 2.0 support PostgreSQL and SQLite?** Its documentation, read directly.
6. **Should the forum be a WordPress core feature or a first-party module?** A decision for the project owner.

## Success Criteria

- [x] Every finding labelled by evidence quality — the notes tag repository, documentation, secondary and memory sources.
- [x] A feature inventory across open-source, other open-source and commercial software — four tiers in the report.
- [x] A recommendation among port, native and plugins, with evidence.
- [ ] A benchmark of postmeta against dedicated tables — not done.
- [ ] The commercial feature lists re-read from primary sources — still open.

---

## Non-Goals

- **Integrating with external forums.** Non-PHP forums were studied for features and design, not as targets.
- **Metered operations.** AI credits, SSO connection counts and retention tiers are business models, not features.
- **Chat as a core feature.** It belongs in an extension on the same ports.

## Alternatives Considered

### Option: Port bbPress faithfully
- **Pros:** Compatibility with about 100,000 sites, existing URLs and templates, and 22 importers.
- **Cons:** Inherits postmeta storage that WordPress.org works around; needs about 1,000 hook names with ordering and `$wp_filter` edits after the PHP Hook API was removed; global-state tricks clash with plain domain objects; no REST contract; one maintainer.
- **Verdict:** Rejected. A port that fixes storage stops being a port.

### Option: Leave forums to plugins
- **Pros:** Cheapest. Demand is modest, and Discourse already has a WordPress bridge ([WP Discourse](https://wordpress.org/plugins/wp-discourse/)).
- **Cons:** Gives up a forum that shares WordPress's users, roles, media, editor and commerce, which is the bundle commercial suites charge for.
- **Verdict:** Rejected.

### Option: Keep posts and postmeta but add indexed side tables
- **Pros:** Keeps WordPress's revisions, REST and export.
- **Cons:** Two sources of truth for every counter; dev4press proposed this for bbPress and it remains a proposal.
- **Verdict:** Rejected in favour of one schema.

## Dependencies

- The Rust database and options ports, and a job queue in the runtime.
- Read access to commercial vendor sites, Codeberg and bbPress Trac, which refused the fetcher during research.

## References

- Source: report "PHP forum software and bbPress"; notes `php_forum_landscape.md`, `other_open_source_forums.md`, `commercial_forum_features.md`, `bbpress_architecture.md`, `bbpress_status_and_scaling.md`.
- [bbPress source](https://github.com/bbpress/bbPress) (trunk `81fd8df`, tag 2.6.22)
- [WordPress.org support-forums performance class](https://github.com/WordPress/wordpress.org/blob/trunk/wordpress.org/public_html/wp-content/plugins/support-forums/inc/class-performance-optimizations.php)
- [dev4press: custom tables for bbPress](https://github.com/dev4press/db-updates-for-bbpress)
- [phpBB 3.0 schema migration](https://github.com/phpbb/phpbb/blob/release-3.3.19/phpBB/phpbb/db/migration/data/v30x/release_3_0_0.php)
- [Discourse `topic_users` model](https://github.com/discourse/discourse/blob/main/app/models/topic_user.rb)
- [Flarum 2.0 migrations](https://github.com/flarum/framework/tree/v2.0.2/framework/core/migrations)
- [Lemmy ranking algorithm](https://join-lemmy.org/docs/contributors/07-ranking-algo.html)
- [NodeBB FEP-1b12](https://docs.nodebb.org/activitypub/fep/1b12)
- [Discourse pricing](https://www.discourse.org/pricing)
- [Native plugin features exploration](../../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
