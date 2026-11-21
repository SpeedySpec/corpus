# PostgreSQL engine

`speedyspec-data-postgres`. From plan 6 (#17).

## Introspection tests flake, and connect errors say little

- **Where:** the introspection tests; connection errors.
- **What happens:**
  - the introspection tests can fail with "out of shared memory" after the corpus drops its schema;
  - connect errors read only "db error".
- **Suggested fix:** give the corpus tests their own schema per run; use `as_db_error` for the message.

## A hex literal loses its "unsupported" error

- **Where:** `SELECT 0x41`.
- **What happens:** the unsupported-construct error is lost inside `try_get`.
- **Suggested fix:** return the error from the translation instead.

## `SELECT ?` returns text and names the column `?1`

- **Where:** result shaping.
- **What happens:** `SELECT ?` bound to the integer 5 returns the text `"5"`, in a column named `?1`.
- **Suggested fix:** keep the bound type, and name the column as MySQL does.

## Seven translation differences from MySQL

- **Where:** the PostgreSQL passes.
- **What happens:**
  - `HAVING` without `GROUP BY`;
  - `SET a=a+1, b=a` uses the old `a` (MySQL uses the new one);
  - `'abc' = 0` raises error 22P02;
  - `||` is not read as `OR`;
  - `MODIFY varchar→int` lacks a `USING` clause;
  - `-1` is accepted into an unsigned bigint;
  - an enum keeps `'X'` under `speedyspec_ci`.
- **Suggested fix:** take each in turn; most need a pass or a `CHECK` constraint.

## A schema-change notification misses one cache

- **Where:** `reset()` on `NOTIFY`.
- **What happens:** it refreshes the schema cache but not the cache of where result columns come from.
- **Suggested fix:** clear both.

## `LOCATE('a')`'s error wording

- **Where:** the `LOCATE` translation.
- **What happens:** the one-argument error message speaks of a start position the call did not give.
- **Suggested fix:** reword it.
