# Options API

`speedyspec-wp-options`, `speedyspec-wp-options-memory`, `speedyspec-wp-options-sql`, `speedyspec-wp-options-php`. From
plan 4 (#15).

## Protected options when `wp_die()` returns

- **Where:** `update_option`/`delete_option` of a protected name.
- **What happens:** when a `wp_die` handler returns instead of exiting, WordPress carries on and writes; the port
  throws and writes nothing.
- **Suggested fix:** follow WordPress, or list the difference in `breaks.json` with the owner's agreement.

## `delete_option(['x'])` returns false

- **Where:** `delete_option`.
- **What happens:** WordPress's `prepare` flattens the array and deletes `x`; the port returns false with no hook.
- **Suggested fix:** follow WordPress.

## Option names that are not UTF-8 become U+FFFD

- **Where:** `OptionName` and the autoload policy (`from_utf8_lossy`).
- **What happens:** two names differing only in invalid bytes are treated as one.
- **Suggested fix:** carry names as bytes.

## `delete_option`'s `SELECT` has `LIMIT 1`

- **Where:** `speedyspec-wp-options-sql`.
- **What happens:** WordPress's query has no `LIMIT`, so a `query` filter matching WordPress's exact text does not
  match.
- **Suggested fix:** drop the `LIMIT`.

## The autoload-change comparison approximates MySQL's collation

- **Where:** `needing_autoload_change`.
- **What happens:** it approximates `utf8mb4_unicode_520_ci` with no PAD SPACE and ASCII-only case folding, so some
  names compare differently than in MySQL.
- **Suggested fix:** compare as the collation does, or let the database compare.

## After-commit callbacks run after PHP exits

- **Where:** the options unit of work.
- **What happens:** after `OptionsError::Php(Null)` (PHP unwinding from `exit`), after-commit callbacks keep running.
- **Suggested fix:** stop running callbacks once PHP is unwinding.

## Wrong error classes in two cases

- **Where:** the PHP surface.
- **What happens:**
  - a non-array from `wp_autoload_values_to_autoload` throws `Error`, not `TypeError`;
  - a PHP exception inside an `OptionsError::Cache` is rethrown as a generic `Error`.
- **Suggested fix:** throw `TypeError`, and rethrow the original exception.

## Deprecated-argument hooks missing from the list a read fires

- **Where:** `hooks_fired_by_get`.
- **What happens:** for deprecated option names, `deprecated_argument_run` and `deprecated_argument_trigger_error`
  are missing from the list.
- **Suggested fix:** add them.

## `speedyspec-wp-options` depends on concrete classes

- **Where:** `OptionsDeps`.
- **What happens:** it takes `DefaultUnitOfWork` and `ObjectCache` directly, which the plan fixed, against the
  ports-and-adapters rule.
- **Suggested fix:** take `&dyn UnitOfWork` and an object-cache port.

## `alloptions` lookups scan linearly

- **Where:** the `alloptions` array.
- **What happens:** each lookup scans the array; large sites pay for it on every option read.
- **Suggested fix:** index by name.
