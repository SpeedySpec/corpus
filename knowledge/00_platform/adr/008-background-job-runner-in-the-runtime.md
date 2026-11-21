# ADR-008: A background job runner inside the runtime

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the project owner
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-003](003-native-features-wait-for-api-ports.md), [ADR-007](007-shared-stores-file-redis-database.md), [ADR-013](013-images-in-rust-on-the-job-runner.md)

---

## Context

Some work takes too long for a request: making image sub-sizes after an upload, or re-compressing an existing media
library. WordPress's cron does not help. It runs on visitors' requests, so scheduled work happens late on quiet sites
and slows a random visitor's page on busy ones.

### Why this matters

Without a place for slow work, it lands in requests. Uploads stall while sub-sizes are made, and scheduled events
depend on traffic. A runner placed in the server would give the server WordPress knowledge (ADR-002).

---

## Decision

A long-lived process inside the PHP runtime, beside the request workers and supervised the same way, runs queued
jobs and scheduled events. It never serves requests.

### Rules

1. The job runner lives in the runtime, not the server.
2. It is supervised the same way as the request workers.
3. It never answers a request.
4. The job queue is a store like the others: file, Redis or database (ADR-007), with an in-memory one for tests.
5. Each queued job is a command object.
6. With `[cluster]`, only one replica runs a given job.
7. It is expected to replace WordPress's request-driven cron.

### What this looks like

✅ **Correct**

```rust
// A request enqueues; the runner does the work later.
queue.push(Job::GenerateSubSizes { attachment_id })?;
return Ok(UploadResult::accepted(attachment_id));

// In the job runner's loop:
while let Some(job) = queue.claim()? {
    job.run(&services)?;
}
```

❌ **Not permitted**

```rust
// Slow work inside the request, or a "cron" that runs on visitors' requests.
fn handle_upload(&self, file: Upload) -> Result<Response> {
    let id = self.media.insert(file)?;
    self.images.make_all_sub_sizes(id)?; // blocks the upload
    Ok(Response::created(id))
}
```

### Patterns

- **Command:** each queued job.
- **Repository:** the job queue.

---

## Consequences

### Positive

1. Uploads return at once; sub-sizes follow (ADR-013).
2. Scheduled events run on time, whatever the traffic.
3. No visitor pays for background work.

### Negative

1. One more supervised process per runtime.
   **Mitigation:** it reuses the request workers' supervision.
2. How one replica claims a job under `[cluster]`, without the server learning about WordPress, is not settled.
   **Mitigation:** none yet. Left to the job runner's own design.
3. Results of background work appear after a delay.
   **Mitigation:** core's resumable sub-size bookkeeping shows progress.

### Trade-offs

**We prioritise:**
- fast requests;
- scheduled work that does not depend on traffic.

**We accept:**
- one more process to run and supervise;
- an open question about job claims in a cluster.

---

## Fleet impact

- **speedyspec-php-runtime:** gains the job runner process and its supervision. Not built.
- **speedyspec-server (repository):** gains the job queue contract and its backends. Not built.
- **Blog, front-end API, admin API:** enqueue jobs instead of doing slow work. Unverified.
- **speedyspec-server (binary):** must not learn about jobs. How cluster claims work is unverified.
- **speedyspec-cli:** unverified; it may need to configure the runner.
- **Plugins:** PHP plugins that schedule cron events expect them to run; whether the runner fires PHP callbacks is
  unverified.
- **Admin site:** unaffected.

---

## Enforcement

### Automated

None. This rule is enforced by review only.

### Review checklist

- [ ] Slow work is queued, not done in the request.
- [ ] The runner lives in the runtime and serves no requests.
- [ ] The queue is reached only through its contract.
- [ ] Under `[cluster]`, a job cannot be run by two replicas.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.
