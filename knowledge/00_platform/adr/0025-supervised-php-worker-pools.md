# 0025. PHP runs in supervised worker pools, one request per worker

- **Status:** Accepted, 2026-11-18. Where the pool runs superseded on 2026-11-19 (see Consequences).
- **Decided by:** the designer

## Context

A PHP worker that crashes, hangs or leaks must not take requests with it, and no request may reach a worker already
known to be poisoned. WordPress expects every request to start clean.

## Decision

- **One request per worker at a time**, as php-fpm does. Each request runs a full PHP request startup and shutdown,
  with `$_SERVER` from the FastCGI params and the body on `php://input`. The response is streamed.
- Every request starts clean: there is no worker mode that keeps state between requests.
- **Pool settings** per app: `workers` (min, max), `max_requests`, `request_timeout`, `memory_limit`, `idle_timeout`,
  in php-fpm's `dynamic` mode.
- **A worker is killed and replaced** when it exits or dies (the request gets 502), runs past `request_timeout`
  (SIGKILL, 504), passes `memory_limit` after a request, has served `max_requests`, or the engine reports it poisoned.
  Each case is logged, and a replacement starts at once.
- **Releases:** `prepare` starts the release's pool and waits for its first worker to answer; `activate` swaps to the
  new pool in one step; `retire` drains the old pool.
- A release declares `document_root`, the `entry` front controller, and per-app `ini` values.

## Alternatives considered

- **A persistent worker mode** (FrankenPHP-style). Rejected: WordPress expects a clean start.

## Consequences

- **Superseded in part:** the pools were first a role of the server binary (`speedyspec-server php-worker`). The PHP
  runtime and admin gate design moved them into their own binary, `speedyspec-php-runtime`, released and upgraded by
  the server like any app, so the server holds no PHP. The behaviour above was kept as designed.
- Pools belong to a server generation, so a handoff starts the new generation's pools before it reports ready.

## Patterns

- **Object Pool**, with supervision.
