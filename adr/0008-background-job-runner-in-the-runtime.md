# 0008. A background job runner inside the runtime

- **Status:** Accepted, 2026-11-19
- **Decided by:** the project owner

## Context

Image sub-sizes and library re-compression take too long for a request. WordPress's cron runs on visitors' requests.

## Decision

A long-lived process inside the runtime, beside the request workers and supervised the same way, runs queued jobs and
scheduled events. It never serves requests.

- The job queue is a store like the others: file, Redis or database (0007).
- With `[cluster]`, only one replica runs a given job.
- It is expected to replace WordPress's request-driven cron.

## Consequences

- Uploads return at once; sub-sizes follow (0013).
- How one replica claims a job, without giving the server WordPress knowledge, is left to the job runner's own design.

## Patterns

- **Command:** each queued job.
- **Repository:** the job queue.
