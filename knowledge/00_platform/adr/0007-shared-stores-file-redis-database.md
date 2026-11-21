# 0007. Shared stores have file, Redis and database implementations

- **Status:** Accepted, 2026-11-19
- **Decided by:** the project owner; the defaults beyond the page cache by the designer

## Context

The features keep state: the page cache, firewall counters and bans, redirect rules, the 404 log, the image job
queue. Several runtime pools touch the same state: the admin API's pool purges cache entries the blog's pool serves.
Replicas must work on their own, and use the cluster's shared state when `[cluster]` is configured.

## Decision

Every shared store is a contract with file, Redis and database implementations, chosen in configuration, plus an
in-memory implementation for tests.

- **The page cache** defaults to files, in a directory every pool of the app can reach.
- **Firewall counters and bans, redirect rules and the 404 log** default to the database, through the ported database
  layer.
- **With `[cluster]`**, Redis or the database is the shared choice; files are per replica.

## Consequences

- A store that is down must not take the site with it. The firewall, for one, lets requests through and logs a
  warning.
- With files and no shared directory, each replica has its own cache and its own counters.

## Patterns

- **Strategy:** the store backends.
- **Repository:** each store's contract.
