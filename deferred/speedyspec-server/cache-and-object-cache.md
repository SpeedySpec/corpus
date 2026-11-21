# Caching store and object cache

`speedyspec-cache-*`, `speedyspec-php-serialize`, `speedyspec-wp-object-cache`, `speedyspec-wp-object-cache-php`. From
plan 3 (#14).

## A literal `sha256:<hex>` key collides with a hashed long key

- **Where:** `speedyspec-cache-sql`'s `stored_key`.
- **What happens:** keys longer than 191 bytes are stored as `sha256:<hex>`. A short key that is literally
  `sha256:<that hex>` lands on the same row.
- **Suggested fix:** mark hashed keys in a way no literal key can take (for example, a prefix byte that is not valid
  UTF-8).

## PHP's `r:` references are not tracked properly

- **Where:** `speedyspec-php-serialize`'s `convert.rs`.
- **What happens:** the converter numbers no slot for `r:`, so a later `r:n` in Rust-only conversion can resolve to
  the wrong object, and materialising loses the outer `R:`/`r:` context.
- **Suggested fix:** number every value slot as PHP does.

## Keys or groups that are not UTF-8 throw

- **Where:** the object cache's key handling.
- **What happens:** WordPress accepts any string as a key or group; the port throws.
- **Suggested fix:** accept bytes, or list the difference in `breaks.json`.

## `get_multiple` with `force` keeps a stale value

- **Where:** `wp_cache_get_multiple`.
- **What happens:** with `force`, a key the second level no longer has keeps its stale first-level value instead of
  reading as missing.
- **Suggested fix:** clear the first-level entry when the forced read misses.

## A first-level entry that fails to decode is read twice

- **Where:** the object cache's first level.
- **What happens:** the bad entry is read, and reported, twice.
- **Suggested fix:** drop it after the first failed decode.

## `decr` with `PHP_INT_MIN` is off by one

- **Where:** `decr`.
- **What happens:** the offset maps to `i64::MAX`, one less than its magnitude.
- **Suggested fix:** saturate after negating, as PHP's arithmetic does.

## `incr` misses some of PHP's casts

- **Where:** `incr`'s offset cast.
- **What happens:** `"1e3"` and out-of-range floats are not cast as PHP casts them.
- **Suggested fix:** use the shared `php_int` cast.

## A cloned `$wp_object_cache` returns false everywhere

- **Where:** the native `WP_Object_Cache` class.
- **What happens:** `clone $wp_object_cache` gives a copy with no registry entry, so all its methods return false.
- **Suggested fix:** register the clone on `__clone`.

## Destroyed `WP_Object_Cache` instances keep their values

- **Where:** the native class.
- **What happens:** values stay in memory until the request ends, after the object is gone.
- **Suggested fix:** free the instance's state in its destructor.

## `MemoryCache` grows on reads of unknown groups

- **Where:** `speedyspec-cache-memory`.
- **What happens:** every read of an unknown group inserts an empty group map, and nothing evicts it.
- **Suggested fix:** look groups up without inserting.

## Nothing schedules `SqlCache::purge_expired`

- **Where:** `speedyspec-cache-sql`; the runtime.
- **What happens:** expired rows stay in the cache table until they are read or overwritten.
- **Suggested fix:** call it from the background job runner, as the runtime hand-over asks for `FileCache`.
