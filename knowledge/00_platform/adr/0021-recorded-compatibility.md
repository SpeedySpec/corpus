# 0021. Recorded compatibility proves a port before WordPress's code is deleted

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner; the format and harness by the designer

## Context

WordPress's implementation of an area is deleted once it is ported (0014). Something has to prove first that the port
behaves exactly as WordPress did, and keep proving it afterwards.

## Decision

1. **A catalogue of scenarios**, as JSON data: sequences of calls to WordPress functions and methods. Callbacks are
   named fixtures, so a scenario needs no PHP code.
2. **A recorder** runs each scenario against WordPress's original implementation in the same static PHP, and commits
   what came back: return values, callback calls in order, and the observable globals after each step. Each fixture
   names the WordPress version it came from, and re-recording fetches that exact upstream release.
3. **A replayer** runs the scenarios against the port and compares exactly. Any difference fails, unless the area's
   design lists it as a documented break.
4. **Deletion:** once the replayer and WordPress's own tests for the area pass, WordPress's implementation is
   deleted. The fixtures and the replayer stay as the compatibility tests.

## Consequences

- `speedyspec-compat` holds this for every area.
- WordPress's own tests for an area (for hooks, `actions.php`, `filters.php` and `hooks/`) run in the embedded PHP and
  outlive the deleted implementation.

## Patterns

- **Golden master** (characterization testing).
