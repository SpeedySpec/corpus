# NoSQL and direct stores

`speedyspec-data-valkey`, `speedyspec-data-redb`, `speedyspec-data-file`, `speedyspec-cache-file`. From plan 7 (#18).

## A huge Valkey TTL wraps

- **Where:** `speedyspec-data-valkey`'s TTL conversion.
- **What happens:** it casts with `as u64`, so a huge TTL wraps around or fails with "not-integer". `KvCache` caps
  TTLs at 100 years, so only direct callers see it.
- **Suggested fix:** saturate the conversion.

## Valkey `timeout_ms=0`, and terse connection errors

- **Where:** `speedyspec-data-valkey`.
- **What happens:** `timeout_ms=0` fails every call, and connection errors show only the I/O kind ("IoError").
- **Suggested fix:** read 0 as "no timeout" or refuse it at configure; include the error's message.

## A ReDB answer over 64 MiB is run twice

- **Where:** `speedyspec-data-redb`'s client and owner.
- **What happens:** an owner answer over 64 MiB closes the connection; the client sees a connection error and has run
  the request twice.
- **Suggested fix:** answer with an error the client does not retry.

## The ReDB client keeps a dead connection

- **Where:** `RedbClient`.
- **What happens:** after an owner "protocol" error, the owner hangs up, but the client keeps the connection and fails
  the next call.
- **Suggested fix:** drop the connection on a protocol error.

## Crash leftovers of the file store are never cleaned

- **Where:** `speedyspec-data-file`.
- **What happens:** `.tmp-*` files left by a crash mid-write stay forever. (`FileCache::flush` already removes leftover
  `.trash-*` directories.)
- **Suggested fix:** remove stale `.tmp-*` files in `FileCache::purge_expired`'s sweep.
