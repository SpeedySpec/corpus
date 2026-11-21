# NoSQL and Direct Stores Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The NoSQL and Direct Store side of the data layer: shared conformance suites, in-memory stores, Valkey, ReDB
behind one owner process per site, a directory-of-files Direct Store, and the `KvCache` and `FileCache` Caching
Stores on top of them.

**Architecture:** Each store is an Adapter behind plan 1's `NoSqlStore` or `DirectStore`, made by its own
`EngineFactory` (scheme `memory`, `valkey`, `redb`, `file`). ReDB locks its file to one process, so
`speedyspec-data-redb` holds both an owner (a library entry point the runtime will supervise) that opens the file and
serves a framed protocol on a private Unix socket, and `RedbClient`, a Remote Proxy that implements `NoSqlStore` over
that socket for every worker. `KvCache` (any `NoSqlStore`) and `FileCache` (the file Direct Store) are Strategies
behind plan 3's `CachingStore`.

**Tech Stack:** Rust 1.99, edition 2024; `redis` (latest release, blocking API only, no async features) against
Valkey; `redb` (latest release); `serde` + `postcard` (`use-std`) for the owner protocol; `sha2` for file names;
`libc` (`flock`); `tempfile` (dev). Hand-written `Display` for errors, as plan 1 does.

**Spec:** `docs/superpowers/specs/2026-11-19-database-layer-design.md` ("Store contracts", "The Caching Store", "ReDB
across processes", "Order of work" step 5), `docs/superpowers/specs/2026-11-19-native-plugin-features-design.md`
("Shared state": read "Redis" as Valkey, the project owner's correction). Plan index:
`docs/superpowers/plans/2026-11-19-data-and-options-ports-index.md`. Runtime context (read only):
`docs/superpowers/specs/2026-11-19-speedyspec-php-runtime-and-admin-gate-design.md`.

## Global Constraints

- **`NoSqlStore::put_if_absent`** (in plan 1's contract) is implemented by every store here: memory (entry check),
  Valkey (`SET key value NX [PX ms]`), ReDB (inside one write transaction), and the owner protocol (a `PutIfAbsent`
  request). `nosql_store_suite` checks it: first call true, second false, value unchanged; with a TTL, true again
  after expiry.

- Work in a new worktree of `SpeedySpec/speedyspec-server`, branch `feat/nosql-and-direct-stores`, from `origin/main`
  once plans 1 and 3 are merged (otherwise from plan 3's merged branch). New crates under `crates/`; no
  `Cargo.toml` workspace edit. Never touch `packages/speedyspec-server-runtime-gate/` or `packages/speedyspec-server/`.
- **Do not edit `crates/speedyspec-php-runtime`.** Supervising the ReDB owner is handed to the runtime's owner (Task 9).
- Each crate's `Cargo.toml`: `edition.workspace = true`, `rust-version.workspace = true`, `license.workspace = true`,
  version `0.1.0`, a one-line `description`.
- Consume plan 1's contracts exactly as written (`NoSqlStore`, `DirectStore`, `Health`, `Capabilities`,
  `EngineHandle`, `EngineKind`, `EngineConfig`, `Engine`, `EngineFactory`, `EngineRegistry`, `DbError`). Add nothing
  to `speedyspec-data`; a gap found is reported, not patched here.
- All access is blocking. Stores are single-threaded (`Rc`, `RefCell`) except the ReDB owner, which is the one place
  threads are used.
- Every NoSQL store's `capabilities()`: `transactions: false, savepoints: false, returning: false, upsert:
  UpsertForm::None, expiry: true, atomic_increment: true`; `ordered_ranges` is `true` for memory and ReDB, `false` for
  Valkey.
- Shared NoSQL rules, pinned by the suite so every engine behaves as Valkey does:
  - `put` with `ttl: Some(Duration::ZERO)` deletes the key; a TTL under 1 ms rounds up to 1 ms.
  - Counters are stored as canonical decimal ASCII (`b"15"`). `increment` on a missing (or expired) key starts from 0;
    on a value that is not canonical decimal (`"abc"`, `"05"`, `"+5"`, `" 5"`, `""`) or on overflow it fails with
    `DbError::Engine { code: Some("not-integer" | "overflow"), .. }`. An increment keeps the key's expiry.
  - An engine without ordered ranges answers `range` with `DbError::Engine { code: Some("unsupported"), .. }`
    (`DbError::Unsupported` names a SQL dialect, so it does not fit).
  - Keyspace names callers use match `[a-z0-9_-]+`; Valkey rejects others with `DbError::Config`.
- Reconnect rule, Valkey and the ReDB client alike: on a connection error, drop the connection, reconnect once and
  retry once — for `get`, `get_many`, `put`, `delete`, `range`, `clear` only. `increment` is never retried (it is not
  idempotent): it returns `DbError::Connection`, and the next call reconnects.
- Patterns used here, and where: Adapter (each store over its driver), Abstract Factory (`MemoryEngineFactory`,
  `ValkeyEngineFactory`, `RedbEngineFactory`, `FileEngineFactory`), Remote Proxy (`RedbClient` for the owner's
  `RedbStore`), Data Transfer Object (the owner protocol's `Request` and `Response`), Strategy (`KvCache`, `FileCache`
  as `CachingStore` backends). Nothing else is named because nothing else is needed.
- Valkey tests: `#[ignore = "needs Valkey: set SPEEDYSPEC_TEST_VALKEY_URL"]`; start with
  `docker run -d --name speedyspec-valkey -p 36379:6379 valkey/valkey:8` and run with
  `SPEEDYSPEC_TEST_VALKEY_URL=valkey://127.0.0.1:36379/15 cargo test -p <crate> -- --ignored`. Each test uses its own
  prefix `speedyspec-test-<pid>-<nanos>:` so runs never collide. ReDB, file and memory tests need nothing external.

## Assumed from plan 3

Plan 3 is written in parallel. The index fixes `CachingStore`, `CacheError`, and the method list; this plan also
assumes the following. If plan 3 differs, use plan 3's names and keep this plan's behaviour.

- `CachingStore` methods: `get(&self, group: &str, key: &str) -> Result<Option<Vec<u8>>, CacheError>`;
  `get_many(&self, group: &str, keys: &[&str]) -> Result<Vec<Option<Vec<u8>>>, CacheError>`; `set(&self, group, key,
  value: &[u8], expiry: Option<Duration>) -> Result<(), CacheError>`; `add` and `replace` with `set`'s parameters
  `-> Result<bool, CacheError>`; `delete(&self, group, key) -> Result<bool, CacheError>`; `increment(&self, group,
  key, by: i64) -> Result<Option<i64>, CacheError>` and `decrement` alike (`None` when the key is absent);
  `flush(&self) -> Result<(), CacheError>`; `flush_group(&self, group: &str) -> Result<(), CacheError>`.
- No `From<DbError> for CacheError` (`speedyspec-cache` depends on nothing): `KvCache` and `FileCache` map a
  `DbError` to `CacheError::Backend { cache: <engine>, message: error.to_string() }`.
- The suite: `speedyspec_cache_conformance::caching_store_suite(store: &dyn CachingStore)` in crate
  `speedyspec-cache-conformance`, which pins the semantics of increment on a non-numeric value and decrement below
  zero. `KvCache` and `FileCache` follow whatever it pins.

## Review Focus

- **A worker's ReDB owner restarts mid-request** (the supervisor replaced it): the next `get` reconnects and succeeds
  without the caller seeing an error; an `increment` in flight returns `DbError::Connection` and is never applied
  twice. Test in Task 6.
- **A counter that is not canonical decimal or would overflow** (`"abc"`, `"05"`, `i64::MAX + 1`): the same
  `not-integer` / `overflow` error on memory, ReDB and Valkey, never a silent reset to 0. Test in the suite, Task 1.
- **A socket path longer than the platform's `sun_path`** (a deep site directory; 104 bytes on macOS, 108 on Linux):
  owner and client fail with `DbError::Config` naming the path and the limit, never a truncated path. Test in Task 5.
- **Cache groups and keys with `/`, `..`, NUL or 1 KiB of text**, and `FileClient` paths with `..` or a leading `/`:
  file names are hashes and never leave the cache directory; bad relative paths are refused with
  `code: Some("invalid-path")`. Tests in Tasks 3 and 8.
- **A TTL of zero or under a millisecond:** zero deletes, sub-millisecond lives at least 1 ms, on every engine (Valkey
  refuses `PX 0`). Test in the suite, Task 1.

---

### Task 1: NoSQL and Direct Store conformance suites

**Files:**
- Create: `crates/speedyspec-data-conformance/src/nosql.rs`, `src/direct.rs`; modify `src/lib.rs`

**Interfaces:**
- Consumes: plan 1's `NoSqlStore`, `DirectStore`, `Health`, `Capabilities`, `DbError`.
- Produces: `pub fn nosql_store_suite(store: &dyn NoSqlStore, keyspace: &str)` and `pub fn direct_store_suite(store:
  &dyn DirectStore, expected_kind: &str)`, both panicking with the failing check's name, as `sql_store_suite` does.
- `nosql_store_suite` checks, in order (it first `clear`s `keyspace` and `<keyspace>-other`; `other` below is that
  second keyspace):
  1. `missing`: `get(k, b"nope")` is `None`.
  2. `roundtrip`: `put(b"a", b"1", None)` then `get` is `b"1"`; `put(b"a", b"2", None)` overwrites.
  3. `binary`: key `[0x00, 0xFF, b'/']` with value `[0x00, 0xFE]` round-trips; an empty value round-trips as
     `Some(vec![])`, distinct from `None`.
  4. `get_many_order`: keys `[a, nope, binary]` give `[Some(2), None, Some(..)]` in that order; `get_many(&[])` is
     empty.
  5. `delete`: `delete(a)` is `true`, then `false`; `get(a)` is `None`.
  6. `keyspaces_isolated`: `put` in `other` under key `b"a"`; `get(keyspace, b"a")` is `None`; `clear(keyspace)`
     leaves `other`'s key.
  7. `increment`: missing key, `by 5` gives `5`; `by -7` gives `-2`; `get` is `b"-2"`; `put(b"c", b"10")` then
     `by 5` is `15`.
  8. `increment_rejects`: on `b"abc"`, `b"05"`, `b" 5"`, `b""` → `Engine { code: Some("not-integer"), .. }`; on
     `i64::MAX` as text, `by 1` → `code: Some("overflow")`; the stored value is unchanged after each.
  9. `expiry`: `put(b"t", b"x", Some(200 ms))` is readable at once and `None` after sleeping 400 ms (also absent from
     `get_many` and `range`); `put(.., Some(Duration::ZERO))` over an existing key leaves it absent;
     `put(.., Some(Duration::from_micros(10)))` does not error; `put(b"n", b"5", Some(300 ms))`, `increment(n, 1)` is
     `6`, and after 500 ms `n` is absent.
  10. `range` (when `capabilities().ordered_ranges`): keys `r1..r5` → `range(Included(r2), Excluded(r5), None)` is
      `r2, r3, r4` in byte order with values; `Unbounded..Unbounded` with `limit Some(2)` gives `r1, r2`;
      otherwise `range` is `Err(Engine { code: Some("unsupported"), .. })`.
  11. `clear`: `clear(keyspace)` empties it; `clear` of a never-used keyspace is `Ok`.
- `direct_store_suite` checks: `kind() == expected_kind`; `connect()` then `health()` is `Ok(Health { ok: true, .. })`;
  a second `connect()` is `Ok`; `close()` then `health()` is `Ok` with `ok == false`; a second `close()` is `Ok`;
  `connect()` again gives `ok == true`.

- [ ] **Step 1: Implement** both suites as above; `engine()` is used in panic messages.
- [ ] **Step 2: Run** `cargo build -p speedyspec-data-conformance` — Expected: builds (the suites run in Task 2).
- [ ] **Step 3: Commit** — `git add crates/speedyspec-data-conformance && git commit -m "feat(data-conformance): NoSQL and Direct Store conformance suites"`

### Task 2: In-memory stores (`speedyspec-data-memory`)

**Files:**
- Create: `crates/speedyspec-data-memory/Cargo.toml`, `src/lib.rs`, `src/nosql.rs`, `src/direct.rs`, `src/factory.rs`;
  `crates/speedyspec-data-counter/Cargo.toml`, `src/lib.rs`
- Test: `crates/speedyspec-data-memory/tests/conformance.rs`, `crates/speedyspec-data-counter/tests/counter.rs`

**Interfaces:**
- Consumes: Task 1; plan 1's `EngineRegistry`.
- Produces:
  - `pub struct MemoryNoSqlStore` with `new(engine: &str) -> MemoryNoSqlStore`, `impl NoSqlStore`. Holds
    `RefCell<HashMap<String, BTreeMap<Vec<u8>, (Vec<u8>, Option<Instant>)>>>`; expired entries are treated as absent
    and dropped when touched.
  - `pub struct MemoryDirectStore` with `new(engine: &str) -> MemoryDirectStore`, `impl DirectStore` (`kind()` is
    `"memory"`; `as_any()` is `self`), plus `get(&self, name: &str) -> Option<Vec<u8>>`, `put(&self, name: &str,
    bytes: &[u8])`, `remove(&self, name: &str) -> bool` for test repositories.
  - `pub struct MemoryEngineFactory` (`scheme() == "memory"`): `memory://` or `memory://nosql` opens
    `EngineHandle::NoSql`, `memory://direct` opens `EngineHandle::Direct` (connected); anything else is `Config`.
    `kind()` is `EngineKind::Memory`. The engine creates its store once and returns clones of the same `Rc`.
  - `speedyspec-data-counter` (depends on `speedyspec-data` only; Valkey's `INCRBY` rules for stores that emulate
    it, shared by memory and ReDB so neither adapter depends on the other): `pub fn parse_counter(bytes: &[u8]) ->
    Option<i64>` and `pub fn checked_increment(engine: &str, current: Option<&[u8]>, by: i64) -> Result<(i64,
    Vec<u8>), DbError>` (returns the new value and its stored bytes; errors per Global Constraints).

- [ ] **Step 1: Write the failing tests** in `tests/conformance.rs`: `memory_nosql_conformance` (open through an
  `EngineRegistry` with `MemoryEngineFactory` and `memory://`, then `nosql_store_suite(&*store, "conf")`);
  `memory_direct_conformance` (`memory://direct`, `direct_store_suite(&*store, "memory")`); in
  `speedyspec-data-counter`, `counter_rules` (`parse_counter` accepts `b"0"`, `b"-12"`, `b"9223372036854775807"`; rejects `b"05"`, `b"-0"`, `b"+1"`, `b"1 "`,
  `b""`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-counter -p speedyspec-data-memory` — Expected: FAIL (crates missing).
- [ ] **Step 3: Implement** the items above. `-0` is rejected, as Valkey's `string2ll` rejects it.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-counter -p speedyspec-data-memory` — Expected: PASS, no suite check failing.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data-counter crates/speedyspec-data-memory && git commit -m "feat(data-memory): in-memory NoSQL and Direct Stores"`

### Task 3: The file Direct Store (`speedyspec-data-file`)

**Files:**
- Create: `crates/speedyspec-data-file/Cargo.toml`, `src/lib.rs`, `src/client.rs`, `src/store.rs`, `src/factory.rs`
- Test: `crates/speedyspec-data-file/tests/file.rs`

**Interfaces:**
- Consumes: Task 1; plan 1's `DirectStore`, `Health`, `EngineFactory`.
- Produces:
  - `#[derive(Clone)] pub struct FileClient` (root `PathBuf`, engine name, and a shared `Rc<Cell<bool>>` connected
    flag) with `root(&self) -> &Path`, `read(&self, path: &str) -> Result<Option<Vec<u8>>, DbError>`, `write(&self,
    path: &str, bytes: &[u8]) -> Result<(), DbError>`, `remove(&self, path: &str) -> Result<bool, DbError>`,
    `remove_dir_all(&self, path: &str) -> Result<bool, DbError>`, `rename(&self, from: &str, to: &str) -> Result<bool,
    DbError>` (`false` when `from` is missing), `list(&self, dir: &str) -> Result<Vec<String>, DbError>` (names, sorted;
    empty for a missing dir), `lock(&self, path: &str) -> Result<FileLock, DbError>` (exclusive `flock` on that file,
    created if missing, released on drop).
  - `pub struct FileStore` with `new(engine: &str, root: PathBuf) -> FileStore`, `client(&self) -> &FileClient`, and
    `impl DirectStore`: `kind()` is `"file"`; `connect` creates the root (`create_dir_all`) and marks connected;
    `health` is `ok: true` only if connected and a probe file `.speedyspec-health-<pid>` can be written and removed in
    the root, `detail` naming the failure otherwise; `close` marks disconnected; **`as_any()` returns the
    `FileClient`**, so a per-system repository does `store.as_any().downcast_ref::<FileClient>()`.
  - `pub struct FileEngineFactory` (`scheme() == "file"`, URL `file:///absolute/dir`; a relative or empty path is
    `Config`); `open` connects and returns `EngineHandle::Direct`; `kind()` is `EngineKind::File`.
- Rules: every `path` is relative, `/`-separated, made of normal components only; empty, absolute, `.` or `..`
  components, or NUL are refused with `DbError::Engine { code: Some("invalid-path"), .. }`. Any operation while not
  connected is `DbError::Connection`. `write` creates parent directories, writes a temp file
  `.<name>.tmp-<pid>-<counter>` in the same directory (`create_new`), calls `sync_data`, then renames it over the
  target, so a reader in another process sees the old file or the new one, never part of one; on error the temp file
  is removed. Permissions follow the process umask.

- [ ] **Step 1: Write the failing tests:** `file_direct_conformance` (registry + `FileEngineFactory` on a tempdir,
  `direct_store_suite(&*store, "file")`); `as_any_exposes_client` (downcast succeeds; `write("a/b.txt", b"x")` then
  `read` is `b"x"`); `write_is_atomic_rename` (after 50 `write`s of 1 MiB to one path, `list` of its dir holds only
  `b.txt`, no `.tmp-` names); `refuses_escaping_paths` (`"../x"`, `"/etc/passwd"`, `"a/../../x"`, `""`, `"a\0b"` →
  `invalid-path`, and nothing is created outside the root); `missing_reads_are_none` (`read` → `None`, `remove` →
  `false`, `list("nope")` → empty); `closed_store_refuses` (`close()` then `read` → `Connection`);
  `lock_is_exclusive` (a second `flock(LOCK_EX | LOCK_NB)` on the same file through `libc` fails while the
  `FileLock` is held, succeeds after drop).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-file` — Expected: FAIL.
- [ ] **Step 3: Implement** per the rules above.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-file` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data-file && git commit -m "feat(data-file): the file Direct Store"`

### Task 4: The Valkey store (`speedyspec-data-valkey`)

**Files:**
- Create: `crates/speedyspec-data-valkey/Cargo.toml`, `src/lib.rs`, `src/store.rs`, `src/factory.rs`
- Test: `crates/speedyspec-data-valkey/tests/valkey.rs`

**Interfaces:**
- Consumes: Task 1; plan 1's contracts.
- Produces: `pub struct ValkeyStore` (`impl NoSqlStore`) with `new(engine: &str, url: &str, prefix: &str, timeout:
  Duration) -> Result<ValkeyStore, DbError>` (validates, does not connect); `pub struct ValkeyEngineFactory`
  (`scheme() == "valkey"`, `kind()` `EngineKind::Valkey`). URL `valkey://[user:password@]host[:port][/db]` becomes
  `redis://…` for the driver (`rediss://` with option `tls=true`); options `prefix` (default `speedyspec:`, matching
  `[A-Za-z0-9_.:-]+`) and `timeout_ms` (default `2000`, connect and read/write timeouts).
- Rules:
  - Keyspaces are key prefixes: the stored key is `<prefix><keyspace>:` followed by the raw key bytes. An invalid
    keyspace is `Config`.
  - Connects lazily on first call and keeps one connection (`RefCell<Option<redis::Connection>>`); reconnects per the
    Global Constraints rule (`err.is_io_error()` or `is_connection_dropped()`).
  - `get` → `GET`; `get_many` → one `MGET` (empty input sends nothing); `put` → `SET key value` or `SET key value PX
    <ms>` with `ms = max(1, ceil(ttl in ms))`, and `Some(ZERO)` → `DEL`; `delete` → `DEL` (`true` when it removed 1);
    `increment` → `INCRBY` (Valkey's "not an integer" → `code: Some("not-integer")`, "would overflow" →
    `code: Some("overflow")`); `range` → `Engine { code: Some("unsupported"), message: "ordered ranges are not
    supported by valkey" }`; `clear` → `SCAN 0 MATCH <prefix><keyspace>:* COUNT 1000` with `UNLINK` per batch until the
    cursor is 0.
  - Other driver errors → `Engine { code: Some(<the error code, e.g. "WRONGTYPE">), .. }`; connection failures →
    `Connection` naming the host, never the password.

- [ ] **Step 1: Write the failing tests:** `url_mapping` (no Valkey: `valkey://:pw@h:1/3` maps to `redis://:pw@h:1/3`,
  `tls=true` to `rediss://`, a bad `prefix` is `Config`, an invalid keyspace `"A b"` is `Config` from any method);
  ignored: `valkey_conformance` (store from `SPEEDYSPEC_TEST_VALKEY_URL` through the registry with a per-test prefix,
  then `nosql_store_suite(&*store, "conf")`); `clear_keeps_other_prefixes` (two stores, prefixes `p1:` and `p2:`,
  same keyspace; `clear` on one leaves the other's keys); `reconnects_after_client_kill` (`CLIENT KILL TYPE normal`
  from a second connection, then `get` succeeds; an `increment` right after a second kill returns `Connection` and
  the counter is unchanged, then the next `increment` succeeds).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-valkey` and, with Valkey up, `… -- --ignored` — Expected: FAIL.
- [ ] **Step 3: Implement** per the rules above.
- [ ] **Step 4: Run** both commands — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data-valkey && git commit -m "feat(data-valkey): the Valkey NoSQL store"`

### Task 5: ReDB store, wire protocol and owner (`speedyspec-data-redb`)

**Files:**
- Create: `crates/speedyspec-data-redb/Cargo.toml`, `src/lib.rs`, `src/store.rs`, `src/protocol.rs`, `src/owner.rs`,
  `src/socket.rs`
- Test: `crates/speedyspec-data-redb/tests/store.rs`, `tests/owner.rs`

**Interfaces:**
- Consumes: Task 1; Task 2's `speedyspec-data-counter` (`checked_increment`); plan 1's `NoSqlStore`.
- Produces:
  - `pub struct RedbStore` (`Arc<redb::Database>`; `Send + Sync`) with `open(engine: &str, file: &Path) ->
    Result<RedbStore, DbError>`, `impl NoSqlStore`, and `sweep_expired(&self, batch: usize) -> Result<usize,
    DbError>`. One table per keyspace, `TableDefinition<&[u8], &[u8]>`; a missing table reads as empty. Each value is
    stored as an envelope: 8 bytes little-endian `expires_at` (Unix ms, `0` = never) then the value. Expired entries
    read as absent. Every write is one write transaction committed with redb's default durability; `increment` reads
    and writes in one write transaction (atomic across the owner's threads). Another process holding the file →
    `Connection { message: "… is open in another process; is a second owner running?" }`.
  - `pub const PROTOCOL_VERSION: u16 = 1;` `pub const MAX_FRAME: u32 = 64 * 1024 * 1024;`
  - `#[derive(Serialize, Deserialize)] pub enum Request { Hello { version: u16 }, Get { keyspace: String, key:
    Vec<u8> }, GetMany { keyspace: String, keys: Vec<Vec<u8>> }, Put { keyspace: String, key: Vec<u8>, value: Vec<u8>,
    ttl_ms: Option<u64> }, Delete { keyspace: String, key: Vec<u8> }, Range { keyspace: String, start: WireBound, end:
    WireBound, limit: Option<u64> }, Increment { keyspace: String, key: Vec<u8>, by: i64 }, Clear { keyspace: String } }`;
    `pub enum WireBound { Included(Vec<u8>), Excluded(Vec<u8>), Unbounded }`;
    `pub enum Response { Ready { version: u16 }, Value(Option<Vec<u8>>), Values(Vec<Option<Vec<u8>>>), Done,
    Deleted(bool), Entries(Vec<(Vec<u8>, Vec<u8>)>), Counter(i64), Error { code: String, message: String } }`.
  - `pub fn write_frame(w: &mut impl Write, msg: &impl Serialize) -> Result<(), DbError>` and `pub fn read_frame<T:
    DeserializeOwned>(r: &mut impl Read) -> Result<T, DbError>`: a frame is a 4-byte big-endian length then that many
    `postcard` bytes; a length over `MAX_FRAME` is `Engine { code: Some("too-large"), .. }`; EOF or IO failure is
    `Connection`.
  - `pub fn owner_socket_for(file: &Path) -> PathBuf` → `<file's dir>/<file stem>.owner/owner.sock` (the client's
    default too); `pub fn check_socket_path(path: &Path) -> Result<(), DbError>` (`Config` naming the path and the
    limit when `path` is longer than `sun_path` less its NUL: 103 bytes on macOS, 107 on Linux).
  - `pub struct OwnerConfig { pub engine: String, pub file: PathBuf, pub socket: PathBuf, pub sweep_every: Duration }`
    with `OwnerConfig::from_engine_config(config: &EngineConfig) -> Result<OwnerConfig, DbError>` (the same `redb://`
    URL and `socket` option the client reads; `sweep_every` from option `sweep_secs`, default 60).
  - `pub struct RedbOwner` with `open(config: OwnerConfig) -> Result<RedbOwner, OwnerError>`, `stop_handle(&self) ->
    StopHandle` (`StopHandle: Clone + Send`, `stop(&self)` sets a flag and connects to the socket once to wake
    `accept`), `serve(&self) -> Result<(), DbError>` (blocks until stopped). `pub enum OwnerError { AlreadyRunning,
    Boot(DbError) }`.
  - **`pub fn run_owner(config: OwnerConfig) -> i32`**: the entry point the runtime will call in its own supervised
    process. Opens, serves, returns `0` when stopped, `1` on a boot failure, `3` when another owner already answers on
    the socket. Logs one JSON line per event to stderr, `{"event", "engine", "socket", "message"}`, events
    `redb_owner_started`, `redb_owner_boot_failed`, `redb_owner_already_running`, `redb_owner_stopped`, as the
    worker's `log` does.
- Owner rules:
  - `open`: `check_socket_path`; create the socket's directory and set it to `0700` (so a `socket` option must name
    a directory used for nothing else); if a stale socket file exists,
    try to connect: success → `AlreadyRunning`; refused → remove it. Open `RedbStore`, bind the `UnixListener`, set
    the socket to `0600` (only the same user — the runtime's workers — may connect, as with the FastCGI sockets).
  - `serve`: one thread per accepted connection (connections are long-lived, one per worker), all sharing the
    `RedbStore`; one more thread calls `sweep_expired(1000)` every `sweep_every`. Per connection: the first frame must
    be `Hello { version: PROTOCOL_VERSION }`, answered `Ready`; any other first frame or version gets `Error { code:
    "protocol" }` and the connection is closed. Then each `Request` gets exactly one `Response`, in order. A `DbError`
    from the store becomes `Error { code, message }` (`code` from `Engine`'s code, else `"storage"`); a frame that does
    not decode gets `Error { code: "protocol" }` and the connection is closed. `ttl_ms: Some(0)` deletes.

- [ ] **Step 1: Write the failing tests:** `tests/store.rs`: `redb_store_conformance` (`RedbStore::open` on a tempdir
  file, `nosql_store_suite(&store, "conf")`); `sweep_removes_expired` (three keys with 10 ms TTL, one without; after
  50 ms `sweep_expired(100)` returns 3). `tests/owner.rs`: `frames_roundtrip` (each `Request` variant through
  `write_frame`/`read_frame` over a `Vec<u8>`; a header of `MAX_FRAME + 1` → `too-large`); `owner_socket_is_private`
  (after `open`, the socket dir mode is `0o700` and the socket `0o600`); `stale_socket_is_replaced` (a leftover
  socket file with no listener → `open` succeeds); `second_owner_is_refused` (second `open` on the same config while
  the first serves → `Err(OwnerError::AlreadyRunning)`, and `run_owner` on it returns `3`); `rejects_bad_hello` (raw
  stream sending `Hello { version: 99 }` gets `Error { code: "protocol", .. }` then EOF); `long_socket_path_refused`
  (a path of 120 bytes → `Config` whose message contains the path and the limit); `stop_ends_serve` (`StopHandle::stop`
  from another thread makes `serve` return `Ok`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-redb` — Expected: FAIL.
- [ ] **Step 3: Implement** the store, protocol, socket helpers and owner per the rules above.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-redb` — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-data-redb && git commit -m "feat(data-redb): ReDB store, owner process and its wire protocol"`

### Task 6: The ReDB client and factory

**Files:**
- Create: `crates/speedyspec-data-redb/src/client.rs`, `src/factory.rs`; modify `src/lib.rs`
- Test: `crates/speedyspec-data-redb/tests/client.rs`

**Interfaces:**
- Consumes: Task 5.
- Produces: `pub struct RedbClient` (`impl NoSqlStore`; `capabilities()` as `RedbStore`'s) with `new(engine: &str,
  socket: PathBuf, timeout: Duration) -> Result<RedbClient, DbError>` (runs `check_socket_path`; connects lazily);
  `pub struct RedbEngineFactory` (`scheme() == "redb"`, `kind()` `EngineKind::Redb`, URL `redb:///absolute/file.redb`,
  options `socket` (default `owner_socket_for(file)`) and `timeout_ms` (default `5000`)), whose `open` returns
  `EngineHandle::NoSql(Rc<RedbClient>)`. The factory never opens the file: only the owner does.
- Rules: the client holds `RefCell<Option<UnixStream>>`; connecting sends `Hello` and expects `Ready`; read and write
  timeouts are `timeout`. Each method sends one `Request` and maps the `Response`: `Error { code, message }` →
  `DbError::Engine { engine, code: Some(code), message }`; an unexpected variant → `Engine { code: Some("protocol") }`
  and the connection is dropped. A missing socket or refused connection → `Connection { message: "the redb owner is
  not reachable at <socket>: …" }`. Reconnect per the Global Constraints rule; `Duration` TTLs go over the wire as
  `ttl_ms = max(1, ceil(ms))`, and `Some(ZERO)` as `Some(0)`.

- [ ] **Step 1: Write the failing tests** (each starts a `RedbOwner` on a tempdir in a thread and stops it at the end):
  `redb_client_conformance` (client through an `EngineRegistry` with `RedbEngineFactory`, `nosql_store_suite(&*store,
  "conf")`); `clients_share_one_owner` (four threads, each with its own `RedbClient`, 250 `increment(by 1)` each →
  final value `1000`; a `put` from one client is read by another); `owner_restart_is_transparent_for_reads` (stop the
  owner, start a new one on the same config, then `get` on the old client succeeds with no error); `increment_is_not_retried`
  (stop and restart the owner between two calls: the first `increment` after the restart returns
  `Err(Connection { .. })` and the stored counter is unchanged; the next `increment` succeeds); `owner_down_is_connection_error`
  (no owner → `get` is `Connection` naming the socket).
- [ ] **Step 2: Run** `cargo test -p speedyspec-data-redb --test client` — Expected: FAIL.
- [ ] **Step 3: Implement** the client and factory.
- [ ] **Step 4: Run** `cargo test -p speedyspec-data-redb` — Expected: PASS.
- [ ] **Step 5: Commit** — `git commit -am "feat(data-redb): NoSqlStore client over the owner's socket"` (add new files first).

### Task 7: `KvCache` over any NoSQL store (`speedyspec-cache-kv`)

**Files:**
- Create: `crates/speedyspec-cache-kv/Cargo.toml`, `src/lib.rs`, `src/keys.rs`, `src/cache.rs`
- Test: `crates/speedyspec-cache-kv/tests/kv.rs`

**Interfaces:**
- Consumes: plan 3's `CachingStore`, `CacheError`, `caching_store_suite` (see "Assumed from plan 3"); plan 1's
  `NoSqlStore`; Tasks 2, 4, 6 in tests.
- Produces: `pub struct KvCache` with `new(store: Rc<dyn NoSqlStore>, keyspace: &str) -> KvCache` and `impl
  CachingStore`. One keyspace holds a cache; callers isolate sites by giving each its own keyspace (`cache-<site>`).
- **Key layout and `flush_group`** (decided: group generation counters, because `NoSqlStore` has no prefix delete and
  Valkey has no ranges):
  - generation key: `b"g"` + group bytes; absent means generation `0`.
  - entry key: `b"e"` + group length as 4-byte big-endian + group bytes + generation as 8-byte big-endian + key bytes.
  - `flush_group` is `increment(generation key, 1)`: atomic, seen by every process at once; old entries are no longer
    addressed and go away by their own expiry or at `flush`.
  - `flush` is `clear(keyspace)`.
  - Every operation reads the group's generation first (one extra `get`; `get_many` reads it once for all keys).
- Other rules: `set` → `put` with the expiry as TTL; `add` → `NoSqlStore::put_if_absent` (atomic; plan 1 has it); `replace` reads then `put` (not
  atomic across processes, as WordPress's own persistent drop-ins); `increment`/`decrement`
  return `None` when the entry is absent, else call `NoSqlStore::increment` (`by` or `-by`) and apply plan 3's suite's
  rules for non-numeric values and results below zero (computed in Rust and `put` when `increment` reports
  `not-integer`).

- [ ] **Step 1: Write the failing tests:** `kv_cache_on_memory` (`caching_store_suite(&KvCache::new(memory store,
  "cache"))`); `kv_cache_on_redb` (owner in a thread, `RedbClient`); ignored `kv_cache_on_valkey`;
  `flush_group_is_seen_by_another_instance` (two `KvCache`s over one store: `set` in `posts` and `users` through A,
  `flush_group("posts")` through B → A's `get("posts", k)` is `None`, `users` intact); `group_and_key_are_unambiguous`
  (group `"a"` key `"bc"` and group `"ab"` key `"c"` hold different values); `entry_key_layout` (`keys::entry(b"g",
  7, b"k")` equals `[b'e', 0,0,0,1, b'g', 0,0,0,0,0,0,0,7, b'k']`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-cache-kv` — Expected: FAIL.
- [ ] **Step 3: Implement** per the rules above.
- [ ] **Step 4: Run** `cargo test -p speedyspec-cache-kv` (and `-- --ignored` with Valkey up) — Expected: PASS.
- [ ] **Step 5: Commit** — `git add crates/speedyspec-cache-kv && git commit -m "feat(cache-kv): Caching Store over any NoSQL store"`

### Task 8: `FileCache` over the file Direct Store (`speedyspec-cache-file`)

**Files:**
- Create: `crates/speedyspec-cache-file/Cargo.toml`, `src/lib.rs`, `src/layout.rs`, `src/cache.rs`
- Test: `crates/speedyspec-cache-file/tests/file_cache.rs`

**Interfaces:**
- Consumes: plan 3's `CachingStore`, `CacheError`, `caching_store_suite`; Task 3's `FileClient`.
- Produces: `pub struct FileCache` with `new(store: &dyn DirectStore, dir: &str) -> Result<FileCache, CacheError>`
  (downcasts `store.as_any()` to `FileClient` and clones it; any other store is `CacheError` from `DbError::Config(
  "FileCache needs a file Direct Store")`; `dir` is the cache's directory under the root) and `impl CachingStore`.
- Layout (`src/layout.rs`, `pub fn group_dir(dir: &str, group: &str) -> String`, `pub fn entry_path(dir: &str, group:
  &str, key: &str) -> String`): group directory `<dir>/g-<first 32 hex of sha256(group)>`; entry
  `<group dir>/<first 2 hex of sha256(group NUL key)>/<the other 62 hex>`. Entry file: 8 bytes little-endian
  `expires_at` (Unix ms, `0` = never), 4 bytes little-endian length of `group NUL key`, those bytes, then the value.
  A `get` whose stored `group NUL key` differs from the one asked (a hash collision) or whose header is short is a
  miss; an expired entry is a miss and is removed.
- Rules: `set` is `FileClient::write` (atomic rename); `add`, `replace`, `increment`, `decrement` hold
  `FileClient::lock("<group dir>/.lock")` for their read-modify-write, so two processes never interleave on one group;
  plain `get` takes no lock. `flush_group` renames the group directory to `<dir>/.trash-<pid>-<nanos>` and then removes
  it, so readers see the whole group vanish at once; `flush` does the same for every `g-*` directory. Integer
  semantics follow plan 3's suite, with values stored as decimal ASCII.

- [ ] **Step 1: Write the failing tests:** `file_cache_conformance` (`FileStore` on a tempdir, connected,
  `caching_store_suite(&FileCache::new(&store, "object-cache")?)`); `hostile_names_stay_inside`
  (group `"../.."`, key `"/etc/passwd\0"` and a 1 KiB key: `set` then `get` round-trips, and every file under the
  tempdir is inside `<root>/object-cache`); `layout_is_stable` (`entry_path("c", "posts", "1")` equals the fixed
  string computed once from sha256, written into the test); `flush_group_leaves_others` (`flush_group("posts")`;
  `users` entries still read; no `.trash-` directory remains); `rejects_non_file_store` (`FileCache::new` on a
  `MemoryDirectStore` is `Err`); `concurrent_increments` (two threads, each with its own `FileStore` on the same root,
  200 `increment(by 1)` each after `set(.., b"0")` → `400`).
- [ ] **Step 2: Run** `cargo test -p speedyspec-cache-file` — Expected: FAIL.
- [ ] **Step 3: Implement** per the rules above.
- [ ] **Step 4: Run** `cargo test -p speedyspec-cache-file` — Expected: PASS.
- [ ] **Step 5: Run** `cargo clippy --workspace --all-targets -- -D warnings` and `cargo fmt --check` — Expected: clean.
- [ ] **Step 6: Commit** — `git add crates/speedyspec-cache-file && git commit -m "feat(cache-file): Caching Store over the file Direct Store"`

### Task 9: Hand the ReDB owner's supervision to the runtime's owner

**Files:**
- Create: `/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/redb-owner-process.md`
  (outside the repository: not committed)

- [ ] **Step 1: Write the note**, titled "ReDB owner process: supervision request for the runtime", stating:
  - What exists: `speedyspec_data_redb::run_owner(OwnerConfig) -> i32` (branch `feat/nosql-and-direct-stores` of
    `SpeedySpec/speedyspec-server`), `OwnerConfig::from_engine_config`, and that `crates/speedyspec-php-runtime` was
    not touched.
  - The request: when a site configures an engine with a `redb://` URL, the runtime starts one owner process per site
    (per replica) beside the background job runner, e.g. a `redb-owner --engine <json>` subcommand that calls
    `run_owner`, before that site's workers in every pool (blog, front-end API, admin API) start; restarts it when it
    exits with `1` or dies; does not restart on `3` (another owner already serves the socket) and logs it; stops it
    after the site's workers; and makes it exit with its parent, as `exit_with_parent` does for workers.
  - The contract the owner keeps: socket in a `0700` directory, socket `0600`, so owner and workers must run as the
    same user; default socket `<file's dir>/<stem>.owner/owner.sock`, which must fit `sun_path`; JSON log events
    `redb_owner_*`; clients survive a restart (reads retry once; an increment in flight fails once).
  - With `[cluster]`: ReDB stays per replica, like files; the file must not be on storage shared between replicas;
    Valkey is the shared choice.
- [ ] **Step 2: Check** — `ls "/Users/jacobsantos/Library/Application Support/Claude/side-session-notes/local_2fb454ad-19fc-4387-9cbf-49b6a3b40750/"` — Expected: `redb-owner-process.md` listed beside `database-and-options-port.md`.
