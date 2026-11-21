# ADR-009: The database port: engines, a raw-SQL builder, a unit of work, plain objects

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the project owner (the port's own design settles how)
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-003](003-native-features-wait-for-api-ports.md), [ADR-007](007-shared-stores-file-redis-database.md)

---

## Context

Step 1 of ADR-003 ports WordPress's database layer to Rust. Two facts shape it. WordPress plugins send raw,
MySQL-flavoured SQL through `$wpdb`, and that SQL must keep working. And the project wants more databases than MySQL.
This record states what the project owner required of the port; the port's own design decides how.

### Why this matters

A port that only speaks MySQL locks every site to MySQL. A port that drops raw SQL breaks every plugin that calls
`$wpdb->query()`. And an Active Record style, where entities save themselves, would tie the domain to storage and
break the project's rule that dependencies point inward.

---

## Decision

The database port supports several engines behind repository contracts, a query builder that also parses raw SQL, a
unit of work, and plain objects that know nothing about storage.

### Rules

1. **Several databases behind repository contracts:** MySQL, PostgreSQL, SQLite and ReDB, with room for others.
2. **Engines, in the manner of SQLAlchemy:** each database is reached through an engine that knows its dialect and
   may apply engine-specific optimisations. Several databases can be used at once.
3. **A query builder in the manner of Eloquent's:** raw SQL can be parsed into a builder instance and changed with
   partials (adding or replacing clauses and fragments), then emitted in the target engine's dialect.
4. **A unit of work** groups changes and commits them together.
5. **Plain objects, not Active Record:** entities are plain PHP objects and plain Rust structs. Data Mappers and
   Repositories load and save them. "In the manner of Eloquent" means the builder's style only, not its models.
6. ReDB has no SQL. Raw SQL and the builder target only the SQL engines; ReDB is reached through repository
   implementations written for it.

### What this looks like

✅ **Correct**

```rust
// A plain struct; a repository and the unit of work handle storage.
pub struct SiteOption { pub name: String, pub value: Vec<u8>, pub autoload: bool }

let mut q = Query::table("options");
q.select(&["option_name", "option_value"]).where_eq("autoload", "yes");
let statement = q.build()?; // emitted in the engine's dialect
uow.register(Box::new(SaveOption(changed)));
uow.commit()?;
```

❌ **Not permitted**

```rust
// Active Record: the entity loads and saves itself.
impl SiteOption {
    pub fn find(name: &str) -> SiteOption { DB.query(...) }
    pub fn save(&self) { DB.execute("UPDATE wp_options SET ...") }
}
```

### Patterns

- **Data Mapper** and **Repository** for persistence; never **Active Record**.
- **Unit of Work** for commits, with an **Identity Map** beside it.
- **Builder** for queries.
- **Strategy** for engine dialects.

---

## Consequences

### Positive

1. `$wpdb`'s MySQL-flavoured SQL can run on PostgreSQL or SQLite, because it is parsed and re-emitted in the target
   dialect.
2. The domain stays free of storage code, so entities are easy to test.
3. A site can mix engines, for example SQL for content and a key-value store for counters.

### Negative

1. Parsing every raw statement costs time, and some MySQL forms may not translate.
   **Mitigation:** recorded corpora (`speedyspec-data-corpus`) and engine conformance suites cover known forms.
2. ReDB needs hand-written repository implementations.
   **Mitigation:** none; accepted as the price of a non-SQL engine.

### Trade-offs

**We prioritise:**
- engine choice;
- plugin SQL that keeps working;
- a domain with no storage knowledge.

**We accept:**
- a SQL parser in the hot path;
- extra implementations for ReDB.

---

## Fleet impact

- **speedyspec-server (repository):** holds the port: `speedyspec-data` and its engine crates (`-mysql`,
  `-postgres`, `-sqlite`, `-redb`, `-valkey`, `-memory`, `-file`), `-query`, `-translate`, `-uow` and
  `speedyspec-wp-db`.
- **speedyspec-php-runtime:** `$wpdb` becomes a shim over the port (`speedyspec-wp-db-php`).
- **Blog, front-end API, admin API:** all database access goes through the port.
- **Plugins:** raw SQL through `$wpdb` keeps working on the SQL engines.
- **speedyspec-cli:** unverified; it may need to configure engines.
- **speedyspec-server (binary), admin site:** unaffected.

---

## Enforcement

### Automated

In SpeedySpec/speedyspec-server, each engine runs the shared suites in `speedyspec-data-conformance` from its
`tests/conformance.rs` (memory, MySQL, PostgreSQL, SQLite). `cargo test -p speedyspec-data-memory --test conformance`
runs in CI's `test` job, and the SQLite suite in the `php` job's `cargo test --workspace`. The MySQL and PostgreSQL
suites are `#[ignore]`d unless `SPEEDYSPEC_TEST_MYSQL_URL` or `SPEEDYSPEC_TEST_POSTGRES_URL` is set, so CI does not
run them. A failing check panics with its name and cargo exits 101. None has been seen failing on a rule violation.
Rule 5 (no Active Record) is enforced by review only.

### Review checklist

- [ ] Entities are plain structs or plain PHP objects with no storage methods.
- [ ] Persistence goes through a repository or data mapper.
- [ ] Multi-row changes go through the unit of work.
- [ ] Dialect-specific SQL lives in an engine, not in a caller.
- [ ] New engines run the conformance suite for their store kind.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-20 to 2026-11-21:** built in SpeedySpec/speedyspec-server#12 to #19.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
