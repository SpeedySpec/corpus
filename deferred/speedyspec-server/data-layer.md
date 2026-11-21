# Data layer

`speedyspec-data`, `speedyspec-data-query`, `speedyspec-data-uow`, `speedyspec-data-mysql`. From plan 1 (#12).

## A bare `?` inside a partial is not caught at build time

- **Where:** the query builder's partials (`speedyspec-data-query`).
- **What happens:** a `?` placeholder written inside a partial is not reported when the query is built. The mismatch
  surfaces later, when the statement runs.
- **Suggested fix:** count placeholders in partials at `build()`, as the builder does for its own clauses.

## A `?` inside `MATCH … AGAINST(?)` is not seen

- **Where:** the expression visitor in `speedyspec-data-query`.
- **What happens:** the visitor does not count a placeholder inside `MATCH … AGAINST(?)`, so `from_statement` reports a
  placeholder-count mismatch for a valid full-text query.
- **Suggested fix:** visit the `AGAINST` argument as an expression.

## `select(&["*"])`, `"t.*"` and `"COUNT(*)"` are quoted as identifiers

- **Where:** the builder's `select`.
- **What happens:** each name is quoted as a column, so `*`, `t.*` and `COUNT(*)` become invalid SQL.
- **Suggested fix:** pass them through, or point callers at `Partial::projection`, which already handles them.

## Empty builders build invalid SQL

- **Where:** the builder's `build()`.
- **What happens:** an insert with no values, or an update with no `SET`, builds SQL the database refuses, instead of
  a configuration error naming the mistake.
- **Suggested fix:** return a `Config` error from `build()` for both.

## A failed `SET NAMES` at connect leaves the connection half set up

- **Where:** `speedyspec-data-mysql`, connection setup.
- **What happens:** if the initial `SET NAMES` fails for a reason other than a lost connection, the connection stays
  open but uninitialised, with the wrong character set.
- **Suggested fix:** close the connection and report the error.

## A connection lost mid-transaction reconnects in autocommit

- **Where:** `speedyspec-data-mysql`.
- **What happens:** the reconnected session is in autocommit, so the rest of the caller's transaction runs statement by
  statement. This matches WordPress's `wpdb::check_connection`, which is why it was deferred.
- **Suggested fix:** report the lost transaction instead of reconnecting silently.

## The unit of work's refusal names only one transaction

- **Where:** `speedyspec-data-uow`.
- **What happens:** when the unit of work refuses to commit across engines, its message names the engine but not both
  transactions involved (the design asks for both).
- **Suggested fix:** include both transactions in the error.

## `DbValue` lacks common `From` conversions

- **Where:** `speedyspec-data`'s `DbValue`.
- **What happens:** there is no `From<i32>`, `From<u32>` or `From<bool>`, so callers write casts by hand.
- **Suggested fix:** add the three conversions.

## The design's `Connection` and `DataMapper` contracts were never built

- **Where:** `speedyspec-data`.
- **What happens:** the database design names a `Connection` contract and a `DataMapper` contract, but no plan built
  them. Code that would use them works around their absence.
- **Suggested fix:** decide whether they are still wanted; if so, add them in their own change.
