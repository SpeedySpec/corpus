# Architecture decision records

Decisions about SpeedySpec's design, one per file, so the reasons outlive the conversations they were made in.

## Layout

```
knowledge/00_platform/adr/
├── README.md              This file, and the index
└── NNNN-<title>.md        One decision per file, numbered in the order recorded
```

## Each record

- **Status:** Proposed, Accepted, or Superseded by a later record or design (named), in whole or in part. A record is never edited to change its
  decision: a new record supersedes it.
- **Decided by:** the project owner, or the designer when the owner left the call to the design.
- **Context:** what forced the decision.
- **Decision:** what was decided.
- **Alternatives considered:** what else was on the table, and why it lost.
- **Consequences:** what follows, good and bad.
- **Patterns:** the design patterns the decision uses and where. Every decision draws on Clean Code, Clean
  Architecture, Hexagonal Architecture, Fowler's Patterns of Enterprise Application Architecture and the Gang of Four.

## Index

Records 0001 to 0013 come from the native plugin features design of 2026-11-19, which makes the work of popular
WordPress plugins part of SpeedySpec's Rust port of WordPress.

| # | Decision | Status |
|---|---|---|
| [0001](0001-plugin-features-are-behaviour-of-ported-apis.md) | Plugin features are behaviour of the ported WordPress APIs | Accepted |
| [0002](0002-native-features-run-in-the-runtime.md) | Native features run in the runtime's Rust side, before PHP | Accepted |
| [0003](0003-native-features-wait-for-api-ports.md) | Native features wait for proper Rust ports of the WordPress APIs | Accepted |
| [0004](0004-runtime-request-stages.md) | The runtime answers requests in four stages before PHP | Accepted |
| [0005](0005-core-hooks-stay.md) | WordPress core's hooks stay; new hooks only for what core lacks | Accepted |
| [0006](0006-php-callback-registry-at-worker-start.md) | A registry of PHP callbacks, built when a worker starts | Accepted |
| [0007](0007-shared-stores-file-redis-database.md) | Shared stores have file, Redis and database implementations | Accepted |
| [0008](0008-background-job-runner-in-the-runtime.md) | A background job runner inside the runtime | Accepted |
| [0009](0009-database-port-requirements.md) | The database port: engines, a raw-SQL builder, a unit of work, plain objects | Accepted |
| [0010](0010-firewall-and-login-security.md) | The firewall runs first; login protection lives in the auth API | Accepted |
| [0011](0011-page-cache-coarse-purge-first.md) | The page cache purges everything first, by tag later | Accepted |
| [0012](0012-redirects-sitemaps-robots-feeds.md) | Redirects, sitemaps, `robots.txt` and feeds are answered in Rust | Accepted |
| [0013](0013-images-in-rust-on-the-job-runner.md) | Images are made by a Rust editor backend, on the job runner | Accepted |

Records 0014 to 0026 come from the PHP foundation design of 2026-11-18, which set out how WordPress is ported to Rust
with PHP embedded for what remains, and how PHP apps run.

| # | Decision | Status |
|---|---|---|
| [0014](0014-port-wordpress-to-rust-and-delete-its-php.md) | WordPress is ported to Rust, and each ported area's PHP is deleted | Accepted |
| [0015](0015-ports-and-adapters-with-cqrs-use-cases.md) | Ports and Adapters, with every WordPress operation a CQRS use case | Accepted; crate naming superseded in part |
| [0016](0016-rust-runs-as-an-embedded-php-extension.md) | Rust runs as a PHP extension compiled into the embedded PHP | Accepted |
| [0017](0017-php-twins-and-the-override-flag.md) | Every port has a PHP twin, and binding it to PHP is the override | Accepted |
| [0018](0018-a-service-container-in-rust.md) | A service container in Rust, with Laravel's container's functionality | Accepted |
| [0019](0019-command-and-query-buses.md) | Command and query buses, with a direct path for hot operations | Accepted |
| [0020](0020-rust-owns-hook-state.md) | Rust owns hook state; WordPress's hook globals are proxies | Accepted |
| [0021](0021-recorded-compatibility.md) | Recorded compatibility proves a port before WordPress's code is deleted | Accepted |
| [0022](0022-fastcgi-over-a-unix-socket.md) | The server talks to PHP workers over FastCGI on a Unix socket | Accepted |
| [0023](0023-shared-crates-in-the-server-workspace.md) | The engine crates live in the server workspace and are copied into the CLI | Accepted |
| [0024](0024-static-php-with-every-wordpress-extension.md) | Static PHP 8.5 with every extension WordPress uses | Accepted |
| [0025](0025-supervised-php-worker-pools.md) | PHP runs in supervised worker pools, one request per worker | Accepted; where the pool runs superseded in part |
| [0026](0026-a-writable-site-directory.md) | Each PHP app has a writable site directory over its read-only releases | Accepted |
