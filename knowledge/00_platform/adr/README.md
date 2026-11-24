# Architecture decision records

Decisions about SpeedySpec's design, one per file, so the reasons outlive the conversations they were made in.

## Layout

```
knowledge/00_platform/adr/
├── README.md              This file, and the index
└── NNN-<title>.md         One decision per file, numbered in the order recorded
```

## Each record

The header names the status, the decision date, who decided, the exploration the decision came from, and related
records. A record is never edited to change its decision: a later record supersedes it, in whole or in part, and the
status says so.

The sections, in order:

- **Context**, with **Why this matters**: the pressure behind the decision, and the cost of getting it wrong.
- **Decision**: the decision in a sentence or two, then:
  - **Rules**: the numbered consequences a reviewer checks against;
  - **What this looks like**: a correct example and one that breaks a rule;
  - **Patterns**: the design patterns it uses. Every decision draws on Clean Code, Clean Architecture, Hexagonal
    Architecture, Fowler's Patterns of Enterprise Application Architecture and the Gang of Four.
- **Consequences**: positive, negative (each with its mitigation), and the trade-offs.
- **Fleet impact**: which repositories, binaries and apps the decision constrains.
- **Enforcement**: the automated check, if one exists, and the review checklist. A rule no tool checks says so.
- **Exceptions**: when the rule may be broken, and who decides.
- **Revision history**: dated changes, including when the decision was built.

## Index

ADR-001 to ADR-013 come from the native plugin features design of 2026-11-19, which makes the work of popular
WordPress plugins part of SpeedySpec's Rust port of WordPress.

| # | Decision | Status |
|---|---|---|
| [ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md) | Plugin features are behaviour of the ported WordPress APIs | Accepted |
| [ADR-002](002-native-features-run-in-the-runtime.md) | Native features run in the runtime's Rust side, before PHP | Accepted |
| [ADR-003](003-native-features-wait-for-api-ports.md) | Native features wait for proper Rust ports of the WordPress APIs | Accepted |
| [ADR-004](004-runtime-request-stages.md) | The runtime answers requests in four stages before PHP | Accepted |
| [ADR-005](005-core-hooks-stay.md) | WordPress core's hooks stay; new hooks only for what core lacks | Accepted |
| [ADR-006](006-php-callback-registry-at-worker-start.md) | A registry of PHP callbacks, built when a worker starts | Accepted |
| [ADR-007](007-shared-stores-file-redis-database.md) | Shared stores have file, Redis and database implementations | Accepted |
| [ADR-008](008-background-job-runner-in-the-runtime.md) | A background job runner inside the runtime | Accepted |
| [ADR-009](009-database-port-requirements.md) | The database port: engines, a raw-SQL builder, a unit of work, plain objects | Accepted |
| [ADR-010](010-firewall-and-login-security.md) | The firewall runs first; login protection lives in the auth API | Accepted |
| [ADR-011](011-page-cache-coarse-purge-first.md) | The page cache purges everything first, by tag later | Accepted |
| [ADR-012](012-redirects-sitemaps-robots-feeds.md) | Redirects, sitemaps, `robots.txt` and feeds are answered in Rust | Accepted |
| [ADR-013](013-images-in-rust-on-the-job-runner.md) | Images are made by a Rust editor backend, on the job runner | Accepted |

ADR-014 to ADR-026 come from the PHP foundation design of 2026-11-18, which set out how WordPress is ported to Rust
with PHP embedded for what remains, and how PHP apps run.

| # | Decision | Status |
|---|---|---|
| [ADR-014](014-port-wordpress-to-rust-and-delete-its-php.md) | WordPress is ported to Rust, and each ported area's PHP is deleted | Accepted |
| [ADR-015](015-ports-and-adapters-with-cqrs-use-cases.md) | Ports and Adapters, with every WordPress operation a CQRS use case | Accepted; crate naming superseded in part |
| [ADR-016](016-rust-runs-as-an-embedded-php-extension.md) | Rust runs as a PHP extension compiled into the embedded PHP | Accepted |
| [ADR-017](017-php-twins-and-the-override-flag.md) | Every port has a PHP twin, and binding it to PHP is the override | Accepted |
| [ADR-018](018-a-service-container-in-rust.md) | A service container in Rust, with Laravel's container's functionality | Accepted |
| [ADR-019](019-command-and-query-buses.md) | Command and query buses, with a direct path for hot operations | Accepted |
| [ADR-020](020-rust-owns-hook-state.md) | Rust owns hook state; WordPress's hook globals are proxies | Accepted |
| [ADR-021](021-recorded-compatibility.md) | Recorded compatibility proves a port before WordPress's code is deleted | Accepted |
| [ADR-022](022-fastcgi-over-a-unix-socket.md) | The server talks to PHP workers over FastCGI on a Unix socket | Accepted |
| [ADR-023](023-shared-crates-in-the-server-workspace.md) | The engine crates live in the server workspace and are copied into the CLI | Accepted |
| [ADR-024](024-static-php-with-every-wordpress-extension.md) | Static PHP 8.5 with every extension WordPress uses | Accepted |
| [ADR-025](025-supervised-php-worker-pools.md) | PHP runs in supervised worker pools, one request per worker | Accepted; where the pool runs superseded in part |
| [ADR-026](026-a-writable-site-directory.md) | Each PHP app has a writable site directory over its read-only releases | Accepted |

ADR-027 comes from the SpeedySpec Guide design of 2026-11-20, and the decisions of 2026-11-21 on doc attributes and
generated Rust docs.

| # | Decision | Status |
|---|---|---|
| [ADR-027](027-the-guide-is-not-built-from-docblocks.md) | The guide is written from behaviour and code, never from docblocks | Accepted |

ADR-028 comes from the SpeedySpec versions discussion of 2026-11-23, which set what each SpeedySpec major promises
plugins.

| # | Decision | Status |
|---|---|---|
| [ADR-028](028-speedyspec-versions.md) | SpeedySpec 1 stays WordPress; later majors replace it | Accepted |
