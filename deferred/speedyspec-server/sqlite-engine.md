# SQLite engine

`speedyspec-data-sqlite`, `speedyspec-data-translate`. From plan 5 (#16).

## Insert ids differ from MySQL

- **Where:** `sqlite_dml`.
- **What happens:**
  - `INSERT … SELECT` into an auto-increment table reports no insert id;
  - `LAST_INSERT_ID()` is `last_insert_rowid()`: the last row of a multi-row insert, and it moves for inserts into
    tables without `AUTO_INCREMENT` and into temporary tables;
  - a multi-row upsert whose first row updates reports that row's id.
- **Suggested fix:** track the first generated id of each insert, as MySQL reports it.

## A SQL `ROLLBACK` does not clear the schema cache

- **Where:** `SqliteStore`.
- **What happens:** `rollback()` clears it, but a `ROLLBACK` statement does not. SQLite also rolls back
  temporary-table DDL, which MySQL does not.
- **Suggested fix:** clear the cache on the statement too.

## `SHOW CREATE TABLE` and `SHOW COLUMNS` of a missing table

- **Where:** the emulated `SHOW` statements.
- **What happens:** they return an empty answer instead of MySQL's error 1146.
- **Suggested fix:** return 1146.

## Every `information_schema` query counts every table

- **Where:** `info_schema::refresh`.
- **What happens:** each refresh runs `COUNT(*)` on every table.
- **Suggested fix:** compute row counts only when `TABLE_ROWS` is asked for, or estimate them.

## Two error-handling slips at open

- **Where:** `SqliteStore::open`.
- **What happens:**
  - `open_connection` maps every error to `Connection`, so the `SQLITE_CANTOPEN` mapping in `open` never runs;
  - WAL mode is set before the busy timeout, so first opens racing on a new file can fail with `SQLITE_BUSY`.
- **Suggested fix:** keep the engine error's code, and set the busy timeout first.

## Type and trigger differences from MySQL

- **Where:** the SQLite type and trigger translation.
- **What happens:**
  - `DECIMAL` has REAL affinity, losing precision past 2^53 and on decimal fractions;
  - `option_id = '1abc'` matches nothing, where MySQL matches 1;
  - the `ON UPDATE` trigger fires when a column is set to its current value;
  - `RENAME TABLE` keeps trigger names, so re-creating the old name with an `ON UPDATE` column can fail.
- **Suggested fix:** take each in turn; the trigger names are the easiest (rename them with the table).
