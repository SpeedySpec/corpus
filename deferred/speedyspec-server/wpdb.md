# wpdb

`speedyspec-wp-db`, `speedyspec-wp-db-php`. From plan 2 (#13).

## Internal byte markers collide with real characters

- **Where:** `speedyspec-wp-db`'s `binary.rs`.
- **What happens:** bytes that are not UTF-8 are carried as markers in the range U+10FF00 to U+10FFFF. A query that
  contains both such bytes and real characters from that range is read back wrongly.
- **Suggested fix:** carry raw bytes alongside the text instead of in-band markers.

## `get_row(…, OBJECT_K)` returns the row

- **Where:** `get_row`.
- **What happens:** WordPress prints an output-type error and returns null for `OBJECT_K`; the port returns the row.
- **Suggested fix:** follow WordPress.

## The filtered `query_statement` path skips checks

- **Where:** `query_statement`.
- **What happens:** on the path where the `query` filter changed the text, the result shape is chosen from the
  statement's kind, the invalid-text check is skipped, and a parse failure is not counted.
- **Suggested fix:** run the same checks and counting on both paths.

## `SET NAMES … COLLATE` puts the collation in unquoted

- **Where:** the charset setter.
- **What happens:** a collation a developer set, containing quotes or spaces, is spliced into the SQL raw.
- **Suggested fix:** validate the collation name, or quote it.

## Left-justified, zero-padded `%F` pads with spaces

- **Where:** `prepare()`'s format handling.
- **What happens:** PHP pads with zeros; the port pads with spaces.
- **Suggested fix:** pad with zeros, as PHP does.

## `prepare()` notices are not translated

- **Where:** `prepare()`.
- **What happens:** its notices skip `__()`, so they are always in English.
- **Suggested fix:** pass the messages through the translation call.

## Identifier escaping uses positions, not the args array's keys

- **Where:** `prepare()`'s `%i` handling.
- **What happens:** WordPress keys identifiers by the args array's keys; the port uses positions, which differ when the
  array has gaps.
- **Suggested fix:** use the keys.

## The placeholder escape lasts per worker, not per request

- **Where:** `speedyspec-wp-db`.
- **What happens:** WordPress makes a fresh placeholder escape string each request; the port keeps one per worker.
- **Suggested fix:** make it at request start.

## Hot-path costs on every request

- **Where:** `speedyspec-wp-db-php`.
- **What happens:**
  - every access to `wpdb`'s state resolves it from the container;
  - `SAVEQUERIES` costs two PHP calls per statement;
  - the facade is evaluated on every request.
- **Suggested fix:** profile first; cache the state handle and the facade per request.

## `__get` of an unknown property raises the wrong warning level

- **Where:** the native `wpdb` class.
- **What happens:** it raises `E_USER_WARNING`; PHP raises `E_WARNING`.
- **Suggested fix:** raise `E_WARNING`.

## One PHPUnit warning in WordPress's database suite is not investigated

- **Where:** the WordPress database tests run against the native `wpdb`.
- **What happens:** the run passes, but one test reports a warning (`W`) whose cause nobody has looked at.
- **Suggested fix:** find the test and decide whether the port should behave differently.
