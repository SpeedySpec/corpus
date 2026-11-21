# 0009. The database port: engines, a raw-SQL builder, a unit of work, plain objects

- **Status:** Accepted, 2026-11-19
- **Decided by:** the project owner; the port's own design settles how

## Context

Step 1 of 0003 ports the database layer. WordPress plugins send raw MySQL-flavoured SQL through `$wpdb`, and the
project wants more databases than MySQL.

## Decision

- **Several databases behind repository contracts:** MySQL, PostgreSQL, SQLite and ReDB, with room for others.
- **Engines, in the manner of SQLAlchemy:** each database is reached through an engine that knows its dialect and can
  apply engine-specific optimizations. Several databases can be used at once.
- **A query builder in the manner of Eloquent's** that also understands raw SQL: raw SQL can be parsed into a builder
  instance and changed with partials (adding or replacing clauses and fragments).
- **A unit of work** that groups changes and commits them together.
- **Plain objects, not Active Record:** entities are plain PHP objects and plain Rust structs that know nothing about
  storage. Data Mappers and Repositories load and save them. "In the manner of Eloquent" refers to the builder's style
  only, not to Eloquent's models.

## Consequences

- Parsing raw SQL into the builder is what lets `$wpdb`'s MySQL run on PostgreSQL or SQLite: it is re-emitted in the
  target engine's dialect.
- ReDB has no SQL. Raw SQL and the builder can only target the SQL engines; ReDB is reached through repository
  implementations written for it.
- Built as speedyspec-server #12 to #19.

## Patterns

- **Data Mapper** and **Repository** for persistence; never **Active Record**.
- **Unit of Work** for commits.
- **Builder** for queries.
- **Strategy** for engine dialects.
