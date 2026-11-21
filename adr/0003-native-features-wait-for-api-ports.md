# 0003. Native features wait for proper Rust ports of the WordPress APIs

- **Status:** Accepted, 2026-11-19
- **Decided by:** the project owner

## Context

The native features need WordPress data: options, permalinks, posts, terms, users, attachments. When this was decided,
only the Hook API was ported (`speedyspec-wp-hooks`), and database access was still PHP's `mysqli`.

## Decision

Port the WordPress APIs to Rust first, then build each feature once the APIs it needs exist:

| Step | Port | Unblocks |
|---|---|---|
| 1 | Database layer, options | Firewall (request rules, login protection) |
| 2 | Permalinks and rewrite | Page cache; redirects |
| 3 | Attachments, background job runner (cron) | Image optimization |
| 4 | Posts, a `WP_Query` subset, terms, users | Sitemaps, feeds, `robots.txt` contributions |

The ports are owned by other work, most likely work devoted to porting. The features' design only says what each port
must include so the features fit.

## Alternatives considered

- **A. A Rust read layer over WordPress's tables, per feature.** Fast, but ties Rust to the table layout and misses
  anything plugins filter in PHP.
- **C. Call into PHP for data.** Runs PHP again, which defeats the purpose.

## Consequences

- Features arrive in the order of the table.
- Ports follow the Hook API port's pattern: commands and queries on the CQRS buses, PHP twins through `#[php_port]`,
  and recorded compatibility (`speedyspec-compat`) against WordPress's real outputs.
- Step 1 landed in speedyspec-server #12 to #19, and the firewall followed in #20.

## Patterns

- **Repository** and **Data Mapper** behind each port, so features never see tables.
- **Command Query Responsibility Segregation:** the ports' commands and queries go over the buses.
