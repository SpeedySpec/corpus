# Persistent object cache, transients and network options

`speedyspec-wp-options*`, `speedyspec-wp-object-cache`. From plan 8 (#19).

## Arrays and objects cast to 0

- **Where:** `speedyspec-wp-options`'s `php_int`.
- **What happens:** PHP's `(int)` gives 1 for a non-empty array or an object; the port gives 0. So
  `set_transient(…, ['x'])` gets no timeout and is autoloaded, where WordPress expires it after one second.
- **Suggested fix:** a non-empty array gives 1, and an object 1 after reading it.

## Transient names that are too long leave an orphan row

- **Where:** `set_transient`.
- **What happens:** as in recorded WordPress, it returns true for a name too long for the column. The 199-byte timeout
  name is stored truncated to 191 bytes, so the timeout row is an orphan and the value never expires. The port matches
  WordPress on purpose.
- **Suggested fix:** none in the port unless WordPress changes; consider a warning.

## `prepare_release` shares one temporary copy

- **Where:** `speedyspec-wp-options-php`'s `release.rs` (tests only).
- **What happens:** two worktrees with different `PORTED_FUNCTIONS` lists keep overwriting the same prepared
  `option.php`, so a PHP process in one can load the other's file.
- **Suggested fix:** key the output directory by a hash of the list.

## Rust adapters sit beside their contracts

- **Where:** `ObjectCachePersistence`, `NoArgsParsing` and `SystemClock` in `speedyspec-wp-options`.
- **What happens:** the adapters live in the same crate as the contracts, against "contracts in their own package".
  Plan 4's `adapters.rs` set the same pattern.
- **Suggested fix:** move the adapters to an adapter crate when the contracts move.

## One multisite test reads the first level

- **Where:** `site_transient_is_global_across_blogs`.
- **What happens:** it reads from the same cache's first level, not from a new request through the second level, so
  it does not prove the site transient is shared.
- **Suggested fix:** end the request and read again, as the cross-process test does.

## Sites sharing one Valkey need their own prefix

- **Where:** the runtime's Valkey route.
- **What happens:** the default prefix `speedyspec:` makes two sites on one Valkey share keys, `alloptions` included.
  The runtime hand-over note says so; nothing enforces it.
- **Suggested fix:** default the prefix to something per site, or refuse two sites with one prefix.

## A failed write during a Valkey blip can be served stale

- **Where:** the object cache's second level.
- **What happens:** a write of `notoptions` or `alloptions` that fails during a blip leaves the old value in Valkey,
  served once it returns. Persistent drop-ins share this weakness.
- **Suggested fix:** after a failed write, write a short TTL on those keys when the store is back.
